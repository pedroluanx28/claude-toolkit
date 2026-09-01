---
name: review-fix
description: Corrige uma pull request do GitHub reprovada em revisão — analisa o diff, o backlog vinculado (Azure Boards) e as reprovações do reviewer, aplica as correções no código respeitando o CLAUDE.md do projeto, comita e dá push na branch da PR, e resolve cada conversation (ou responde o comentário de reprovação, se não houver conversations) explicando a correção. Use quando o usuário disser que uma PR foi reprovada/tem changes requested e pedir para corrigir e resolver as conversations. Dispara com o argumento <link-da-pull-request>.
---

# review-fix

Fecha o ciclo de uma PR reprovada: entende o que foi pedido (backlog), entende o que o reviewer
apontou, corrige o código de verdade (não só responde), e deixa a PR num estado onde o reviewer só
precisa reler o resultado. Nunca resolva uma conversation sem ou corrigir o código ou justificar por
escrito por que não vai corrigir — "resolver" sem nenhuma das duas coisas é esconder a reprovação,
não endereçá-la.

Push na branch da PR e resolver conversations são ações visíveis para o time. O gatilho é o próprio
usuário pedindo para corrigir esta PR — não precisa reconfirmar a cada commit, mas nunca rode esta
skill "de brinde" sobre uma PR que o usuário não apontou.

## Passo 1 — Resolver a PR e o projeto

Extraia `owner`, `repo` e `number` do `<link-da-pull-request>`. `repo` provavelmente é um dos
projetos do `~/.claude/CLAUDE.md` global (SIRegula, saudeintegrada, DigitalSignAPI,
telemedicina-api, CnsApi, essentials-api, SiAdmin, indicadores-esus-client, html-to-pdf-buffer) —
identifique o clone local pelo índice desse arquivo e `cd` até lá; se o projeto tiver `CLAUDE.md`
próprio, leia-o agora (convenções de arquitetura, camadas, fluxo de PR — julgar ou corrigir código
sem isso é trabalhar no vácuo).

```bash
gh pr view $NUMBER --repo $OWNER/$REPO --json title,body,url,headRefName,baseRefName,state,reviews,comments
```

Confira se a branch local bate com a da PR (`git status`, `git branch --show-current`). Se não
bater, `git fetch origin <headRefName>` e `git checkout <headRefName>` — sempre rode `git status`
antes de qualquer checkout/reset para não perder trabalho não commitado que já esteja na árvore.

Guarde do `body` da PR o(s) link(s) de card (`AB#NNNN`, formato Azure Boards) — é o que valida o
passo 2.

## Passo 2 — Ler o backlog vinculado

Busque cada work item citado no body da PR:

```bash
az boards work-item show --id $CARD_ID --org https://dev.azure.com/multintegrada --output json
```

Extraia de `fields`: `System.Title`, `System.Description` (o HTML tem "O quê", "Por quê", "Regras
de Negócio", "Regras de Funcionalidade" e "Critérios de Aceite" — é a fonte de verdade sobre o
comportamento esperado) e `System.State`. Se `az` não estiver autenticado ou o card não existir mais,
não trave a skill por isso — prossiga só com o que der para inferir do body da PR e do diff, e avise
o usuário que não validou contra o backlog.

O objetivo aqui não é decorar o card, é ter munição para o passo 4: quando o reviewer aponta um
problema, você precisa saber se a correção sugerida é consistente com a regra de negócio pedida, ou
se o reviewer (ou o autor original) interpretou mal um critério de aceite.

## Passo 3 — Ler o histórico completo de revisão

Nunca corrija só a partir do resumo da review — o comentário-resumo costuma ser um diagnóstico
compacto; o detalhe real (arquivo, linha, sugestão) está nas conversations.

```bash
gh api repos/$OWNER/$REPO/pulls/$NUMBER/reviews              # veredito + corpo de cada review
gh api --paginate repos/$OWNER/$REPO/pulls/$NUMBER/comments  # conversations (comentários de linha)
gh api repos/$OWNER/$REPO/issues/$NUMBER/comments            # comentários soltos (aba Conversation)
```

O REST não expõe se a conversation já foi resolvida — para isso, GraphQL (guarde `id` de cada
thread e `databaseId` de cada comment, você precisa dos dois: o primeiro para resolver a thread no
passo 6, o segundo para responder o comentário no passo 5):

```bash
gh api graphql -f query='
  query($owner: String!, $repo: String!, $number: Int!) {
    repository(owner: $owner, name: $repo) {
      pullRequest(number: $number) {
        reviewThreads(first: 100) {
          nodes {
            id
            isResolved
            isOutdated
            path
            line
            comments(first: 20) { nodes { id databaseId author { login } body } }
          }
        }
      }
    }
  }' -f owner="$OWNER" -f repo="$REPO" -F number="$NUMBER"
```

