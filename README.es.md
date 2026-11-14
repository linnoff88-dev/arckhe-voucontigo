# 💚 Vou Contigo — v2.0.0

**🇧🇷 [Português](README.md) · 🇺🇸 [English](README.en.md) · 🇪🇸 [Español](README.es.md)**

> **Cuando la familia no puede estar presente, nosotros estamos.**

Plataforma de **Vou Contigo** — servicio de acompañamiento y apoyo en la rutina de personas mayores (consultas, exámenes, mercado, banco, farmacia, paseos), ofrecido a hijos y familiares que no pueden estar presentes.

El sistema saca el negocio de la libreta y del WhatsApp desorganizado **sin quitarle WhatsApp al cliente**: en la v1 organiza agenda, ejecución e informes; en la **v2.0.0 ("Automatización")**, el propio WhatsApp empieza a agendar, cancelar, mostrar horarios disponibles y enviar recordatorios automáticamente, el cobro por PIX pasa a Asaas y los familiares obtienen un portal web.

## 📖 Guía de uso

Guía completa (landing + paso a paso): **https://inematds.github.io/voucontigo/guia/es/**

Plan de implementación completo (v1, v2 con lo que quedó ✅/⚠️, v3, modelo de datos, plantillas): [PLANO.md](PLANO.md).

---

## Qué incluye

### v1.0.0 — "Libreta digital"

- **Landing promocional** (`/`) — mobile-first, con la paleta del logo, CTA fijo de WhatsApp y formulario breve que crea un lead y notifica a la gestora por Telegram.
- **Panel** (`/painel`, login de Supabase, roles de gestora/acompañante) — agenda, clientes y personas acompañadas, paquetes y saldo de horas, finanzas, leads, configuraciones y métricas con exportación a CSV.
- **Ejecución del servicio** — Iniciar/Finalizar con registro obligatorio de minutos de espera, km recorridos, estacionamiento, peaje y nivel de esfuerzo (1–5). Esto permite revisar el precio después de 10–20 servicios.
- **Informe de WhatsApp copiable** — texto prearmado con el modelo de la marca.
- **Bot de Telegram para la gestión** — notificaciones y comandos de agenda/ejecución.
- **Cron de recordatorios** — ruta protegida por `CRON_SECRET`.

### v2.0.0 — "Automatización" (nuevo)

- **Bot de WhatsApp** (`lib/whatsapp/fluxo.ts` + `textos.ts`), menú numérico:
  `1` Agendar · `2` Cancelar o cambiar la fecha · `3` Ver horarios disponibles · `4` Mi saldo y próxima visita · `5` Hablar con una persona.
  Agendar ofrece los horarios disponibles calculados a partir de la agenda y registra la solicitud mediante la RPC `solicitar_atendimento` (queda en estado `solicitado`, a la espera de la gestora). Cancelar usa `cancelar_atendimento_familiar`, que aplica la política e informa la tarifa. Tres situaciones ponen la conversación en estado **`humano`**: la opción `5`, texto libre en el menú y número sin registro (que también se convierte en lead). En ese estado el bot **guarda silencio por completo** — ni siquiera "menú" lo reactiva — hasta que la gestión envía `/liberar` por Telegram. Si la respuesta no es válida dentro de un flujo (elegir tipo, fecha u horario), no escala: vuelve a pedir el número.
