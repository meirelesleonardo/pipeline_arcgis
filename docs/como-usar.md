# Como usar

O passo a passo de instalação (uv, Google Cloud SDK, ADC) está no [README da raiz](../README.md). Este guia assume ambiente já validado com `dbt debug`.

## Setup (lembrete)

Na raiz do repo:

```powershell
uv sync
gcloud auth application-default login --project rj-smas-dev
dbt debug --project-dir queries --profiles-dir queries
```

Se o terminal não achar `dbt`, use `uv run dbt ...`. Codespaces: `uv sync` e `dbt deps` já rodam; falta só o login gcloud.

Sempre passe `--project-dir queries --profiles-dir queries` (o projeto dbt não está na raiz).

## Dia a dia — dbt

```powershell
# Um produto / pasta
dbt run --select pic --project-dir queries --profiles-dir queries

# Um modelo e seus pais
dbt run --select +mart_rma_cras__indicadores --project-dir queries --profiles-dir queries

# Só o que mudou em relação a main (quando houver manifest)
dbt build --select state:modified+ --project-dir queries --profiles-dir queries

# Parse (mesmo check do CI)
dbt parse --target ci --project-dir queries --profiles-dir queries
```

Target local padrão: `dev` → projeto `rj-smas-dev`. Não use `--target prod` no notebook.

Vars úteis (definidas em [`dbt_project.yml`](../queries/dbt_project.yml)): `competencia` (RMA, `AAAA-MM`; vazio = mês corrente), `corte_extrema_pobreza`. Evite hardcode desses valores no SQL.

## Dia a dia — Prefect

Flows podem rodar in-process (sem servidor Prefect):

```powershell
cd c:\laragon\www\pipelines
$env:MODE = "staging"
uv run python pipelines/datalake/transform/dbt/flows.py
```

Flows ArcGIS precisam das variáveis `SIURB_URL`, `SIURB_USER` e `SIURB_PWD` (não commitar; `.env` está no `.gitignore`).

| Flow | Arquivo |
|------|---------|
| Abordagem | `pipelines/arcgis/abordagem/flows.py` |
| Equipamento | `pipelines/arcgis/equipamento/flows.py` |
| Polígonos CRAS/CAS | `pipelines/arcgis/cras_cas_poligonos/flows.py` |
| Primeira Infância | `pipelines/arcgis/primeira_infancia_carioca/flows.py` |
| Meta AR/CPF | `pipelines/arcgis/primeira_infancia_carioca/meta_ar_cpf/flows.py` |
| Bolsa Família | `pipelines/bolsa_familia/flows.py` |
| dbt por tag | `pipelines/datalake/transform/dbt/flows.py` |

Deployments oficiais (cron, imagem, env) estão em [`prefect.yaml`](../prefect.yaml): pares `*-staging` e `*-prod`. Flow novo que deve ir para o cluster precisa de entrada aí.

## CI e branches

| Workflow | Quando | O que faz |
|----------|--------|-----------|
| `CI - Lint` | PR em `pipelines/`, `queries/`, etc. | Ruff + SQLFluff **só nas linhas tocadas** |
| `CI - DBT` | PR em `queries/` | `dbt deps` + `dbt parse --target ci` |
| `CI - Recce Data Review` | PR que muda `queries/models/**/*.sql` | Slim build no dataset `ci_<PR>__<SHA>` |
| `CI - Tests` | PR em `pipelines/` | pytest (hoje há pouco ou nenhum teste Python) |
| `CI - Security` | push | Gitleaks, `uv audit`, Hadolint |
| CD | `main` → prod; `staging/**` → staging | Imagem GHCR + `prefect deploy` |

PRs que fecham disparam drop dos datasets `ci_<PR>__*`.

Implicação para review: lint legado fora do diff não quebra o CI; o que você alterou, sim. Mudança de modelo SQL pode gerar dataset Recce no comentário da PR para validar dados.

Rotina de fork, overlay das docs pessoais e checkout de PR: [fluxo-fork-e-review.md](fluxo-fork-e-review.md).
