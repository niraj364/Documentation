# WhatsApp Bot — Grey-Collar Capture

This document describes the **current** WapHire WhatsApp bot implementation in this repository: architecture, conversation flow, integrated APIs, configuration, and how to run/test it.

It matches the Django app `backend.whatsapp_bot` as it exists today.

---

## Primary code paths

| Area | Path |
|------|------|
| Webhook | `backend/src/backend/whatsapp_bot/views.py` |
| Orchestrator (inbound logic) | `backend/src/backend/whatsapp_bot/orchestrator.py` |
| SOP loader (role-aware paths) | `backend/src/backend/whatsapp_bot/sop_loader.py` |
| SOP / role question config | `backend/src/backend/whatsapp_bot/sops/driver_four_wheeler_v1.json` |
| Graph send helpers | `backend/src/backend/whatsapp_bot/services.py` |
| Media download / upload | `backend/src/backend/whatsapp_bot/media.py` |
| Welcome promo image | `backend/src/backend/whatsapp_bot/welcome_image.py` |
| Inactivity reminder | `backend/src/backend/whatsapp_bot/inactivity.py` |
| Celery tasks | `backend/src/backend/whatsapp_bot/tasks.py` |
| i18n / system copy | `backend/src/backend/whatsapp_bot/i18n.py` |
| ASR (Sarvam) | `backend/src/backend/whatsapp_bot/asr.py` |
| Claude extraction | `backend/src/backend/whatsapp_bot/extraction.py` |
| Profile sync | `backend/src/backend/whatsapp_bot/profile_sync.py` |
| Models | `backend/src/backend/whatsapp_bot/models.py` |
| Promo asset | `backend/src/backend/whatsapp_bot/assets/waphire_promo.jpg` |
| URL mount | `backend/src/backend/urls.py` → `{API_PREFIX}/whatsapp/` |
| Env template | `backend/.env.example` |

**Webhook URL:** `GET/POST /waphire-api/v1/whatsapp/webhook/`

---

## 1. Overview

### What it is

Grey-collar **voice-to-profile** capture over the **WhatsApp Business Cloud API**. Workers chat on WhatsApp; the bot collects:

1. Language  
2. Occupation (“What do you do?”)  
3. **Role-specific** questions  
4. Shared questions (area, availability, confirm, ID verification stub)  

Answers are stored on a `ConversationSession`, upserted into `WhatsAppWorkerProfile`, and can sync onto an existing or synthetic candidate `User`.

### Goals

- No new worker mobile app — WhatsApp only.  
- Fast webhook ACK; heavy voice work in Celery.  
- Hindi / English / Hinglish from the first language choice onward.  
- Role-aware questions (Driver, Carpenter, Electrician, Sweeper).  
- WapHire promo image once at conversation entry (and after full restart).  
- 30-second inactivity reminder; `Hi` after the reminder restarts the full welcome flow.  
- Reuse Django backend, Celery, Claude helpers — no parallel Node WhatsApp service.

---

## 2. High-level architecture

```
WhatsApp user
    │
    │  inbound message (text / button / list / voice)
    ▼
Meta WhatsApp Cloud API
    │
    │  HTTPS webhook POST
    ▼
Django ── WhatsAppWebhookView
    │         GET  = verify token (Meta subscription)
    │         POST = parse payload → orchestrator → HTTP 200
    │
    ├─ claim_message()          idempotency (ProcessedWhatsAppMessage)
    ├─ ConversationSession      answers, step, locale, inactivity tokens
    ├─ sop_loader               JSON SOP + role_questions banks
    ├─ welcome_image            promo JPG via Graph media upload / send
    ├─ services                 Graph send text / buttons / list / image
    ├─ inactivity               Celery countdown reminder
    │
    └─ Celery worker (async)
          ├─ process_experience_voice       download media → Sarvam ASR → Claude extract
          ├─ process_experience_transcript  typed fallback → Claude extract
          └─ send_inactivity_reminder       evaluate + send idle copy
```

**Design principle:** conversation logic lives in an **SOP JSON config** plus a thin orchestrator. New occupations or questions are added in JSON, not by rewriting Python flows.

---

## 3. Conversation flow (user journey)

```
Hi / Hello
  → WapHire promotional image (once per session entry)
  → Greeting: "I am the Job Assistant of WapHire…"
  → Language list (हिंदी / English / Hinglish)
  → Occupation list ("What do you do?")
       ├─ Driver      → vehicle, voice experience, licence, driving work type
       ├─ Carpenter   → years, specialty, tools, work looking for
       ├─ Electrician → years, work type, certification, work looking for
       └─ Sweeper     → years, cleaning type, shift, work looking for
  → Shared: work area → availability → confirm profile → ID verification stub
  → Profile saved (WhatsAppWorkerProfile + optional User sync)
```

### Inactivity + restart

