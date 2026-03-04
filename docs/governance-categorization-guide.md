# Guia de Categorização de Políticas (OCM)

Este documento define os critérios para classificar novas políticas de governança no ecossistema multi-cluster.
Cada categoria é um **módulo auto-descoberto pelo ArgoCD** via Git Generator — basta criar a pasta, e o ArgoCD cria a Application automaticamente.

---

## Hierarquia de Governança

```
governance/
├── platform/        → estrutura base do cluster
├── security/        → hardening e restrições
├── observability/   → telemetria e visibilidade
├── capacity/        → quotas e limites de recursos
├── compliance/      → frameworks externos (NIST, CIS)
└── infrastructure/  → componentes e storage
```

---

## Categorias

### 1. `platform` — Plataforma Base
Foco na estrutura básica necessária para que outros times operem no cluster.

| Critério | Exemplo |
|---|---|
| Cria namespaces mandatórios | `platform-ops`, `shared-services` |
| Define RBAC global | ClusterRoles para times de infra |
| Configura ServiceAccounts de plataforma | agents, operators |

**Políticas atuais:**
- `policy-platform-namespace` (`enforce`, medium) — Garante que o namespace `platform-ops` existe com labels `managed-by: ocm`

---

### 2. `security` — Segurança
Hardening do cluster e proteção contra workloads maliciosos ou mal configurados.

| Critério | Exemplo |
|---|---|
| Restringe comportamentos de containers | Sem `privileged: true` |
| Controla tráfego de rede | NetworkPolicies default-deny |
| Valida origem de imagens | Registry whitelist |

**Políticas atuais:**
- `policy-security-privileged-denial` (`enforce`, high) — Bloqueia pods com `privileged: true` em todos os namespaces

---

### 3. `observability` — Observabilidade
Garante que todos os clusters tenham a infraestrutura de telemetria funcional.

| Critério | Exemplo |
|---|---|
| Cria namespaces de monitoramento | `observability-ops`, `monitoring` |
| Instala agentes de coleta | Prometheus node-exporter, Fluentbit |
| Configura destinos de logs | Log Forwarder para SIEM central |

**Políticas atuais:**
- `policy-observability-namespace` (`enforce`, low) — Garante que o namespace `observability-ops` existe nos clusters

---

### 4. `capacity` — Capacidade
Evita o problema de "noisy neighbor" e garante sustentabilidade do cluster em multi-tenancy.

| Critério | Exemplo |
|---|---|
| Define limites de CPU/memória | LimitRange, ResourceQuota por namespace |
| Configura prioridades de workload | PriorityClasses |
| Monitora consumo excessivo | Alertas de quota |

**Políticas atuais:**
- `policy-capacity-default-limits` (`enforce`, medium) — Aplica LimitRange nos namespaces de workloads
- `policy-capacity-default-quota` (`enforce`, medium) — Aplica ResourceQuota nos namespaces de workloads

---

### 5. `compliance` — Conformidade
Aplicação de benchmarks e frameworks de conformidade externos ou regulatórios.

| Critério | Exemplo |
|---|---|
| Controles NIST SP 800-53 | Auditoria de criação de recursos privilegiados |
| CIS Kubernetes Benchmark | Configurações de API server |
| Requisitos de auditoria interna | Logs de acesso habilitados |

**Políticas atuais:**
- `policy-compliance-nist-baseline` (`enforce`, critical) — Aplica controles NIST SP 800-190 nos clusters

---

### 6. `infrastructure` — Infraestrutura
Gestão de componentes e do ciclo de vida de recursos de infraestrutura do cluster.

| Critério | Exemplo |
|---|---|
| Garante StorageClasses disponíveis | `gp2`, `standard` |
| Instala Operators via OLM | CatalogSource, Subscription |
| Configura recursos de infra | sysctl, kernel parameters |

**Políticas atuais:**
- `policy-infra-storage-class` (`enforce`, medium) — Garante que a StorageClass `standard` existe no cluster

---

## Como Criar uma Nova Política

### Passo a Passo

**1. Identifique a categoria** usando a tabela acima.

**2. Crie o manifest da Policy** em `governance/<categoria>/base/policy-<nome>.yaml`:

```yaml
apiVersion: policy.open-cluster-management.io/v1
kind: Policy
metadata:
  name: policy-<categoria>-<descricao>
spec:
  remediationAction: enforce   # ou inform (apenas reportar, não corrigir)
  disabled: false
  policy-templates:
    - objectDefinition:
        apiVersion: policy.open-cluster-management.io/v1
        kind: ConfigurationPolicy
        metadata:
          name: policy-<categoria>-<descricao>
        spec:
          remediationAction: enforce
          severity: low | medium | high | critical
          object-templates:
            - complianceType: musthave
              objectDefinition:
                # o recurso Kubernetes que deve existir nos clusters
```

**3. Adicione ao PolicySet** em `governance/<categoria>/base/policyset.yaml`:

```yaml
apiVersion: policy.open-cluster-management.io/v1beta1
kind: PolicySet
metadata:
  name: policy-set-<categoria>
spec:
  policies:
    - policy-<categoria>-<existente>
    - policy-<categoria>-<nova>   # adicionar aqui
```

**4. Atualize o `kustomization.yaml`** da categoria (`base/`):

```yaml
resources:
  - policyset.yaml
  - policy-<existente>.yaml
  - policy-<nova>.yaml       # adicionar aqui
```

> Se a categoria já existe, os passos de binding não são necessários — o binding já existe e cobre novas políticas automaticamente via PolicySet.

---

## Boas Práticas

| Prática | Descrição |
|---|---|
| **Sem namespace fixo nas políticas** | Nunca hardcode `namespace:` dentro dos YAMLs em `governance/`. O ArgoCD injeta o namespace via ApplicationSet |
| **Uma responsabilidade por Policy** | Mantenha políticas atômicas |
| **`inform` antes de `enforce`** | Em ambientes novos, comece com `remediationAction: inform` para ver o impacto antes de enforçar |
| **Nome consistente** | Padrão: `policy-<categoria>-<descricao-curta>` |
| **Severity proporcional** | `critical` apenas para conformidade regulatória; `high` para segurança; `medium`/`low` para operacional |
