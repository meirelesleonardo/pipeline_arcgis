# Estrutura do repositório

## Pastas de topo

```
pipelines/                 # Flows Prefect (Python)
queries/                   # Projeto dbt
.github/workflows/         # CI/CD
sqlfluff_libs/             # Helpers do SQLFluff (templater Jinja/dbt)
prefect.yaml               # Deployments Prefect (staging + prod)
pyproject.toml / uv.lock   # Dependências Python
Dockerfile                 # Imagem do worker Prefect
.devcontainer.json         # Codespaces
```

| Pasta / arquivo | Função |
|-----------------|--------|
| [`pipelines/arcgis/`](../pipelines/arcgis/) | Extração SIURB → `arcgis_raw.*_raw`; alguns flows escrevem de volta no ArcGIS (PIC, meta AR/CPF). |
| [`pipelines/bolsa_familia/`](../pipelines/bolsa_familia/) | ZIP no GCS → staging → WAP → dbt. |
| [`pipelines/datalake/transform/dbt/`](../pipelines/datalake/transform/dbt/) | `dbt build` por seletor/tag. |
| [`pipelines/utils/`](../pipelines/utils/) | `MODE` → projeto/bucket; auth GCP. |
| [`queries/models/`](../queries/models/) | Modelos Medallion + `old_architecture/`. |
| [`queries/macros/`](../queries/macros/) | Schema CI, CadÚnico, AcolheRio, nomes, etc. |
| [`queries/snapshots/`](../queries/snapshots/) | Histórico (ex.: status Cartão PIC). |
| [`queries/tests/`](../queries/tests/) | Testes singulares e fixtures. |

## Camadas dbt (Medallion)

Detalhe normativo: [queries/ARCHITECTURE.md](../queries/ARCHITECTURE.md). Resumo:

| Camada | Caminho | Materialização típica | Responsabilidade |
|--------|---------|----------------------|------------------|
| Raw (Bronze) | `queries/models/raw/<sistema>/` | `view` (CadÚnico: `table`, tag `weekly`) | Único ponto com `source()`. Rename, cast, trim. Sem filtro de negócio. |
| Intermediate | `queries/models/intermediate/{social,atividades,bolsa_familia,cartao_pic}/` | `ephemeral` (PIC/bolsa podem ser `table`) | Lógica modular, agregações, joins de domínio. |
| Core | `queries/models/intermediate/core/` | `table` | Dimensões e fatos conformados (`dim_*`, `fct_*`), surrogate keys. |
| Marts (Gold) | `queries/models/marts/<produto>/` | `table` | Agregações e recortes para BI/operação. |

Não existe pasta `staging/` na arquitetura nova: raw absorve rename/cast.

### Fontes em `raw/`

- `prontuario_carioca_assistencia_social/` — Prontuário Carioca (Airbyte)
- `cadunico/`
- `arcgis/`
- `bolsa_familia/`
- `sheets/`

### Domínios em `intermediate/` e produtos em `marts/`

- Intermediate: `core`, `social`, `atividades`, `cartao_pic`, `bolsa_familia`
- Marts: `acolhimento`, `atividades`, `atendimentos_acolherio`, `bolsa_familia`, `cartao_pic`, `centro_pop`, `core`, `estrutura_assistencial`, `pic`, `poc_paif`, `risco_social`, `rma_cras`

## Nomenclatura

| Prefixo | Camada | Exemplo |
|---------|--------|---------|
| `raw_<entidade>` ou `raw_<sistema>__<entidade>` | Raw | `raw_unidades.sql`, `raw_bolsa_familia__folha.sql` |
| `int_<descricao>` | Intermediate | `int_usuarios_violacoes.sql` |
| `dim_<entidade>`, `fct_<evento>` | Core | `dim_familias.sql`, `fct_atendimentos.sql` |
| `mart_<produto>__<descricao>` | Marts | `mart_rma_cras__indicadores.sql` |

Na arquitetura nova, usuários são `usuarios` (não `pacientes`). O sistema é `prontuario_carioca_assistencia_social` (não pastas novas chamadas “acolherio”). Prefixo `stg_*` só existe em `old_architecture/`.

YAML 1:1: cada `modelo.sql` novo deve ter `modelo.yml` ao lado (descrição + testes de chave).

## `old_architecture/`

Layout antigo: um “projeto” por dashboard (`dashboard_acolherio`, `rma/cras`, …), cada um com raw/staging/intermediate/mart próprios — lógica de dimensões duplicada.

Em [`dbt_project.yml`](../queries/dbt_project.yml):

- `old_architecture` default: **`+enabled: false`**
- Ainda **ligados**: `dashboard_arcgis`, `equipamentos`, `old_architecture/bolsa_familia`
- `dashboard_acolherio` e a maior parte do RMA legado estão desligados; RMA novo está em `marts/rma_cras`

Ao revisar PR: conferir se a mudança é no caminho novo ou no legado ainda habilitado. Não reativar pasta desligada sem alinhamento.

## Prefect ↔ dbt

1. Python grava tabelas com nomes estáveis (`arcgis_raw.equipamento_raw`, `bolsa_familia_staging.folha`, …).
2. dbt declara essas tabelas em `_sources.yml` e constrói as camadas.
3. O flow chama `dbt run` / `dbt build` com `--select` alinhado ao produto (`+abordagem`, `+tag:daily`, `+folha+`, …).

Exemplo de raw enxuto (só `source()`, rename e tipagem): [`queries/models/raw/prontuario_carioca_assistencia_social/raw_unidades.sql`](../queries/models/raw/prontuario_carioca_assistencia_social/raw_unidades.sql).
