# Estudo e code review

Para versionar este material no seu fork e baixar uma PR sem misturar `docs/` e `.cursor/` no código oficial, siga [fluxo-fork-e-review.md](fluxo-fork-e-review.md). Os checklists abaixo valem no GitHub da SMAS (comentar e aprovar).

Depois de revisar, registre o que importa para consulta futura em [reviews/](reviews/README.md) (template em [reviews/_template.md](reviews/_template.md)).

## Trilha de estudo (nessa ordem)

1. [README da raiz](../README.md) — levantar o ambiente e rodar `dbt debug`.
2. [Visão geral](visao-geral.md) e [estrutura](estrutura.md) — mapa mental do monorepo.
3. [queries/ARCHITECTURE.md](../queries/ARCHITECTURE.md) — regras das camadas (obrigatório antes de revisar SQL).
4. Um raw curto: [`raw_unidades.sql`](../queries/models/raw/prontuario_carioca_assistencia_social/raw_unidades.sql) e o `_sources.yml` da mesma pasta.
5. Um modelo de domínio: [`int_usuarios_violacoes.sql`](../queries/models/intermediate/social/int_usuarios_violacoes.sql) (só `ref('raw_*')` / outros `int_`).
6. Um core: [`fct_atendimentos.sql`](../queries/models/intermediate/core/fct_atendimentos.sql) + [`dim_familias.sql`](../queries/models/intermediate/core/dim_familias.sql) — grain, surrogate key, YAML de testes.
7. Um mart: [`mart_rma_cras__indicadores.sql`](../queries/models/marts/rma_cras/mart_rma_cras__indicadores.sql) — consome `dim_`/`fct_`/`int_`, tags `monthly`, vars.
8. Um flow: [`pipelines/arcgis/abordagem/flows.py`](../pipelines/arcgis/abordagem/flows.py) — extract → tabela `*_raw` → `dbt run --select …`. Conferir o par em [`prefect.yaml`](../prefect.yaml).
9. [`dbt_project.yml`](../queries/dbt_project.yml) — o que está enabled no legado e quais tags/schemas valem.
10. [Padrão espelho dbt → GCS](padrao-espelho-dbt-prefect.md) — quando o PR materializa um mart e exporta CSV. O código de referência está na branch da feature (PRs #154 / #158), não nesta branch.

Depois disso, pegue um PR real e aplique os checklists abaixo. Não comece pelo `old_architecture/` a menos que o PR mexa nele.

## Checklist — PR dbt (`queries/`)

- [ ] **`source()` só em `raw/`**, salvo exceção já existente e documentada (ex.: bootstrap PIC em `int_status`). Downstream usa `ref()`.
- [ ] **Sem lógica de negócio duplicada** que já exista em `intermediate/core` ou `int_*`. Estenda o core; não recrie dimensão no mart.
- [ ] **Nomenclatura:** `raw_` / `int_` / `dim_` / `fct_` / `mart_`. Pastas novas de Prontuário: `prontuario_carioca_assistencia_social`.
- [ ] **YAML irmão** com descrição e testes de grain (`unique`, `not_null`, `unique_combination_of_columns` quando couber).
- [ ] **Materialização e schema** batem com `dbt_project.yml` (tags `daily` / `weekly` / `monthly`, overrides de `cartao_pic`, etc.).
- [ ] **Incremental:** campo de partição, predicado `is_incremental()`, overwrite idempotente.
- [ ] **CadÚnico:** macros de partição (`filtro_particao_cadunico` / `obter_particao_cadunico`) — não varrer histórico inteiro.
- [ ] **Mart de contrato externo** (espelho): sem join, sem `where`, sem agregação; isso fica no intermediate. Ver [padrao-espelho-dbt-prefect.md](padrao-espelho-dbt-prefect.md).
- [ ] **Vars** (`competencia`, `corte_extrema_pobreza`) em vez de número mágico no SQL.
- [ ] **Ciclos:** respeitar a direção das dependências (ver comentários em `int_usuarios.sql`).
- [ ] **Snapshots:** mudança de grain/colunas em `mart_status` exige olhar `snapshots/snapshot_status_cartao_pic.yml`.
- [ ] **Legado:** PR em `old_architecture/` — o modelo está `+enabled`? Evitar build duplo com o caminho novo.

## Checklist — PR Python (`pipelines/`)

- [ ] Flow novo ou que deve rodar no cluster tem **par staging + prod em [`prefect.yaml`](../prefect.yaml)** (cron, `MODE`, env).
- [ ] Nome da tabela que o Python grava **bate com o `source()` / `_sources.yml`** do dbt.
- [ ] `--select` do dbt no flow cobre os modelos/tags certos.
- [ ] **`MODE`** consistente (`rj-smas` vs `rj-smas-dev`, datasets em `constants.py` / `BaseSettings`).
- [ ] **Sem credenciais no código.** `SIURB_*` e `GCP_CREDENTIALS` só por env.
- [ ] Load ArcGIS: `item_id` / layer documentados; entender o replace atômico (camada vazia pode recriar tabela).
- [ ] Bolsa Família: não quebrar WAP nem `identify_pending_files`; tasks com efeito colateral não devem ganhar cache Prefect.
- [ ] Write-back ArcGIS (feedback / meta AR/CPF): nomes de dataset/tabela delta diferem prod vs staging.
- [ ] **Espelho mart → GCS:** SQL de negócio fora do Python; `dbt build` só no caminho de exportação; estado gravado só depois do export; staging fora do prefixo e da tabela de prod. Checklist em [padrao-espelho-dbt-prefect.md](padrao-espelho-dbt-prefect.md).

## Como comentar em review

1. Diga a camada e o impacto (ex.: “isso filtra no raw — regra de negócio deveria ir ao intermediate”).
2. Aponte o modelo canônico se a lógica já existe (`dim_unidades`, `int_usuarios`, …).
3. Peça teste de grain quando a chave não estiver no YAML.
4. Em pipeline, peça o seletor dbt e o bloco do `prefect.yaml` no mesmo PR.

## Leitura extra por domínio

| Assunto | Onde começar |
|---------|----------------|
| RMA / encaminhamentos | `queries/models/marts/rma_cras/` (inclui notas de negócio `.md`) |
| Cartão PIC operação | `queries/models/marts/cartao_pic/operacao/` + flow PIC |
| Acolhimento | `intermediate/core/fct_acolhimento_*` → `marts/acolhimento/` |
| Atividades de grupo | `intermediate/atividades/` → `marts/atividades/` |
| Centro POP / planilha unificada | `marts/centro_pop/mart_planilha_unificada_centro_pop` — review [PR #139](reviews/2026-10-pr-139-centro-pop-planilha-unificada.md) |