- **Webhook de WhatsApp** (`app/api/whatsapp/route.ts`) — Evolution API por defecto; Meta Cloud API opcional con `WHATSAPP_PROVIDER=meta`.
- **Recordatorios y avisos automáticos al cliente** (`lib/automacao/**`) — confirmación al aprobar la solicitud, **D-1**, **2h antes**, **informe enviado automáticamente** al finalizar y envío semanal opcional de los horarios disponibles (config `enviar_horarios_semanal`). El aviso de **saldo bajo** llega al Telegram de la gestión, con el texto listo para enviar.
- **Radio de atención** — geocodificación del destino (Nominatim) y comparación con `lat_base`/`lng_base`. Es **informativo**: si no hay ubicación base o geocodificador, responde "radio no verificado" y **no bloquea nada**.
- **Cobro por PIX vía Asaas** (`lib/asaas/**`) — cobro con QR y opción de copiar y pegar, webhook en `/api/asaas/webhook` autenticado mediante el header **`asaas-access-token`**, y `/api/cron/renovacoes` para avisar sobre el vencimiento de los paquetes mensuales. El paquete nuevo solo se crea cuando el webhook confirma el pago.
- **Portal familiar** (`app/(portal)/**`) — `/entrar` mediante **enlace mágico por correo electrónico**, `/minha-conta` (saldo, próximas visitas, historial, datos de la persona acompañada, solicitudes y cancelaciones), `/minha-conta/horarios` con "recibir por correo electrónico / por WhatsApp", `/minha-conta/relatorios/[mes]` en **página imprimible** (Imprimir → Guardar como PDF) y `/minha-conta/calendario.ics` (descarga con la sesión abierta).
- **Telegram v2** (`lib/telegram/comandos-v2.ts`) — botones inline ✅ aprobar / ✏️ ajustar / ❌ rechazar y los comandos `/solicitacoes`, `/conversas`, `/responder`, `/liberar`, `/resumo`, `/financeiro`.
- **Panel v2** — `/painel/inbox` (bandeja de entrada de WhatsApp + leads), `/painel/solicitacoes`, cronología por cliente y PIX de Asaas en la pantalla de finanzas.
- **Correo electrónico transaccional** (`lib/email/cliente.ts`) — Resend vía REST, sin SDK. Sin `RESEND_API_KEY`, usa un cliente Fake que solo registra.
- **Base de datos** — `supabase/migrations/0004_v2_schema.sql` (conversaciones y mensajes de WhatsApp, `webhook_processado`) y `0005_v2_rls_funcoes.sql` (rol familiar mediante `auth_user_id` y las RPC `solicitar_atendimento`, `cancelar_atendimento_familiar`, `vincular_familiar`, `horarios_ocupados`).

**Pendiente (v3):** varias acompañantes, app PWA con check-in, evaluaciones e IA de asistencia.

## Stack

| Capa | Elección |
|---|---|
| Web (landing, panel, portal) | Next.js 15 (App Router) + TypeScript + Tailwind 4 |
| Base de datos, auth, almacenamiento | Supabase (Postgres + RLS + Auth) |
| WhatsApp | **Evolution API** autoalojada (número común mediante QR) · Meta Cloud API opcional |
| Bot de gestión | grammY en una ruta de Next.js (webhook) |
| Cobro | Asaas (PIX, webhook) |
| Correo electrónico | Resend (REST) |
| Tareas programadas | Vercel Cron (Hobby: solo diario) o cron del sistema en la VPS |
| Validación | Zod · pruebas con Vitest |
| Alojamiento | VPS con Docker + Caddy, o Vercel — siempre mediante `git push` |

## Cómo ejecutar localmente

Requisitos previos: Node 20+, npm y Docker (para Supabase local). El flujo de la v1 no cambió.

```bash
git clone https://github.com/inematds/voucontigo.git
cd voucontigo
npm install

# 1. variables de entorno
cp .env.example .env.local

# 2. Supabase local (imprime la URL y las claves — cópielas en .env.local)
npx supabase start

# 3. aplica las migraciones de supabase/migrations (0001 → 0005)
npx supabase db reset

# 4. inicia la app
npm run dev        # http://localhost:3000 · panel en /painel · portal en /minha-conta
```

Studio local de Supabase: `http://localhost:54323` — allí se crea la **usuaria gestora** (Authentication → Add user). Un disparador crea el `perfil` con el rol predeterminado `acompanhante`; promueva la primera cuenta a `gestora` desde SQL Editor:

```sql
update public.perfil set papel = 'gestora'
  where id = (select id from auth.users where email = 'voce@exemplo.com');
```

La app funciona sin Evolution, Asaas ni Resend: WhatsApp y el correo electrónico usan clientes Fake que solo registran los envíos.

Otros scripts: `npm run build`, `npm run lint`, `npm run typecheck`, `npm test`.

### Variables de entorno

Todas están listadas en [`.env.example`](.env.example).

**v1 (sin cambios)**

| Variable | De dónde viene |
|---|---|
| `NEXT_PUBLIC_SUPABASE_URL` / `NEXT_PUBLIC_SUPABASE_ANON_KEY` | Supabase → Settings → API (o la salida de `npx supabase start`) |
| `SUPABASE_SERVICE_ROLE_KEY` | Supabase → Settings → API — **solo en el servidor** |
| `TELEGRAM_BOT_TOKEN` | `@BotFather` |
| `TELEGRAM_WEBHOOK_SECRET` | cadena larga que inventas; valida las llamadas de Telegram |
| `TELEGRAM_CHAT_GESTAO` | id del grupo privado de gestión |
| `CRON_SECRET` | secreto que envía el cron como `Authorization: Bearer ...` |
| `NEXT_PUBLIC_SITE_URL` | URL pública del sitio |
| `NEXT_PUBLIC_WHATSAPP_EMPRESA` | número de WhatsApp Business, formato `55DDDNÚMERO` |
| `TZ` | **`America/Sao_Paulo`** — zona horaria del proceso; defínela también en las variables de entorno del proyecto en Vercel |

