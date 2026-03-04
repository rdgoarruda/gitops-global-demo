# ADR-002: Branch Única (main) + Overlays por Ambiente

- **Status:** Aceito
- **Data:** 2026-02-21
- **Decisores:** Time de Plataforma

---

## Contexto

Com múltiplos ambientes (`ho`, `pr`) sendo gerenciados a partir do mesmo repositório
de políticas, precisávamos definir como controlar o que vai para cada ambiente de forma
segura, auditável e sem introduzir complexidade operacional desnecessária.

---

## Decisão

**Adotamos uma única branch protegida (`main`) com separação de ambientes por diretório
(Kustomize overlays) e controle de acesso via `CODEOWNERS`.**

```
main (branch protegida)
├── governance/
│   └── <categoria>/
│       ├── base/               # Recursos base da categoria
│       └── overlays/
│           ├── ho/             # Configurações de homologação
│           └── pr/             # Configurações de produção (CODEOWNERS restrito)
├── config/
│   ├── ho/                     # ConfigMaps e Placements para HO
│   └── pr/                     # ConfigMaps e Placements para PR
└── .github/
    └── CODEOWNERS
```

### Configuração de CODEOWNERS
```
# Produção exige aprovação explícita do time SRE
**/overlays/pr/**    @org/sre-team
config/pr/**         @org/sre-team

# Base afeta todos os ambientes — aprovação ampla
**/base/**           @org/platform-team

# Não-produção — qualquer membro do time de plataforma
**/overlays/ho/**    @org/platform-team
config/ho/**         @org/platform-team
```

### GitHub Branch Protection Rules (`main`)
- ✅ Require pull request before merging
- ✅ Require approvals: mínimo 1 (ho) / 2 (pr via CODEOWNERS)
- ✅ Require review from CODEOWNERS
- ✅ Dismiss stale reviews when new commits are pushed
- ✅ Require status checks (lint, `kustomize build`)

---

## Consequências

**Positivas:**
- **História linear:** Todo o estado da plataforma visível numa única branch
- **Promoção explícita:** Ir de ho para pr exige um segundo PR deliberado
- **Auditoria por path:** Visível quem aprovou cada mudança em pr via histórico de PR
- **Sem merge hell:** Não há necessidade de manter branches sincronizadas

**Negativas / Trade-offs aceitos:**
- Requer disciplina de PR: mudanças em `overlays/pr/` devem ser PRs separados de `overlays/ho/`

---

## Alternativas Consideradas

### ❌ Multi-branch por ambiente (main, ho, pr)
**Por que descartado:** Complexidade operacional, divergência inevitável e histórico confuso.

### ❌ Tags por ambiente
**Por que descartado:** Inviável para configurações que mudam frequentemente.

### ❌ Sem CODEOWNERS
**Por que descartado:** Sem rastreabilidade de quem é responsável por cada path.

---

## Referências

- [ArgoCD Best Practices — Repository Structure](https://argo-cd.readthedocs.io/en/stable/user-guide/best_practices/)
- [OpenGitOps Principles — CNCF](https://opengitops.dev/)
- [GitHub CODEOWNERS Documentation](https://docs.github.com/en/repositories/managing-your-repositorys-settings-and-features/customizing-your-repository/about-code-owners)
