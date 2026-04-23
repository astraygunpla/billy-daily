# Bot de Daily Assíncrona para Discord

Bot automatizado de daily para Discord. No horário configurado (padrão: 09h30, seg–sex), o bot envia uma DM para cada membro de um cargo específico com três perguntas de stand-up, aguarda as respostas e publica um resumo formatado como página no ClickUp Docs.

O formato da daily é baseado no método de alinhamentos assíncronos de **[Daniel Kossmann](https://www.danielkossmann.com/pt/como-fazer-alinhamentos-diarios-sem-reunioes-assincronamente/)**, adaptado para funcionar via Discord com envio automático ao ClickUp.

---

## Estrutura de arquivos

```
discord-standup-bot/
├── src/
│   ├── index.js          Ponto de entrada
│   ├── bot.js            Cliente Discord + registro de eventos
│   ├── standup.js        Lógica principal (sendDailyDMs, handleDMReply, …)
│   ├── storage.js        Persistência em JSON
│   ├── clickup.js        Integração com a API REST do ClickUp
│   ├── scheduler.js      Jobs com node-cron (suporta reagendamento dinâmico)
│   ├── commands.js       Comandos do servidor (!start-standup, !show-answers, …)
│   ├── scheduleStore.js  Persistência do horário configurado
│   ├── formatStore.js    Persistência do formato do ClickUp
│   └── utils.js          Utilitários de data e divisão de mensagens
├── data/
│   ├── standup.json      Criado em tempo de execução — respostas armazenadas aqui
│   ├── schedule.json     Horário configurado (criado ao usar !set-time ou !set-deadline)
│   └── format.json       Formato do ClickUp (criado ao usar !set-format)
├── config.js             Carrega variáveis de ambiente em BOT_CONFIG / CLICKUP_CONFIG
├── .env.example          Template — copie para .env e preencha
└── package.json
```

---

## Schema de armazenamento

`data/standup.json` é indexado por data e depois por ID de usuário:

```json
{
  "2024-01-15": {
    "123456789": {
      "userId": "123456789",
      "username": "joao",
      "displayName": "João Silva",
      "date": "2024-01-15",
      "state": "complete",
      "yesterday": "Finalizei o módulo de pagamento",
      "today": "Vou fazer o code review do PR #42",
      "blockers": "Nenhum",
      "dmChannelId": "987654321",
      "sentAt": "2024-01-15T12:30:00.000Z",
      "completedAt": "2024-01-15T12:35:00.000Z",
      "lastUpdatedAt": "2024-01-15T12:35:00.000Z"
    }
  }
}
```

**Máquina de estados:**
`sent` → `answered_yesterday` → `answered_today` → `complete`

---

## Configuração

### 1. Criar o bot no Discord

1. Acesse <https://discord.com/developers/applications> e clique em **New Application**.
2. Abra a aba **Bot** → clique em **Add Bot**.
3. Copie o **Token** → será o valor de `DISCORD_TOKEN`.
4. Role até **Privileged Gateway Intents** e ative:
   - **Server Members Intent** (necessário para buscar membros do cargo)
   - **Message Content Intent** (necessário para ler o texto das DMs)
5. Vá em **OAuth2 → URL Generator**, selecione os escopos:
   - `bot` com as permissões: **Send Messages**, **Read Message History**, **Use Slash Commands**
6. Abra a URL gerada e convide o bot para o seu servidor.

### 2. Obter os IDs necessários

Ative o **Modo Desenvolvedor** no Discord (Configurações → Avançado → Modo Desenvolvedor) e então:

| Valor | Como obter |
|---|---|
| `DISCORD_GUILD_ID` | Clique com botão direito no ícone do servidor → Copiar ID do servidor |
| `DISCORD_ROLE_ID` | Configurações do servidor → Cargos → botão direito no cargo → Copiar ID do cargo |
| `DISCORD_OWNER_ID` | Clique com botão direito no seu nome → Copiar ID do usuário |

### 3. Configurar o ClickUp

1. No ClickUp: clique no seu **avatar** → **Configurações** → **Apps** → **API Token** → copie o token.
2. Abra o documento onde deseja que as páginas de daily sejam criadas.
   - A URL do workspace aparece como `app.clickup.com/{WORKSPACE_ID}/...` — copie o ID do workspace.
   - Abra o documento e copie o ID do doc da URL: `app.clickup.com/{ws}/docs/{DOC_ID}-...`

### 4. Criar o `.env`

```bash
cp .env.example .env
# edite o .env com seus valores reais
```

### 5. Instalar e executar

```bash
npm install
npm start
```

---

## Comandos (envie em qualquer canal do servidor)

Apenas o usuário cujo ID corresponde a `DISCORD_OWNER_ID` pode usar estes comandos.

### Stand-up

| Comando | Descrição |
|---|---|
| `!start-standup` | Dispara as DMs de stand-up manualmente |
| `!show-answers [YYYY-MM-DD]` | Exibe todas as respostas do dia |
| `!missing [YYYY-MM-DD]` | Lista quem ainda não respondeu |
| `!post-clickup [YYYY-MM-DD]` | Envia as respostas ao ClickUp manualmente |
| `!help` | Exibe a lista de comandos |

A data padrão é hoje quando omitida.

### Configuração de horário

| Comando | Descrição |
|---|---|
| `!set-time HH:MM` | Define o horário do disparo automático da daily (seg–sex) |
| `!set-deadline N` | Define quantos minutos após o stand-up o ClickUp é atualizado automaticamente |
| `!show-config` | Exibe a configuração atual de horários |

Os horários são salvos em `data/schedule.json` e persistem após reinicializações do bot.

### Formato do ClickUp

| Comando | Descrição |
|---|---|
| `!set-format language pt\|en` | Idioma das seções (Português ou English) |
| `!set-format legend on\|off` | Ativa/desativa a legenda de emojis no rodapé |
| `!set-format header on\|off` | Ativa/desativa o cabeçalho automático |
| `!set-format title <prefixo>` | Define o prefixo do título da página (ex: `Daily`) |
| `!show-format` | Exibe as configurações atuais de formato |
| `!preview-format` | Mostra uma pré-visualização do markdown que será enviado |

As configurações de formato são salvas em `data/format.json`.

## Comandos de DM (participantes)

Qualquer participante da daily pode digitar estes comandos na DM com o bot:

| Comando | Descrição |
|---|---|
| `!restart` | Apaga as respostas de hoje e recomeça (válido antes do prazo) |

---

## Configuração de horário (via variáveis de ambiente)

`STANDUP_TIME` usa sintaxe cron padrão: `MINUTO HORA * * DIAS`

```
30 9 * * 1-5   →  09h30, segunda a sexta  (padrão)
0 10 * * 1-5   →  10h00, segunda a sexta
0 9 * * *      →  09h00, todos os dias
```

O prazo dispara `DEADLINE_MINUTES` após `STANDUP_TIME`. Padrão: 30 min → encerra às 10h00.

Você também pode alterar o horário sem reiniciar o bot usando `!set-time` e `!set-deadline`.

---

## Formato da página no ClickUp

```
> Gerado automaticamente pelo bot de stand-up.

---

## 👤 João Silva (@joao)

Ontem eu fiz:

🟨 Módulo de pagamento;
✅ Testes de integração;

Hoje vou focar em:

* Code review do PR #42;
* Reunião com o designer;

Bloqueios:

Nenhum.

---
[✅=Feito][🟨=Fazendo][🟫=Não trabalhado]
```

**Dica:** Os usuários podem prefixar linhas de tarefas com `✅`, `🟨` ou `🟫` nas respostas por DM e o bot preservará esses emojis na página do ClickUp.

O formato é configurável via comandos `!set-format` sem precisar reiniciar o bot.

---

## Deploy

### Opção A — VPS com PM2 (recomendado)

```bash
# No seu servidor
npm install -g pm2
git clone <seu-repositorio> discord-standup-bot
cd discord-standup-bot
cp .env.example .env && nano .env   # preencha os valores
npm install
pm2 start src/index.js --name standup-bot
pm2 save
pm2 startup   # siga o comando exibido para auto-iniciar no boot
```

### Opção B — Railway (plano gratuito)

1. Faça fork do repositório GitHub.
2. Crie um novo **Projeto** no railway usando o repositório como fonte.
5. Adicione todas as variáveis de ambiente em **Variables → New Variable**.

---

## Testes locais

```bash
# 1. Defina STANDUP_TIME para 2 minutos a partir de agora
#    ex: se forem 14h23, defina:  STANDUP_TIME=25 14 * * *
#    e:                            DEADLINE_MINUTES=2

# 2. Inicie o bot
npm start

# 3. No Discord, envie em um canal do servidor (como dono):
!start-standup

# 4. O bot enviará DM para cada membro com o cargo configurado. Responda as 3 perguntas.

# 5. Verifique as respostas coletadas:
!show-answers

# 6. Envie manualmente ao ClickUp (ou aguarde o cron do prazo):
!post-clickup
```

---

## Extensões possíveis

- **Armazenamento com SQLite:** substitua `src/storage.js` por queries com `better-sqlite3` mantendo a mesma interface (`getEntry`, `setEntry`, `getDayEntries`).
- **Lembrete para quem não respondeu:** adicione um cron na metade do tempo entre `standupTime` e `deadlineCron` que chame `missingUsersReport` e envie DMs de lembrete.
- **Suporte a múltiplos servidores:** parametrize `guildId` / `roleId` por servidor em um mapa de configuração.
