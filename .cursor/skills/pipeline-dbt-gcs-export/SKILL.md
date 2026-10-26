---
name: pipeline-dbt-gcs-export
description: >-
  Implements a Prefect flow that materializes a dbt mart and exports CSV to
  GCS only when a new partition or competência exists. Use when creating or
  changing an espelho, EXPORT DATA, watermark de competência, export de mart
  para o GCS, or a flow that should follow pipelines/cadunico_bq_to_prontuario_eletronico.
---

# Espelho: mart dbt → CSV no GCS

Procedimento. A explicação e o checklist estão em [docs/padrao-espelho-dbt-prefect.md](../../../docs/padrao-espelho-dbt-prefect.md). Ler esse arquivo antes de editar. Não repetir o ensaio aqui e não inventar bucket, dataset ou regra de competência que a nota não registre.

O código de referência não está na branch `local/docs`. Está na branch da feature (PRs #154 / #158): `pipelines/cadunico_bq_to_prontuario_eletronico/`.

## Ordem obrigatória

1. Partição disponível via macro dbt (`dbt show --inline`). Nenhum SQL de join ou filtro no Python.
2. Competências já exportadas, lidas do estado no GCS.
3. Decisão pura em `utils` (`should_export`), com teste sem BigQuery.
4. Se não exportar: `status: skipped`. Sem `dbt build`, sem CSV, sem estado.
5. `dbt build --select +<mart> --target <MODE>` (`project_dir` e `profiles_dir` = `queries`).
6. `COUNT(*)`; tabela vazia aborta sem publicar e sem gravar estado.
7. `EXPORT DATA` por `pipelines/utils/export.py`.
8. Estado JSON só depois do export bem-sucedido, fora da pasta que o consumidor lê.

## Onde cada coisa mora

- Joins e filtro de negócio: intermediate.
- Mart de contrato externo: só tradução, truncamento e constantes. Sem join, sem `where`, sem agregação. YAML 1:1 com o grão.
- Schema por target: `queries/dbt_project.yml`, não no SQL.
- Projeto, dataset, bucket e prefixo: `constants.py` via `MODE`. Staging nunca usa tabela nem prefixo de prod.
- Deploy: par `*-staging` (sem schedule) e `*-prod` em `prefect.yaml`.

## Proibido

- SQL de negócio dentro de `pipelines/`.
- Gravar estado antes do dbt e do export terminarem.
- Copiar o bucket `rj-smas-dev` em produção. Isso é exceção do app do prontuário, registrada na nota.

## Testes

Cobrir a decisão e a montagem do `EXPORT DATA` sem chamar BigQuery. Referência: `tests/test_cadunico_espelho.py` na branch da feature.
