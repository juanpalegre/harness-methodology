# Mapeo de Plataformas — Equivalencias Agnósticas

> El harness es agnóstico. La implementación varía por plataforma. Esta tabla te permite traducir.

---

## Governance y Branch Protection

| Concepto agnóstico | GitHub | Azure DevOps | GitLab |
|---|---|---|---|
| **Protección de rama principal** | Branch protection rules | Branch policies | Protected branches |
| **Requerir N aprobaciones** | Required reviews (N) | Minimum reviewers (N) | Approvals (N) |
| **Ownership por área** | CODEOWNERS | Automatic reviewers (path-based) | CODEOWNERS |
| **Bloquear push directo** | Restrict push to matching branches | Block direct push | Push rules: no direct push |
| **Requerir CI verde** | Required status checks | Build validation policy | Pipeline must succeed |
| **Firmar commits** | Require signed commits | — (no nativo) | Reject unsigned commits |
| **Reglas granulares** | Repository rulesets | Branch policies + pipeline gates | Push rules + approval rules |

---

## CI/CD Pipeline

| Concepto agnóstico | GitHub Actions | Azure Pipelines | GitLab CI |
|---|---|---|---|
| **Archivo de config** | `.github/workflows/*.yml` | `azure-pipelines.yml` | `.gitlab-ci.yml` |
| **Trigger en MR/PR** | `on: pull_request` | `trigger: pr` | `rules: - if: $CI_MERGE_REQUEST_IID` |
| **Trigger en merge a main** | `on: push: branches: [main]` | `trigger: branches: [main]` | `rules: - if: $CI_COMMIT_BRANCH == "main"` |
| **Stages** | `jobs:` con `needs:` | `stages:` → `jobs:` | `stages:` → `jobs:` |
| **Variables/secrets** | `secrets.*` (repo/org/env) | Variable groups + pipeline vars | CI/CD variables (project/group) |
| **Cache** | `actions/cache` | `Cache@2` task | `cache:` keyword |
| **Artefactos** | `actions/upload-artifact` | `PublishBuildArtifacts@1` | `artifacts:` keyword |
| **Aprobación pre-deploy** | Environment protection rules | Environment approvals | Protected environments |
| **Matrix/parallel** | `strategy: matrix` | `strategy: matrix` | `parallel: matrix` |
| **Runner auto-hosted** | Self-hosted runners | Self-hosted agents | Runners (shell/docker) |

### Ejemplo: CI mínimo en cada plataforma

**GitHub Actions**:
```yaml
name: CI
on: [pull_request]
jobs:
  validate:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: npm ci
      - run: npm run lint
      - run: npm test
```

**Azure Pipelines**:
```yaml
trigger: none
pr:
  branches:
    include: [main]
pool:
  vmImage: ubuntu-latest
steps:
  - script: npm ci
  - script: npm run lint
  - script: npm test
```

**GitLab CI**:
```yaml
stages: [validate]
validate:
  stage: validate
  image: node:20
  script:
    - npm ci
    - npm run lint
    - npm test
  rules:
    - if: $CI_MERGE_REQUEST_IID
```

---

## Instrucciones de Agente

| Concepto agnóstico | GitHub Copilot | Claude (Anthropic) | Cursor | Genérico |
|---|---|---|---|---|
| **Instrucciones globales del repo** | `.github/copilot-instructions.md` | `.claude/CLAUDE.md` | `.cursorrules` | `.ai/instructions.md` |
| **Instrucciones por directorio** | `*.instructions.md` (con frontmatter) | Soportado via `CLAUDE.md` | `.cursorrules` por carpeta | `.ai/instructions.md` por carpeta |
| **Agentes especializados** | `.github/agents/*.md` | `.claude/agents/*.md` | — | `.ai/agents/*.md` |
| **Setup de entorno pre-agente** | `.github/copilot-setup-steps.yml` | — (manual/hooks) | — | `setup.sh` / `setup.ps1` |
| **Memoria persistente del servicio** | Copilot Memory (28 días sin uso) | Claude Memory | — | `tasks/lessons.md` (manual) |

