# Documentação — pipelines (RJ SMAS)

Guia de onboarding para quem vai estudar o repositório e participar de code review. Complementa o setup do [README da raiz](../README.md) e a arquitetura dbt em [queries/ARCHITECTURE.md](../queries/ARCHITECTURE.md). Não substitui esses arquivos.

## O que ler primeiro

1. [Visão geral](visao-geral.md) — para que serve o projeto, stack e ambientes.
2. [Estrutura](estrutura.md) — pastas, camadas Medallion e nomenclatura.
3. [Como usar](como-usar.md) — comandos do dia a dia, Prefect local e CI.
4. [Estudo e code review](estudo-e-code-review.md) — trilha de leitura e checklists.
5. [Fluxo fork e review](fluxo-fork-e-review.md) — fork, branch `local/docs` e como puxar uma PR.

## Mapa rápido

| Precisa de… | Arquivo |
|-------------|---------|
| Instalar uv, gcloud e validar dbt | [README.md](../README.md) |
| Fork, docs pessoais e rotina de PR | [fluxo-fork-e-review.md](fluxo-fork-e-review.md) |
| Princípios das camadas raw / intermediate / marts | [queries/ARCHITECTURE.md](../queries/ARCHITECTURE.md) |
| História da migração AcolheRio → Medallion | [queries/refatoracao-arquitetura-acolherio.md](../queries/refatoracao-arquitetura-acolherio.md) |
| Targets BigQuery (`dev`, `staging`, `prod`, `ci`) | [queries/profiles.yml](../queries/profiles.yml) |
| Tags, schemas e legado ligado/desligado | [queries/dbt_project.yml](../queries/dbt_project.yml) |
| Deployments Prefect | [prefect.yaml](../prefect.yaml) |

## Base da IA no Cursor

Regras persistentes em [`.cursor/rules/`](../.cursor/rules/): visão geral sempre ativa, convenções dbt ao editar SQL/YAML e convenções Prefect ao editar Python.
