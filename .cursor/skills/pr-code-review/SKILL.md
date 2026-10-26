---
name: pr-code-review
description: >-
  Reviews pull requests in the RJ SMAS pipelines monorepo (Prefect ELT + dbt
  Medallion on BigQuery) using the project knowledge base. Weights data
  semantics (grain, lineage, incrementality, contracts) over style. Use when
  reviewing a PR, analyzing a git diff, or when the user asks for code review,
  review de PR, or revisão de código neste repositório.
---

# Code Review — pipelines (RJ SMAS)

Pergunta central:

> Esta alteração está tecnicamente correta e compatível com a arquitetura e as regras deste projeto?

A Skill é o **procedimento**. A fonte de regras é a base de conhecimento — não duplicar arquitetura aqui e **não inventar** o que não estiver documentado ou no código.

Durante o review: **não** editar código, criar arquivos, commitar, pushar ou corrigir automaticamente. Só analisar e reportar.

## 1. Recuperar contexto (obrigatório)

Ler nesta ordem, só o necessário para o diff:

1. [docs/README.md](../../../docs/README.md) — índice
2. [queries/ARCHITECTURE.md](../../../queries/ARCHITECTURE.md) — camadas
3. [docs/estrutura.md](../../../docs/estrutura.md) e [docs/visao-geral.md](../../../docs/visao-geral.md) — pastas, fontes, o que o Python não ingere
4. [docs/estudo-e-code-review.md](../../../docs/estudo-e-code-review.md) — checklists
5. Regras: [.cursor/rules/projeto-smas.mdc](../../rules/projeto-smas.mdc), [dbt-medallion.mdc](../../rules/dbt-medallion.mdc), [pipelines-prefect.mdc](../../rules/pipelines-prefect.mdc), [espelho-mart-gcs.mdc](../../rules/espelho-mart-gcs.mdc)
6. [queries/dbt_project.yml](../../../queries/dbt_project.yml) — tags, schemas, `+enabled` do legado
7. Código tocado **e** consumidores (`ref()`, `source()`, `--select` em flows, YAML irmão, snapshots)

Para baixar a PR sem misturar docs pessoais: [docs/fluxo-fork-e-review.md](../../../docs/fluxo-fork-e-review.md).

Se a regra não estiver nesses arquivos nem no código: escrever `Contexto não encontrado na base de conhecimento.` e não inventar.

### Conflitos conhecidos na base

| Tema | Documentos | Como decidir |
|------|------------|--------------|
| Nome raw | ARCHITECTURE: `raw_<sistema>__<tabela>`; árvore real também usa `raw_<entidade>` | Ambos válidos (ver `docs/estrutura.md`) |
| Materialização intermediate | ARCHITECTURE: ephemeral; `dbt_project.yml` materializa `cartao_pic` (e bolsa) como table | Preferir `dbt_project.yml` + o modelo |
| Snapshots | ARCHITECTURE cita Unidades/Evoluções; ativo no repo: `snapshot_status_cartao_pic` | Só exigir snapshot onde já existe ou o doc de auditoria pedir |

## 2. Nomes reais (não usar Bronze/Silver/Gold no relatório)

| Nome do projeto | Alias só para leitura |
|-----------------|------------------------|
| `raw/` | Bronze |
| `intermediate/` + `intermediate/core/` | Silver |
| `marts/` | Gold |

Fluxo real:

```text
Fonte (Airbyte / ArcGIS / GCS / BQ / Sheets)
  ↓
Ingestão Prefect (só ArcGIS e Bolsa Família) ou source() direto
  ↓
raw  →  intermediate / core  →  marts
  ↓
Consumo (BI, operação, write-back ArcGIS)
```

Não existe `staging/` na arquitetura nova. `stg_*` e `old_architecture/` só se o PR mexer nisso. CadÚnico, Prontuário e Sheets **não** têm flow de ingestão (entram por `source()`). O espelho `cd_cidadao_pac` é flow de saída (mart → CSV): padrão em [docs/padrao-espelho-dbt-prefect.md](../../../docs/padrao-espelho-dbt-prefect.md).

## 3. Procedimento

```
Progresso:
- [ ] Diff + arquivos além do diff (refs, sources, prefect.yaml, YAML, snapshots)
- [ ] Camada e eixo (queries vs pipelines)
- [ ] Checklists em docs/estudo-e-code-review.md
- [ ] Semântica: grão, joins, lineage, incremental/idempotência
- [ ] Testes e CI que ESTE repo realmente tem
- [ ] Relatório no formato da seção 6
```

1. Obter o diff (`git diff base...HEAD` ou a PR). Identificar branch, base, área (`queries/`, `pipelines/`, ambos).
2. Classificar camadas e pipelines afetados. Uma mudança em `int_*` / `dim_*` / `fct_*` exige olhar marts e flows que fazem `ref` ou `--select` nela.
3. Priorizar **semântica dos dados** sobre estilo. SQL/Python “bonito” ainda pode estar errado se mudou o grão, multiplicou join, quebrou incremental ou alterou métrica de mart.
4. Aplicar o checklist **dbt** e/ou **Python** de `docs/estudo-e-code-review.md` — não reescrever essas regras aqui.
5. Investigar só o que o diff tornar aplicável (seção 4). Não marcar item genérico sem evidência.
6. Emitir o relatório. Sem nota numérica.

## 4. O que investigar (quando o PR tocar nisso)

### Arquitetura e contrato

