# 💚 Vou Contigo — v2.0.0

**🇧🇷 [Português](README.md) · 🇺🇸 [English](README.en.md) · 🇪🇸 [Español](README.es.md)**

> **When family can’t be there, we are.**

Platform for **Vou Contigo** — a service that accompanies and supports older adults in their daily routines (appointments, tests, grocery shopping, banking, pharmacy visits, outings), sold to their children and relatives who can’t be there.

The system moves the business out of notebooks and scattered WhatsApp chats **without taking WhatsApp away from the client**: v1 organizes scheduling, service delivery, and reports; in **v2.0.0 (“Automation”)**, WhatsApp itself can schedule, cancel, show available time slots, and send reminders automatically, PIX billing switches to Asaas, and family members get a web portal.

## 📖 User guide

Complete guide (landing page + step-by-step instructions): **https://inematds.github.io/voucontigo/guia/en/**

Complete implementation plan (v1, v2 with what’s ✅/⚠️, v3, data model, templates): [PLANO.md](PLANO.md).

---

## What’s included

### v1.0.0 — “Digital notebook”

- **Promotional landing page** (`/`) — mobile-first, using the logo’s color palette, with a fixed WhatsApp CTA and a short form that creates a lead and notifies the manager on Telegram.
- **Dashboard** (`/painel`, Supabase login, manager/companion roles) — schedule, clients and care recipients, packages and hours balance, finances, leads, settings, and metrics with CSV export.
- **Service delivery** — Start/Finish with required capture of wait time in minutes, kilometers driven, parking, tolls, and effort level (1–5). This makes it possible to review pricing after 10–20 appointments.
- **Copyable WhatsApp report** — preformatted text using the brand’s template.
- **Management Telegram bot** — notifications and scheduling/service commands.
- **Reminder cron** — route protected by `CRON_SECRET`.

### v2.0.0 — “Automation” (new)

- **WhatsApp bot** (`lib/whatsapp/fluxo.ts` + `textos.ts`), numeric menu:
  `1` Schedule · `2` Cancel or reschedule · `3` See available time slots · `4` My balance and next visit · `5` Talk to a person.
  Scheduling offers the available slots calculated from the schedule and saves the request through the `solicitar_atendimento` RPC (it starts as `solicitado`, awaiting the manager). Cancellation uses `cancelar_atendimento_familiar`, which applies the policy and provides the fee. Three situations put the conversation in the **`humano`** state: option `5`, free text in the menu, and an unregistered number (which also becomes a lead). In this state, the bot **goes completely silent** — not even “menu” reactivates it — until management sends `/liberar` on Telegram. Within a flow (choosing type, date, or time), an invalid response does not escalate: the bot asks for the number again.
- **WhatsApp webhook** (`app/api/whatsapp/route.ts`) — Evolution API by default; Meta Cloud API optional with `WHATSAPP_PROVIDER=meta`.
- **Automatic client reminders and notifications** (`lib/automacao/**`) — confirmation when the request is approved, **D-1**, **2 hours before**, **report sent automatically** when service is completed, and optional weekly delivery of available time slots (config `enviar_horarios_semanal`). The **low balance** alert goes to management’s Telegram, with the message ready to send.
- **Service radius** — destination geocoding (Nominatim) and comparison with `lat_base`/`lng_base`. This is **informational**: without a base location or geocoder, it responds “radius not verified” and **doesn’t block anything**.
- **PIX billing via Asaas** (`lib/asaas/**`) — billing with QR code and copy-and-paste code, webhook at `/api/asaas/webhook` authenticated by the **`asaas-access-token`** header, and `/api/cron/renovacoes` notifying clients when monthly packages are due. The new package is created only when the webhook confirms payment.
- **Family portal** (`app/(portal)/**`) — `/entrar` via **magic email link**, `/minha-conta` (balance, upcoming visits, history, care recipient details, request and cancel), `/minha-conta/horarios` with “receive by email / by WhatsApp”, `/minha-conta/relatorios/[mes]` as a **printable page** (Print → Save as PDF), and `/minha-conta/calendario.ics` (download with the session open).
- **Telegram v2** (`lib/telegram/comandos-v2.ts`) — inline buttons ✅ approve / ✏️ adjust / ❌ reject and the commands `/solicitacoes`, `/conversas`, `/responder`, `/liberar`, `/resumo`, `/financeiro`.
- **Dashboard v2** — `/painel/inbox` (WhatsApp inbox + leads), `/painel/solicitacoes`, per-client timeline, and Asaas PIX on the finance screen.
- **Transactional email** (`lib/email/cliente.ts`) — Resend via REST, without an SDK. Without `RESEND_API_KEY`, it uses a Fake client that only logs.
- **Database** — `supabase/migrations/0004_v2_schema.sql` (WhatsApp conversations and messages, `webhook_processado`) and `0005_v2_rls_funcoes.sql` (family role via `auth_user_id` and the RPCs `solicitar_atendimento`, `cancelar_atendimento_familiar`, `vincular_familiar`, `horarios_ocupados`).

