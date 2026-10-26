# Documentação — pipelines (RJ SMAS)

Guia de onboarding para quem vai estudar o repositório e participar de code review. Complementa o setup do [README da raiz](../README.md) e a arquitetura dbt em [queries/ARCHITECTURE.md](../queries/ARCHITECTURE.md). Não substitui esses arquivos.

## O que ler primeiro

1. [Visão geral](visao-geral.md) — para que serve o projeto, stack e ambientes.
2. [Estrutura](estrutura.md) — pastas, camadas Medallion e nomenclatura.
3. [Como usar](como-usar.md) — comandos do dia a dia, Prefect local e CI.
4. [Estudo e code review](estudo-e-code-review.md) — trilha de leitura e checklists.
5. [Fluxo fork e review](fluxo-fork-e-review.md) — fork, branch `local/docs` e como puxar uma PR.
6. [Histórico de reviews](reviews/README.md) — registros locais de PRs revisadas (referência futura).
7. [Padrão espelho dbt → GCS](padrao-espelho-dbt-prefect.md) — mart no dbt, Prefect só decide e exporta (referência: PRs #154 e #158).

## Mapa rápido

| Precisa de… | Arquivo |
|-------------|---------|
| Instalar uv, gcloud e validar dbt | [README.md](../README.md) |
| Fork, docs pessoais e rotina de PR | [fluxo-fork-e-review.md](fluxo-fork-e-review.md) |
| Histórico de code reviews (fork) | [reviews/README.md](reviews/README.md) |
| Princípios das camadas raw / intermediate / marts | [queries/ARCHITECTURE.md](../queries/ARCHITECTURE.md) |
| História da migração AcolheRio → Medallion | [queries/refatoracao-arquitetura-acolherio.md](../queries/refatoracao-arquitetura-acolherio.md) |
| Targets BigQuery (`dev`, `staging`, `prod`, `ci`) | [queries/profiles.yml](../queries/profiles.yml) |
| Tags, schemas e legado ligado/desligado | [queries/dbt_project.yml](../queries/dbt_project.yml) |
| Deployments Prefect | [prefect.yaml](../prefect.yaml) |
| Espelho: mart dbt → CSV no GCS | [padrao-espelho-dbt-prefect.md](padrao-espelho-dbt-prefect.md) |

## Base da IA no Cursor

Regras persistentes em [`.cursor/rules/`](../.cursor/rules/): visão geral sempre ativa, convenções dbt ao editar SQL/YAML, convenções Prefect ao editar Python e o padrão de espelho (mart → GCS) ao editar esses arquivos. A skill `pipeline-dbt-gcs-export` aplica o procedimento quando a tarefa for criar ou alterar um export desse tipo.
