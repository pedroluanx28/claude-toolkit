---
name: clean-review
description: Revisa uma pull request do GitHub como um especialista sênior (impacto no sistema, performance, segurança, qualidade de código) e publica o veredito diretamente na PR via gh — cada ponto reprovado vira uma conversation (comentário de linha) na aba "Files changed", e a PR é marcada como aprovada (com "LGTM") ou como changes requested. Use quando o usuário pedir para revisar/avaliar uma PR e quiser o resultado publicado no GitHub, não apenas relatado no chat. Dispara com o argumento <link-da-pull-request>.
---

# clean-review

Revisão de PR ponta a ponta: busca a PR no GitHub, revisa o diff como um especialista, e publica o
resultado — aprovação com "LGTM" ou reprovação com uma **conversation por ponto** — via `gh`.

Publicar review em uma PR é uma ação visível para o time e difícil de desfazer completamente
(dá pra dismiss, não pra apagar o rastro). O usuário (luanzn) deu permissão permanente pra publicar
o veredito no passo 6 sem pedir confirmação extra — o gatilho já é ele invocar esta skill com o
link da PR. Não peça confirmação de novo antes do passo 6. Mas nunca rode isso "de brinde" durante
outra tarefa sem o usuário ter pedido explicitamente uma revisão desta PR.

## Passo 1 — Resolver a PR

Extraia `owner`, `repo` e `number` do `<link-da-pull-request>` (formato
`github.com/<owner>/<repo>/pull/<number>`). Busque os metadados e o SHA do head, que serão
necessários depois para o payload da review:

```bash
gh api repos/$OWNER/$REPO/pulls/$NUMBER
```

Guarde `.head.sha` (vira `commit_id` na review), `.title`, `.body` e `.base.ref`/`.head.ref`.

Sempre busque esses metadados de novo no momento da revisão, mesmo que a PR já tenha aparecido
antes nesta conversa — a PR pode ter recebido commits novos, comentários ou reviews desde então.
Nunca reaproveite um `head.sha` ou diff de uma leitura anterior: a revisão é sempre sobre a versão
mais atual da PR.

## Passo 2 — Ler o histórico de reviews e conversations

Antes de revisar o diff, veja o que já foi discutido nesta PR — senão o risco é repetir um ponto
que o autor já resolveu, ou reprovar algo que uma review anterior (sua ou de outra pessoa) já
tratou como aceitável. Puxe:

```bash
gh api repos/$OWNER/$REPO/pulls/$NUMBER/reviews          # reviews anteriores (veredito + corpo)
gh api repos/$OWNER/$REPO/pulls/$NUMBER/comments         # comentários de linha (conversations)
gh api repos/$OWNER/$REPO/issues/$NUMBER/comments        # comentários soltos na aba Conversation
```

O endpoint REST de `pulls/.../comments` não expõe se a conversation foi marcada como **resolvida** —
para isso, uma consulta GraphQL:

```bash
gh api graphql -f query='
  query($owner: String!, $repo: String!, $number: Int!) {
    repository(owner: $owner, name: $repo) {
      pullRequest(number: $number) {
        reviewThreads(first: 100) {
          nodes {
            isResolved
            comments(first: 20) { nodes { path line body author { login } } }
          }
        }
      }
    }
  }' -f owner="$OWNER" -f repo="$REPO" -F number="$NUMBER"
```

Use isso para entender o contexto completo: o que já foi apontado, o que já foi corrigido (thread
resolvida) e o que ainda está em aberto (thread não resolvida) — inclusive se um ponto que você
identificaria no passo 4 já foi levantado e resolvido, não o repita; se ainda está aberto e não
resolvido, pode reforçá-lo, mas deixe claro que é um ponto já sinalizado antes.

## Passo 3 — Pegar o diff com números de linha reais

Comentários de linha só podem ser ancorados em linhas que aparecem nos hunks do diff. Busque os
arquivos alterados com patch incluído (pagine se houver mais de 30 arquivos):

```bash
gh api --paginate repos/$OWNER/$REPO/pulls/$NUMBER/files
```

Cada item traz `filename`, `status`, `patch` (o hunk unificado, com os números de linha da versão
nova/antiga) e contadores de `additions`/`deletions`. É a partir desse `patch` que você escolhe a
linha (`line`) e o lado (`side: RIGHT` para linha adicionada/contexto na versão nova, `LEFT` para
linha removida) de cada comentário.

## Passo 4 — Puxar contexto local

