---
description: Busca um work item no Azure DevOps e orquestra tech-lead + dev-backend + dev-frontend para planejar e implementar a tarefa de ponta a ponta.
argument-hint: <numero-da-backlog>
---

O usuário quer implementar o work item **#$1** do Azure DevOps. Execute os passos abaixo, nesta ordem.

## 1. Buscar o work item

Determine `org` e `project` do Azure DevOps: procure variáveis `AZURE_DEVOPS_ORG` / `AZURE_DEVOPS_PROJECT` no ambiente ou em `.claude/settings.json` do projeto atual. Se não encontrar nenhuma das duas, pergunte ao usuário antes de continuar (não assuma).

**Prioridade 1 — MCP do Azure DevOps:** use `ToolSearch` com a query `"select:mcp__azure-devops__wit_get_work_item,mcp__azure-devops__wit_list_work_item_comments,mcp__azure-devops__wit_get_work_items_batch_by_ids"`. Se as ferramentas existirem (MCP conectado), use-as para buscar:
- O work item `#$1` completo (título, tipo, descrição, critérios de aceite — campo tipicamente `Microsoft.VSTS.Common.AcceptanceCriteria` — e relações de tarefas filhas)
- Comentários relevantes do item (`wit_list_work_item_comments`)
- Se houver tarefas filhas (relations do tipo child), busque os detalhes delas também (`wit_get_work_items_batch_by_ids`)

**Prioridade 2 — Azure CLI (fallback, só se o MCP não estiver disponível):** antes de rodar qualquer comando, avise o usuário explicitamente:
> Não encontrei o MCP do Azure DevOps conectado. Vou usar o Azure CLI — isso requer `az` instalado, autenticado (`az login`) e com a extensão `azure-devops` (`az extension add --name azure-devops`).

Depois, busque com:
```bash
az boards work-item show --id $1 --org https://dev.azure.com/<org> --output json
```
Para comentários (não há comando dedicado no `az boards`), use:
```bash
az rest --method get --url "https://dev.azure.com/<org>/<project>/_apis/wit/workItems/$1/comments?api-version=7.1"
```
Para tarefas filhas, repita `az boards work-item show` para cada ID encontrado em `relations` (do tipo `System.LinkTypes.Hierarchy-Forward`).

Se nenhuma das duas opções funcionar (sem MCP e sem `az` configurado), pare e explique ao usuário exatamente o que falta configurar — não invente dados do work item.

Guarde o **tipo do work item** (ex.: `Bug`, `User Story`, `Improvement`, `Product Backlog Item`) e o **título** exatos — serão usados no passo 6 para montar o link do card na PR.

## 2. Resumir a tarefa

Produza um resumo claro e objetivo contendo: título, tipo do item (User Story/Bug/PBI/etc.), o que precisa ser feito, escopo, critérios de aceite, tarefas filhas (se houver) e comentários relevantes que mudem o entendimento do escopo. Guarde os critérios de aceite originais — serão usados na revisão final.

## 3. Plano do tech-lead

Acione o agente `tech-lead` (Agent tool, `subagent_type: tech-lead`, em foreground — o resultado é necessário antes de continuar) passando o resumo do passo 2. Peça explicitamente o "Plano de Execução" no formato que esse agente já usa (contexto, back-end, front-end, contrato de API, ordem de execução, riscos).

Mostre o plano ao usuário antes de seguir para a implementação.

## 4. Criar a branch de trabalho

Antes de acionar qualquer agente de implementação, crie a branch onde o trabalho será feito:

- **Branch base — padrão `dev`:** a menos que o usuário tenha pedido explicitamente para partir de outra branch nesta conversa, sempre parta de `dev` atualizada:
  ```bash
  git fetch origin dev
  git checkout -b <branch> origin/dev
  ```
- Se o usuário indicou outra branch base, substitua `origin/dev` por ela (ex.: `origin/<branch-indicada>`), mas mantenha o restante do fluxo igual.
- **Nome da branch** segue a convenção do `CLAUDE.md`: `<prefixo>/<descricao-curta-em-kebab>-ab#$1`.
  - Infira o `<prefixo>` a partir do tipo do work item: `Bug` → `fix`; `User Story`/`Feature`/`Improvement`/`Product Backlog Item` → `feat`; tarefas de manutenção/dependências → `chore`; refatoração pura → `refactor`. Na dúvida, pergunte ao usuário.
  - `<descricao-curta-em-kebab>` vem do título do work item, resumido.

Se já existir uma branch local com uncommitted changes relevantes no diretório (verifique com `git status` antes), não descarte nada — avise o usuário e peça orientação antes de trocar de branch.

## 5. Implementação

