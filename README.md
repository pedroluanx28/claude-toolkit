# claude-toolkit

Repo coringa de configuração pessoal do Claude Code.

## Estrutura

- `.claude/` — pasta real do Claude Code (skills, agents, commands), pronta pra usar.
  - `skills/` — clean-review, forge-ssh, review-fix
  - `agents/` — tech-lead, dev-backend, dev-frontend
  - `commands/` — build-task
- `projects/` — pasta livre para clonar/colar qualquer projeto local. Ignorada pelo git (`.gitignore`), cada projeto mantém seu próprio controle de versão.

## Skills

| Skill | Descrição |
|---|---|
| `clean-review` | Revisa uma PR do GitHub como especialista sênior e publica o veredito direto na PR via `gh`. |
| `forge-ssh` | Conecta via SSH em servidor Forge (Laravel Forge) usando chave dedicada. |
| `review-fix` | Corrige uma PR reprovada (analisa diff + backlog Azure Boards + comentários do reviewer) e resolve as conversations. |
| `/build-task` (command) | Busca work item no Azure DevOps e orquestra os agents `tech-lead`, `dev-backend` e `dev-frontend` para planejar e implementar a tarefa. |

## Agents

| Agent | Papel |
|---|---|
| `tech-lead` | Quebra a tarefa em subtarefas, define contrato de API, decide ordem back/front, revisão final. |
| `dev-backend` | PHP/Laravel: migrations, models, controllers, rotas, regras de negócio. |
| `dev-frontend` | JS/TS/React: UI, hooks, consumo de API, testes de componente. |

## Uso

Ativa tudo pra qualquer projeto com um symlink — sem copiar arquivo por arquivo:

```bash
ln -s ~/claude-toolkit/.claude ~/algum-projeto/.claude
```

Se o projeto já tem `.claude/` próprio, faz merge manual (linka só as subpastas que faltam, ex.: `ln -s ~/claude-toolkit/.claude/skills/forge-ssh ~/algum-projeto/.claude/skills/forge-ssh`).

Pra deixar global (todos os projetos, sem symlink por repo), linka direto dentro de `~/.claude/`:

```bash
ln -s ~/claude-toolkit/.claude/skills/clean-review ~/.claude/skills/clean-review
ln -s ~/claude-toolkit/.claude/skills/forge-ssh ~/.claude/skills/forge-ssh
ln -s ~/claude-toolkit/.claude/skills/review-fix ~/.claude/skills/review-fix
ln -s ~/claude-toolkit/.claude/commands/build-task.md ~/.claude/commands/build-task.md
ln -s ~/claude-toolkit/.claude/agents/tech-lead.md ~/.claude/agents/tech-lead.md
ln -s ~/claude-toolkit/.claude/agents/dev-backend.md ~/.claude/agents/dev-backend.md
ln -s ~/claude-toolkit/.claude/agents/dev-frontend.md ~/.claude/agents/dev-frontend.md
```