**Recomendación**: mantener una estructura `.ai/` agnóstica como fuente de verdad, y generar los archivos específicos de cada herramienta a partir de ella. Así, si cambiás de herramienta, solo traducís el formato.

---

## Herramientas Externas (MCP y equivalentes)

| Concepto agnóstico | GitHub | Otros |
|---|---|---|
| **Protocolo de extensión** | MCP (Model Context Protocol) | MCP (estándar abierto, no es exclusivo de GitHub) |
| **Config local** | `.vscode/mcp.json` | `.vscode/mcp.json` (VS Code) |
| **Allowlist organizacional** | MCP Allowlists (org-level) | Depende de la org — manual o policy-as-code |
| **Transporte local** | `stdio` | `stdio` |
| **Transporte remoto** | `SSE` / `HTTP` | `SSE` / `HTTP` |

---

## Merge/Pull Requests

| Concepto agnóstico | GitHub | Azure DevOps | GitLab |
|---|---|---|---|
| **Unidad de revisión** | Pull Request | Pull Request | Merge Request |
| **Aprobaciones** | Reviews (approve/request changes) | Votes (approve/reject/wait) | Approvals (approve) |
| **Checks automáticos** | Status checks | Build validation | Pipeline status |
| **Auto-merge** | Auto-merge (con conditions) | Auto-complete | Auto-merge (con approvals) |
| **Templates** | PR templates (`.github/PULL_REQUEST_TEMPLATE.md`) | PR templates (`.azuredevops/pull_request_template.md`) | MR templates (`.gitlab/merge_request_templates/`) |
| **Labels/tags** | Labels | Tags | Labels |

---

## Modelo de Permisos

| Concepto agnóstico | GitHub | Azure DevOps | GitLab |
|---|---|---|---|
| **Mínimo privilegio para agente** | Token con `contents:read` + `pull-requests:write` | PAT con Code(Read) + PR(Contribute) | Token con `read_repository` + `write_merge_request` |
| **Organización** | Organization | Organization / Project | Group |
| **Roles** | Owner / Admin / Write / Read | Project Admin / Contributor / Reader | Owner / Maintainer / Developer / Reporter / Guest |
| **Políticas a nivel org** | Org-level policies + rulesets | Org policies + project settings | Group-level settings |

---

## Hooks y Automation

| Concepto agnóstico | GitHub | Azure DevOps | GitLab |
|---|---|---|---|
| **Pre-commit hook (local)** | git hooks / husky / lefthook | git hooks / husky / lefthook | git hooks / husky / lefthook |
| **Server-side hook** | GitHub Actions + branch rules | Service hooks + policies | Server hooks + push rules |
| **Webhook** | Webhooks (repo/org) | Service hooks | Webhooks (project/group) |
| **Bot/automation** | GitHub Apps | Azure DevOps extensions | GitLab integrations |
| **Scheduled tasks** | `on: schedule: cron` | `schedules:` en pipeline | `rules: - if:` + `schedules:` |

---

## Observabilidad

| Concepto agnóstico | GitHub | Azure DevOps | GitLab |
|---|---|---|---|
| **Historial de cambios** | Git log + blame | Git log + blame | Git log + blame |
| **Timeline de revisión** | PR conversation + timeline | PR activity + comments | MR discussion + activity |
| **Logs de pipeline** | Actions run logs | Pipeline logs (stages/tasks) | Job logs (stages/jobs) |
| **Artefactos de build** | Actions artifacts | Pipeline artifacts | Job artifacts |
| **Audit log** | Org audit log (Enterprise) | Org audit log | Audit events (Premium+) |
| **Métricas de agente** | Copilot metrics API | — (custom) | — (custom) |

---

## Resumen: qué se migra 1:1 y qué se traduce

| Se migra 1:1 (agnóstico) | Se traduce (específico) |
|---|---|
| `.context/` (architecture, conventions, stack) | CI/CD pipeline syntax |
| `tasks/lessons.md` | Branch protection config |
| `tasks/todo.md` | Archivo de instrucciones (path/nombre) |
| `.ai/agents/*.md` (contenido) | Permisos y tokens |
| Hooks (lógica/scripts) | Hooks (mecanismo de trigger) |
| Sprint contracts | MR/PR templates |
| Plan files | Variables/secrets config |
