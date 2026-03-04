# ADR-004: ArgoCD como Ferramenta de Entrega Contínua GitOps

- **Status:** Aceito
- **Data:** 2026-02-21
- **Decisores:** Time de Plataforma

---

## Contexto

A equipe precisava de uma ferramenta GitOps para sincronizar o estado dos repositórios
com os clusters Kubernetes. Os requisitos eram:

1. **Pull-based delivery** — o cluster puxa as mudanças, não o pipeline empurra
2. **Multi-cluster** — capaz de gerenciar deploys em `bu-acme-ho` e `bu-acme-pr` a partir de um ponto central
3. **UI visual** — observabilidade do estado de sincronização sem uso de CLI
4. **Kustomize/Helm nativo** — suporte aos padrões de templating já usados
5. **Drift detection** — alertar se o estado do cluster divergir do repositório

---

## Decisão

**Adotamos o ArgoCD como ferramenta de entrega contínua GitOps.**

ArgoCD roda no cluster de gerenciamento (`gerencia-ho`) e é responsável por:
- Sincronizar os manifestos dos repositórios para os clusters worker
- Servir como UI de observabilidade do estado da plataforma
- Ser a "cola" entre o repositório Git e o OCM Hub

---

## Consequências

**Positivas:**
- **Pull-based nativo:** O agente ArgoCD no Hub puxa as mudanças sem credenciais expostas em pipelines
- **ApplicationSet:** Permite criar N aplicações a partir de um template + lista de clusters
- **Drift detection automático:** ArgoCD detecta e corrige qualquer mudança manual no cluster
- **Integração com OCM:** ArgoCD usa clusters registrados no OCM como destinos de deploy
- **Projeto CNCF Graduated:** Usado em produção por Intuit, Red Hat, Tesla, Alibaba

**Negativas / Trade-offs aceitos:**
- ArgoCD adiciona ~200MB de RAM no cluster Hub
- Requer manutenção de atualização do próprio ArgoCD

---

## Modelo de Uso no Projeto

```
gitops-global-demo (repositório)
        │
        │  (polling / webhook)
        ▼
   ArgoCD (em gerencia-ho)
        │
        │  aplica manifests (OCM Policies, Placements)
        ▼
   OCM Hub (em gerencia-ho)
        │
        │  distribui para clusters selecionados por label
        ├──▶ bu-acme-ho  (label: env=ho, bu=bu-acme)
        └──▶ bu-acme-pr  (label: env=pr, bu=bu-acme)
```

---

## Alternativas Consideradas

### ❌ Flux CD
**Por que não escolhido:** Sem UI nativa; menor adoção corporativa no Brasil.
> Nota: Flux seria uma escolha igualmente válida. A decisão é contextual, não técnica.

### ❌ Pipeline CI/CD push-based (GitHub Actions / Jenkins)
**Por que descartado:** Credenciais expostas, sem drift detection, não é GitOps puro.

### ❌ Helm standalone
**Por que descartado:** Sem reconcile loop, sem multi-cluster nativo, não integra com OCM.

---

## Referências

- [ArgoCD — CNCF Graduated Project](https://www.cncf.io/projects/argo/)
- [ArgoCD Multi-Cluster Documentation](https://argo-cd.readthedocs.io/en/stable/operator-manual/declarative-setup/#clusters)
- [ArgoCD ApplicationSet Controller](https://argo-cd.readthedocs.io/en/stable/user-guide/application-set/)
- [GitOps Principles — CNCF OpenGitOps](https://opengitops.dev/)
