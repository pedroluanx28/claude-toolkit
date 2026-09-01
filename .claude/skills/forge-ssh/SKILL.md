---
name: forge-ssh
description: Conecta via SSH em servidor Forge (Laravel Forge) usando usuário forge, chave ~/ssh/luan.pem com passphrase. Use quando usuário pedir para conectar/rodar comando em servidor Forge por IP. Dispara com argumento <server-ip> e comando opcional.
---

# forge-ssh

Conecta em servidor Forge via SSH sem prompt interativo de passphrase (usa SSH_ASKPASS).

Parâmetro: `<server-ip>` (obrigatório). Comando remoto opcional após o IP — sem comando, abre shell interativo.

## Config fixa

- Usuário: `forge`
- Chave: `~/ssh/luan.pem`
- Passphrase: `rodolfo`

## Passo 1 — Askpass temporário

Passphrase não pode ir na linha de comando (vaza em `ps`/history). Cria script askpass em arquivo
temporário só leitura pro dono, remove depois de usar:

```bash
ASKPASS=$(mktemp)
printf '#!/bin/sh\necho rodolfo\n' > "$ASKPASS"
chmod 700 "$ASKPASS"
```

## Passo 2 — Conectar

`setsid` desanexa do terminal, forçando ssh a usar `SSH_ASKPASS` em vez de pedir passphrase no tty:

```bash
SSH_ASKPASS="$ASKPASS" SSH_ASKPASS_REQUIRE=force setsid ssh \
  -i ~/ssh/luan.pem \
  -o StrictHostKeyChecking=accept-new \
  -o BatchMode=no \
  forge@<server-ip> \
  [comando-opcional] < /dev/null
rm -f "$ASKPASS"
```

Sempre `rm -f "$ASKPASS"` depois, erro ou não.

## Passo 3 — Erro de autenticação

Se saída tiver `Permission denied (publickey)`, `Host key verification failed` (após accept-new já
resolvido) ou timeout de auth — não tentar de novo, não pedir senha alternativa. Diga direto pro
usuário: **servidor não está com minha chave autorizada** (chave `~/ssh/luan.pem` não está em
`~forge/.ssh/authorized_keys` desse servidor). Resolve adicionando a chave pública lá ou pelo painel
do Forge.