**v2 (nuevas)**

| Variable | De dónde viene |
|---|---|
| `WHATSAPP_PROVIDER` | `evolution` (por defecto) o `meta` |
| `EVOLUTION_API_URL` | `http://evolution:8080` cuando la app y Evolution comparten la red Docker |
| `EVOLUTION_API_KEY` | la **misma** cadena que `AUTHENTICATION_API_KEY` de Evolution |
| `EVOLUTION_INSTANCE` | nombre de la instancia creada en el manager (`voucontigo`) |
| `WHATSAPP_WEBHOOK_TOKEN` | secreto que inventas; Evolution no firma el cuerpo, así que viaja en la cadena de consulta |
| `WHATSAPP_TOKEN` / `WHATSAPP_PHONE_NUMBER_ID` / `WHATSAPP_VERIFY_TOKEN` / `WHATSAPP_APP_SECRET` | solo con `WHATSAPP_PROVIDER=meta` |
| `RESEND_API_KEY` / `EMAIL_FROM` | Resend → API Keys; el dominio del remitente debe estar verificado |
| `ASAAS_API_KEY` / `ASAAS_WEBHOOK_TOKEN` / `ASAAS_BASE_URL` | Asaas → Integraciones; empieza en `https://sandbox.asaas.com/api/v3` |
| `GEOCODER_PROVIDER` / `GEOCODING_URL` / `GEOCODER_EMAIL` | Nominatim de OpenStreetMap; la política de uso **exige** un correo electrónico de contacto |

Vacío = desactivado, no = error: sin `ASAAS_API_KEY` no hay PIX automático, sin `RESEND_API_KEY` no hay correo electrónico, sin geocodificador nunca se verifica el radio.

## Iniciar Evolution API en la VPS

Evolution se ejecuta junto a la app, en la misma red Docker (`inemadlp_default`), y conversa con WhatsApp mediante un **número común leído por QR** — sin verificación de Meta ni plantillas aprobadas.

**1. Configurar e iniciar**

```bash
cp .env.evolution.example .env.evolution
# edite: EVOLUTION_SERVER_URL, EVOLUTION_API_KEY (string longa), EVOLUTION_DB_PASSWORD

docker compose -p voucontigo -f docker-compose.evolution.yml --env-file .env.evolution up -d
```

Usa siempre `-p voucontigo` (o ejecútalo desde un directorio llamado `voucontigo`): `deploy/Caddyfile.snippet` hace proxy a los contenedores `voucontigo-app-1` y `voucontigo-evolution-1`, nombres que dependen de ese prefijo de proyecto. La imagen está fijada en `atendai/evolution-api:v2.2.3`; el Postgres de Evolution es propio y no tiene relación con Supabase.

**2. Crear la instancia y leer el QR**

Abre `https://evolution.inema.club/manager` e inicia sesión con `EVOLUTION_API_KEY`. Crea la instancia con el nombre **`voucontigo`** (el mismo valor de `EVOLUTION_INSTANCE`), solicita el código QR y escanéalo con el **celular del número de Vou Contigo** (WhatsApp → Dispositivos vinculados). La sesión queda en el volumen `evolution_instances` y persiste tras reinicios.

**3. Configurar el webhook de la instancia**

`WEBHOOK_GLOBAL_ENABLED` está en `false` intencionalmente: el webhook se registra **en la instancia**, en el manager.

| Campo | Valor |
|---|---|
| URL | `https://voucontigo.inema.club/api/whatsapp?token=<WHATSAPP_WEBHOOK_TOKEN>` |
| Eventos | `MESSAGES_UPSERT` e `MESSAGES_UPDATE` |
| `webhook_by_events` | **false** (todos los eventos en la misma URL) |
| `webhook_base64` | **false** (no queremos medios en base64) |