Acione `dev-backend` e `dev-frontend` respeitando a ordem definida pelo tech-lead no passo 3. Em toda chamada, inclua no prompt a seção **"Arquivos relevantes já identificados"** do plano do tech-lead (passo 3) — evita que o agente reexplore do zero o que já foi localizado:
- Se o plano indicar dependência (contrato de API novo/alterado) → acione `dev-backend` primeiro, em foreground, pegue o contrato de API que ele documentar, e só então acione `dev-frontend` passando esse contrato + os arquivos relevantes do front-end já listados pelo tech-lead.
- Se o plano indicar que as partes são independentes → acione os dois em paralelo, em uma única mensagem com duas chamadas do Agent tool.
- Se a tarefa for só back-end ou só front-end, acione apenas o agente correspondente.

Se `dev-frontend` reportar contrato insuficiente, volte ao `dev-backend` (ou ao `tech-lead`, se for uma decisão de arquitetura) para resolver antes de finalizar.

**Não commite nada neste passo.** As alterações ficam apenas no working tree, aguardando o teste manual do usuário (passo 7).

> Se em qualquer momento a partir daqui o usuário disser que precisa parar para tratar outra atividade, pare o fluxo normal e siga a seção **"Interrupção no meio da tarefa"** no final deste documento em vez dos passos 6–7.

## 6. Revisão final

Acione novamente o `tech-lead` (pode ser via `SendMessage` continuando a mesma instância do passo 3, ou uma nova chamada do Agent com o contexto necessário) passando: os critérios de aceite originais (passo 2) e o que `dev-backend`/`dev-frontend` reportaram ter implementado. Peça a "Revisão Final" no formato já definido por esse agente (critérios cobertos, critérios não cobertos, recomendação).

## 7. Reportar ao usuário e aguardar teste manual

Apresente ao usuário, em português: o resumo da tarefa, o plano executado, os arquivos criados/alterados por cada agente, e a revisão final do tech-lead — destacando com clareza qualquer critério de aceite não coberto.

Deixe explícito que as alterações ainda **não foram commitadas** e que você está aguardando o usuário testar manualmente antes de prosseguir com commit, push e abertura da PR.

## Ajustes pós-teste manual

É comum o usuário pedir ajustes depois de testar manualmente. **Nem todo ajuste precisa reabrir o pipeline completo** (tech-lead → dev-backend/dev-frontend) — isso gera overhead desnecessário para mudanças pequenas.

**Ajuste direto** (você mesmo edita com Read/Edit, sem acionar sub-agentes) quando o pedido:
- Fica contido em um arquivo (ou poucos arquivos da mesma camada) e não muda o contrato de API entre back-end e front-end;
- É mecânico/de baixo risco: condicional de exibição, texto, estilo, ordem de elementos, ajuste visual, nome de campo, etc.;
- Você já entende o código envolvido (leu o arquivo nesta sessão ou é trivial de ler agora).

Rode o teste/lint relevante se aplicável e informe objetivamente o que mudou — sem reabrir o plano do tech-lead nem acionar dev-backend/dev-frontend para isso.

**Volte ao pipeline completo** (tech-lead e/ou dev-backend/dev-frontend, repetindo os passos 3/5/6) quando o pedido:
- Mudar o contrato de API entre back-end e front-end;
- Exigir decisão de arquitetura ou reabrir uma escolha que o tech-lead já validou;
- Tocar múltiplas camadas/módulos de forma não trivial;
- Você não tiver certeza do impacto sem investigar mais a fundo.

Na dúvida, prefira o ajuste direto — é mais barato corrigir depois do que pagar o overhead de sub-agentes para algo trivial.

## 8. Aprovação do usuário → commit, push e PR

Só execute este passo depois que o usuário confirmar, após o teste manual, que as alterações estão aprovadas (frases como "aprovado", "pode seguir", "tá funcionando", "pode commitar", etc.). Não commite nem abra PR antes disso.

