# ðŸ’š Vou Contigo â€” v2.0.0

**ðŸ‡§ðŸ‡· [PortuguÃªs](README.md) Â· ðŸ‡ºðŸ‡¸ [English](README.en.md) Â· ðŸ‡ªðŸ‡¸ [EspaÃ±ol](README.es.md)**

> **Quando a famÃ­lia nÃ£o pode estar presente, nÃ³s estamos.**

Plataforma do **Vou Contigo** â€” serviÃ§o de acompanhamento e apoio Ã  rotina de pessoas mais velhas (consultas, exames, mercado, banco, farmÃ¡cia, passeios), vendido para os filhos e familiares que nÃ£o conseguem estar presentes.

O sistema tira o negÃ³cio do caderno e do WhatsApp solto **sem tirar o WhatsApp do cliente**: na v1 organiza agenda, execuÃ§Ã£o e relatÃ³rio; na **v2.0.0 ("AutomaÃ§Ã£o")** o prÃ³prio WhatsApp passa a agendar, cancelar, mostrar horÃ¡rios livres e mandar lembretes sozinho, a cobranÃ§a PIX vira Asaas e o familiar ganha um portal web.

## ðŸ“– Guia de uso

Guia completo (landing + passo a passo): **https://inematds.github.io/voucontigo/guia/**

Plano de implementaÃ§Ã£o completo (v1, v2 com o que ficou âœ…/âš ï¸, v3, modelo de dados, templates): [PLANO.md](PLANO.md).

---

## O que tem

### v1.0.0 â€” "Caderno digital"

- **Landing promocional** (`/`) â€” mobile-first, na paleta do logo, com CTA fixo de WhatsApp e formulÃ¡rio curto que cria um lead e notifica a gestora no Telegram.
- **Painel** (`/painel`, login Supabase, papÃ©is gestora/acompanhante) â€” agenda, clientes e acompanhados, pacotes e saldo de horas, financeiro, leads, configuraÃ§Ãµes e mÃ©tricas com exportaÃ§Ã£o em CSV.
- **ExecuÃ§Ã£o do atendimento** â€” Iniciar/Finalizar com captura obrigatÃ³ria de minutos de espera, km rodados, estacionamento, pedÃ¡gio e nÃ­vel de esforÃ§o (1â€“5). Ã‰ isso que permite revisar o preÃ§o apÃ³s 10â€“20 atendimentos.
- **RelatÃ³rio de WhatsApp copiÃ¡vel** â€” texto prÃ©-montado no modelo da marca.
- **Bot de Telegram da gestÃ£o** â€” notificaÃ§Ãµes e comandos de agenda/execuÃ§Ã£o.
- **Cron de lembretes** â€” rota protegida por `CRON_SECRET`.

### v2.0.0 â€” "AutomaÃ§Ã£o" (novo)

- **Bot de WhatsApp** (`lib/whatsapp/fluxo.ts` + `textos.ts`), menu numÃ©rico:
  `1` Agendar Â· `2` Cancelar ou remarcar Â· `3` Ver horÃ¡rios livres Â· `4` Meu saldo e prÃ³xima visita Â· `5` Falar com uma pessoa.
  Agendar oferece os slots livres calculados da agenda e grava pela RPC `solicitar_atendimento` (nasce `solicitado`, Ã  espera da gestora). Cancelar usa `cancelar_atendimento_familiar`, que aplica a polÃ­tica e informa a taxa. TrÃªs situaÃ§Ãµes pÃµem a conversa no estado **`humano`**: a opÃ§Ã£o `5`, texto livre no menu e nÃºmero sem cadastro (que tambÃ©m vira lead). Nesse estado o bot **silencia por completo** â€” nem "menu" reativa â€” atÃ© a gestÃ£o mandar `/liberar` no Telegram. JÃ¡ dentro de um fluxo (escolher tipo, data, horÃ¡rio) a resposta invÃ¡lida nÃ£o escala: ele pede o nÃºmero de novo.
