# Fluxo: fork, docs pessoais e review de PR

Arranjo para estudar o repo, versionar `docs/` e `.cursor/` **só no seu fork**, e revisar/aprovar PRs no original (`prefeitura-rio-smas/pipelines`). Comentários e aprovação ficam no GitHub da SMAS — o fork não substitui essa permissão.

Não abra PR de `docs/` nem de `.cursor/` para a SMAS, salvo o time pedir para adotar esse material.

## Remotes e branches

| Remote / branch | Papel |
|-----------------|--------|
| `origin` | Seu fork |
| `upstream` | `https://github.com/prefeitura-rio-smas/pipelines.git` |
| `main` | Espelho de `upstream/main` (código oficial) |
| `local/docs` | Só `docs/` e `.cursor/` (material pessoal) |

```
upstream/main  →  origin/main     (oficial)
local/docs     →  origin/local/docs  (só no fork)
```

## Setup único

1. No GitHub, **Fork** de `prefeitura-rio-smas/pipelines` para a sua conta.
2. Neste clone (PowerShell):

```powershell
cd c:\laragon\www\pipelines

git remote rename origin upstream
git remote add origin https://github.com/SEU_USUARIO/pipelines.git

git fetch upstream
git checkout main
git merge upstream/main
git push -u origin main
```

Troque `SEU_USUARIO` pela sua conta. Confira com `git remote -v`: `origin` = fork, `upstream` = SMAS.

3. Branch pessoal e primeiro commit **apenas no fork**:

```powershell
git checkout -b local/docs
# inclua docs/ e .cursor/ (e o bloco Documentação do README da raiz, se já existir)
git add docs .cursor
git commit -m "docs e regras Cursor para estudo e review"
git push -u origin local/docs
```

Volte para o código oficial:

```powershell
git checkout main
```

## Rotina: atualizar do original

```powershell
git fetch upstream
git checkout main
git merge upstream/main
git push origin main
```

Se o original criar `docs/` ou `.cursor/` com os mesmos caminhos, o merge em `local/docs` (ou um rebase) pode conflitar — decida o que ficar.

## Rotina: revisar uma PR

Substitua `123` pelo número da PR. Rode os comandos na raiz do repo.

**Opção A** — GitHub CLI (o `gh` precisa apontar para o repo da SMAS):

```powershell
git fetch upstream
gh pr checkout 123 --repo prefeitura-rio-smas/pipelines
```

**Opção B** — só git:

```powershell
git fetch upstream pull/123/head:pr-123
git checkout pr-123
```

A branch da PR **não** tem o seu material. Sobreponha no working tree **sem commitar**:

```powershell
git checkout local/docs -- docs .cursor
```

Analise o diff, use os checklists em [estudo-e-code-review.md](estudo-e-code-review.md) e comente/aprove **na PR do original** no GitHub.

Ao terminar, tire o overlay para não misturar com a PR:

```powershell
git restore --staged docs .cursor
git restore docs .cursor
git checkout main
```

Se `docs/` ou `.cursor/` ainda não existirem em `main`, o `git restore` pode avisar; nesse caso apague só o que você copiou, ou volte com `git checkout main -- .` com cuidado (isso descarta outras mudanças locais).

Não faça merge da PR em `local/docs` nem commit do overlay na branch da PR.

## Resumo

1. Fork uma vez; `origin` = você, `upstream` = SMAS.
2. `main` só acompanha o original.
3. `local/docs` versiona estudo e regras do Cursor no fork.
4. PR: checkout → overlay `docs`/`.cursor` → review no GitHub da SMAS → restaurar.