**Not done yet (v3):** multiple companions, PWA app with check-in, ratings, and assistive AI.

## Stack

| Layer | Choice |
|---|---|
| Web (landing page, dashboard, portal) | Next.js 15 (App Router) + TypeScript + Tailwind 4 |
| Database, auth, storage | Supabase (Postgres + RLS + Auth) |
| WhatsApp | **Evolution API** self-hosted (regular number via QR) · Meta Cloud API optional |
| Management bot | grammY in a Next.js route (webhook) |
| Billing | Asaas (PIX, webhook) |
| Email | Resend (REST) |
| Scheduled jobs | Vercel Cron (Hobby: daily only) or system cron on the VPS |
| Validation | Zod · tests with Vitest |
| Hosting | VPS with Docker + Caddy, or Vercel — always via `git push` |

## Running locally

Prerequisites: Node 20+, npm, and Docker (for local Supabase). The v1 workflow hasn’t changed.

```bash
git clone https://github.com/inematds/voucontigo.git
cd voucontigo
npm install

# 1. environment variables
cp .env.example .env.local

# 2. local Supabase (prints the URL and keys — copy them to .env.local)
npx supabase start

# 3. apply migrations from supabase/migrations (0001 → 0005)
npx supabase db reset

# 4. start the app
npm run dev        # http://localhost:3000 · dashboard at /painel · portal at /minha-conta
```

Local Supabase Studio: `http://localhost:54323` — create the **manager user** there (Authentication → Add user). A trigger creates the `perfil` with the default `acompanhante` role; promote the first account to `gestora` in the SQL Editor:

```sql
update public.perfil set papel = 'gestora'
  where id = (select id from auth.users where email = 'voce@exemplo.com');
```

The app runs without Evolution, Asaas, or Resend: WhatsApp and email use Fake clients that only log messages.

Other scripts: `npm run build`, `npm run lint`, `npm run typecheck`, `npm test`.

### Environment variables

All are listed in [`.env.example`](.env.example).

**v1 (unchanged)**

| Variable | Where it comes from |
|---|---|
| `NEXT_PUBLIC_SUPABASE_URL` / `NEXT_PUBLIC_SUPABASE_ANON_KEY` | Supabase → Settings → API (or the output of `npx supabase start`) |
| `SUPABASE_SERVICE_ROLE_KEY` | Supabase → Settings → API — **server only** |
| `TELEGRAM_BOT_TOKEN` | `@BotFather` |
| `TELEGRAM_WEBHOOK_SECRET` | A long string you make up; validates Telegram requests |
| `TELEGRAM_CHAT_GESTAO` | ID of the private management group |
| `CRON_SECRET` | Secret sent by cron as `Authorization: Bearer ...` |
| `NEXT_PUBLIC_SITE_URL` | Public site URL |
| `NEXT_PUBLIC_WHATSAPP_EMPRESA` | Business WhatsApp number, format `55DDDNÚMERO` |
| `TZ` | **`America/Sao_Paulo`** — process timezone; also set it in the project’s Vercel environment variables |

**v2 (new)**

| Variable | Where it comes from |
|---|---|
| `WHATSAPP_PROVIDER` | `evolution` (default) or `meta` |
| `EVOLUTION_API_URL` | `http://evolution:8080` when the app and Evolution share the Docker network |
| `EVOLUTION_API_KEY` | The **same** string as Evolution’s `AUTHENTICATION_API_KEY` |
| `EVOLUTION_INSTANCE` | Name of the instance created in the manager (`voucontigo`) |
| `WHATSAPP_WEBHOOK_TOKEN` | A secret you make up; Evolution doesn’t sign the body, so it’s sent in the query string |
| `WHATSAPP_TOKEN` / `WHATSAPP_PHONE_NUMBER_ID` / `WHATSAPP_VERIFY_TOKEN` / `WHATSAPP_APP_SECRET` | Only with `WHATSAPP_PROVIDER=meta` |
| `RESEND_API_KEY` / `EMAIL_FROM` | Resend → API Keys; the sender domain must be verified |
| `ASAAS_API_KEY` / `ASAAS_WEBHOOK_TOKEN` / `ASAAS_BASE_URL` | Asaas → Integrations; start with `https://sandbox.asaas.com/api/v3` |
| `GEOCODER_PROVIDER` / `GEOCODING_URL` / `GEOCODER_EMAIL` | OpenStreetMap Nominatim; its usage policy **requires** a contact email |

Empty = disabled, not an error: without `ASAAS_API_KEY` there’s no automatic PIX, without `RESEND_API_KEY` there’s no email, and without a geocoder the radius is never verified.