- **Webhook de WhatsApp** (`app/api/whatsapp/route.ts`) â€” Evolution API por padrÃ£o; Meta Cloud API opcional com `WHATSAPP_PROVIDER=meta`.
- **Lembretes e avisos automÃ¡ticos ao cliente** (`lib/automacao/**`) â€” confirmaÃ§Ã£o ao aprovar a solicitaÃ§Ã£o, **D-1**, **2h antes**, **relatÃ³rio enviado sozinho** ao finalizar e envio semanal opcional dos horÃ¡rios livres (config `enviar_horarios_semanal`). O aviso de **saldo baixo** vai para o Telegram da gestÃ£o, com o texto pronto para ela mandar.
- **Raio de atendimento** â€” geocodificaÃ§Ã£o do destino (Nominatim) e comparaÃ§Ã£o com `lat_base`/`lng_base`. Ã‰ **informativo**: sem base ou sem geocoder, responde "raio nÃ£o verificado" e **nÃ£o bloqueia nada**.
- **CobranÃ§a PIX via Asaas** (`lib/asaas/**`) â€” cobranÃ§a com QR e copia-e-cola, webhook em `/api/asaas/webhook` autenticado pelo header **`asaas-access-token`**, e `/api/cron/renovacoes` avisando o vencimento dos pacotes mensais. O pacote novo sÃ³ nasce quando o webhook confirma o pagamento.
- **Portal do familiar** (`app/(portal)/**`) â€” `/entrar` por **link mÃ¡gico por e-mail**, `/minha-conta` (saldo, prÃ³ximas visitas, histÃ³rico, dados do acompanhado, solicitar e cancelar), `/minha-conta/horarios` com "receber por e-mail / por WhatsApp", `/minha-conta/relatorios/[mes]` em **pÃ¡gina imprimÃ­vel** (Imprimir â†’ Salvar como PDF) e `/minha-conta/calendario.ics` (download com a sessÃ£o aberta).
- **Telegram v2** (`lib/telegram/comandos-v2.ts`) â€” botÃµes inline âœ… aprovar / âœï¸ ajustar / âŒ recusar e os comandos `/solicitacoes`, `/conversas`, `/responder`, `/liberar`, `/resumo`, `/financeiro`.
- **Painel v2** â€” `/painel/inbox` (caixa de entrada WhatsApp + leads), `/painel/solicitacoes`, timeline por cliente e PIX do Asaas na tela de financeiro.
- **E-mail transacional** (`lib/email/cliente.ts`) â€” Resend via REST, sem SDK. Sem `RESEND_API_KEY` vira um cliente Fake que sÃ³ registra.
- **Banco** â€” `supabase/migrations/0004_v2_schema.sql` (conversas e mensagens de WhatsApp, `webhook_processado`) e `0005_v2_rls_funcoes.sql` (papel familiar por `auth_user_id` e as RPCs `solicitar_atendimento`, `cancelar_atendimento_familiar`, `vincular_familiar`, `horarios_ocupados`).

**Ainda nÃ£o feito (v3):** mÃºltiplas acompanhantes, app PWA com check-in, avaliaÃ§Ãµes e IA assistiva.

## Stack

| Camada | Escolha |
|---|---|
| Web (landing, painel, portal) | Next.js 15 (App Router) + TypeScript + Tailwind 4 |
| Banco, auth, storage | Supabase (Postgres + RLS + Auth) |
| WhatsApp | **Evolution API** self-hosted (nÃºmero comum via QR) Â· Meta Cloud API opcional |
| Bot de gestÃ£o | grammY em rota Next.js (webhook) |
| CobranÃ§a | Asaas (PIX, webhook) |
| E-mail | Resend (REST) |
| Jobs agendados | Vercel Cron (Hobby: sÃ³ diÃ¡rio) ou cron do sistema na VPS |
| ValidaÃ§Ã£o | Zod Â· testes com Vitest |
| Hospedagem | VPS com Docker + Caddy, ou Vercel â€” sempre por `git push` |

## Como rodar local

PrÃ©-requisitos: Node 20+, npm e Docker (para o Supabase local). O fluxo da v1 nÃ£o mudou.

```bash
git clone https://github.com/inematds/voucontigo.git
cd voucontigo
npm install

# 1. variÃ¡veis de ambiente
cp .env.example .env.local

# 2. Supabase local (imprime a URL e as chaves â€” copie para o .env.local)
npx supabase start

# 3. aplica as migraÃ§Ãµes de supabase/migrations (0001 â†’ 0005)
npx supabase db reset

# 4. sobe o app
npm run dev        # http://localhost:3000 Â· painel em /painel Â· portal em /minha-conta
```