**Limite de escopo:** contexto extra vem só dos arquivos que já aparecem na lista do passo 3. Nunca
busque um arquivo que não está no diff só porque a lógica parece apontar pra lá (um tipo importado,
uma função chamada, uma config global, o call site de um método) — isso é o que faz a revisão
disparar em tempo para PRs pequenas. Se notar um risco que só dá pra confirmar com contexto de fora
do diff, não vá atrás: registre a incerteza no comentário (ex.: "depende de como X é usado alhures —
não verificado") em vez de investigar.

`$REPO` provavelmente é um dos projetos listados no `~/.claude/CLAUDE.md` global (SIRegula,
saudeintegrada, DigitalSignAPI, telemedicina-api, CnsApi, essentials-api, SiAdmin,
indicadores-esus-client, html-to-pdf-buffer). Se o nome bater com um clone local:

- Leia o `CLAUDE.md` próprio do projeto (se existir) para conhecer convenções, arquitetura e
  regras específicas — julgar qualidade de código sem isso é julgar no vácuo.
- Dê um `git fetch` e olhe a versão completa (não só o diff) dos arquivos **já alterados pela PR**
  direto do clone — o método pode parecer bom isolado e ser um problema pelo resto do arquivo ao
  redor, mesmo sem sair dele.
- Se o projeto tiver linter/análise estática configurado (PHPStan, ESLint, etc.), rodar contra os
  arquivos alterados reforça achados, mas não é obrigatório — não trave a review por isso.

Se o repo não estiver clonado localmente, siga só com o que `gh api` do passo 3 já retornou (patch
de cada arquivo) — é suficiente para a maioria das PRs. Só quando o patch não for suficiente (ex.:
precisa ver uma função inteira que o diff só mostra pela metade) é que vale buscar a versão completa
de um arquivo **que já está na lista do passo 3** — nunca um arquivo novo. Para 1 arquivo,
`gh api repos/$OWNER/$REPO/contents/<path>?ref=<head.sha>` direto resolve. Para mais de 1-2, prefira
um clone raso único do branch da PR a repetir essa chamada arquivo por arquivo (cada uma é um
round-trip sequencial — a maior fonte de lentidão quando o repo não está local):

```bash
gh repo clone $OWNER/$REPO /tmp/claude-*/.../scratchpad/pr-$NUMBER -- --depth 1 --branch <head.ref>
```

e leia os arquivos localmente a partir daí.

## Passo 5 — Revisar como um especialista

Avalie o diff considerando, na ordem de prioridade:

1. **Corretude** — bugs, edge cases não tratados, lógica que quebra em cenários plausíveis.
2. **Impacto no sistema** — o que essa mudança afeta além do arquivo tocado: contratos de API
   quebrados, side effects em outras rotinas, migrations irreversíveis, mudanças em integrações
   externas (e-SUS, CADSUS, IBGE, CNES), dados sensíveis de saúde (LGPD).
3. **Performance** — N+1 queries, loops desnecessários, queries sem índice, payloads grandes sem
   paginação, chamadas síncronas que deveriam ser assíncronas/filas.
4. **Segurança** — injeção (SQL/XSS), validação de entrada ausente, checagem de autorização
   ausente em rotas sensíveis, segredos hardcoded, HMAC/tokens mal validados.
5. **Qualidade e manutenibilidade** — nomes ruins, duplicação evitável, abstração
   desnecessária, inconsistência com o padrão já usado no resto do projeto.
6. **Cobertura de testes** — mudança de regra de negócio sem teste correspondente.

Não invente problema para preencher a lista — se a PR estiver limpa, aprove. O objetivo é rigor,
não implicância.

Mantenha a análise dentro do limite de escopo do passo 4: o julgamento é sobre o que o diff (mais o
resto dos arquivos já alterados) mostra, não sobre uma investigação aberta pelo resto do repositório.
Uma hipótese de bug que só se confirma lendo um arquivo fora da lista do passo 3 vira uma ressalva no
comentário, não um motivo para buscar esse arquivo.

## Passo 6 — Publicar o resultado

Monte **uma única review** com `gh api`, combinando as conversations e o veredito em uma chamada
atômica (assim tudo aparece publicado junto, não em ordem estranha).

Escreva o payload em `/tmp/claude-*/.../scratchpad/review-payload.json` (nunca no repo do
usuário). Cada ponto reprovado é um item em `comments`, direto e sem enrolação — só o essencial
para o autor entender o que está errado; se tiver dúvida, ele pergunta.

**Se algo foi reprovado** (`event: REQUEST_CHANGES`):

```json
{
  "commit_id": "<head.sha>",
  "event": "REQUEST_CHANGES",
  "body": "Alterações solicitadas — ver comentários.",
  "comments": [
    {
      "path": "app/Services/FooService.php",
      "line": 42,
      "side": "RIGHT",
      "body": "Essa query roda dentro do loop — N+1."
    }
  ]
}
```

**Se tudo passou** (`event: APPROVE`, sem comments):

```json
{
  "event": "APPROVE",
  "body": "LGTM"
}
```

Envie:

```bash
gh api --method POST repos/$OWNER/$REPO/pulls/$NUMBER/reviews --input /caminho/para/review-payload.json
```

Regras do payload:
- `comments` exige `path` relativo à raiz do repo e `line` dentro de um hunk visível no patch do
  passo 3 — linha fora do diff é rejeitada pela API.
- Cada item de `comments` é uma conversation independente na aba "Files changed" — é isso, e não
  um comentário solto na aba "Conversation", que corresponde a uma reprovação.
- `REQUEST_CHANGES`/`COMMENT` exigem `body` não vazio; `APPROVE` não exige, mas mande "LGTM" assim
  mesmo, como pedido.

## Passo 7 — Reportar ao usuário

Resuma no chat: veredito final, quantos pontos reprovados (se houver) e um link direto pra PR.
Não repita o conteúdo de cada conversation no chat — elas já estão publicadas e visíveis lá.

Termine a mensagem com uma linha de emojis marcando o veredito: se **APPROVE**, uma sequência de
✅ (ex.: `✅✅✅✅✅`); se **REQUEST_CHANGES**, uma sequência de ❌ (ex.: `❌❌❌❌❌`).
