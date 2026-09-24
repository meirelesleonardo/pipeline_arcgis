# Visão geral

## Para que serve

Este repositório é a plataforma de dados da **Secretaria Municipal de Assistência Social (SMAS)** do Rio de Janeiro. Consolida fontes operacionais e produz tabelas para relatórios, dashboards e operação de programas:

- Prontuário Carioca (AcolheRio): unidades CRAS/CREAS, usuários, famílias, atendimentos, evoluções, atividades.
- CadÚnico: cadastro único federal (renda, moradia, deficiência, situação de rua, etc.).
- Cartão PIC / Primeira Infância Carioca: elegibilidade, status e operação (controle CAS, meta AR/CPF).
- Bolsa Família: folha de pagamentos.
- RMA CRAS, acolhimento institucional, Centro POP, risco social, ArcGIS (abordagem, equipamentos, polígonos).

Organização no GitHub: `prefeitura-rio-smas/pipelines`. Equipe: Dados — RJ SMAS.

## Dois eixos

| Eixo | Pasta | Papel |
|------|-------|--------|
| Ingestão e orquestração | [`pipelines/`](../pipelines/) | Flows **Prefect 3**: extraem dados (ArcGIS/SIURB, GCS) para o BigQuery e disparam dbt. |
| Transformação | [`queries/`](../queries/) | Projeto **dbt** (`name: queries`) em arquitetura **Medallion** no BigQuery. |

```
Fontes (Airbyte, ArcGIS, GCS, BQ, Sheets)
        │
        ▼
   BigQuery (tabelas brutas)
        │
        ▼
   dbt: raw → intermediate/core → marts
        │
        ▼
   Relatórios, dashboards, operação
```

Prefect também agenda builds dbt por tag (`daily`, `weekly`, `hourly`, `monthly`) em [`pipelines/datalake/transform/dbt/flows.py`](../pipelines/datalake/transform/dbt/flows.py).

## Stack

| Camada | Tecnologia |
|--------|------------|
| Linguagem | Python 3.13 (`uv` + `uv.lock`) |
| Orquestração | Prefect 3 (`prefect.yaml`, work pool Docker) |
| Transformação | dbt + `dbt-bigquery` ≥ 1.8, pacote `dbt_utils` |
| Warehouse | Google BigQuery (`rj-smas` / `rj-smas-dev`, location US) |
| Qualidade | Ruff, SQLFluff, Recce (slim CI em PRs de SQL) |
| Runtime | Imagem Docker → GHCR; CD em `main` (prod) e `staging/**` |

Não há Makefile, docker-compose, Airflow ou Dagster neste repo.

## Ambientes

| Contexto | Projeto GCP | Como autentica | Target dbt |
|----------|-------------|----------------|------------|
| Desenvolvimento local | `rj-smas-dev` | OAuth (`gcloud auth application-default login`) | `dev` (padrão em [`profiles.yml`](../queries/profiles.yml)) |
| Pipeline staging | `rj-smas-dev` | Credencial de serviço no Prefect (`GCP_CREDENTIALS`) | `staging` (`MODE=staging`) |
| Produção | `rj-smas` | Service account | `prod` (`MODE=prod`) |
| CI de PR | `rj-smas-dev` | OAuth / secrets do GitHub | `ci` — dataset `ci_<PR>__<SHA>` |

`MODE` vive em [`pipelines/constants.py`](../pipelines/constants.py) (`staging` ou `prod`, default `staging`). Flows Prefect usam `--target` igual a `MODE`. No notebook/terminal local o alvo padrão do dbt é `dev`, não `staging`.

Em produção, schemas separam camadas (`raw`, `core`, `marts`, mais overrides de produto). Fora de prod, boa parte cai em `gerenciamento__dbt` ou `relatorio` — ver [`dbt_project.yml`](../queries/dbt_project.yml).

## O que o Python não ingere

CadÚnico, Prontuário Carioca e Google Sheets **não** têm flow Prefect neste repo. Entram no dbt só via `source()` em [`queries/models/raw/`](../queries/models/raw/):

- Prontuário: Airbyte → `rj-smas.brutos_acolherio_staging`
- CadÚnico: dataset `rj-smas.protecao_social_cadunico`
- Sheets: fontes declaradas em `raw/sheets/_sources.yml`

O Python cobre ArcGIS/SIURB, Bolsa Família (ZIP no GCS → BigQuery WAP) e o flow genérico de dbt.
