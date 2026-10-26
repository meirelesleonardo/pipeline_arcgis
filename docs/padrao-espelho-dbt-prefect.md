# Padrão: mart dbt → CSV no GCS (espelho CadÚnico)

Modelo a copiar quando um consumidor externo (app, planilha, outro sistema) precisa de um CSV publicado a partir de um mart, e o job só deve rodar quando há partição ou competência nova.

O código de referência **não está nesta branch**. Ele vive na branch da feature / PRs [#154](https://github.com/prefeitura-rio-smas/pipelines/pull/154) e [#158](https://github.com/prefeitura-rio-smas/pipelines/pull/158), no pacote `pipelines/cadunico_bq_to_prontuario_eletronico/` e nos modelos citados abaixo. Esta nota descreve o desenho final para estudo e para a IA seguir nas próximas tarefas do mesmo tipo.

## O que a refatoração separou

No PR #154 original (`pipelines/cadunico_espelho/`, commit `db4e4d8`) um script SQL fazia joins, filtro, materialização e export no mesmo job BigQuery. O Prefect só comparava `MAX(data_particao)`.

O desenho final separa três papéis:

| Papel | Onde | O que faz |
|-------|------|-----------|
| Transformação | dbt (`queries/`) | Joins, filtro, contrato de colunas |
| Decisão e orquestração | Prefect (`pipelines/<produto>/`) | Há competência nova? Se sim, `dbt build`, export, estado |
| Export genérico | `pipelines/utils/export.py` | `EXPORT DATA` de uma tabela já materializada para `gs://` |

```text
macro obter_particao_cadunico()
        │
        ▼
competências já exportadas  (JSON em gs://<bucket>/<prefixo>/_state/)
        │
        ▼
should_export?  (AAAA-MM nova, ou nunca exportou)
        │ não → status skipped (sem dbt, sem CSV, sem estado)
        ▼ sim
dbt build --select +<mart> --target <MODE>
        │
        ▼
COUNT(*) > 0?  senão aborta (não publica CSV, não grava estado)
        │
        ▼
EXPORT DATA  (pipelines/utils/export.py)
        │
        ▼
grava estado JSON  (só depois do export bem-sucedido)
```

Falha no `dbt build` ou no export **não** grava estado. O próximo run reexporta (idempotente, `overwrite=true`).

## Camadas dbt deste espelho

Os raws de CadÚnico já aplicam `filtro_particao_cadunico()`. A macro `obter_particao_cadunico()` (em `queries/macros/dominios/cadunico/filtro_particao_cadunico.sql`) devolve `max(data_particao)` de `rj-smas.protecao_social_cadunico.prefeitura`. O Python não reescreve esse SQL: chama a macro com `dbt show --inline`.

| Camada | Arquivo | Responsabilidade |
|--------|---------|------------------|
| Intermediate | `queries/models/intermediate/cadunico/int_cadunico_membros_familia.sql` | Os 5 joins, normalização de texto e o filtro (`estado_cadastral <> 'Excluido'` e `estado_cadastro = 'Cadastrado'`). Grão: 1 linha por `id_membro_familia`. |
| Mart de contrato | `queries/models/marts/cadunico/prontuario_eletronico/cd_cidadao_pac.sql` | Só tradução de código, truncamento e constantes. **31 colunas, nesta ordem.** Sem join, sem `where`, sem agregação. |

O schema de destino não fica hardcoded no SQL. Em `queries/dbt_project.yml`, a pasta `prontuario_eletronico` usa `prontuario_eletronico` em prod e `relatorio` nos outros targets.

Contrato externo: renomear, acrescentar ou reordenar coluna quebra o app do prontuário. O `.yml` irmão documenta o grão e as colunas.

## Pacote Prefect

`pipelines/cadunico_bq_to_prontuario_eletronico/`:

| Arquivo | Função |
|---------|--------|
| `constants.py` | `MODE` → projeto, dataset, bucket, prefixo GCS. Nome do mart (`cd_cidadao_pac`). |
| `utils.py` | Funções puras: competência `AAAA-MM`, `should_export`, URI do CSV, caminho do estado. A leitura da macro fica aqui, sem SQL de negócio. |
| `tasks.py` | `dbt build`, contagem, `export_table_to_gcs`, gravação do JSON. |
| `flows.py` | Só a ordem do diagrama. Devolve um `dict` (`status`, `rows`, `uri`, `job_id`, `state_uri`). |

Regra de produto em `should_export`: exporta se não há estado, ou se a competência `AAAA-MM` da partição disponível é **maior** que a última exportada. Recarga corretiva no mesmo mês não reexporta.

Estado (fora da pasta que o app lê):

```text
gs://<bucket>/<prefixo>/_state/cd_cidadao_pac_YYYY-MM-DD.json
```

CSV (contrato do app, shards gzip):

```text
gs://<bucket>/<prefixo>/cd_cidadao_pac_YYYY-MM-DD/cd_cidadao_pac_*.csv.gz
```

## Ambientes

O mesmo flow. O que muda é `MODE` no `prefect.yaml` (`cadunico-espelho-staging` sem cron; `cadunico-espelho-prod` com `0 6 * * *` em `America/Sao_Paulo`).

| | Produção | Staging |
|--|----------|---------|
| Tabela | `rj-smas.prontuario_eletronico.cd_cidadao_pac` | `rj-smas-dev.relatorio.cd_cidadao_pac` |
| Prefixo GCS | `raw/cadunico` | `raw/cadunico-staging` |
| Bucket | `rj-smas-dev` | `rj-smas-dev` |

O bucket de produção continua `rj-smas-dev` **de propósito**: o app do prontuário lê esse caminho e está fora do alcance deste repositório. É exceção deste produto, documentada em `constants.py`. Não copiar esse bucket para o próximo job. Staging nunca escreve no prefixo nem na tabela de produção.

O CSV carrega PII (CPF, NIS, nome, endereço, telefone).

## O que não fazer

- SQL de negócio (join, filtro, `CASE` de domínio) dentro de `pipelines/`.
- Join, `where` ou agregação no mart que é só contrato de colunas.
- Gravar o estado antes do `dbt build` e do export terminarem.
- Staging apontando para o prefixo GCS ou a tabela de produção.
- Tratar o bucket `rj-smas-dev` em produção como padrão dos próximos exports.
- Comparar a data cheia quando a regra do produto for competência mensal.

## Checklist do próximo espelho do mesmo tipo

1. Raw (ou fonte já existente) com a macro de partição. Sem filtro de negócio no raw.
2. Intermediate com joins e filtro. Mart só com o contrato do consumidor, mais o `.yml` (grão e colunas).
3. Schema por target em `dbt_project.yml`, não no SQL.
4. Pacote Prefect fino: `constants`, `utils` (decisão pura, com teste), `tasks`, `flow` na ordem do diagrama.
5. Export via `pipelines/utils/export.py`. Contar linhas antes; tabela vazia aborta sem estado.
6. Estado no GCS, fora da pasta consumida, gravado só depois do export.
7. Par `*-staging` (sem schedule) e `*-prod` em `prefect.yaml`. Staging isolado de prod.
8. Testes da decisão e do SQL de export sem chamar BigQuery (`tests/test_cadunico_espelho.py` é a referência).

Ingestão do CadÚnico continua **sem** flow Python: a fonte entra por `source()` em `queries/models/raw/cadunico/`. Este padrão é flow de **saída** (mart já modelado → CSV).
