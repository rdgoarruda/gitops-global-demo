# ADR-001: Estratégia de 3 Repositórios GitOps

- **Status:** Aceito
- **Data:** 2026-02-21
- **Decisores:** Time de Plataforma

---

## Contexto

Ao estruturar uma plataforma multi-cluster com unidade de negócio (BU ACME),
é necessário definir onde e como os artefatos GitOps são armazenados.

O ambiente alvo consiste em:
- Um cluster de gerenciamento central (`gerencia-ho`)
- Clusters worker por ambiente de negócio (`bu-acme-ho`, `bu-acme-pr`)
- Equipes distintas: Infraestrutura, Plataforma/SecOps, e Desenvolvimento de Aplicações

---

## Decisão

**Adotamos a estrutura de 3 repositórios com responsabilidade claramente separada:**

| Repositório | Responsabilidade | Ferramental |
|---|---|---|
| `gitops-ocm-foundation-demo` | Provisionamento de clusters e bootstrap do ambiente | Scripts + Kind + clusteradm |
| `gitops-global-demo` | Governança, segurança e configurações mandatórias | ArgoCD + OCM |
| `gitops-bu-acme-demo` | Aplicações de negócio e ferramentas por BU | ArgoCD ApplicationSet |

---

## Consequências

**Positivas:**
- **Separação de responsabilidades (SoC):** Times de Infra, Plataforma e Dev operam independentemente
- **Blast radius controlado:** Um erro em `gitops-bu-acme-demo` não afeta políticas de segurança em `gitops-global-demo`
- **Permissões granulares:** O repositório de políticas pode ter acesso restrito ao time de SecOps
- **Audit trail limpo:** O histórico de um repositório reflete apenas mudanças do seu domínio

**Negativas / Trade-offs aceitos:**
- Necessita de 3 fluxos de PR separados para mudanças que afetam camadas diferentes
- Coordenação entre repositórios pode ser necessária

---

## Alternativas Consideradas

### ❌ Monorepo único
**Por que descartado:** Mistura de responsabilidades cria conflitos entre times e permissões all-or-nothing.

### ❌ Repositório por cluster
**Por que descartado:** Drift inevitável e não escala com múltiplos clusters.

### ❌ 2 repositórios (Infra + App)
**Por que descartado:** Políticas de governança têm ciclo de vida completamente diferente de workloads.

---

## Referências

- [GitOps Working Group — Best Practices](https://github.com/open-gitops/documents/blob/main/PRINCIPLES.md)
- [ArgoCD User Guide — Repository Structure](https://argo-cd.readthedocs.io/en/stable/user-guide/best_practices/)
- [OpenGitOps Principles — CNCF](https://opengitops.dev/)