1. After every real inbound message, Celery schedules a check in **30 seconds** (`WHATSAPP_INACTIVITY_SECONDS`).
2. If the user stays idle, a locale-aware reminder is sent (“type Hi to continue…”).
3. Reminder is also allowed after **profile completed** (not while voice is processing).
4. If the user sends **Hi / hi / HI / Hello** **after** that reminder, the session is **fully reset** and the **same** welcome path runs again (image → greeting → language → occupation → …).
5. Mid-flow `Hi` **without** a prior reminder **resumes** the current step (does not wipe language).

### Idempotency

Webhook retries use Meta `message.id`. `ProcessedWhatsAppMessage` ensures the same inbound event is handled once (no duplicate image / menus).

---

## 4. Module map

| File | Role |
|------|------|
| `views.py` | Thin webhook adapter (verify + dispatch) |
| `orchestrator.py` | Inbound routing, ask/advance, restart, interactive replies |
| `sop_loader.py` | Load SOP JSON; build **role-aware** linear question path |
| `sops/driver_four_wheeler_v1.json` | Shared questions + `role_questions` banks |
| `i18n.py` | Locale helpers + system copy (errors, inactivity, etc.) |
| `services.py` | WhatsApp Cloud API **outbound** (text, buttons, list, image) |
| `media.py` | Download inbound media; **upload** promo image to Graph |
| `welcome_image.py` | Send promo image once per entry; media-id cache |
| `inactivity.py` | Activity tokens + schedule/evaluate reminder |
| `tasks.py` | Celery: voice, transcript, inactivity reminder |
| `asr.py` | Sarvam speech-to-text (mockable) |
| `extraction.py` | Claude structured extract from transcript |
| `profile_sync.py` | Upsert `WhatsAppWorkerProfile`; sync to `User` |
| `models.py` | Session, processed messages, worker profile |
| `assets/waphire_promo.jpg` | Bundled welcome image |

All paths above are under `backend/src/backend/whatsapp_bot/`.

---

## 5. Data model (session state)

Stored mainly on `ConversationSession.answers` (JSON), plus status fields:

| Key / field | Meaning |
|-------------|---------|
| `language` / `locale` | `lang_en` / `lang_hi` / … and `en` / `hi` |
| `occupation` | e.g. `role_driver`, `role_carpenter` |
| Role answer fields | e.g. `vehicle_type`, `carpentry_years`, … |
| `work_area`, `availability`, … | Shared tail of the interview |
| `_welcome_image_sent` | Prevents repeat promo image in the same entry |
| `current_question_id` | Current SOP step |
| `inactivity_token` / `inactivity_reminder_token` | Generation tokens for the 30s reminder |
| `last_inbound_at` | Last user activity timestamp |

`WhatsAppWorkerProfile` is the portable capture artifact (`answers_raw` keeps the full JSON). Confirming the profile can sync onto an existing or synthetic candidate `User`.

Migrations:

- `0001_voice_to_profile` — session, processed messages, worker profile  
- `0002_inactivity_watch` — inactivity timestamp / tokens  

---

## 6. Role-based questions (scalable config)

Shared path in SOP:

```text
language → occupation → [role bank] → work_area → availability → confirm → verify
```

Role banks live under `role_questions` in the SOP JSON:

```json
"role_questions": {
  "role_driver": [ /* vehicle, experience, licence, work type */ ],
  "role_carpenter": [ /* … */ ],
  "role_electrician": [ /* … */ ],
  "role_sweeper": [ /* … */ ]
}
```

`sop_loader.linear_questions(sop, answers)` inserts the selected occupation’s bank **immediately after** `occupation`. Changing occupation clears previous role answers (`clear_role_specific_answers`) so Driver questions never leak into Carpenter.

**To add a new occupation later:**

1. Add an option on the occupation question.  
2. Add a new bank under `role_questions`.  
3. No orchestrator rewrite required.

---

## 7. APIs integrated

### 7.1 Meta WhatsApp Cloud API (Graph `v21.0`)

| Use | How |
|-----|-----|
| Webhook verify | `GET …/whatsapp/webhook/` with `hub.verify_token` |
| Inbound messages | `POST` same URL — statuses ignored; messages handled |
| Send text / buttons / list / image | `POST https://graph.facebook.com/v21.0/{PHONE_NUMBER_ID}/messages` |
| Upload media (promo image) | `POST …/{PHONE_NUMBER_ID}/media` |
| Download voice note | `GET /{media_id}` then download `url` with Bearer token |

Auth: `Authorization: Bearer {WHATSAPP_TOKEN}`.

```text
GET/POST /waphire-api/v1/whatsapp/webhook/
```

No JWT on the webhook (Meta cannot send app tokens). CSRF / DRF auth are disabled for this view; Meta’s verify token + HTTPS tunnel protect the endpoint in practice.

### 7.2 Sarvam ASR (speech-to-text)

- Used for Driver **voice experience** notes.  
- `POST` to `SARVAM_ASR_URL` (default Saaras).  
- Local demos: `WHATSAPP_ASR_MOCK=true` skips Sarvam.

### 7.3 Anthropic Claude (via existing resume-parse helper)

