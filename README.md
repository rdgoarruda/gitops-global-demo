# gitops-global-demo

Repositório central de **governança, políticas e infraestrutura lógica** da plataforma multi-cluster.

Faz parte de uma estratégia de [3 repositórios GitOps](docs/ADR-001-three-repo-gitops-strategy.md), sendo
o responsável pela camada de plataforma e segurança — tudo que é mandatório independente de qual
aplicação roda no cluster.

> ⚠️ **Importante:** Substitua `rdgoarruda` em todos os arquivos YAML pelo seu usuário/organização GitHub.
> Execute: `grep -r "rdgoarruda" . --include="*.yaml" -l`

---

## Ambiente de Referência

O ambiente é composto por **4 clusters Kubernetes** (Kind em laboratório), organizados em 2 ambientes isolados:

| Cluster | Papel | Contexto kubectl | Ambiente |
|---|---|---|---|
| `gerencia-ho` | Hub HO — ArgoCD + OCM Hub + Headlamp | `kind-gerencia-ho` | Homologação |
| `bu-acme-ho` | Worker HO — BU ACME | `kind-bu-acme-ho` | Homologação |
| `gerencia-pr` | Hub PR — ArgoCD + OCM Hub + Headlamp | `kind-gerencia-pr` | Produção |
| `bu-acme-pr` | Worker PR — BU ACME | `kind-bu-acme-pr` | Produção |

> **Dual-Hub:** Cada ambiente (HO/PR) tem seu próprio cluster de gerência com ArgoCD e OCM Hub
> independentes. Garante isolamento total entre homologação e produção.

> **Nota OCM/RHACM:** O ambiente usa [OCM](https://open-cluster-management.io/) (open-source) como
> engine de governança. As APIs (`Policy`, `Placement`, `PlacementBinding`) são 100% compatíveis com
> Red Hat ACM — o código funciona em produção sem alteração.

---

## Estrutura do Repositório

```text
gitops-global-demo/
├── bootstrap/                       # 🚀 Ponto de entrada do ArgoCD — App-of-Apps
│   ├── ho/
│   │   ├── root-app.yaml            # Application instalada manualmente 1x no hub HO
│   │   ├── app-argocd-pull-integration.yaml  # ArgoCD pull integration para OCM
│   │   ├── app-ocm-config.yaml      # Infraestrutura lógica OCM (namespaces, placements)
│   │   ├── appset-governance.yaml   # ApplicationSet: autodescobre governance/*
│   │   ├── app-haproxy-bu-acme-ho.yaml  # HAProxy Ingress no cluster bu-acme-ho
│   │   ├── appset-haproxy-ho.yaml   # ApplicationSet: instala HAProxy nos workers
│   │   ├── appset-headlamp-ho.yaml  # ApplicationSet: instala Headlamp nos workers
│   │   └── kustomization.yaml
│   └── pr/                          # Mesma estrutura para produção
│
├── config/                          # ⚙️ Infraestrutura lógica do Hub OCM
│   ├── ho/
│   │   ├── namespace.yaml           # Namespace "ocm-policies-ho" no Hub
│   │   ├── acm-placement-configmap.yaml  # Duck-typing: ensina ArgoCD a ler PlacementDecisions
│   │   ├── placement.yaml           # Placement compartilhado (governance/PolicySets)
│   │   ├── clusterset-binding.yaml
│   │   ├── binding-*.yaml (×6)      # Liga cada PolicySet ao Placement ho
│   │   └── kustomization.yaml
│   └── pr/                          # Mesma estrutura para produção
│
├── domains/                         # 📁 ApplicationSets e Bootstraps por BU
│   └── bu-acme/
│       ├── bootstrap/ho/            # root-bu-acme-ho.yaml → domains/bu-acme/ho
│       ├── bootstrap/pr/            # root-bu-acme-pr.yaml → domains/bu-acme/pr
│       ├── ho/
│       │   ├── appset-tools-ho.yaml # ApplicationSet: clusterDecisionResource + Matrix Git
│       │   ├── placement.yaml       # Placement bu-acme-placement-ho no namespace argocd
│       │   └── kustomization.yaml
│       └── pr/
│           ├── appset-tools-pr.yaml
│           └── placement.yaml
│
├── governance/                      # ⚖️ Políticas OCM (auto-descobertas pelo ArgoCD)
│   ├── capacity/                    # ResourceQuotas, LimitRanges mandatórios
│   ├── compliance/                  # Controles NIST SP 800-190, CIS
│   ├── infrastructure/              # StorageClasses, Operators
│   ├── observability/               # Monitoramento e log
│   ├── platform/                    # Namespaces base, RBAC
│   └── security/                    # Pod Security Standards, NetworkPolicies
│
├── infrastructure/                  # 🏗️ Componentes de infraestrutura dos workers
│   └── headlamp/
│       ├── base/                    # Ingress base (template)
│       └── overlays/                # Por cluster: bu-acme-ho, bu-acme-pr
│
└── docs/                            # 📚 ADRs e guias de arquitetura
```

---

## Arquitetura de Deploy — `clusterDecisionResource`

A segregação de deployments por BU é feita através de um **Matrix Generator** no ApplicationSet,
combinando um **Git Generator** (autodescobre pastas de tools) com um **clusterDecisionResource
Generator** (seleciona o cluster-alvo via OCM Placement):

```
gitops-bu-acme-demo/ho/tools/sample-app/    ← Git Generator detecta pasta
         +
bu-acme-placement-ho (Placement OCM)        ← seleciona apenas bu-acme-ho
         ↓
ApplicationSet gera Application             → destino: name: bu-acme-ho
         ↓
ArgoCD deploya no cluster bu-acme-ho        ← usando o cluster secret correto
```

---

## Como Inicializar (Bootstrap)

### 1. Aplicar root-app no Hub HO

```bash
# Aplicar o App-of-Apps raiz (único comando manual!)
kubectl --context kind-gerencia-ho apply -f bootstrap/ho/root-app.yaml

# Aguardar sync no ArgoCD: http://argocd-ho.local
```

### 2. Aplicar roots de domínio

```bash
# Root do domínio BU ACME — HO
kubectl --context kind-gerencia-ho apply -f domains/bu-acme/bootstrap/ho/root-bu-acme-ho.yaml
```

### 3. Repetir para PR

```bash
kubectl --context kind-gerencia-pr apply -f bootstrap/pr/root-app.yaml
kubectl --context kind-gerencia-pr apply -f domains/bu-acme/bootstrap/pr/root-bu-acme-pr.yaml
```

---

## Políticas Mandatórias (OCM Governance)

| Categoria | Política | Severidade | Ação |
|---|---|---|---|
| `security` | Bloqueia containers privileged | high | enforce |
| `platform` | Garante namespace platform-ops | medium | enforce |
| `compliance` | NIST SP 800-190 (hostPID/IPC/Network) | critical | enforce |
| `capacity` | LimitRange padrão nos workloads | medium | enforce |
| `observability` | Namespace observability-ops | low | enforce |
| `infrastructure` | StorageClass padrão | medium | enforce |
