# PR #139 — Centro POP: planilha unificada no grão de atendimento

- **Link:** https://github.com/prefeitura-rio-smas/pipelines/pull/139
- **Branch:** `staging/centro-pop-planilha-unificada` → `main`
- **Data da revisão:** 2026-10-02
- **Commit HEAD revisado:** `cbfa4b7` (5 commits; título final: *fix(centro-pop): align spreadsheet fields at attendance grain*)

## Contexto

- **Área:** `queries/` (dbt apenas)
- **Arquivos:** `queries/models/marts/centro_pop/mart_planilha_unificada_centro_pop.sql`, `.yml`
- **Tag:** `daily` (`dbt_project.yml` → `marts.centro_pop`)
- **Dependências relevantes:** `fct_atendimentos` (`id_atendimento_modulo`), `fct_evolucoes`, `dim_usuarios`, macros `ultimo_registro_ate_evento`, `extrair_campos_html_evolucao`, `calc_idade`
- **Consumo:** Looker / planilha Google ([URL no YAML do mart](https://docs.google.com/spreadsheets/d/1me-dxFcE-SZ_WgZ75_sq1t92Se1zwxk9YPgZ_YYMKu8/edit)); sem `ref()` downstream no repo

## Resumo da mudança

A mart deixa de ser **1 linha por usuário × unidade × mês** (com arrays e totais mensais embutidos) e passa a **1 linha por atendimento** (`id_atendimento_modulo`), com profissionais compartilhados agregados no evento. Contagens por pessoa, mês e tipo vão para o Looker (`COUNT(DISTINCT id_atendimento_modulo)`, etc.). O YAML mapeia as 50 colunas da planilha (`meta.colunas_planilha`) e documenta campos mensais repetidos em cada linha de atendimento.

Prévia Recce (staging): ~794 → ~1536 linhas, coerente com o novo grão.

## Conclusão

`EXISTEM PONTOS QUE MERECEM AVALIAÇÃO ANTES DO MERGE`

```text
BLOCKER: 0
HIGH: 2
MEDIUM: 4
LOW: 2
INFO: 1
```

## Findings

### [HIGH] Teste `accepted_values` vs `flag_situacao_rua`

- **Problema:** SQL usa `coalesce(u.flag_situacao_rua, 'Não Informado')`; YAML aceita só `Sim` e `Não`.
- **Impacto:** `dbt test` pode falhar se houver cadastro sem situação de rua.
- **Evidência:** `mart_planilha_unificada_centro_pop.sql` (select final) + coluna no `.yml`.
- **Follow-up:** Incluir `'Não Informado'` no teste ou mapear NULL conforme planilha.

### [HIGH] Flags PAI — regra mais estrita

- **Problema:** PAI só conta com formulário real *Centro POP - Plano de Atendimento Individual (PAI)*; evoluções genéricas da abrangência 29 não entram.
- **Impacto:** Queda forte em `flag_possui_plano_individual` / `flag_inserido_acompanhamento` (Recce citou ordem de magnitudes ~799 → ~27 na prévia — validar após materialização).
- **Follow-up:** Validação com negócio/Centro POP antes de prod.

### [MEDIUM] `flag_atendido_pontualmente` por evento

- **Problema:** Deixou de ser agregado mensal; cada linha: `Sim` se `nome_atendimento` não é tipo PAI.
- **Impacto:** Mesmo mês pode misturar `Sim` e `Não` entre atendimentos; colunas `*_fim_mes` removidas.
- **Follow-up:** Confirmar coluna 7 da planilha com área de negócio; ajustar explores Looker.

### [MEDIUM] `motivo_principal_permanencia_rua`

- **Problema:** Sem fallback no formulário Acolhimento; só questionário template 3 / questão 18 (sem respostas na auditoria 2026-10-01).
- **Impacto:** Predomínio de `Não Informado`.
- **Follow-up:** Aceite documentado ou reintroduzir fallback com regra explícita.

### [MEDIUM] Label exato `Data do atendimento`

- **Problema:** De `label like 'Data do atendimento'` para `label = 'Data do atendimento'`.
- **Impacto:** Labels HTML com variação podem forçar só fallback `data_evolucao` na inclusão PAI.
- **Follow-up:** Amostra de HTML em produção.

### [MEDIUM] Migração Looker

- **Problema:** Removidos `qtd_atendimentos_*_mes`, arrays `atendimentos` / `atendimentos_por_tipo`.
- **Impacto:** Somas diretas nas linhas inflam métricas; oficinas: `MAX(qtd_oficinas_participadas_mes)` por `id_usuario_unidade_mes`, não `SUM`.
- **Follow-up:** Checklist na descrição da PR — aplicar no deploy.

### [LOW] `accepted_values` sem aspas no YAML

- Risco de parsing YAML; preferir `'Sim'`, `'Não'`, `'Não Informado'` como no `main`.

### [LOW] Coluna `numero`

- `row_number()` de apresentação; não é chave persistente.

### [INFO] Recce `ano_cursando` × `flag_estuda`

- Com a regra atual, não estudantes recebem `'Não Informado'` (não NULL) em `ano_cursando`; alertas Recce só por `IS NOT NULL` podem ser falso positivo. Validar com `ano_cursando NOT IN ('Não Informado')`.

## Contrato de dados (pós-PR)

| Tema | Valor |
|------|--------|
| **Grão** | `id_atendimento_modulo` (`unique`, `not_null`) |
| **Agrupamento mensal** | `id_usuario_unidade_mes` (repetido; não único na mart) |
| **Contar atendimentos** | `COUNT(DISTINCT id_atendimento_modulo)` |
| **Contar pessoas** | `COUNT(DISTINCT id_usuario)` |
| **Idade** | `calc_idade` no **fim do mês** (`data_referencia`) |
| **Formulários** | Snapshot até fechamento do mês; demandas/encaminhamentos agregados no mês |
| **Cadastro / CadÚnico** | Snapshot atual (sem histórico na data do atendimento) |
| **Oficinas** | Sem presenças Centro POP na auditoria 2026-10-01 → `Não Informado` / `NULL` |

## Lacunas de coleta (documentadas na PR)

Empregabilidade, habilidades, interesse em curso, resultado/acesso/descrição, motivo principal (questionário sem respostas), observações (bloco não exportado), oficinas (sem registro na fonte auditada).

## Testes recomendados (pós-materialização)

1. `dbt build --select mart_planilha_unificada_centro_pop` (staging/prod conforme alias `centro_pop_mart_alias`).
2. Schema tests do YAML (atenção `flag_situacao_rua`).
3. Sanidade: contagens distinct de chaves; PAI por `mes_referencia` vs versão anterior.
4. Recce: refresh manifest base; `query_diff` em `flag_possui_plano_individual`.

## Pontos positivos (referência)

- `id_atendimento_modulo` alinhado a `fct_atendimentos` e RMA.
- YAML com mapeamento das 50 colunas e orientação Looker.
- Macro `ultimo_registro_ate_evento` no `main` (já existente antes do diff de arquivos da PR).
- Lacunas sem inventar `Não` em oficinas/documentos.

## Pós-merge (checklist)

- [ ] Negócio validou contagem PAI e semântica de “pontual”.
- [ ] Looker atualizado (medidas e `MAX` para contexto mensal).
- [ ] CI/dbt test verde em `flag_situacao_rua`.
- [ ] Atualizar esta nota com resultado do merge e data de deploy prod.