## Running Evolution API on the VPS

Evolution runs alongside the app on the same Docker network (`inemadlp_default`), and it communicates with WhatsApp using a **regular number scanned via QR** — no Meta verification or approved templates.

**1. Configure and start**

```bash
cp .env.evolution.example .env.evolution
# edit: EVOLUTION_SERVER_URL, EVOLUTION_API_KEY (long string), EVOLUTION_DB_PASSWORD

docker compose -p voucontigo -f docker-compose.evolution.yml --env-file .env.evolution up -d
```

Always use `-p voucontigo` (or run it from a directory called `voucontigo`): `deploy/Caddyfile.snippet` proxies to the `voucontigo-app-1` and `voucontigo-evolution-1` containers, whose names depend on this project prefix. The image is pinned to `atendai/evolution-api:v2.2.3`; Evolution’s Postgres is its own and is unrelated to Supabase.

**2. Create the instance and scan the QR**

Open `https://evolution.inema.club/manager` and sign in with the `EVOLUTION_API_KEY`. Create an instance named **`voucontigo`** (the same value as `EVOLUTION_INSTANCE`), request the QR Code, and scan it with the **Vou Contigo number’s phone** (WhatsApp → Linked devices). The session is stored in the `evolution_instances` volume and survives restarts.

**3. Configure the instance webhook**

`WEBHOOK_GLOBAL_ENABLED` is `false` on purpose: the webhook is registered **on the instance**, in the manager.

| Field | Value |
|---|---|
| URL | `https://voucontigo.inema.club/api/whatsapp?token=<WHATSAPP_WEBHOOK_TOKEN>` |
| Events | `MESSAGES_UPSERT` and `MESSAGES_UPDATE` |
| `webhook_by_events` | **false** (all events use the same URL) |
| `webhook_base64` | **false** (we don’t want media in base64) |

The same `WHATSAPP_WEBHOOK_TOKEN` must be in the app’s `.env`: the route compares it in constant time and returns 401 if it doesn’t match. Group messages, messages sent by you, and messages without text are ignored with 200; `wa_message_id` ensures idempotency.

**4. Point the app to it**

```env
WHATSAPP_PROVIDER=evolution
EVOLUTION_API_URL=http://evolution:8080   # service name on the shared Docker network
EVOLUTION_API_KEY=<the same AUTHENTICATION_API_KEY>
EVOLUTION_INSTANCE=voucontigo
WHATSAPP_WEBHOOK_TOKEN=<the same as the webhook URL>
```

## Asaas (PIX billing)

1. Create the account and start in **sandbox** (`https://sandbox.asaas.com`): generate the API key and set it as `ASAAS_API_KEY`, with `ASAAS_BASE_URL=https://sandbox.asaas.com/api/v3`.
2. Register the webhook at `https://voucontigo.inema.club/api/asaas/webhook`, with an **authentication token** — Asaas sends it in the **`asaas-access-token`** header, and the route compares it with `ASAAS_WEBHOOK_TOKEN` (401 if it doesn’t match).
3. Events that matter: `PAYMENT_RECEIVED` / `PAYMENT_CONFIRMED` (marks as paid) and `PAYMENT_DELETED` / `PAYMENT_REFUNDED` (reverses the payment). The route always responds with 200 when authenticated — Asaas blocks the queue for any non-2xx response — and deduplicates using `webhook_processado`.
4. Switch to production only after completing an end-to-end charge in the sandbox: set `ASAAS_BASE_URL=https://api.asaas.com/v3` and use the production key (the `asaas_ambiente` config records which environment is in use).

## Resend (optional email)

Create the API key in Resend, verify the domain used in `EMAIL_FROM`, and fill in both variables. Without `RESEND_API_KEY`, the system keeps working: email uses a Fake client, and everything is sent only by WhatsApp.

## Supabase Auth — Redirect URLs

The project has **two** magic link logins. In **Authentication → URL Configuration → Redirect URLs**, register both, or the email link will return an error:

```
${NEXT_PUBLIC_SITE_URL}/entrar/callback     # family portal
${NEXT_PUBLIC_SITE_URL}/login/callback      # team dashboard
```

Family members can sign in only if their **email (or WhatsApp number) is already registered in the client’s record in the dashboard** — that’s how the `vincular_familiar` RPC links the account to the client. If there’s no match, login goes to `/entrar/sem-cadastro`. Anyone who isn’t a family member gets the `acompanhante` role by default; the manager is promoted via SQL.

## Deploy

Publishing is **`git push`** — the rest is a webhook.

```bash
git add -A
git commit -m "feat: ..."
git push
```

### VPS (recommended — where Evolution runs)

```bash
docker compose -p voucontigo -f docker-compose.vps.yml --env-file .env up -d --build
```

