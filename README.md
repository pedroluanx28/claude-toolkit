# claude-toolkit

Repo coringa de configuração pessoal do Claude Code.

## Estrutura

- `skills/` — skills personalizadas (SKILL.md) usadas em todos os projetos.
- `projects/` — pasta livre para clonar/colar qualquer projeto local. Ignorada pelo git (`.gitignore`), cada projeto mantém seu próprio controle de versão.

## Skills

| Skill | Descrição |
|---|---|
| `clean-review` | Revisa uma PR do GitHub como especialista sênior e publica o veredito direto na PR via `gh`. |
| `forge-ssh` | Conecta via SSH em servidor Forge (Laravel Forge) usando chave dedicada. |
| `review-fix` | Corrige uma PR reprovada (analisa diff + backlog Azure Boards + comentários do reviewer) e resolve as conversations. |
| `build-task` | Busca work item no Azure DevOps e orquestra os agents `tech-lead`, `dev-backend` e `dev-frontend` (em `build-task/agents/`) para planejar e implementar a tarefa. |

## Uso

Para ativar essas skills num projeto, copie ou linke a pasta `skills/<nome>` para `~/.claude/skills/<nome>` (ou para `.claude/skills/` do projeto). Para `build-task`, copie também `build-task/agents/*.md` para `~/.claude/agents/` e `build-task/build-task.md` para `~/.claude/commands/`.