Studio local do Supabase: `http://localhost:54323` â€” Ã© lÃ¡ que se cria a **usuÃ¡ria gestora** (Authentication â†’ Add user). Um gatilho cria o `perfil` com papel padrÃ£o `acompanhante`; promova a primeira conta a `gestora` pelo SQL Editor:

```sql
update public.perfil set papel = 'gestora'
  where id = (select id from auth.users where email = 'voce@exemplo.com');
```

Sem Evolution, sem Asaas e sem Resend o app roda: o WhatsApp e o e-mail caem em clientes Fake que apenas registram o envio.

Outros scripts: `npm run build`, `npm run lint`, `npm run typecheck`, `npm test`.

### VariÃ¡veis de ambiente

Todas listadas em [`.env.example`](.env.example).

**v1 (inalteradas)**

| VariÃ¡vel | De onde vem |
|---|---|
| `NEXT_PUBLIC_SUPABASE_URL` / `NEXT_PUBLIC_SUPABASE_ANON_KEY` | Supabase â†’ Settings â†’ API (ou a saÃ­da do `npx supabase start`) |
| `SUPABASE_SERVICE_ROLE_KEY` | Supabase â†’ Settings â†’ API â€” **sÃ³ no servidor** |
| `TELEGRAM_BOT_TOKEN` | `@BotFather` |
| `TELEGRAM_WEBHOOK_SECRET` | string longa inventada por vocÃª; valida as chamadas do Telegram |
| `TELEGRAM_CHAT_GESTAO` | id do grupo privado da gestÃ£o |
| `CRON_SECRET` | segredo enviado pelo cron como `Authorization: Bearer ...` |
| `NEXT_PUBLIC_SITE_URL` | URL pÃºblica do site |
| `NEXT_PUBLIC_WHATSAPP_EMPRESA` | nÃºmero do WhatsApp Business, formato `55DDDNÃšMERO` |
| `TZ` | **`America/Sao_Paulo`** â€” fuso do processo; defina tambÃ©m nas env vars do projeto na Vercel |

**v2 (novas)**

| VariÃ¡vel | De onde vem |
|---|---|
| `WHATSAPP_PROVIDER` | `evolution` (padrÃ£o) ou `meta` |
| `EVOLUTION_API_URL` | `http://evolution:8080` quando o app e a Evolution dividem a rede Docker |
| `EVOLUTION_API_KEY` | a **mesma** string de `AUTHENTICATION_API_KEY` da Evolution |
| `EVOLUTION_INSTANCE` | nome da instÃ¢ncia criada no manager (`voucontigo`) |
| `WHATSAPP_WEBHOOK_TOKEN` | segredo que vocÃª inventa; a Evolution nÃ£o assina o corpo, entÃ£o ele viaja na query string |
| `WHATSAPP_TOKEN` / `WHATSAPP_PHONE_NUMBER_ID` / `WHATSAPP_VERIFY_TOKEN` / `WHATSAPP_APP_SECRET` | sÃ³ com `WHATSAPP_PROVIDER=meta` |
| `RESEND_API_KEY` / `EMAIL_FROM` | Resend â†’ API Keys; o domÃ­nio do remetente precisa estar verificado |
| `ASAAS_API_KEY` / `ASAAS_WEBHOOK_TOKEN` / `ASAAS_BASE_URL` | Asaas â†’ IntegraÃ§Ãµes; comece em `https://sandbox.asaas.com/api/v3` |
| `GEOCODER_PROVIDER` / `GEOCODING_URL` / `GEOCODER_EMAIL` | Nominatim do OpenStreetMap; a polÃ­tica de uso **exige** um e-mail de contato |

Vazio = desligado, nÃ£o = erro: sem `ASAAS_API_KEY` nÃ£o hÃ¡ PIX automÃ¡tico, sem `RESEND_API_KEY` nÃ£o hÃ¡ e-mail, sem geocoder o raio nunca Ã© verificado.