- The host’s Caddy handles TLS and proxying: add `deploy/Caddyfile.snippet` to the Caddyfile (`voucontigo.inema.club` → `voucontigo-app-1:3000`, `evolution.inema.club` → `voucontigo-evolution-1:8080`).
- The `NEXT_PUBLIC_*` variables are passed as **build args** (embedded in the bundle); the rest come from `.env` at runtime.
- Jobs become system cron jobs: copy `deploy/vps-cron.example` to `/etc/cron.d/voucontigo`, replace `CRON_SECRET` and the domain. It uses `CRON_TZ=America/Sao_Paulo` and runs reminders at `07:00`, renewals at `06:30`, and the 2-hour reminder **every hour**.

### Vercel

`vercel.json` already includes the three crons — in **UTC**, equivalent to the VPS times: `0 10 * * *` (07:00 BRT, D-1 reminders + summary), `30 9 * * *` (06:30 BRT, renewals), and `0 * * * *` (2-hour reminder).

> ⚠️ **The hourly cron requires the Pro plan.** On Hobby, Vercel runs crons only once a day and with no guaranteed schedule, so the “2 hours before” reminder doesn’t work properly. This isn’t a problem on the VPS: it uses system cron.

Before the first deploy, register the variables (including `TZ=America/Sao_Paulo`) in the project, and run `npx tsx scripts/telegram-set-webhook.ts` with the final URL.

## Telegram commands

**v1 — scheduling and service delivery**

| Command | What it does |
|---|---|
| `/start`, `/ajuda` | Lists the available commands |
| `/hoje` | Today’s schedule, with IDs |
| `/amanha` | Tomorrow’s schedule |
| `/semana` | The next seven days |
| `/agendar` | Step-by-step scheduling assistant |
| `/cancelar <id>` | Cancels and records the reason, applying the policy |
| `/iniciar <id>` | Check-in: records the actual start time |
| `/finalizar <id>` | Check-out: asks for wait time, kilometers, parking, tolls, and effort level 1–5 |
| `/relatorio <id>` | Creates the report using the brand’s template |
| `/saldo <cliente>` | Contracted, used, and remaining hours |
| `/lead` | New leads from the landing page, with a WhatsApp link |

**v2 — requests and conversations**

| Command | What it does |
|---|---|
| `/solicitacoes` | Pending WhatsApp requests, with buttons ✅ approve · ✏️ adjust · ❌ reject |
| `/conversas` | Who is waiting for human assistance |
| `/responder <whatsapp> <texto>` | Replies to the person on WhatsApp without leaving Telegram |
| `/liberar <whatsapp>` | Returns the conversation to the bot (removes the `humano` state) |
| `/resumo` | Today’s operational summary |
| `/financeiro` | Financial summary for the past week |

In addition to commands, the bot automatically notifies management about new leads, created or canceled appointments, new WhatsApp requests, requests for human assistance, reports pending for more than 2 hours, and packages with low balances. The 7 a.m. cron sends the day’s summary; on Mondays, it includes the financial summary for the completed week.

## Service limitations

**No medical care.** Vou Contigo doesn’t administer medication, provide nursing care, or perform procedures. That’s why the system has no medication fields or medical records — only free-text “declared restrictions,” with the minimum necessary information.

**LGPD.** Older adults’ data is sensitive: minimal collection, consent recorded during registration, photos only with explicit authorization, deletion upon request, and isolation through Supabase RLS. Family members can see only their own client; the team sees only what their role allows.

**Risk of WhatsApp number being blocked.** Evolution API uses a **regular number** connected via QR, not Meta’s official API. It’s inexpensive and has little red tape, but Meta may block the number if there’s spam-like behavior. Usage rules:

- reply only to conversations initiated by the client;
- send **only messages about real appointments** (confirmation, reminder, report, billing for contracted services);
- no mass messages, broadcast lists, promotions, or marketing;
- keep a backup number and, if volume grows, migrate to Meta Cloud API (`WHATSAPP_PROVIDER=meta`, already supported in the code).

## Roadmap — v3.0.0 “Network”

- Multiple companions: registration, documents, service areas, availability, and payment distribution.
- Matching by service area, availability, and history with the same care recipient.
- Companion PWA app with check-in/check-out and guided reporting.
- Family ratings after service and alerts for declining quality.
- Assistive AI: draft/voice message becomes a report, history summary before the visit, route suggestions.
- Complete financial management (payment distribution, overdue payments, invoices) and referral program.

Before that, the known v2 gaps (§7.2 of PLANO): WhatsApp OTP for the portal, `.ics` via signed URL, “I’m on my way” notification at check-in, and report PDF generated on the server.

---

Documents: [PLANO.md](PLANO.md) · [docs/conversa-original.md](docs/conversa-original.md)