1. Rode `git status` e revise o que será incluído — adicione os arquivos relevantes por nome (evite `git add -A`/`git add .` sem revisão) e confira se não há nada sensível (`.env`, credenciais) sendo staged.
2. Commit com mensagem no padrão do repositório (Conventional Commits, mesmo prefixo da branch — `feat:`, `fix:`, etc.), descrevendo o "porquê" da mudança. Consulte `git log` para manter o estilo.
   - **Repositório `saudeintegrada`:** use `git commit --no-verify`. Os hooks Husky de pre-commit têm falhas pré-existentes no host, não relacionadas às mudanças feitas nesta tarefa — é assim que as PRs recentes desse repo já vêm sendo commitadas (confirmado em histórico de PRs mergeadas, ex. #4703). Não vale a pena tentar corrigir o hook quebrado como parte da tarefa, a menos que o usuário peça isso explicitamente.
3. Push da branch para `origin` com upstream:
   ```bash
   git push -u origin <branch>
   ```
   - **Repositório `saudeintegrada`:** use `git push --no-verify` pelo mesmo motivo do item 2 (pre-push também está quebrado no host).
4. Monte a descrição da PR usando o template do `CLAUDE.md`, preenchendo **"Link(s) do(s) card(s):"** neste formato exato — o texto do link precisa ser literalmente `AB#<ID>` (maiúsculo, sem espaço), pois é esse padrão que a integração GitHub↔Azure Boards reconhece para vincular a PR ao work item automaticamente (ver passo 6):
   ```
   [AB#<ID>](https://dev.azure.com/<org>/<project-url-encoded>/_workitems/edit/<ID>): <título do work item>
   ```
   - `<org>` = valor de `AZURE_DEVOPS_ORG` (ex.: `multintegrada`)
   - `<project-url-encoded>` = valor de `AZURE_DEVOPS_PROJECT` com URL-encoding (ex.: `Saúde Integrada` → `Sa%C3%BAde%20Integrada`)
   - `<título>` = exatamente o valor guardado no passo 1
   - `<ID>` = `$1`

   Exemplo real (work item #11876):
   ```
   [AB#11876](https://dev.azure.com/multintegrada/Sa%C3%BAde%20Integrada/_workitems/edit/11876): Enriquecer contexto de log de exceções (trace filtrado + dados da requisição) para Grafana/Loki
   ```
5. Crie a PR com `gh pr create`, base = a branch usada como base no passo 4, título curto e descritivo, corpo = template preenchido (via HEREDOC para preservar formatação).
   - Se `gh pr edit` for necessário depois (ex. corrigir o corpo), e falhar com um erro de GraphQL tipo `Projects (classic) is being deprecated` (não relacionado ao conteúdo do body), a edição não foi de fato aplicada — use `gh api repos/<org>/<repo>/pulls/<numero> -X PATCH -f body="..."` como alternativa, que evita essa query quebrada.
6. **Vincule a PR ao work item no Azure DevOps:**
   - **Se o repositório for hospedado no GitHub** (todos os projetos deste índice são — remote `origin` em `github.com/multintegradabr/...`): a vinculação acontece **automaticamente** pela integração Azure Boards↔GitHub assim que o texto `AB#<ID>` aparece no título ou corpo da PR (feito no passo 4) — **não** use `az repos pr work-item add` aqui, esse comando é exclusivo de PRs de Azure Repos Git (`dev.azure.com/.../_git/...`) e não tem efeito nenhum em PRs do GitHub, mesmo rodando sem erro aparente de sintaxe. Depois de criar/editar a PR, confirme a vinculação com:
     ```bash
     az boards work-item show --id $1 --org https://dev.azure.com/<org> --output json
     ```
     e cheque se `relations` contém uma entrada `"name": "GitHub Pull Request"`. Pode levar alguns segundos para o webhook processar.
   - Se por algum motivo o `AB#<ID>` não gerar o link automático (ex.: app do Azure Boards não instalado nesse repo), avise o usuário explicitamente que a vinculação precisa ser feita manualmente no Azure DevOps e informe por quê — não tente `az repos pr work-item add` como fallback nesse caso, ele não resolve.
7. Informe o usuário: link da PR criada e confirmação de que ela foi vinculada ao work item #$1.

---

## Interrupção no meio da tarefa

A qualquer momento entre o passo 4 (branch criada) e a aprovação do passo 8, se o usuário indicar que precisa parar para tratar outra atividade (frases como "preciso ir para outra tarefa", "vou pausar isso e depois volto", "surgiu uma prioridade", etc.), **não** siga os passos 6–8 normais. Em vez disso:

1. Rode `git status` para revisar tudo que foi alterado até o momento (mesmo que incompleto).
2. Stage e commit de tudo que já existe, com mensagem clara indicando que é trabalho em andamento (ex.: prefixo da branch + descrição do que foi feito até aqui — não descreva como concluído o que não foi). No repositório `saudeintegrada`, use `git commit --no-verify` (ver nota no item 2 do passo 8).
3. Push da branch:
   ```bash
   git push -u origin <branch>
   ```
   No repositório `saudeintegrada`, use `git push --no-verify` (ver nota no item 3 do passo 8).
4. Abra a PR em **modo draft**:
   ```bash
   gh pr create --draft --title "..." --body "..."
   ```
   Use o mesmo template e o mesmo formato de link do card — texto literal `AB#<ID>` (item 4 do passo 8), mas deixe explícito na seção "Observações" que o trabalho está incompleto/em andamento e que a PR está em draft por esse motivo.
5. Tente vincular a PR ao work item no Azure DevOps, seguindo o mesmo procedimento do item 6 do passo 8 (link automático via `AB#<ID>` em repos GitHub → confirmar via `az boards work-item show` → aviso manual só se o automático falhar).
6. Informe o usuário: link da PR draft, o que já foi feito, e o que ainda falta (com base no plano do tech-lead) para retomar depois.