- `source()` fora de `raw/` sem exceção já documentada (ex.: PIC `int_status`).
- Filtro/regra de negócio no raw; dimensão recriada no mart.
- Nome/pasta fora da convenção (`raw_` / `int_` / `dim_` / `fct_` / `mart_`; Prontuário = `prontuario_carioca_assistencia_social`).
- YAML 1:1 e testes de grain ausentes ou enfraquecidos.
- Path em `old_architecture/`: está `+enabled`? Risco de build duplo?

### Lineage e ELT

Não limitar aos arquivos do diff. Seguir `ref()` / `source()` / `--select` / tags (`daily`, `weekly`, `hourly`, `monthly`, `controle_cas`, `meta_ar_cpf`).

Python: tabela gravada = `_sources.yml`; seletor dbt alinhado; flow de cluster tem par em `prefect.yaml`.

### Granularidade (alta prioridade em modelos analíticos)

Perguntar: qual é o grão documentado (comentário, YAML, `unique` / `unique_combination_of_columns`)?

Joins, `GROUP BY`, fatos/dims: risco de duplicar linhas, inflar métrica ou perder registro.

### Incrementalidade e idempotência (estratégia real, não genérica)

No código deste repo aparece:

- dbt `incremental` + `insert_overwrite` + `is_incremental()` em modelos PIC/bolsa/acolhimento (e legado RMA/bolsa).
- CadÚnico: macros de partição (`filtro_particao_cadunico` / `obter_particao_cadunico`).
- ArcGIS: replace atômico da tabela `*_raw`.
- Bolsa Família: WAP (`folha_wap` → promote); não cachear task com side effect.
- Snapshot ativo: `snapshot_status_cartao_pic` (mudança de grain/colunas em `mart_status`).

**Não assumir** CDC, watermark genérico, merge/upsert universal. Perguntar: se rodar duas vezes com a mesma entrada, o resultado continua correto?

Vars de negócio (`competencia`, `corte_extrema_pobreza`) — não hardcode.

### Data quality

Completude, unicidade, NULL, chaves, consistência, temporalidade — **se** o modelo/YAML/teste existente for afetado. Destacar se o PR **remove ou enfraquece** teste de schema ou teste singular.

### Testes que existem aqui

| Existe | Onde |
|--------|------|
| Schema tests | `unique`, `not_null`, `accepted_values`, `dbt_utils.unique_combination_of_columns` no YAML 1:1 |
| Testes singulares | `queries/tests/validar_capacidade_unidades.sql`; pasta `tests/dashboard_acolherio/` (legado; muitos `stg_*` desligados) |
| Recce slim | CI em PR que muda `queries/models/**/*.sql` |
| Lint | Ruff/SQLFluff **só linhas tocadas** |
| pytest | decisão e export do espelho em `tests/test_cadunico_espelho.py` (branch da feature / PRs #154 e #158); no restante do repo, quase ausente |

**Não existem** testes `relationships:` no YAML. Não exigir suíte unitária/integração genérica. Pedir teste só se proteger comportamento que o PR altera (grain, métrica, partição).

### Performance

Só com impacto plausível: scan sem macro de partição CadÚnico; incremental sem predicado; join que explode cardinalidade; ephemeral pesado promovido sem necessidade. Sem micro-otimização estética.

### Segurança (o que o projeto documenta)

Credenciais no código; `SIURB_*` / `GCP_CREDENTIALS` fora de env; commit de `.env`. Não inventar ameaças (ex. SQL injection) sem evidência no diff.

## 5. Findings e severidade

Não registrar preferência pessoal, estilo, “eu faria diferente”, refatoração fora do PR ou arquitetura alternativa sem evidência.

Todo finding:

```text
Problema
Impacto
Evidência
Localização
Correção sugerida
```

Sem evidência suficiente: `Não foi possível confirmar este comportamento apenas pela análise estática.`

Severidade pelo impacto real:

```text
BLOCKER
HIGH
MEDIUM
LOW
INFO
```

Tratar como alto impacto **quando comprovados**: corrupção, perda ou duplicação de dados; quebra de pipeline ou contrato; métrica de mart errada; raw filtrando negócio; incremental/WAP/partição quebrados; secret no código.

## 6. Formato do resultado

# Code Review

## Contexto

* Branch:
* Base:
* Área afetada:
* Pipelines afetados:
* Camadas afetadas:

## Resumo

```text
BLOCKER: X
HIGH: X
MEDIUM: X
LOW: X
INFO: X
```

## Findings

Para cada problema:

```text
[SEVERIDADE] arquivo:linha

Problema:
...

Impacto:
...

Evidência:
...

Correção sugerida:
...
```

## Impacto no pipeline

Quando aplicável, usar os **nomes reais** (`raw` / `intermediate` / `core` / `marts`):

```text
Fonte
 ↓
Ingestão (Prefect ou source)
 ↓
raw
 ↓
intermediate / core  ← alteração (exemplo)
 ↓
marts
 ↓
Consumo
```

Explicar onde o comportamento pode propagar.

## Data Quality

Registrar possíveis impactos em:

* completude;
* unicidade;
* consistência;
* integridade;
* granularidade;
* temporalidade.

## Testes recomendados

Listar os testes específicos necessários para validar o PR (YAML de grain, singular existente, Recce, run local com `--select`). Não inventar tipo de teste que o repo não usa.

## Pontos positivos

Somente registrar aspectos tecnicamente relevantes.

## Conclusão

Utilizar somente:

```text
SEM BLOQUEIOS IDENTIFICADOS

EXISTEM PONTOS QUE DEVEM SER CORRIGIDOS ANTES DO MERGE

EXISTEM PONTOS QUE MERECEM AVALIAÇÃO ANTES DO MERGE
```

Não atribuir nota ao PR.
