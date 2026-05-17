# VPS humilde — Infraestrutura Kubernetes

Manifestos Kubernetes production-ready para a stack da VPS humilde,
organizados com Kustomize (base + overlays/prod) e prontos para GitOps
(ArgoCD / FluxCD).

> Stack: **k3s 1.30+** · **Traefik 3.2** · **cert-manager 1.16** · **Let's Encrypt** · **Kustomize 5**
> Target VPS: Hostinger KVM2 · Ubuntu 26.04 · 2 vCPU · 8 GB RAM · 100 GB

---

## Documento principal

Para instalar do zero, siga **[INSTALL.md](./INSTALL.md)** (passo-a-passo
desde a VPS Ubuntu limpa até todos os serviços rodando em produção, com
DNS, Let's Encrypt e validação).

---

## Visão rápida da arquitetura

```text
Cliente HTTPS
    │
    ▼
DNS *.example.com  ─►  IP da VPS  ─►  k3s ServiceLB (80/443)
                                       │
                                       ▼
                                  Traefik (Ingress Controller)
                                       │  IngressRoute (traefik.io/v1alpha1)
                        ┌──────────────┬──────────────────────┐
                        ▼              ▼                      ▼
                      n8n-main     n8n-webhook            Evolution
                      │ │              │                     │ │
                      └─┴──────────────┴── PostgreSQL 16 ────┘ │
                       Redis 7 (cache + queue)                 │
                                                               │
                       cert-manager (Let's Encrypt HTTP-01 via Traefik)
```

**Princípios arquiteturais**

| Componente   | Responsabilidade                                                      | Não faz                                         |
|--------------|-----------------------------------------------------------------------|-------------------------------------------------|
| Traefik      | Roteamento HTTP(S), middlewares, TLS termination (Secret pronto)      | NÃO emite/renova certificados (ACME desativado) |
| cert-manager | Emite/renova Certificate via Let's Encrypt; grava Secret de nome fixo | NÃO faz proxy / NÃO termina TLS                 |
| Postgres     | Banco único, databases lógicos separados por app                      | (sem replicação ainda)                          |
| Redis        | Cache + filas Bull do n8n + cache do Evolution                        | (sem cluster ainda)                             |

---

## Estrutura

```text
infrastructure/
├── INSTALL.md                  ← guia operacional ponta-a-ponta
├── README.md                   ← este arquivo
├── base/
│   ├── traefik/                ← Ingress Controller + dashboard
│   ├── cert-manager/           ← namespace + ClusterIssuers (prod/staging)
│   ├── databases/
│   │   ├── postgres/           ← StatefulSet + PVC + init de DBs/roles
│   │   └── redis/              ← StatefulSet + PVC + senha por Secret
│   ├── n8n/                    ← main + webhook (modo queue)
│   ├── evolution/              ← Evolution API v2
└── overlays/
    └── prod/
        ├── domain-patches/     ← substitui example.com pelos seus domínios
        └── resource-patches/   ← ajustes de recursos para VPS pequena
```

---

## Comandos essenciais

```bash
# Validar o build dos overlays sem aplicar
kubectl kustomize infrastructure/overlays/prod

# Aplicar tudo (depois que cert-manager + Traefik já estão instalados)
kubectl apply -k infrastructure/overlays/prod

# Status geral
kubectl get pods,certificates,ingressroute -A
kubectl get events -A --sort-by=.lastTimestamp | tail -30
```

---

## Regras de ouro (do projeto)

1. **NUNCA** usar ACME interno do Traefik.
2. **NUNCA** usar `secretGenerator` para TLS.
3. **NUNCA** Secrets TLS com hash/sufixo aleatório — sempre nome fixo
   (`<app>-tls`).
4. Todos os Certificates são gerenciados pelo cert-manager.
5. Traefik = APENAS ingress / reverse proxy.
6. Kustomize com `generatorOptions: { disableNameSuffixHash: true }`.
7. Cada serviço em seu próprio namespace.
8. Resource requests/limits, readiness + liveness + startup probes,
   `runAsNonRoot`, `readOnlyRootFilesystem` quando possível.
9. NetworkPolicies por namespace, restritivas por padrão.
10. Versões fixas de imagem (jamais `latest` em produção).

---

## Próximos passos sugeridos

- Substituir o Postgres single-node por **CloudNativePG** (replicação +
  backup S3 nativos).
- Implantar **Prometheus + Grafana** consumindo o entrypoint `metrics`
  do Traefik (já habilitado).
- Migrar Secrets para **SealedSecrets** ou **External Secrets Operator**.
- Adicionar **Loki + Promtail** para agregação de logs.
- Trocar o backup local do Postgres por **Velero** (snapshots+S3).