- `extraction.py` reuses `_claude_messages` from `backend.accounts.resume_parse`.  
- Turns transcript → structured fields (`experience_yrs`, cities, summary, confidence).

### 7.4 Internal (not external HTTP)

- Django ORM / Postgres — sessions & profiles  
- Celery + RabbitMQ — voice + inactivity  
- Promo image uses Graph upload by default (not Azure)

---

## 8. How outbound WhatsApp sending works

`services.py` builds Graph JSON payloads:

- **Text** — plain body  
- **Buttons** — max 3 reply buttons  
- **List** — max 10 rows (language, occupation, areas, etc.)  
- **Image** — by Graph `media_id` or public HTTPS `link`

Promo image resolution order (`welcome_image.py`):

1. `WHATSAPP_PROMO_IMAGE_MEDIA_ID` if set  
2. Else public `https://…` `WHATSAPP_PROMO_IMAGE_URL` (localhost rejected)  
3. Else upload bundled `assets/waphire_promo.jpg` once and cache the media id  

If image send fails, **text + menus still continue**.

---

## 9. Environment variables

Set in `backend/.env` (see `backend/.env.example`):

| Variable | Purpose |
|----------|---------|
| `WHATSAPP_TOKEN` | Temporary or system-user Graph access token |
| `WHATSAPP_PHONE_NUMBER_ID` | Cloud API phone number id |
| `WHATSAPP_VERIFY_TOKEN` | Must match Meta webhook verify token |
| `WHATSAPP_INACTIVITY_SECONDS` | Idle delay before reminder (default `30`) |
| `WHATSAPP_VOICE_MIN_CONFIDENCE` | Min Claude confidence for voice accept |
| `WHATSAPP_ASR_MOCK` | Skip Sarvam in local/demo |
| `WHATSAPP_PROMO_IMAGE_MEDIA_ID` | Optional pre-uploaded media id |
| `WHATSAPP_PROMO_IMAGE_URL` | Optional public HTTPS image URL |
| `SARVAM_API_KEY` / `SARVAM_*` | ASR |
| `ANTHROPIC_API_KEY` | Claude extraction |

**Note:** Meta **temporary** tokens expire often. If the bot receives `Hi` (webhook 200) but never replies, check logs for Graph `401` / session expired and regenerate `WHATSAPP_TOKEN`, then recreate `web` + `worker` so env reloads.

---

## 10. Local run (Docker)

From `backend/`:

```powershell
docker compose up -d --build
docker compose up -d --force-recreate web worker   # after .env / code changes that need reload
```

Expose the webhook publicly (Cloudflare quick tunnel or ngrok) to:

```text
https://<public-host>/waphire-api/v1/whatsapp/webhook/
```

In Meta Developer → WhatsApp → Configuration:

- Callback URL = that URL  
- Verify token = `WHATSAPP_VERIFY_TOKEN`  
- Subscribe to **messages** on the WhatsApp Business Account  

Run tests:

```powershell
docker compose exec -T web python manage.py test backend.whatsapp_bot --verbosity=1
```

---

## 11. What was intentionally not changed

- Candidate / employer website UI  
- Existing JWT-protected REST APIs (except registering this webhook under `/whatsapp/`)  
- Unrelated Celery beat jobs  
- No separate Node WhatsApp service — everything stays in Django  

---

## 12. Extending safely

1. **New role** → `role_questions` + occupation option in the SOP JSON.  
2. **New shared question** → add to `questions` (after occupation, before or after area as needed).  
3. **New language string** → bilingual `prompt` / `title` objects or `i18n.MESSAGES`.  
4. Prefer config over hardcoding in `orchestrator.py`.

---

## 13. Quick troubleshooting

| Symptom | Likely cause |
|---------|----------------|
| No reply at all | Expired `WHATSAPP_TOKEN`, or tunnel/callback URL mismatch |
| Reminder never arrives | Celery `worker` down / RabbitMQ unhealthy |
| Reminder OK but `Hi` resumes old step | After reminder, greetings should restart — redeploy if running old code |
| Same questions for every job | Occupation not selected / old SOP cached — restart web; check `role_questions` |
| Image missing, text OK | Graph media upload failed (logged); text flow is designed to continue |
| Duplicate menus | Should not happen if `message.id` present — check `ProcessedWhatsAppMessage` |

---

Related docs in this monorepo

| Doc | Path |
|-----|------|
| **Scale architecture (~1.2M users)** | [`documentation/whatsapp-bot/SCALE_ARCHITECTURE.md`](./SCALE_ARCHITECTURE.md) |
| Candidate onboarding | [`documentation/onboarding/README.md`](../onboarding/README.md) |
| Bulk upload | [`documentation/bulk-upload/README.md`](../bulk-upload/README.md) |
| Backend overview | [`backend/README.md`](../../backend/README.md) |
| Env template | [`backend/.env.example`](../../backend/.env.example) |
| App-local pointer | [`backend/src/backend/whatsapp_bot/README.md`](../../backend/src/backend/whatsapp_bot/README.md) |