## Subir a Evolution API na VPS

A Evolution roda ao lado do app, na mesma rede Docker (`inemadlp_default`), e Ã© ela que conversa com o WhatsApp por um **nÃºmero comum lido via QR** â€” sem verificaÃ§Ã£o Meta e sem templates aprovados.

**1. Configurar e subir**

```bash
cp .env.evolution.example .env.evolution
# edite: EVOLUTION_SERVER_URL, EVOLUTION_API_KEY (string longa), EVOLUTION_DB_PASSWORD

docker compose -p voucontigo -f docker-compose.evolution.yml --env-file .env.evolution up -d
```

Use sempre `-p voucontigo` (ou rode de um diretÃ³rio chamado `voucontigo`): o `deploy/Caddyfile.snippet` faz proxy para os containers `voucontigo-app-1` e `voucontigo-evolution-1`, nomes que dependem desse prefixo de projeto. A imagem estÃ¡ fixada em `atendai/evolution-api:v2.2.3`; o Postgres da Evolution Ã© prÃ³prio e nÃ£o tem relaÃ§Ã£o com o Supabase.

**2. Criar a instÃ¢ncia e ler o QR**

Abra `https://evolution.inema.club/manager` e entre com a `EVOLUTION_API_KEY`. Crie a instÃ¢ncia com o nome **`voucontigo`** (o mesmo valor de `EVOLUTION_INSTANCE`), peÃ§a o QR Code e leia com o **celular do nÃºmero do Vou Contigo** (WhatsApp â†’ Aparelhos conectados). A sessÃ£o fica no volume `evolution_instances` e sobrevive a reinÃ­cios.

**3. Configurar o webhook da instÃ¢ncia**

`WEBHOOK_GLOBAL_ENABLED` estÃ¡ `false` de propÃ³sito: o webhook Ã© cadastrado **na instÃ¢ncia**, no manager.

| Campo | Valor |
|---|---|
| URL | `https://voucontigo.inema.club/api/whatsapp?token=<WHATSAPP_WEBHOOK_TOKEN>` |
| Eventos | `MESSAGES_UPSERT` e `MESSAGES_UPDATE` |
| `webhook_by_events` | **false** (todos os eventos na mesma URL) |
| `webhook_base64` | **false** (nÃ£o queremos mÃ­dia em base64) |

O mesmo `WHATSAPP_WEBHOOK_TOKEN` tem de estar no `.env` do app: a rota compara em tempo constante e devolve 401 se nÃ£o bater. Mensagens de grupo, enviadas por vocÃª mesma e sem texto sÃ£o ignoradas com 200; `wa_message_id` garante idempotÃªncia.

**4. Apontar o app para ela**

```env
WHATSAPP_PROVIDER=evolution
EVOLUTION_API_URL=http://evolution:8080   # nome do serviÃ§o na rede Docker compartilhada
EVOLUTION_API_KEY=<a mesma AUTHENTICATION_API_KEY>
EVOLUTION_INSTANCE=voucontigo
WHATSAPP_WEBHOOK_TOKEN=<o mesmo da URL do webhook>
```

## Asaas (cobranÃ§a PIX)

1. Crie a conta e comece em **sandbox** (`https://sandbox.asaas.com`): gere a API key e ponha em `ASAAS_API_KEY`, com `ASAAS_BASE_URL=https://sandbox.asaas.com/api/v3`.
2. Cadastre o webhook apontando para `https://voucontigo.inema.club/api/asaas/webhook`, com um **token de autenticaÃ§Ã£o** â€” o Asaas o envia no header **`asaas-access-token`**, e a rota compara com `ASAAS_WEBHOOK_TOKEN` (401 se nÃ£o bater).
3. Eventos que importam: `PAYMENT_RECEIVED` / `PAYMENT_CONFIRMED` (marca pago) e `PAYMENT_DELETED` / `PAYMENT_REFUNDED` (desfaz). A rota responde 200 sempre que autenticada â€” o Asaas trava a fila em qualquer resposta nÃ£o-2xx â€” e deduplica por `webhook_processado`.
4. SÃ³ depois de uma cobranÃ§a de ponta a ponta no sandbox troque para produÃ§Ã£o: `ASAAS_BASE_URL=https://api.asaas.com/v3` e a chave de produÃ§Ã£o (a config `asaas_ambiente` registra qual ambiente estÃ¡ em uso).