Separe as threads em: já resolvidas (ignore — outra correção já tratou), e abertas (é isso que o
passo 4 precisa corrigir). Se **não houver nenhuma conversation** (revisão só com o comentário-resumo
de reprovação, sem comentários de linha), pule os passos 5/6 de thread e vá direto para responder o
comentário de reprovação no passo 7.

## Passo 4 — Entender o diff e planejar a correção

```bash
gh pr diff $NUMBER --repo $OWNER/$REPO
```

Para cada ponto reprovado (bloqueante ou não), releia o arquivo **completo** no clone local — não só
o hunk do diff — para entender o contexto ao redor antes de decidir a correção. Preste atenção
especial em pontos que se repetem sob a mesma causa raiz (comentários diferentes que apontam o mesmo
problema de ângulos distintos) — corrigir a causa raiz costuma resolver vários de uma vez, e a
correção fica mais coerente do que remendar sintoma por sintoma.

Nem todo comentário não-bloqueante precisa virar código: se a sugestão do reviewer conflita com o
escopo do card (passo 2), com o `CLAUDE.md` do projeto ("NÃO refatorar código fora do escopo da
tarefa"), ou expõe um problema pré-existente mais amplo que essa PR não introduziu, é legítimo optar
por não alterar o código — mas isso exige justificar por escrito na resposta (passo 5), nunca apenas
resolver a thread em silêncio.

Monte um plano curto (mentalmente ou com a ferramenta de tasks) antes de editar: quais arquivos,
qual a causa raiz de cada bloqueante, o que vai virar código e o que vai virar resposta justificada.

## Passo 5 — Corrigir

Aplique as correções seguindo as convenções do `CLAUDE.md` do projeto (camadas, nomenclatura,
padrões de service/request/rule, etc.) e o padrão já existente no restante do arquivo. Não
aproveite para refatorar nada fora do que os comentários pedem.

Depois de editar, rode o que estiver disponível no projeto antes de commitar — `php -l` nos arquivos
PHP tocados, o linter/type-checker do front se configurado, e a suíte de testes relevante se houver
ambiente rodando (Docker/`make`) e testes cobrindo a área tocada. Não deixe de commitar só porque
não dá para rodar a suíte completa neste ambiente — registre a limitação para o usuário no relatório
final (passo 8), mas não pule a checagem sintática básica.

## Passo 6 — Commit e push

```bash
git add <arquivos-alterados>
git commit -m "fix: <resumo> (AB#NNNN)"
git push origin <headRefName>
```

Siga a convenção de commit/PR do `CLAUDE.md` do projeto se ela especificar algo (idioma do título,
prefixo, referência ao card). Guarde o SHA do commit (`git rev-parse --short HEAD`) — vai ser citado
nas respostas do passo 7.

## Passo 7 — Responder cada conversation

Para cada thread aberta do passo 3, responda o comentário **específico** (não um comentário solto)
explicando o que mudou e por quê, citando o SHA do passo 6. Use `in_reply_to` com o `databaseId` do
comentário original — isso ancora a resposta na mesma thread em vez de criar uma solta:

```bash
gh api repos/$OWNER/$REPO/pulls/$NUMBER/comments \
  -f body="Corrigido em <sha>. <explicação objetiva da causa raiz e da correção>." \
  -F in_reply_to=<databaseId>
```

Para um ponto que você decidiu **não** corrigir (passo 4), a resposta é o lugar de justificar —
seja específico sobre por que (escopo do card, padrão pré-existente no resto do arquivo/rota,
CLAUDE.md, etc.), não apenas "vou deixar assim".

Se a PR não tinha nenhuma conversation (revisão só com comentário-resumo), responda esse comentário
de reprovação diretamente (comentário solto na aba Conversation, via `gh pr comment $NUMBER --repo
$OWNER/$REPO --body-file <arquivo>`), cobrindo todos os pontos levantados no resumo.

## Passo 8 — Resolver as threads

Só resolva uma thread depois de ter respondido a ela (passo 7) — resolver antes de responder é
enterrar o ponto sem registro do que foi feito.

```bash
gh api graphql -f query='
  mutation($id: ID!) { resolveReviewThread(input: {threadId: $id}) { thread { id isResolved } } }
  ' -f id="<thread-id>"
```

Repita para cada thread aberta do passo 3. Ao final, confira via GraphQL (mesma query do passo 3)
que todas as threads relevantes voltaram com `isResolved: true`.

## Passo 9 — Relatar ao usuário

Poste um comentário-resumo na PR (`gh pr comment`) listando o que foi corrigido, ponto a ponto, e
citando o SHA. No chat, resuma de forma objetiva: quantos bloqueantes/não-bloqueantes foram
corrigidos, o que foi respondido em vez de corrigido (e por quê), limitações de validação (ex.: não
foi possível rodar a suíte de testes neste ambiente), e o link da PR. Não repita o conteúdo de cada
conversation no chat — elas já estão publicadas e visíveis lá.