El mismo `WHATSAPP_WEBHOOK_TOKEN` debe estar en el `.env` de la app: la ruta compara en tiempo constante y devuelve 401 si no coincide. Los mensajes de grupo, enviados por ti misma y sin texto se ignoran con 200; `wa_message_id` garantiza la idempotencia.

**4. Configurar la app para usarla**

```env
WHATSAPP_PROVIDER=evolution
EVOLUTION_API_URL=http://evolution:8080   # nome do serviço na rede Docker compartilhada
EVOLUTION_API_KEY=<a mesma AUTHENTICATION_API_KEY>
EVOLUTION_INSTANCE=voucontigo
WHATSAPP_WEBHOOK_TOKEN=<o mesmo da URL do webhook>
```

## Asaas (cobro por PIX)

1. Crea la cuenta y empieza en **sandbox** (`https://sandbox.asaas.com`): genera la API key y colócala en `ASAAS_API_KEY`, con `ASAAS_BASE_URL=https://sandbox.asaas.com/api/v3`.
2. Registra el webhook apuntando a `https://voucontigo.inema.club/api/asaas/webhook`, con un **token de autenticación** — Asaas lo envía en el header **`asaas-access-token`**, y la ruta lo compara con `ASAAS_WEBHOOK_TOKEN` (401 si no coincide).
3. Eventos relevantes: `PAYMENT_RECEIVED` / `PAYMENT_CONFIRMED` (marca como pagado) y `PAYMENT_DELETED` / `PAYMENT_REFUNDED` (revierte el cambio). La ruta siempre responde 200 si está autenticada — Asaas bloquea la cola ante cualquier respuesta que no sea 2xx — y elimina duplicados mediante `webhook_processado`.
4. Solo después de completar un cobro de punta a punta en sandbox, cambia a producción: `ASAAS_BASE_URL=https://api.asaas.com/v3` y la clave de producción (la config `asaas_ambiente` registra qué entorno está en uso).

## Resend (correo electrónico, opcional)

Crea la API key en Resend, verifica el dominio usado en `EMAIL_FROM` y completa ambas variables. Sin `RESEND_API_KEY`, el sistema sigue funcionando: el correo electrónico usa un cliente Fake y todo se envía solo por WhatsApp.

## Supabase Auth — Redirect URLs

El proyecto tiene **dos** inicios de sesión mediante enlace mágico. En **Authentication → URL Configuration → Redirect URLs**, registra ambas URL; de lo contrario, el enlace del correo electrónico volverá con un error:

```
${NEXT_PUBLIC_SITE_URL}/entrar/callback     # portal del familiar
${NEXT_PUBLIC_SITE_URL}/login/callback      # panel del equipo
```

El familiar solo puede iniciar sesión si el **correo electrónico (o WhatsApp) ya está registrado en la ficha del cliente en el panel** — así es como la RPC `vincular_familiar` vincula la cuenta con el cliente. Si no coincide, el inicio de sesión va a `/entrar/sem-cadastro`. Quien no sea familiar queda como `acompanhante` por defecto; la gestora se promueve mediante SQL.

## Deploy

Publicar es **`git push`** — el resto es un webhook.

```bash
git add -A
git commit -m "feat: ..."
git push
```

### VPS (recomendado — es donde está Evolution)

```bash
docker compose -p voucontigo -f docker-compose.vps.yml --env-file .env up -d --build
```

- Caddy del host se encarga de TLS y del proxy: añade `deploy/Caddyfile.snippet` al Caddyfile (`voucontigo.inema.club` → `voucontigo-app-1:3000`, `evolution.inema.club` → `voucontigo-evolution-1:8080`).
- Las variables `NEXT_PUBLIC_*` se incluyen como **build args** (quedan incorporadas en el bundle); las demás vienen de `.env` en tiempo de ejecución.
- Los jobs pasan a ser cron del sistema: copia `deploy/vps-cron.example` a `/etc/cron.d/voucontigo`, cambia `CRON_SECRET` y el dominio. Usa `CRON_TZ=America/Sao_Paulo` y ejecuta recordatorios a las `07:00`, renovaciones a las `06:30` y el recordatorio de 2h **cada hora**.

### Vercel

`vercel.json` ya incluye los tres cron — en **UTC**, equivalentes a los horarios de la VPS: `0 10 * * *` (07:00 BRT, recordatorios D-1 + resumen), `30 9 * * *` (06:30 BRT, renovaciones) y `0 * * * *` (recordatorio de 2h).

