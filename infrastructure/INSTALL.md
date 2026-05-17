# INSTALL.md — Instalação ponta-a-ponta na VPS Hostinger KVM2

Guia operacional **passo a passo** para subir toda a stack do zero numa VPS
**Hostinger KVM2 (2 vCPU / 8 GB RAM / 100 GB SSD)** rodando **Ubuntu 26.04**.

> Você precisa apenas de acesso `root` (SSH) ao IP da VPS e controle do DNS
> do domínio que vai usar. Tudo o mais é construído por este guia.

---

## 0. Índice

1. [Arquitetura final](#1-arquitetura-final)
2. [Estrutura de diretórios](#2-estrutura-de-diretórios)
3. [Fluxo TLS](#3-fluxo-tls-lets-encrypt--cert-manager--traefik)
4. [Fluxo de tráfego](#4-fluxo-de-tráfego)
5. [Responsabilidades — Traefik vs cert-manager](#5-responsabilidades--traefik-vs-cert-manager)
6. [Análise de capacidade da VPS](#6-análise-de-capacidade-da-vps)
7. [Preparação da VPS](#7-preparação-da-vps)
8. [Instalação do k3s](#8-instalação-do-k3s)
9. [kubectl + kustomize + helm na sua workstation](#9-kubectl--kustomize--helm-na-sua-workstation)
10. [DNS — registros A](#10-dns--registros-a)
11. [Instalação do cert-manager](#11-instalação-do-cert-manager)
12. [Instalação do Traefik (CRDs + base)](#12-instalação-do-traefik)
13. [Criação dos Secrets reais](#13-criação-dos-secrets-reais)
14. [Deploy Postgres + Redis](#14-deploy-postgres--redis)
15. [Deploy n8n](#15-deploy-n8n)
16. [Deploy Evolution API](#16-deploy-evolution-api)
19. [Validação TLS](#19-validação-tls)
20. [GitOps com ArgoCD (opcional)](#20-gitops-com-argocd-opcional)
21. [Backup e manutenção](#21-backup-e-manutenção)
22. [Troubleshooting](#22-troubleshooting)

---

## 1. Arquitetura final

```text
                                  Internet
                                     │  DNS A *.example.com.br -> IP da VPS
                                     ▼
                        ┌─────────────────────────────┐
                        │     k3s ServiceLB (80/443)  │
                        │  (klipper-lb expõe hostPort)│
                        └──────────────┬──────────────┘
                                       │
                                       ▼
                            ┌──────────────────────┐
                            │     Traefik 3.2      │
                            │ Ingress + Reverse PX │
                            │ TLS termination      │
                            │ (consome Secrets TLS │
                            │  do cert-manager)    │
                            └──────────┬───────────┘
                                       │  IngressRoute (traefik.io/v1alpha1)
                       ┌───────────────┼────────────────────┐
                       ▼               ▼                    ▼                 
                ┌────────────┐   ┌────────────┐    ┌───────────────┐
                │  n8n-      │   │ Evolution  │    │      n8n      │
                │  webhook   │   │   API      │    │      main     │
                │ (stateless)│   │            │    │     (queue)   │
                └─────┬──────┘   └──────┬─────┘    └───────┬───────┘ 
                      │                 │                  │                  
                      └───────────┬─────┴──────────────────┘
                                  │
                                  ▼
                       ┌──────────────────────┐
                       │  namespace databases │
                       │  ┌──────┐  ┌──────┐  │
                       │  │ PG16 │  │ Rd 7 │  │
                       │  └──────┘  └──────┘  │
                       └──────────────────────┘

                       ┌──────────────────────┐
                       │   cert-manager       │
                       │ - emite Certificates │
                       │ - HTTP-01 via Traefik│
                       │ - escreve Secret TLS │
                       │   com NOME FIXO      │
                       └──────────────────────┘
```

**Princípios:**

- O Traefik NUNCA emite certificados — só consome Secrets TLS já prontos.
- O cert-manager NUNCA faz proxy — só emite/renova certificados e grava
  Secrets com nome fixo.
- Postgres e Redis são compartilhados (databases lógicos separados por app).
- Tudo é GitOps-friendly: aplicar com `kubectl apply -k` ou ArgoCD.

---

## 2. Estrutura de diretórios

```text
infrastructure/
├── README.md
├── INSTALL.md
├── base/
│   ├── kustomization.yaml                # agrega tudo
│   ├── traefik/                          # Ingress Controller + dashboard
│   │   ├── namespace.yaml
│   │   ├── rbac.yaml
│   │   ├── deployment.yaml
│   │   ├── service.yaml
│   │   ├── middleware-basicauth.yaml     # + outros middlewares globais
│   │   ├── dashboard-secret.yaml         # placeholder htpasswd
│   │   ├── certificate.yaml              # cert do dashboard
│   │   ├── ingressroute-dashboard.yaml
│   │   └── kustomization.yaml
│   ├── cert-manager/                     # SÓ os ClusterIssuers
│   │   ├── namespace.yaml
│   │   ├── clusterissuer-prod.yaml
│   │   ├── clusterissuer-staging.yaml
│   │   └── kustomization.yaml
│   ├── databases/
│   │   ├── namespace.yaml
│   │   ├── postgres/                     # StatefulSet + PVC + Secret + init
│   │   │   ├── pvc.yaml
│   │   │   ├── secret.yaml
│   │   │   ├── configmap-init.yaml
│   │   │   ├── statefulset.yaml
│   │   │   ├── service.yaml
│   │   │   └── kustomization.yaml
│   │   ├── redis/
│   │   │   ├── pvc.yaml
│   │   │   ├── secret.yaml
│   │   │   ├── configmap.yaml
│   │   │   ├── statefulset.yaml
│   │   │   ├── service.yaml
│   │   │   └── kustomization.yaml
│   │   └── kustomization.yaml
│   ├── n8n/                              # main + webhook em modo queue
│   │   ├── namespace.yaml
│   │   ├── configmap.yaml
│   │   ├── secret.yaml
│   │   ├── pvc.yaml
│   │   ├── deployment-main.yaml
│   │   ├── deployment-webhook.yaml
│   │   ├── service-main.yaml
│   │   ├── service-webhook.yaml
│   │   ├── middlewares.yaml
│   │   ├── certificate.yaml              # 2 certs (main + webhook)
│   │   ├── ingressroute.yaml
│   │   ├── networkpolicy.yaml
│   │   └── kustomization.yaml
│   ├── evolution/
│   │   └── (mesma estrutura)
└── overlays/
    └── prod/
        ├── kustomization.yaml
        ├── domain-patches/               # patches JSON-6902 + SMP
        │   ├── traefik-ingressroute.yaml
        │   ├── traefik-certificate.yaml
        │   ├── n8n-main-ingressroute.yaml
        │   ├── n8n-main-certificate.yaml
        │   ├── n8n-webhook-ingressroute.yaml
        │   ├── n8n-webhook-certificate.yaml
        │   ├── n8n-configmap.yaml
        │   ├── evolution-ingressroute.yaml
        │   ├── evolution-certificate.yaml
        │   ├── evolution-configmap.yaml
        └── resource-patches/
            └── traefik-deployment.yaml
```

---

## 3. Fluxo TLS (Let's Encrypt + cert-manager + Traefik)

```text
 1. Você aplica um Certificate (ex: n8n-tls) no namespace n8n.
       spec.secretName : n8n-tls         (NOME FIXO)
       spec.issuerRef  : letsencrypt-prod (ClusterIssuer)

 2. cert-manager cria Order + Challenge HTTP-01.

 3. Para resolver HTTP-01, cert-manager cria um Ingress TEMPORÁRIO com
    ingressClassName=traefik expondo /.well-known/acme-challenge/<token>.

 4. Traefik (porque é o controller da IngressClass `traefik`) roteia esse
    path para o pod solver do cert-manager.

 5. Let's Encrypt acessa http://<dominio>/.well-known/acme-challenge/<token>,
    valida, e emite o certificado.

 6. cert-manager grava o certificado no Secret kubernetes.io/tls com
    o nome FIXO (n8n-tls). O Ingress temporário é removido.

 7. A IngressRoute do app já referencia este Secret:
       tls:
         secretName: n8n-tls
    Traefik faz TLS termination usando o Secret.

 8. ~15 dias antes do vencimento (90d emitido) o cert-manager renova
    sobrescrevendo o MESMO Secret. Zero downtime, zero alteração no app.
```

---

## 4. Fluxo de tráfego

```text
Cliente HTTPS (browser/app)
   │
   ▼
DNS resolve n8n.example.com.br -> IP da VPS
   │
   ▼
VPS recebe na porta 443
   │  k3s ServiceLB (klipper-lb) faz hostPort 443 -> Service Traefik
   ▼
Pod Traefik
   │  - termina TLS usando Secret n8n-tls
   │  - aplica middlewares (security-headers, rate-limit, compress)
   │  - faz match: IngressRoute com Host(`n8n.example.com.br`)
   ▼
Service ClusterIP n8n-main:5678
   │
   ▼
Pod n8n-main  ──> resposta ──> caminho inverso até o cliente

Tráfego HTTP (80) entra no entryPoint `web` e é redirecionado 308 para
`https` pelo middleware nativo configurado no Traefik.
```

---

## 5. Responsabilidades — Traefik vs cert-manager

| Componente   | Faz                                                                       | NÃO faz                                |
|--------------|---------------------------------------------------------------------------|----------------------------------------|
| Traefik      | Roteamento HTTP(S), middlewares, TLS termination via Secret pronto, métricas | NÃO emite/renova certificados (ACME OFF) |
| cert-manager | Emite/renova Certificate via Let's Encrypt; grava Secret TLS nome fixo    | NÃO faz proxy, NÃO termina TLS         |

Vantagens de separar:
- Renovação automática transparente (mesmo Secret).
- Pode-se trocar o Ingress Controller sem perder certificados.
- Funciona para qualquer aplicação que precise de Secret TLS, não só HTTP
  (ex.: TCP TLS).

---

## 6. Análise de capacidade da VPS

| Componente                                   | CPU req | CPU lim | MEM req | MEM lim |
|----------------------------------------------|---------|---------|---------|---------|
| Traefik                                      |  75m    | 700m    | 128Mi   | 320Mi   |
| cert-manager (controller+webhook+cainjector) |  60m    | 300m    | 192Mi   | 384Mi   |
| Postgres 16                                  | 200m    | 1000m   | 512Mi   | 1536Mi  |
| Redis 7                                      |  50m    | 500m    | 128Mi   | 384Mi   |
| n8n-main                                     | 150m    | 800m    | 384Mi   | 768Mi   |
| n8n-webhook                                  | 100m    | 500m    | 256Mi   | 512Mi   |
| Evolution API                                | 100m    | 800m    | 256Mi   | 768Mi   |
| **Σ requests**                               | **855m**|         | **2112Mi (~2.1Gi)** |
| **Σ limits (teórico)**                       | **6550m**|        | **5440Mi (~5.4Gi)** |

VPS = 2000m CPU / 8192Mi RAM.

- **Requests (garantido)**: ~0.85 vCPU / 2.1 GiB → folga grande p/ k3s
  (kube-apiserver, kubelet, containerd, etcd embutido ≈ 0.3 vCPU / 600 MiB)
  e sistema (~300 MiB).
- **Limits (pico)**: a soma dos `limits` é maior que o nó (overcommit
  proposital). Como cargas reais raramente sobem juntas, está ok. O k3s
  vai estrangular CPU (cgroup throttling) e priorizar pelo `requests`.
- **Disco**: PVCs somam 40 GiB (Postgres 20 + Redis 5 + n8n 5 + Evolution
  10), sobra >50 GiB para imagens, logs, snapshots.

Se você ativar HPA e ele subir réplicas (configurado `maxReplicas=2` em
APIs), ainda cabe. Para folga maior, mantenha 1 réplica em prod até
adicionar um segundo node.

---

## 7. Preparação da VPS

> **Premissa**: você está logado como `root@<IP-VPS>` via SSH.

### 7.1 Atualizar o sistema

```bash
apt update && apt -y upgrade
apt -y install curl wget gnupg ca-certificates lsb-release \
                software-properties-common ufw fail2ban unattended-upgrades \
                htop iotop iftop tmux git jq
```

### 7.2 Timezone

```bash
timedatectl set-timezone America/Sao_Paulo
```

### 7.3 Criar usuário não-root

```bash
adduser --gecos "" <user-name>        # responda à senha
usermod -aG sudo <user-name>
mkdir -p /home/<user-name>/.ssh
cp /root/.ssh/authorized_keys /home/<user-name>/.ssh/authorized_keys 2>/dev/null || true
chown -R <user-name>:<user-name> /home/<user-name>/.ssh
chmod 700 /home/<user-name>/.ssh
chmod 600 /home/<user-name>/.ssh/authorized_keys 2>/dev/null || true
```

### 7.4 SSH hardening

Edite `/etc/ssh/sshd_config.d/99-hardening.conf`:

```text
PermitRootLogin no
PasswordAuthentication no
PubkeyAuthentication yes
ChallengeResponseAuthentication no
KbdInteractiveAuthentication no
X11Forwarding no
AllowUsers <user-name>
MaxAuthTries 3
ClientAliveInterval 300
ClientAliveCountMax 2
```

```bash
systemctl reload ssh
```

A partir daqui, abra um **NOVO terminal** e entre como `<user-name>@<IP>` antes
de fechar a sessão root.

### 7.5 Firewall (ufw)

```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow 22/tcp
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
sudo ufw allow 8000/tcp
sudo ufw allow 8443/tcp
sudo ufw --force enable
sudo ufw status verbose
```

### 7.6 Swap (importante com 8 GB)

```bash
sudo fallocate -l 4G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile
echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab
sudo sysctl vm.swappiness=10
echo 'vm.swappiness=10' | sudo tee -a /etc/sysctl.conf
```

### 7.7 Atualizações automáticas de segurança

```bash
sudo dpkg-reconfigure --priority=low unattended-upgrades
```

---

## 8. Instalação do k3s

```bash
# Instala k3s SEM o Traefik que vem por padrão (vamos usar o nosso).
# servicelb (klipper) FICA habilitado — ele expõe Service LoadBalancer
# em hostPort na VPS.
curl -sfL https://get.k3s.io | sh -s - \
  --disable=traefik \
  --write-kubeconfig-mode=644 \
  --node-name=$(hostname) \
  --cluster-init
```

Verifique:

```bash
sudo systemctl status k3s --no-pager
sudo k3s kubectl get nodes
```

Configure `kubectl` para o usuário não-root:

```bash
mkdir -p ~/.kube
sudo cp /etc/rancher/k3s/k3s.yaml ~/.kube/config
sudo chown $(id -u):$(id -g) ~/.kube/config
chmod 600 ~/.kube/config
# Se quiser usar de fora da VPS, edite ~/.kube/config local e troque
# `server: https://127.0.0.1:6443` por `server: https://<IP-VPS>:6443`
echo 'export KUBECONFIG=$HOME/.kube/config' >> ~/.bashrc
source ~/.bashrc

kubectl get nodes
kubectl get pods -A
```

---

## 9. kubectl + kustomize + helm na sua workstation

Você pode operar tudo direto da VPS (`kubectl` já instalado), mas a forma
recomendada é da sua máquina local (workstation Linux/Mac/WSL).

```bash
# kubectl
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
chmod +x kubectl && sudo mv kubectl /usr/local/bin/

# kustomize (CLI é opcional, kubectl já tem `kubectl kustomize`)
curl -s "https://raw.githubusercontent.com/kubernetes-sigs/kustomize/master/hack/install_kustomize.sh" | bash
sudo mv kustomize /usr/local/bin/

# helm (só para instalar cert-manager via Helm, se preferir)
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
bash get_helm.sh && rm get_helm.sh

# Copie o kubeconfig da VPS:
scp lucas@<IP-VPS>:~/.kube/config ~/.kube/config
sed -i "s|server: https://127.0.0.1:6443|server: https://<IP-VPS>:6443|" ~/.kube/config

kubectl get nodes
```

---

## 10. DNS — registros A

Crie estes registros **A** apontando para o IP público da VPS. Substitua
`example.com.br` pelo seu domínio real (e ajuste os patches em
`overlays/prod/domain-patches/` antes de aplicar).

| Subdomínio                     | Tipo | Destino    | Serviço                       |
|--------------------------------|------|------------|-------------------------------|
| `traefik.example.com.br`          | A    | `<YOUR-IP>` | Dashboard do Traefik          |
| `n8n.example.com.br`              | A    | `<YOUR-IP>` | n8n UI/API                    |
| `webhook.n8n.example.com.br`      | A    | `<YOUR-IP>` | n8n webhooks                  |
| `evolution.example.com.br`        | A    | `<YOUR-IP>` | Evolution API                 |

**Aguarde** a propagação (`dig +short n8n.example.com.br` retornando o IP)
antes de aplicar Certificates de produção, senão Let's Encrypt rate-limita.

Para trocar o domínio em massa **antes** de aplicar:

```bash
cd k8s-structure
grep -rl 'example.com' infrastructure/ | xargs sed -i 's/example.com/example.com.br/g'
```

(ou edite só `overlays/prod/domain-patches/*.yaml` e os campos `email:`
em `base/cert-manager/clusterissuer-*.yaml`.)

---

## 11. Instalação do cert-manager

Use o manifest oficial (mais simples e inclui CRDs):

```bash
CERT_MANAGER_VERSION=v1.16.2
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/${CERT_MANAGER_VERSION}/cert-manager.yaml

# Aguarde tudo ficar Ready
kubectl -n cert-manager wait --for=condition=Available deploy --all --timeout=300s
kubectl get pods -n cert-manager
```

Aplique os ClusterIssuers (depois de revisar o e-mail em
`base/cert-manager/clusterissuer-*.yaml`):

```bash
kubectl apply -k infrastructure/base/cert-manager
kubectl get clusterissuers
```

Deve aparecer `letsencrypt-prod` e `letsencrypt-staging` com `READY=True`.

---

## 12. Instalação do Traefik

### 12.1 CRDs do Traefik

Os CRDs do Traefik **não** fazem parte do Kustomize (são objetos
cluster-scoped que sobrevivem a uninstall). Aplique uma única vez:

```bash
TRAEFIK_VERSION=v3.2
kubectl apply -f https://raw.githubusercontent.com/traefik/traefik/${TRAEFIK_VERSION}/docs/content/reference/dynamic-configuration/kubernetes-crd-definition-v1.yml
kubectl get crds | grep traefik.io
```

### 12.2 Base do Traefik

```bash
kubectl apply -k infrastructure/base/traefik
kubectl -n traefik rollout status deploy/traefik --timeout=180s
kubectl -n traefik get svc traefik
```

O `EXTERNAL-IP` do Service `traefik` deve assumir o IP da VPS (k3s
ServiceLB) ou aparecer como `<pending>` por alguns segundos antes de
ganhar IP. Quando ele tiver IP, **teste com curl HTTP**:

```bash
curl -I http://<YOUR-IP>
# Deve responder 404 Not Found (Traefik responde, mas sem rota = 404)
```

### 12.3 BasicAuth do dashboard

```bash
# Gere o htpasswd:
sudo apt -y install apache2-utils
HASH=$(htpasswd -nbB admin '<YOUR_PASSWORD_HERE>')
echo "$HASH"
# admin:$2y$05$...

kubectl -n traefik create secret generic traefik-dashboard-auth \
  --from-literal=users="$HASH" \
  --dry-run=client -o yaml | kubectl apply -f -
```

---
# Parei por aqui!!!
## 13. Criação dos Secrets reais

Os Secrets em Git são **placeholders**. Substitua antes de aplicar os
overlays. O comando abaixo gera tudo de forma segura.

```bash
# 13.1 Postgres credentials (DEPENDÊNCIA: vai ser usado pelas apps).
PG_PASS=$(openssl rand -base64 32 | tr -d '/+=')
N8N_PG_PASS=$(openssl rand -base64 24 | tr -d '/+=')
EVO_PG_PASS=$(openssl rand -base64 24 | tr -d '/+=')
ARCO_PG_PASS=$(openssl rand -base64 24 | tr -d '/+=')
FORMS_PG_PASS=$(openssl rand -base64 24 | tr -d '/+=')

kubectl create namespace databases --dry-run=client -o yaml | kubectl apply -f -
kubectl -n databases create secret generic postgres-credentials \
  --from-literal=POSTGRES_USER=postgres \
  --from-literal=POSTGRES_PASSWORD=$PG_PASS \
  --from-literal=POSTGRES_DB=postgres \
  --from-literal=APP_N8N_PASSWORD=$N8N_PG_PASS \
  --from-literal=APP_EVOLUTION_PASSWORD=$EVO_PG_PASS \
  --from-literal=APP_ARCOBATRIP_PASSWORD=$ARCO_PG_PASS \
  --from-literal=APP_FORMS_PASSWORD=$FORMS_PG_PASS \
  --dry-run=client -o yaml | kubectl apply -f -

# 13.2 Redis credentials
REDIS_PASS=$(openssl rand -base64 32 | tr -d '/+=')
kubectl -n databases create secret generic redis-credentials \
  --from-literal=REDIS_PASSWORD=$REDIS_PASS \
  --dry-run=client -o yaml | kubectl apply -f -

# 13.3 n8n
N8N_ENC=$(openssl rand -hex 32)
N8N_UI_PASS=$(openssl rand -base64 24 | tr -d '/+=')
kubectl create namespace n8n --dry-run=client -o yaml | kubectl apply -f -
kubectl -n n8n create secret generic n8n-secrets \
  --from-literal=N8N_ENCRYPTION_KEY=$N8N_ENC \
  --from-literal=DB_POSTGRESDB_PASSWORD=$N8N_PG_PASS \
  --from-literal=QUEUE_BULL_REDIS_PASSWORD=$REDIS_PASS \
  --from-literal=N8N_BASIC_AUTH_USER=admin \
  --from-literal=N8N_BASIC_AUTH_PASSWORD=$N8N_UI_PASS \
  --dry-run=client -o yaml | kubectl apply -f -

# 13.4 Evolution API
EVO_API_KEY=$(openssl rand -hex 32)
kubectl create namespace evolution --dry-run=client -o yaml | kubectl apply -f -
kubectl -n evolution create secret generic evolution-secrets \
  --from-literal=AUTHENTICATION_API_KEY=$EVO_API_KEY \
  --from-literal=DATABASE_CONNECTION_URI="postgresql://evolution_user:${EVO_PG_PASS}@postgres-rw.databases.svc.cluster.local:5432/evolution?schema=public" \
  --from-literal=CACHE_REDIS_URI="redis://:${REDIS_PASS}@redis.databases.svc.cluster.local:6379/1" \
  --dry-run=client -o yaml | kubectl apply -f -

# 13.7 Anote tudo num cofre (1Password / Bitwarden / Vault)!
cat <<EOF
==================== SENHAS GERADAS ====================
Postgres superuser : $PG_PASS
n8n DB password    : $N8N_PG_PASS
Evolution DB pass  : $EVO_PG_PASS
Redis password     : $REDIS_PASS
n8n encryption key : $N8N_ENC
n8n UI password    : $N8N_UI_PASS (user: admin)
Evolution API key  : $EVO_API_KEY
========================================================
EOF
```

> **Importante — ordem dos applies**: `kubectl apply -k` **SUBSTITUI** as
> chaves de qualquer Secret cujo manifest exista em `base/`. Isso significa
> que se você rodar primeiro `kubectl create secret ...` (com senhas reais)
> e depois aplicar a base, os placeholders `CHANGE_ME_*` do `secret.yaml`
> versionado vão SOBRESCREVER os valores reais. Soluções, escolha UMA:
>
> 1. **(Recomendado para o primeiro deploy)** Remova o `secret.yaml` do
>    `resources:` do `kustomization.yaml` de cada módulo (n8n, evolution,
>    arcobatrip, receive-forms, databases) **antes** de aplicar a base.
>    Os Secrets vivem só no cluster — gerenciados fora do Git.
> 2. **(GitOps puro)** Adote **SealedSecrets** (Bitnami) ou **External
>    Secrets Operator** (ESO) e versione `SealedSecret`/`ExternalSecret`
>    em vez de `Secret`. Os placeholders deste repo devem ser apagados.
> 3. **(Caminho rápido para single-node)** Edite manualmente o `secret.yaml`
>    de cada módulo com `stringData:` real ANTES do `kubectl apply -k`,
>    e mantenha esses arquivos **fora** de `git` (`.gitignore`). Não é
>    GitOps de verdade — só serve para começar.

---

## 14. Deploy Postgres + Redis

```bash
kubectl apply -k infrastructure/base/databases

# Aguardar
kubectl -n databases rollout status statefulset/postgres --timeout=180s
kubectl -n databases rollout status statefulset/redis --timeout=120s

# Verificar
kubectl -n databases get pods,svc,pvc
```

Teste de conexão:

```bash
# Postgres
kubectl -n databases exec -it postgres-0 -- \
  psql -U postgres -c "\l"   # deve listar n8n, evolution, arcobatrip, forms

# Redis
kubectl -n databases exec -it redis-0 -- \
  sh -c 'redis-cli -a "$REDIS_PASSWORD" --no-auth-warning PING'
# -> PONG
```

---

## 15. Deploy n8n

```bash
kubectl apply -k infrastructure/base/n8n

kubectl -n n8n rollout status deploy/n8n-main --timeout=300s
kubectl -n n8n rollout status deploy/n8n-webhook --timeout=180s
kubectl -n n8n get pods,svc,ingressroute,certificate
```

Aguardar o Certificate ficar `READY=True`:

```bash
kubectl -n n8n get certificate -w
```

Quando o Certificate sair de `Issuing` para `Ready`, acesse:
`https://n8n.example.com.br` — pede BasicAuth (admin / senha do passo 13.3).

---

## 16. Deploy Evolution API

```bash
kubectl apply -k infrastructure/base/evolution
kubectl -n evolution rollout status deploy/evolution --timeout=300s
kubectl -n evolution get pods,certificate,ingressroute
```

Teste:

```bash
curl -H "apikey: $EVO_API_KEY" https://evolution.example.com.br/instance/fetchInstances
```

---

## 17. Validação TLS

```bash
# 1. Todos os Certificates devem estar Ready
kubectl get certificates -A
# NAMESPACE       NAME              READY  SECRET                  AGE
# evolution       evolution         True   evolution-tls           1h
# n8n             n8n-main          True   n8n-tls                 1h
# n8n             n8n-webhook       True   n8n-webhook-tls         1h
# traefik         traefik-dashboard True   traefik-tls             1h

# 2. Inspecionar challenges em andamento (se algum não ficou Ready)
kubectl get challenges -A
kubectl get orders -A
kubectl describe certificate <NAME> -n <NS>

# 3. Validar HTTPS (cadeia válida pela Let's Encrypt)
for host in traefik n8n webhook.n8n evolution; do
  echo "== $host.example.com.br =="
  curl -sI "https://${host}.example.com.br/" | head -1
done

# 4. Inspecionar o cert real entregue
openssl s_client -connect n8n.example.com.br:443 -servername n8n.example.com.br </dev/null 2>/dev/null \
  | openssl x509 -noout -issuer -subject -dates
```

---

## 20. GitOps com ArgoCD (opcional)

```bash
# Instalar ArgoCD
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# Senha inicial
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d

# Port-forward só para o setup inicial (depois exponha via IngressRoute)
kubectl -n argocd port-forward svc/argocd-server 8080:443
```

Crie uma Application apontando para o overlay:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: example-prod
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/<seu-user>/k8s-structure.git
    path: infrastructure/overlays/prod
    targetRevision: main
  destination:
    server: https://kubernetes.default.svc
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
      - ServerSideApply=true
```

> Em produção, NUNCA commite Secrets. Use **SealedSecrets** (Bitnami) ou
> **External Secrets Operator** (ESO). Manter os `secret.yaml` em base/
> apenas como **placeholder não-aplicado** (remova do `kustomization.yaml`
> após o primeiro setup e gerencie os Secrets reais por outro caminho).

---

## 21. Backup e manutenção

### 21.1 Backup do Postgres (cron diário)

Crie `/usr/local/bin/backup-postgres.sh`:

```bash
#!/usr/bin/env bash
set -euo pipefail
TS=$(date +%Y%m%d-%H%M%S)
BACKUP_DIR=/var/backups/postgres
mkdir -p "$BACKUP_DIR"
kubectl -n databases exec postgres-0 -- \
  sh -c 'pg_dumpall -U "$POSTGRES_USER" | gzip' > "$BACKUP_DIR/all-$TS.sql.gz"
# Mantém 14 dias
find "$BACKUP_DIR" -name 'all-*.sql.gz' -mtime +14 -delete
```

Crontab:

```bash
sudo chmod +x /usr/local/bin/backup-postgres.sh
echo "0 3 * * * root /usr/local/bin/backup-postgres.sh" | sudo tee /etc/cron.d/postgres-backup
```

### 21.2 Restore

```bash
gunzip -c /var/backups/postgres/all-YYYYMMDD-HHMMSS.sql.gz | \
  kubectl -n databases exec -i postgres-0 -- psql -U postgres
```

### 21.3 Atualização de imagens (sem downtime)

```bash
# Editar a tag no manifest e:
kubectl -n n8n set image deployment/n8n-main n8n=n8nio/n8n:1.67.0
kubectl -n n8n rollout status deployment/n8n-main
# Rollback:
kubectl -n n8n rollout undo deployment/n8n-main
kubectl -n n8n rollout status deployment/n8n-main
```

### 21.4 Rotação de logs do k3s

k3s já usa journald (systemd). Para limitar:

```bash
sudo mkdir -p /etc/systemd/journald.conf.d
cat <<EOF | sudo tee /etc/systemd/journald.conf.d/k3s.conf
[Journal]
SystemMaxUse=2G
MaxFileSec=1week
EOF
sudo systemctl restart systemd-journald
```

### 21.5 Renovação certificados — verificar

```bash
kubectl get certificate -A -o custom-columns=NAMESPACE:.metadata.namespace,NAME:.metadata.name,READY:.status.conditions[0].status,EXPIRY:.status.notAfter
```

cert-manager renova automaticamente 15 dias antes (`renewBefore: 360h`).

---

## 22. Troubleshooting

### 22.1 Pod não fica pronto

```bash
kubectl -n <NS> describe pod <POD>
kubectl -n <NS> logs <POD> --previous
kubectl -n <NS> get events --sort-by=.lastTimestamp
```

### 22.2 Certificate stuck em `Issuing`

```bash
kubectl describe certificate <NAME> -n <NS>
kubectl describe certificaterequest -n <NS>
kubectl describe order -n <NS>
kubectl describe challenge -n <NS>

# Verifique se o domínio resolve ao IP da VPS:
dig +short <host>

# Verifique se o solver HTTP-01 está acessível externamente:
curl -v http://<host>/.well-known/acme-challenge/test
```

Se persistir, force a recriação:

```bash
kubectl delete certificate <NAME> -n <NS>
kubectl delete secret <SECRET-TLS> -n <NS>
kubectl apply -k infrastructure/base/<MODULE>
```

### 22.3 Traefik 404 mesmo com DNS apontado

```bash
kubectl -n traefik logs deploy/traefik --tail=200 -f
# Procure por "Host(...) matched" ou erros de TLS
kubectl get ingressroute -A
kubectl describe ingressroute <NAME> -n <NS>
```

### 22.4 Logs do cert-manager

```bash
kubectl -n cert-manager logs deploy/cert-manager -f
kubectl -n cert-manager logs deploy/cert-manager-webhook
```

### 22.5 Tudo do cluster de uma vez (visão geral)

```bash
kubectl get all -A
kubectl get pv,pvc -A
kubectl top nodes
kubectl top pods -A --sort-by=memory | head -20
```

### 22.6 Reset emergencial de um deployment

```bash
kubectl -n <NS> rollout restart deployment/<NAME>
kubectl -n <NS> rollout undo deployment/<NAME>
```

### 22.7 Acessar Postgres por psql local (sem expor publicamente)

```bash
kubectl -n databases port-forward svc/postgres-rw 5432:5432
# Em outro terminal:
psql "host=127.0.0.1 port=5432 user=postgres dbname=n8n"
```

---

## Apêndice A — Resumo de comandos para o primeiro deploy

```bash
# 1. VPS preparada (passos 7.x)
# 2. k3s instalado (passo 8)
# 3. DNS apontado (passo 10)

# 4. Substituir example.com.br pelos seus domínios:
sed -i 's/example.com.br/seudominio.com.br/g' $(grep -rl example.com.br infrastructure/)
sed -i 's/admin@example.com.br/seu-email@dominio.com/g' \
  infrastructure/base/cert-manager/clusterissuer-*.yaml

# 5. cert-manager
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.16.2/cert-manager.yaml
kubectl -n cert-manager wait --for=condition=Available deploy --all --timeout=300s
kubectl apply -k infrastructure/base/cert-manager

# 6. Traefik CRDs + base
kubectl apply -f https://raw.githubusercontent.com/traefik/traefik/v3.2/docs/content/reference/dynamic-configuration/kubernetes-crd-definition-v1.yml
kubectl apply -k infrastructure/base/traefik

# 7. BasicAuth dashboard
HASH=$(htpasswd -nbB admin 'SUA_SENHA_AQUI')
kubectl -n traefik create secret generic traefik-dashboard-auth \
  --from-literal=users="$HASH" --dry-run=client -o yaml | kubectl apply -f -

# 8. Secrets reais (passo 13) — RODE TUDO

# 9. Databases
kubectl apply -k infrastructure/base/databases

# 10. Apps
kubectl apply -k infrastructure/base/n8n
kubectl apply -k infrastructure/base/evolution
kubectl apply -k infrastructure/base/arcobatrip
kubectl apply -k infrastructure/base/receive-forms

# 11. Validação
kubectl get certificates -A
kubectl get pods -A
```

Ou, depois de tudo estabilizado, em um único shot via overlay:

```bash
kubectl apply -k infrastructure/overlays/prod
```

---

Bom deploy. Em caso de dúvida, comece sempre por `kubectl get events -A
--sort-by=.lastTimestamp | tail -30`.