## Resend (e-mail, opcional)

Crie a API key em Resend, verifique o domÃ­nio usado em `EMAIL_FROM` e preencha as duas variÃ¡veis. Sem `RESEND_API_KEY` o sistema segue funcionando: o e-mail vira um cliente Fake e tudo sai sÃ³ por WhatsApp.

## Supabase Auth â€” Redirect URLs

O projeto tem **dois** logins por link mÃ¡gico. Em **Authentication â†’ URL Configuration â†’ Redirect URLs**, cadastre os dois, ou o link do e-mail volta com erro:

```
${NEXT_PUBLIC_SITE_URL}/entrar/callback     # portal do familiar
${NEXT_PUBLIC_SITE_URL}/login/callback      # painel da equipe
```

O familiar sÃ³ entra se o **e-mail (ou o WhatsApp) jÃ¡ estiver cadastrado na ficha do cliente no painel** â€” Ã© assim que a RPC `vincular_familiar` liga a conta ao cliente. Sem casar, o login cai em `/entrar/sem-cadastro`. Quem nÃ£o Ã© familiar vira `acompanhante` por padrÃ£o; a gestora Ã© promovida por SQL.

## Deploy

Publicar Ã© **`git push`** â€” o resto Ã© webhook.

```bash
git add -A
git commit -m "feat: ..."
git push
```

### VPS (recomendado â€” Ã© onde a Evolution vive)

```bash
docker compose -p voucontigo -f docker-compose.vps.yml --env-file .env up -d --build
```

- O Caddy do host faz TLS e proxy: acrescente `deploy/Caddyfile.snippet` ao Caddyfile (`voucontigo.inema.club` â†’ `voucontigo-app-1:3000`, `evolution.inema.club` â†’ `voucontigo-evolution-1:8080`).
- As variÃ¡veis `NEXT_PUBLIC_*` entram como **build args** (ficam embutidas no bundle), as demais vÃªm do `.env` em runtime.
- Os jobs viram cron do sistema: copie `deploy/vps-cron.example` para `/etc/cron.d/voucontigo`, troque o `CRON_SECRET` e o domÃ­nio. Ele usa `CRON_TZ=America/Sao_Paulo` e roda `07:00` lembretes, `06:30` renovaÃ§Ãµes e **de hora em hora** o lembrete de 2h.

### Vercel

`vercel.json` jÃ¡ traz os trÃªs crons â€” em **UTC**, equivalentes aos horÃ¡rios da VPS: `0 10 * * *` (07:00 BRT, lembretes D-1 + resumo), `30 9 * * *` (06:30 BRT, renovaÃ§Ãµes) e `0 * * * *` (lembrete de 2h).

> âš ï¸ **O cron de hora em hora exige plano Pro.** No Hobby a Vercel sÃ³ executa crons uma vez por dia e sem horÃ¡rio garantido, entÃ£o o lembrete "2h antes" nÃ£o funciona direito. Na VPS isso nÃ£o Ã© problema: o cron Ã© do sistema.

Antes do primeiro deploy, cadastre as variÃ¡veis (incluindo `TZ=America/Sao_Paulo`) no projeto, e rode `npx tsx scripts/telegram-set-webhook.ts` jÃ¡ com a URL final.

## Comandos do Telegram

**v1 â€” agenda e execuÃ§Ã£o**

| Comando | O que faz |
|---|---|
| `/start`, `/ajuda` | Lista os comandos disponÃ­veis |
| `/hoje` | Agenda de hoje, com ids |
| `/amanha` | Agenda de amanhÃ£ |
| `/semana` | PrÃ³ximos sete dias |
| `/agendar` | Assistente passo a passo de agendamento |
| `/cancelar <id>` | Cancela e registra o motivo, aplicando a polÃ­tica |
| `/iniciar <id>` | Check-in: grava a hora real de inÃ­cio |
| `/finalizar <id>` | Check-out: pede espera, km, estacionamento, pedÃ¡gio e esforÃ§o 1â€“5 |
| `/relatorio <id>` | Monta o relatÃ³rio no modelo da marca |
| `/saldo <cliente>` | Horas contratadas, usadas e restantes |
| `/lead` | Leads novos da landing, com link de WhatsApp |