> ⚠️ **El cron cada hora requiere el plan Pro.** En Hobby, Vercel solo ejecuta cron una vez al día y sin horario garantizado, por lo que el recordatorio de "2h antes" no funciona bien. En la VPS esto no es un problema: cron lo ejecuta el sistema.

Antes del primer deploy, registra las variables (incluido `TZ=America/Sao_Paulo`) en el proyecto y ejecuta `npx tsx scripts/telegram-set-webhook.ts` con la URL final.

## Comandos de Telegram

**v1 — agenda y ejecución**

| Comando | Qué hace |
|---|---|
| `/start`, `/ajuda` | Muestra los comandos disponibles |
| `/hoje` | Agenda de hoy, con ids |
| `/amanha` | Agenda de mañana |
| `/semana` | Próximos siete días |
| `/agendar` | Asistente paso a paso para agendar |
| `/cancelar <id>` | Cancela y registra el motivo, aplicando la política |
| `/iniciar <id>` | Check-in: registra la hora real de inicio |
| `/finalizar <id>` | Check-out: solicita espera, km, estacionamiento, peaje y esfuerzo 1–5 |
| `/relatorio <id>` | Prepara el informe con el modelo de la marca |
| `/saldo <cliente>` | Horas contratadas, usadas y restantes |
| `/lead` | Nuevos leads de la landing, con enlace de WhatsApp |

**v2 — solicitudes y conversaciones**

| Comando | Qué hace |
|---|---|
| `/solicitacoes` | Solicitudes pendientes de WhatsApp, con botones ✅ aprobar · ✏️ ajustar · ❌ rechazar |
| `/conversas` | Quién está esperando atención humana |
| `/responder <whatsapp> <texto>` | Responde a la persona por WhatsApp sin salir de Telegram |
| `/liberar <whatsapp>` | Devuelve la conversación al bot (la saca del estado `humano`) |
| `/resumo` | Resumen operativo de hoy |
| `/financeiro` | Resumen financiero de la última semana |

Además de los comandos, el bot avisa automáticamente sobre nuevos leads, servicios creados o cancelados, nuevas solicitudes de WhatsApp, pedidos de atención humana, informes pendientes por más de 2h y paquetes con saldo bajo. El cron de las 7h envía el resumen del día; los lunes incluye el resumen financiero de la semana cerrada.

## Límites del servicio

**Sin cuidados médicos.** Vou Contigo no administra medicamentos, no brinda cuidados de enfermería ni realiza procedimientos. Por eso el sistema no tiene campos de medicación ni expediente médico — solo "restricciones declaradas" en texto libre, con lo mínimo necesario.

**LGPD.** Los datos de las personas mayores son sensibles: recopilación mínima, consentimiento registrado en el alta, fotos solo con autorización explícita, eliminación a pedido y aislamiento mediante RLS en Supabase. El familiar solo ve a su propio cliente; el equipo, solo lo que permite su rol.

**Riesgo de bloqueo del número en WhatsApp.** Evolution API usa un **número común** conectado mediante QR, no la API oficial de Meta. Es barato y sencillo, pero Meta puede bloquear el número si hay actividad de spam. Por eso, las reglas de uso son:

- responder solo a conversaciones iniciadas por el cliente;
- enviar **solo mensajes sobre compromisos reales** (confirmación, recordatorio, informe, cobro de lo contratado);
- nada de envíos masivos, listas de difusión, promociones ni marketing;
- mantener un número de reserva y, si aumenta el volumen, migrar a Meta Cloud API (`WHATSAPP_PROVIDER=meta`, ya compatible con el código).

## Roadmap — v3.0.0 "Red"

- Varias acompañantes: registro, documentos, regiones, disponibilidad y asignación.
- Matching por región, disponibilidad e historial con la misma persona acompañada.
- App PWA de la acompañante con check-in/check-out e informe guiado.
- Evaluación de la familia después del servicio y alerta de caída en la calidad.
- IA de asistencia: un borrador/audio se convierte en informe, resumen del historial antes de la visita, sugerencia de ruta.
- Finanzas completas (asignación, morosidad, factura) y programa de referidos.

Antes de eso, las brechas conocidas de la v2 (§7.2 del PLANO): OTP por WhatsApp en el portal, `.ics` por URL firmada, aviso "estoy en camino" durante el check-in y PDF del informe generado en el servidor.

---

Documentos: [PLANO.md](PLANO.md) · [docs/conversa-original.md](docs/conversa-original.md)