**v2 â€” solicitaÃ§Ãµes e conversas**

| Comando | O que faz |
|---|---|
| `/solicitacoes` | SolicitaÃ§Ãµes pendentes do WhatsApp, com botÃµes âœ… aprovar Â· âœï¸ ajustar Â· âŒ recusar |
| `/conversas` | Quem estÃ¡ aguardando atendimento humano |
| `/responder <whatsapp> <texto>` | Responde a pessoa pelo WhatsApp sem sair do Telegram |
| `/liberar <whatsapp>` | Devolve a conversa ao bot (tira do estado `humano`) |
| `/resumo` | Resumo operacional de hoje |
| `/financeiro` | Resumo financeiro da Ãºltima semana |

AlÃ©m dos comandos, o bot avisa sozinho: novo lead, atendimento criado ou cancelado, nova solicitaÃ§Ã£o do WhatsApp, pedido de atendimento humano, relatÃ³rio pendente hÃ¡ mais de 2h e pacote com saldo baixo. O cron das 7h manda o resumo do dia; Ã s segundas ele inclui o resumo financeiro da semana fechada.

## Limites do serviÃ§o

**Sem cuidados mÃ©dicos.** O Vou Contigo nÃ£o aplica medicaÃ§Ã£o, nÃ£o faz enfermagem nem procedimentos. Por isso o sistema nÃ£o tem campos de medicaÃ§Ã£o nem prontuÃ¡rio â€” apenas "restriÃ§Ãµes declaradas" em texto livre, com o mÃ­nimo necessÃ¡rio.

**LGPD.** Dados de pessoas idosas sÃ£o sensÃ­veis: coleta mÃ­nima, consentimento registrado no cadastro, fotos sÃ³ com autorizaÃ§Ã£o explÃ­cita, exclusÃ£o a pedido e isolamento por RLS no Supabase. O familiar enxerga apenas o prÃ³prio cliente; a equipe, sÃ³ o que o papel permite.

**Risco de bloqueio do nÃºmero no WhatsApp.** A Evolution API usa um **nÃºmero comum** conectado por QR, nÃ£o a API oficial da Meta. Ã‰ barato e sem burocracia, mas a Meta pode bloquear o nÃºmero se houver comportamento de spam. Regras de uso, portanto:

- responder apenas conversas iniciadas pelo cliente;
- enviar **sÃ³ mensagens sobre compromissos reais** (confirmaÃ§Ã£o, lembrete, relatÃ³rio, cobranÃ§a do que foi contratado);
- nada de disparo em massa, lista de transmissÃ£o, promoÃ§Ã£o ou marketing;
- manter um nÃºmero de reserva e, se o volume crescer, migrar para a Meta Cloud API (`WHATSAPP_PROVIDER=meta`, jÃ¡ suportado no cÃ³digo).

## Roadmap â€” v3.0.0 "Rede"

- MÃºltiplas acompanhantes: cadastro, documentos, regiÃµes, disponibilidade e repasse.
- Matching por regiÃ£o, disponibilidade e histÃ³rico com o mesmo acompanhado.
- App PWA da acompanhante com check-in/check-out e relatÃ³rio guiado.
- AvaliaÃ§Ã£o da famÃ­lia pÃ³s-atendimento e alerta de queda de qualidade.
- IA assistiva: rascunho/Ã¡udio vira relatÃ³rio, resumo do histÃ³rico antes da visita, sugestÃ£o de rota.
- Financeiro completo (repasse, inadimplÃªncia, nota fiscal) e programa de indicaÃ§Ã£o.

Antes disso, as lacunas conhecidas da v2 (Â§7.2 do PLANO): OTP por WhatsApp no portal, `.ics` por URL assinada, aviso "estou a caminho" no check-in e PDF do relatÃ³rio gerado no servidor.

---

Documentos: [PLANO.md](PLANO.md) Â· [docs/conversa-original.md](docs/conversa-original.md)

---

Projeto adaptado da comunidade INEMA TDS - Marca Arckhe