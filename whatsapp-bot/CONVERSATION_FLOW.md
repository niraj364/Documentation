# WapHire WhatsApp Bot — Role-Based Conversation Flow (Developer Reference)

> **Scope.** This document describes the **current implementation** of the WhatsApp
> candidate bot, traced function-by-function through the code in
> `backend/src/backend/whatsapp_bot/`. It does not describe PRD intentions.
> Where the code has a gap or an inconsistency, it is listed in
> [§10 Known gaps and quirks](#10-known-gaps-and-quirks-observed-in-code) rather than
> being papered over.
>
> All paths below are relative to `backend/src/backend/` unless stated otherwise.
> Prompts are quoted in English; each SOP prompt also has a Hindi variant in the JSON.

---

## Table of contents

1. [Module inventory](#1-module-inventory)
2. [Entry flow: candidate message → bot reply](#2-entry-flow-candidate-message--bot-reply)
3. [Session model and state](#3-session-model-and-state)
4. [SOP engine (question selection)](#4-sop-engine-question-selection)
5. [Onboarding flow: language → resume or no resume](#5-onboarding-flow-language--resume-or-no-resume)
6. [Role resolution and role-specific question banks](#6-role-resolution-and-role-specific-question-banks)
7. [Common profile questions, confirmation and edits](#7-common-profile-questions-confirmation-and-edits)
8. [Job matching and applying](#8-job-matching-and-applying)
9. [Background paths: voice, resume, inactivity, delivery status, errors](#9-background-paths-voice-resume-inactivity-delivery-status-errors)
10. [Known gaps and quirks observed in code](#10-known-gaps-and-quirks-observed-in-code)
11. [Configuration reference](#11-configuration-reference)

---

## 1. Module inventory

| File | Responsibility |
|---|---|
| `urls.py` (project) + `whatsapp_bot/urls.py` | Routes `API_PREFIX + "whatsapp/"` → `webhook/`, `demo/inbound/`, `demo/session/`, `sessions/` (read-only viewset). Live webhook path: `/waphire-api/v1/whatsapp/webhook/`. |
| `whatsapp_bot/views.py` | `WhatsAppWebhookView` (GET verify, POST inbound), `_verify_meta_signature`. |
| `whatsapp_bot/tasks.py` | All Celery tasks (inbound, status, resume, voice, reminders, purge, re-engage). |
| `celery.py` | `task_routes` (queues) and beat schedule. |
| `whatsapp_bot/orchestrator.py` | Lock + idempotency + session + router; `ask_question` (renders any SOP question); legacy linear SOP handlers. |
| `whatsapp_bot/candidate_flow.py` | The `candidate_onboarding_v1` flow: interactive/text/voice/document handlers, resume parse mapping, role + profile question progression, confirmation, edits, matching, apply. |
| `whatsapp_bot/sop_loader.py` | Loads SOP JSON, role-bank resolution, condition evaluation, `next_role_question`, `next_profile_question`, regex field extraction. |
| `whatsapp_bot/sops/candidate_onboarding_v1.json` | The question bank (shared questions, 13 role banks, role aliases, profile order). |
| `whatsapp_bot/services.py` | Meta Graph API sends (`send_text`, `send_buttons`, `send_list`, `send_image`, `send_template`, `send_graph_payload`). |
| `whatsapp_bot/models.py` | `ConversationSession`, `ProcessedWhatsAppMessage`, `WhatsAppDeliveryStatus`, `WhatsAppBlockList`, `WhatsAppMediaAttachment`, `WhatsAppWorkerProfile`. |
| `whatsapp_bot/cache_store.py` | Redis (Django cache): conversation lock, rate limits, session snapshot, inactivity de-dupe, promo media id. |
| `whatsapp_bot/blocklist.py` | `is_blocked` (DB + 60 s cache). |
| `whatsapp_bot/inactivity.py` | Idle reminder scheduling and evaluation. |
| `whatsapp_bot/welcome_image.py` | Promo image before the greeting. |
| `whatsapp_bot/i18n.py` | `MESSAGES` strings, `greeting_text`, `session_locale`, `persist_locale`, `sop_field`. |
| `whatsapp_bot/asr.py` | `transcribe` / `transcribe_audio` (local faster-whisper, or mock). |
| `whatsapp_bot/extraction.py` | `extract_experience_from_transcript` (Claude JSON extraction). |
| `whatsapp_bot/media.py` / `azure_media.py` | Download media from Graph API; store in Azure blob. |
| `accounts/resume_parse.py` | `parse_resume_pdf_bytes` (Anthropic Messages API, PDF beta), `store_resume_bytes`, `experience_years_from_work`. |
| `whatsapp_bot/profile_sync.py` | `upsert_worker_profile`, `sync_to_candidate_user`, legacy `build_profile_summary`. |
| `whatsapp_bot/profile_edit.py` | `clean_role`, `match_job_function`, `role_option_id`, `parse_experience_years`, `canonical_location`, `apply_skill_edit`, `classify_other`. |
| `whatsapp_bot/job_bridge.py` | `match_jobs_for_answers`, `other_city_options`, `format_job_card_text`, `apply_to_job`, `list_applications_for_user`. |
| `whatsapp_bot/reengage.py` | 24-hour-window text vs. template re-engagement (see §10: no caller). |
| `whatsapp_bot/phone.py` | `normalize_wa_id`, `mobile_lookup_variants`. |
| `whatsapp_bot/demo_views.py` | `POST demo/inbound/` (only when `DEBUG` or `WHATSAPP_DEMO_API`). |
| `whatsapp_bot/admin.py` | Session admin with `mark_handed_off` action. |

**Runtime processes** (`backend/docker-compose.yml`):

| Service | Command / queues |
|---|---|
| `web` | gunicorn Django (host `8080` → container `8000`) |
| `worker` | `-Q celery,whatsapp_chat,whatsapp_voice,whatsapp_outbound` |
| `worker-wa` | `-Q whatsapp_chat,whatsapp_outbound`, concurrency `WA_CHAT_CONCURRENCY` (8) |
| `worker-wa-voice` | `-Q whatsapp_voice`, concurrency `WA_VOICE_CONCURRENCY` (4), prefetch 1 |
| `worker-wa-chat`, `worker-wa-outbound` | Only with compose profile `wa-scale` |
| `broker` | RabbitMQ (Celery broker) |
| `redis` | Django cache backend (locks, rate limits, snapshots) |
| `db` | Postgres |

**Celery routing** (`celery.py` `task_routes`): `process_inbound`, `process_status_update`,
`process_experience_transcript`, `send_inactivity_reminder` → `whatsapp_chat`;
`process_experience_voice`, `process_onboarding_voice`, `process_onboarding_edit_voice`,
`process_whatsapp_resume`, `process_id_document_ocr` → `whatsapp_voice`;
`send_outbound_payload`, `reengage_session` → `whatsapp_outbound`.
Beat: `whatsapp_purge_processed_messages` daily at 03:15.

---

## 2. Entry flow: candidate message → bot reply

### 2.1 One-line trace (asynchronous mode, the default)

```
Candidate sends "Hi"
 → Meta Cloud API POST /waphire-api/v1/whatsapp/webhook/
 → views.py  WhatsAppWebhookView.post()          (signature check, split statuses/messages)
 → tasks.py  process_inbound.delay(sender, message)    [RabbitMQ queue: whatsapp_chat]
 → HTTP 200 {"status":"ok"} to Meta
 ...worker...
 → tasks.py  process_inbound()
 → orchestrator.py  handle_inbound()             (normalize wa_id, Redis conversation lock)
 → orchestrator.py  _handle_inbound_locked()     (blocklist → rate limit → claim_message → session)
 → orchestrator.py  get_or_create_session()
 → inactivity.py    mark_user_activity()
 → orchestrator.py  _handle_text()               (greeting detection)
 → orchestrator.py  start_or_resume() → candidate_flow.py start_onboarding()
 → welcome_image.py send_welcome_image_if_needed() → services.send_image()
 → services.py      send_text(greeting)
 → candidate_flow.py _ask("language") → orchestrator.py ask_question() → services.send_buttons()
 → services.py      send_graph_payload() → POST graph.facebook.com/v21.0/{PHONE_NUMBER_ID}/messages
 → inactivity.py    schedule_inactivity_watch()  → send_inactivity_reminder.apply_async(countdown)
 → cache_store.py   set_session_snapshot()
 → Candidate receives: promo image, greeting, [🇮🇳 Hindi] [🇬🇧 English]
```

When `WHATSAPP_WEBHOOK_SYNC=True`, the webhook calls `orchestrator.handle_inbound()` inline
instead of queuing, so every reply is sent before Meta receives the HTTP 200.

### 2.2 Stage-by-stage

#### Stage 1 — Webhook verification (one-time, GET)
- **File / function:** `whatsapp_bot/views.py` → `WhatsAppWebhookView.get`
- **What it does:** Checks `hub.mode == "subscribe"` and `hub.verify_token == WHATSAPP_VERIFY_TOKEN`.
- **Input:** Query params `hub.mode`, `hub.verify_token`, `hub.challenge`.
- **Output:** `hub.challenge` echoed (200) or 403.
- **DB changes:** none. **Queue:** none. **Next:** none.

#### Stage 2 — Webhook receive (POST)
- **File / function:** `whatsapp_bot/views.py` → `WhatsAppWebhookView.post`, `_verify_meta_signature`
- **What it does:**
  1. Validates `X-Hub-Signature-256` as HMAC-SHA256 of the raw body with `WHATSAPP_APP_SECRET`.
     **If `WHATSAPP_APP_SECRET` is empty, verification is skipped.** A failure → 403.
  2. Iterates `entry[].changes[].value`.
  3. `value.statuses` (delivery receipts) with no messages → `process_status_update.delay(status)`.
  4. Each `value.messages[]` item → sender = `message["from"]`;
     sync mode → `orchestrator.handle_inbound(sender, message)`,
     otherwise → `tasks.process_inbound.delay(sender, message)`.
- **Input:** Meta JSON payload.
- **Output:** 200 `{"status":"ok"}`; any exception → 503 `{"status":"temporarily_unavailable"}` (Meta retries).
- **DB changes:** none in async mode. **Raw messages are not stored anywhere** (see §10).
- **Queue:** `whatsapp_chat` (inbound), `whatsapp_chat` (status).
- **Next:** `tasks.process_inbound` (async) or `orchestrator.handle_inbound` (sync).

#### Stage 3 — Celery task
- **File / function:** `whatsapp_bot/tasks.py` → `process_inbound(self, wa_id, message)`
- **What it does:** Calls `handle_inbound`; on any exception retries (`max_retries=3`, `default_retry_delay=2` s).
- **Input:** `wa_id` string, Meta message dict.
- **Output:** none. **DB changes:** none directly. **Queue:** consumed from `whatsapp_chat`.
- **Next:** `orchestrator.handle_inbound`.

#### Stage 4 — Conversation lock
- **File / function:** `orchestrator.py` → `handle_inbound`; `cache_store.py` → `acquire_conversation_lock`, `release_conversation_lock`
- **What it does:** Normalizes the number (`phone.normalize_wa_id`) and takes a per-number Redis lock
  `wa:conversation:{wa_id}` (`cache.add`, polling every 0.05 s, wait `WHATSAPP_CONVERSATION_LOCK_WAIT_SECONDS`=30,
  TTL `WHATSAPP_CONVERSATION_LOCK_TTL_SECONDS`=180). Messages from one number are therefore processed one at a time.
- **Output:** If the lock can't be taken → `TimeoutError` → Celery retry (Stage 3).
- **DB changes:** none. **Next:** `_handle_inbound_locked`.

#### Stage 5 — Guards and idempotency
- **File / function:** `orchestrator.py` → `_handle_inbound_locked`
- **Steps, in order:**
  1. `blocklist.is_blocked(wa_id)` — `WhatsAppBlockList` row (cached `wa:block:{wa_id}` 60 s) → silently return.
  2. `cache_store.allow_inbound_rate(wa_id)` — `wa:rate:in:{wa_id}`, `WHATSAPP_INBOUND_RATE_PER_MIN` (30) per 60 s; fails **open** if Redis errors → over limit: silently return (message dropped).
  3. `claim_message(msg_id, wa_id)` — `ProcessedWhatsAppMessage.objects.get_or_create(message_id=…)`; existing row → duplicate, return.
- **DB changes:** inserts `ProcessedWhatsAppMessage(message_id, wa_id, processed_at)`.
- **Next:** `get_or_create_session`.

#### Stage 6 — Session load/create
- **File / function:** `orchestrator.py` → `get_or_create_session`
- **What it does:** Latest `ConversationSession` for `wa_id` excluding `abandoned`. If none, or the latest is
  `completed` (its `inactivity_token` is bumped first), creates a new one with `sop_id=get_default_sop_id()`
  (`candidate_onboarding_v1`), `current_question_id=""`, `answers={"_flow": "candidate_onboarding"}`, `status="active"`.
- **Then in `_handle_inbound_locked`:** `handed_off` sessions → ignore. Saves `last_inbound_message_id`.
  Computes `after_reminder` (reminder already sent for this idle period) **before** calling
  `inactivity.mark_user_activity(session)` (bumps `inactivity_token`, sets `last_inbound_at`).
- **DB changes:** possible `ConversationSession` insert; update of `last_inbound_message_id`, `inactivity_token`, `last_inbound_at`.

#### Stage 7 — Router (by message type)
- **File / function:** `orchestrator.py` → `_handle_inbound_locked`

| `message.type` | Onboarding SOP handler | Legacy SOP handler |
|---|---|---|
| `text` | `_handle_text` → `candidate_flow.handle_onboarding_text` | `_handle_text` |
| `interactive` (`button_reply` / `list_reply`) | `candidate_flow.handle_onboarding_interactive(session, reply_id)` | `_handle_interactive` |
| `audio` / `voice` | `candidate_flow.handle_onboarding_voice` | `_handle_voice` |
| `image` / `document` | `candidate_flow.handle_onboarding_document` | `_handle_media` |
| anything else | `send_text(i18n "unexpected_input")` | same |

- Any exception inside the handler is caught: `send_text(i18n "friendly_error")`. It is **not** re-raised, so Celery does not retry these.
- `finally`: `schedule_inactivity_watch(session)` and `set_session_snapshot(wa_id, {session_id, status, step, locale})` (`wa:session:{wa_id}`, TTL 3600 s); logs `WhatsApp handle done … elapsed_ms`.

#### Stage 8 — Text pre-routing (greeting handling)
- **File / function:** `orchestrator.py` → `_handle_text`
- `GREET_TOKENS = {"hi","hello","hii","hey","नमस्ते"}` (case-insensitive exact match).

| Condition | Action |
|---|---|
| Greeting **and** reminder already sent | `reset_conversation_state` (answers, retries, step wiped) → `start_or_resume` (fresh start) |
| Greeting mid-flow (step set, not completed) | `start_or_resume` → re-asks the **current** question |
| No current step, or greeting on a completed session | reset (if completed/empty) → `start_or_resume` |
| Otherwise (onboarding SOP) | `candidate_flow.handle_onboarding_text(session, text)` |
| Not handled + current question `mode == "voice"` | "voice ack" + `process_experience_transcript.delay` (legacy path) |
| Not handled | `send_text(i18n "use_buttons")` |

#### Stage 9 — SOP/question engine
- **File / function:** `candidate_flow.py` → `_advance_candidate_profile` (details in §4)
- **What it does:** Decides the next question from the saved `answers` + SOP JSON.
- **Next:** `candidate_flow._ask(session, question_id)` → `orchestrator.ask_question`.

#### Stage 10 — Rendering the next question
- **File / function:** `orchestrator.py` → `ask_question(session, question)`
- **DB changes:** sets `current_question_id`; status → `awaiting_voice` (mode voice), `confirming` (`confirm_profile`), otherwise `active`.
- **Rendering rules:**
  - **Open questions** (`mode` text/voice, except `resume_upload`, `job_matching`, `field_confirmation`) are sent as a **button message**: prompt + "💬 Type your answer, or choose Voice." with a single button `🎤 Answer by Voice` (`mode_voice`); in voice mode the button is `💬 Type Instead` (`mode_chat`) and the text says "🎤 Send a voice note to answer."
  - `mode == "button"` → `send_buttons` (first 3 options). For `confirm_profile` the body is replaced by `profile_sync.build_profile_summary`.
  - `mode == "list"` → `send_list` (≤10 rows, titles cut to 24 chars). If `resume_available == "resume_no"`, a **second** button message is sent: "You can switch between voice and chat at any time." + mode switch button.
  - Any other mode (e.g. `resume_upload`) → `send_text`.
- **Dynamic options** (`_question_options`):
  - `desired_role` → up to 10 active `JobFunction` rows (`desired_job_function_{pk}`); falls back to static SOP options if none.
  - `skills` → up to 9 top-level active `Skills` for `answers.current_job_function_id` + `✅ Done` (`skill_done`). With no catalog skills: static SOP driving skills only when `current_role_bank` is empty, `role_driver` or `role_delivery`; any other bank → question becomes free text "What are your main skills? Type them separated by commas (e.g. Cooking, Hygiene)."
  - `change_field_selection` → the resume path shows only `chg_role, chg_experience, chg_skills, chg_location, chg_other, chg_done`; the no-resume path shows all ten.

#### Stage 11 — Outgoing API
- **File / function:** `services.py` → `send_text` / `send_buttons` / `send_list` / `send_image` / `send_template` → `send_graph_payload`
- **What it does:** `POST https://graph.facebook.com/v21.0/{WHATSAPP_PHONE_NUMBER_ID}/messages` with `Authorization: Bearer {WHATSAPP_TOKEN}`.
  - Skips if credentials are missing, the recipient is blocklisted, or the global outbound limit is hit (`wa:rate:out:global`, `WHATSAPP_OUTBOUND_RATE_PER_SEC` = 80).
  - Up to 4 attempts; retries on 408/429/5xx/network errors with backoff (honours `Retry-After`); no retry on 400/401/403.
  - `send_buttons`: max 3 buttons, titles ≤20 chars; falls back to text "Reply with: …" if interactive send fails.
- **Runs synchronously inside the same worker** as the inbound task (the `send_outbound_payload` task exists but is not used by the flow).
- **DB changes:** none. Delivery receipts arrive later via Stage 2 → `process_status_update` → `WhatsAppDeliveryStatus` row.

---

## 3. Session model and state

`models.ConversationSession` — one row per conversation:

| Field | Meaning |
|---|---|
| `wa_id` | Normalized phone number |
| `sop_id` | `candidate_onboarding_v1` (or a legacy SOP if `WHATSAPP_DEFAULT_SOP_ID` is changed) |
| `current_question_id` | The state; an SOP question id, a role-bank question id, `resume_confirm`, `confirm_profile`, `field_confirmation`, `change_field_selection`, `edit_*`, `job_matching` |
| `answers` (JSON) | Every captured value, including labels, `field_confidence`, `input_mode`, `language`, matching state |
| `retry_counts` (JSON) | Used by legacy voice retries |
| `status` | `active`, `awaiting_voice`, `processing_voice`, `confirming`, `completed`, `abandoned`, `handed_off` |
| `user` / `worker_profile` | FK to `accounts.User` and `WhatsAppWorkerProfile` once linked |
| `last_inbound_message_id`, `last_inbound_at` | Last message bookkeeping |
| `inactivity_token`, `inactivity_reminder_token` | Idle-reminder generation counters |
| `handed_off_at`, `handoff_note` | Set by admin action `mark_handed_off` |

The onboarding flow **never sets `status = completed`** (only the legacy `advance_after_answer` does). A finished
candidate stays at `current_question_id = "job_matching"` with `status = active`, and the same session is reused.

Related tables: `ProcessedWhatsAppMessage` (idempotency; purged after `WHATSAPP_PROCESSED_RETENTION_DAYS`=90),
`WhatsAppDeliveryStatus` (purged after 30 days), `WhatsAppMediaAttachment` (resume blobs, `parse_status`, `parsed_json`),
`WhatsAppWorkerProfile` (mobile, language, vehicle_type, work_area, experience_yrs, availability, display_name,
`answers_raw`, `is_confirmed`).

---

## 4. SOP engine (question selection)

`candidate_flow._advance_candidate_profile(session)` is called after almost every answer. Its order is fixed:

1. **Prefill** from an existing WapHire `User` with the same mobile (`_prefill_no_resume_answers`): name (unless it
   starts with "worker "), `curr_pos` → current role, experience, skills, current location, preferred location,
   `job_function` → desired role. Links `session.user`. Only empty answers are filled.
2. **Current role fallback:** if `current_role_label` is empty but `curr_pos` / `role_label` / `desired_role_label`
   exists, it is copied and `current_role_bank` is resolved with `role_bank_key`.
3. **Current role missing** → ask `current_work_voice` (voice mode) or `current_role` (chat mode).
4. No-resume path: `_persist_no_resume_profile` (partial User sync once a name exists).
5. **Low role confidence:** `field_confidence.current_role_label < confidence_threshold (0.75)` → `field_confirmation` (Yes / Change buttons, §7.3).
6. **Role questions:** `sop_loader.next_role_question(sop, answers)` — first question of the resolved bank that
   applies (see below) and is not yet answered. An answered field whose confidence is below 0.75 is returned with
   `needs_confirmation` → `field_confirmation`.
7. **Common profile questions:** `sop_loader.next_profile_question(sop, answers)` in `profile_question_order`.
8. Nothing left → `_show_confirm` (final summary).

**Question applicability** (`sop_loader`): skipped when `required: false`, when `required_if` is false, when
`when` is false, or when `depends_on` is unmet. Conditions support `equals`, `not_equals`, `in`, `not_in`,
`gt/gte/lt/lte`, `contains`, `exists`, `truthy`. A field counts as answered if the field **or any
`satisfied_by` field** has a value.

**Regex extraction** (`extract_role_fields`): a role question with an `extract` block runs SOP regexes on the
answer text and may fill several fields at once (confidence 0.92), which then skips those questions.
Free-text answers also run through `candidate_flow._extract_profile_fields` (name, experience, desired role,
current-role claims like "I am a driver", availability, locations).

---

## 5. Onboarding flow: language → resume or no resume

```
"Hi" → [promo image] → greeting → language
language:            lang_hi / lang_en   → "Great! 👍" → resume_available
resume_available:    resume_yes          → resume_upload
                     resume_no           → input_mode_selection
input_mode_selection: mode_voice / mode_chat → _advance_candidate_profile → current_work_voice / current_role
```

| Step (`current_question_id`) | Sent as | Prompt (EN) | Options → next |
|---|---|---|---|
| (entry) | image + text | Promo image (`welcome_image`, once per session via `answers._welcome_image_sent`), then `greeting_text` — Hindi + English before a language is chosen | → `language` |
| `language` | buttons | "Choose your language:" | `lang_hi` 🇮🇳 Hindi, `lang_en` 🇬🇧 English → `persist_locale`, "Great! 👍", → `resume_available` |
| `resume_available` | buttons | "Great! 👍 First, let's see if you have a resume ready. Do you have a resume?" | `resume_yes` → `resume_upload`; `resume_no` → `input_mode_selection` |
| `input_mode_selection` | buttons | "That's okay! You can create your WapHire profile without a resume. How would you like to answer?" | `mode_voice` / `mode_chat` → sets `answers.input_mode` → `_advance_candidate_profile` |
| `current_role` | open text | "What kind of work do you currently do? Type your role." | see §6 |
| `current_work_voice` | open voice | "What kind of work do you currently do? 🎤 Send a voice message." | voice note → ASR → same handler as `current_role` |

**Mode switch at any time:** tapping `mode_voice` / `mode_chat` sets `answers.input_mode`. On
`input_mode_selection`, `resume_available`, `current_role` or `current_work_voice` it asks
`current_work_voice` / `current_role`; on any other step it re-asks the same question and keeps existing answers.

### 5.1 Resume path

| Step | What happens |
|---|---|
| `resume_upload` | "Awesome! Please send your resume here 📄 (PDF)". Only on this step is a document/image accepted (`handle_onboarding_document`; elsewhere → i18n `media_not_needed`). MIME must contain `pdf`, `msword`, `officedocument` or `image/`, otherwise "Please send a PDF resume." Reply: "Resume received ✅ I'm reading your skills and experience." |
| background | `process_whatsapp_resume` (queue `whatsapp_voice`; runs inline in sync mode). See §9.2. |
| `resume_confirm` | `_show_profile_review`: resume summary + `resume_ok` "✅ Looks good" / `resume_fix` "✏️ Change". |
| `resume_ok` | `_advance_after_resume_ok` → `_advance_candidate_profile` → **only missing** role/profile questions, then `confirm_profile`. |
| `resume_fix` | `_open_change_menu(return_to="resume_confirm")` (§7.3). |
| parse failure | "I couldn't read that resume…" → asks the legacy `current_work` list (see §10). |

---

## 6. Role resolution and role-specific question banks

### 6.1 How a typed role is resolved (`handle_onboarding_text`, step `current_role` / `current_work_voice`)

1. Empty → "Please type your role or send a voice note." and re-ask.
2. `role = profile_edit.clean_role(text)`; `jf = profile_edit.match_job_function(role)` (existing active `JobFunction`, never creates rows);
   `bank_key = sop_loader.role_bank_key(sop, role)`.
3. Neither `jf` nor `bank_key` → "I couldn't match that role. Type the closest option: " + up to 8 active JobFunction names → re-ask `current_role`.
4. Otherwise saves `current_role_label`, `curr_pos` (JobFunction name if matched, else the bank's canonical name),
   `current_job_function_id`, `current_role_bank`, `current_work` (`role_option_id` or `work_other`), runs
   `_extract_profile_fields`, then `_advance_candidate_profile`.

**`role_bank_key` rules:** exact bank id → normalized role equals a bank id → exact match against a bank's
`aliases` / `job_function_names` → otherwise substring match (names ≥4 chars), choosing the bank whose longest
name is longest.

Verified results against the current SOP (run in the `web` container):

| Typed role | Bank |
|---|---|
| Driver | `role_driver` |
| Delivery Driver | `role_delivery` |
| Guard, Security | `security_guard` |
| Cleaner | `housekeeping` |
| Cook | `cook_chef` |
| Sales | `sales_executive` |
| Customer Support | `customer_support` |
| Developer | `software_developer` |
| Electrician | `electrician` |
| Plumber | `plumber` |
| **Technician** | **`electrician`** (substring of "electrical technician") |
| Warehouse, Helper, Delivery Boy, Telecaller, Receptionist | none |

A role with no bank still proceeds **if a matching JobFunction exists**; it simply gets no role-specific
questions and goes straight to the common profile questions (§7).

### 6.2 Role aliases configured in the SOP

| Bank | job_function_names | aliases |
|---|---|---|
| `role_driver` | Driver, 4-Wheeler Driver, Cab Driver | driver, car driver, cab driver, taxi driver, 4 wheeler driver, 4-wheeler driver |
| `role_delivery` | Delivery Executive, Delivery Driver | delivery executive, delivery driver, courier, delivery rider |
| `security_guard` | Security Guard, Security Officer | security guard, guard, security officer |
| `electrician` | Electrician | electrician, electrical technician |
| `plumber` | Plumber | plumber, plumbing technician |
| `housekeeping` | Housekeeping, Housekeeper | housekeeping, housekeeper, cleaner |
| `cook_chef` | Cook, Chef | cook, chef, cooking |
| `sales_executive` | Sales Executive, Sales Representative | sales executive, sales representative, salesperson |
| `customer_support` | Customer Support, Customer Service Executive | customer support, customer service, call center executive, support agent |
| `software_developer` | Software Developer, Software Engineer, IT Engineer | software developer, software engineer, software/it, it engineer, programmer |

`role_technician`, `role_warehouse`, `role_helper` banks exist but have **no aliases** (see §10).

### 6.3 Role banks (exact order, from `candidate_onboarding_v1.json`)

"Open" = sent as an open text/voice question; "List" = interactive list. Tapped list answers get
confidence 1.0 and `option_values` mapping where configured.

#### `role_driver`
| # | Question id → field | Type | Prompt | Options / condition |
|---|---|---|---|---|
| 1 | `driver_vehicle_type` → `vehicle_type_text` | Open | What type of vehicle do you drive? | Regex extract may also fill experience, license, own vehicle, shift |
| 2 | `driver_experience` → `experience_yrs` | Open | How many years of driving experience do you have? Say 0 if you are a fresher. | |
| 3 | `driver_license_status` → `license_status` | List | Do you have a valid driving license? | `license_yes`=true, `license_no`=false |
| 4 | `driver_license_type` → `license_type` | Open | What type of driving license do you have? | **only when** `license_status == true` |
| 5 | `driver_commercial_experience` → `commercial_experience` | Open | Have you driven commercially? For how many years? | **only if** `license_type` contains "commercial" |
| 6 | `driver_own_vehicle` → `own_vehicle` | List | Do you own a vehicle? | `own_vehicle_yes`/`own_vehicle_no` |
| 7 | `driver_preferred_shift` → `driver_preferred_shift` | List | Which shift do you prefer? | `shift_day`, `shift_night`, `shift_any` |

#### `role_delivery`
| # | id → field | Type | Prompt | Options / condition |
|---|---|---|---|---|
| 1 | `delivery_experience` → `experience_yrs` | Open | How many years of delivery experience do you have? | |
| 2 | `delivery_two_wheeler` → `two_wheeler_available` | List | Do you have a two-wheeler for delivery work? | yes/no |
| 3 | `delivery_license` → `license_status` | List | Do you have a valid driving license? | yes/no |
| 4 | `delivery_own_vehicle` → `own_vehicle` | List | Do you own a bike? | yes/no |
| – | `delivery_platforms` → `platforms_text` | Open | Which delivery platforms have you worked with? | `required: false` → **never asked** |
| 5 | `delivery_shift` → `preferred_shift` | List | Which shift do you prefer? | day/night/any |

#### `security_guard`
| # | id → field | Type | Prompt | Options / condition |
|---|---|---|---|---|
| 1 | `security_experience` → `experience_yrs` | Open | How many years of security experience do you have? | |
| 2 | `security_shift` → `security_shift` | List | Which shifts can you work? | `security_day`, `security_night`, `security_both` |
| 3 | `security_night_shift` → `night_shift_available` | List | Are you available for night shifts? | **only if** shift is `security_night` or `security_both` |
| – | `security_certification` | Open | Do you have security training or certification? | `required: false` → never asked |

#### `housekeeping`
1. `housekeeping_experience` → `experience_yrs` (Open) — How many years of housekeeping experience do you have?
2. `housekeeping_workplace` (List) — Which workplaces have you worked in? `workplace_hotel`, `_hospital`, `_office`, `_home`, `_other`
3. `housekeeping_shift` → `preferred_shift` (List) — day/night/any

#### `cook_chef`
1. `cooking_experience` → `experience_yrs` (Open)
2. `cooking_specialty` → `cuisine_specialty` (Open) — Which cuisines or dishes can you prepare? (extract may fill diet, experience)
3. `cooking_diet` → `dietary_preference` (List) — `diet_veg`=veg, `diet_non_veg`=non_veg, `diet_both`=both
4. `cooking_workplace` (List) — `cook_restaurant`, `cook_hotel`, `cook_home`
- `cooking_shift` — `required: false` → never asked

#### `sales_executive`
1. `sales_experience` → `experience_yrs` (Open)
2. `sales_customer_type` (List) — `sales_b2b`=B2B, `sales_b2c`=B2C, `sales_both`=B2B/B2C
3. `sales_channel` (List) — `sales_field`, `sales_inside`, `sales_both_channels`
4. `sales_industry` (Open) — Which industries have you sold in?
5. `sales_lead_generation` → `lead_generation` (List) — yes/no

#### `customer_support`
1. `support_experience` → `experience_yrs` (Open)
2. `support_languages` (Open) — Which languages can you support customers in?
3. `support_channel` (List) — `support_voice`=voice, `support_chat`=non_voice, `support_both`=both
4. `support_market` (List) — domestic / international / both
5. `support_shift` → `preferred_shift` (List) — day/night/any
- `support_industry` — `required: false` → never asked

#### `software_developer`
1. `tech_stack` → `skills_labels` (Open) — Which technologies or programming languages do you work with? (parsed with `apply_skill_edit`; also satisfies the common `skills` question)
2. `tech_work_mode` → `work_mode` (List) — `work_mode_onsite`, `work_mode_hybrid`, `work_mode_remote`
- `tech_secondary_skills`, `tech_notice_period`, `tech_compensation` — `required: false` → never asked

#### `electrician` / `plumber`
- One Open question each: "What electrical work can you do?" → `electrical_skills` / "What plumbing work can you do?" → `plumbing_skills` (regex extract may also fill `experience_yrs` and `*_work_types`).

#### `role_technician` / `role_warehouse` / `role_helper`
- One Open question each → `specialty_text`. Not reachable through the typed-role path (§10).

---

## 7. Common profile questions, confirmation and edits

### 7.1 Common questions (`profile_question_order`)

Asked after the role bank, **skipping any already satisfied** (by role answers, resume, prefill or extraction):

| Order | id → field | Type | Prompt | Satisfied by |
|---|---|---|---|---|
| 1 | `candidate_name` → `name` | Open | What is your full name? | `parsed_name` |
| 2 | `experience` → `experience` | List | How much experience do you have? (`exp_fresher`, `exp_1_2`, `exp_3_5`, `exp_5_plus`) | `experience_yrs`, `experience_yrs_known`, `experience_label` — so skipped for every bank that asks experience |
| 3 | `skills` → `skills` | List (multi-select) or free text | Which of these can you do? (see Stage 10 for option source) | `skills_labels`, `skills_text` |
| 4 | `current_location` → `current_location_label` | Open | Which city do you currently live in? | `location_label`, `location` |
| 5 | `desired_role` → `desired_role` | List | Which role are you looking for? (JobFunctions from DB, else static list) | `desired_role_label`, `desired_job_function_id`, `job_function_id` |
| 6 | `preferred_location` → `preferred_location_label` | Open | Where would you prefer to work? | `preferred_cities` |
| 7 | `availability` → `availability` | List | When can you start? (`avail_now`, `avail_7d`, `avail_15d`, `avail_30d`, `avail_notice`, `avail_custom`, `avail_later`) | — |
| 7a | `availability_date` | Open | What date can you start? | only after `avail_custom` |
| 8 | `work_type` → `work_type` | List | What type of work are you looking for? (`work_full_time`, `work_part_time`, `work_contract`, `work_temporary`) | `work_type_label` |

**Skills multi-select:** each tapped skill is added to `skills_selected` and the list is re-sent; `skill_done`
builds `skills_labels` and advances.

### 7.2 Example: full chat-mode path for a driver without resume

```
Hi → [image] greeting → language(lang_en) → resume_available(resume_no) → input_mode_selection(mode_chat)
→ current_role "Driver"
→ driver_vehicle_type → driver_experience → driver_license_status
    ├─ license_yes → driver_license_type ──(contains "commercial")──> driver_commercial_experience
    └─ license_no  → (both skipped)
→ driver_own_vehicle → driver_preferred_shift
→ candidate_name → [experience skipped] → skills → current_location → desired_role
→ preferred_location → availability → work_type
→ confirm_profile → confirm_yes → job matching
```

### 7.3 Confirmation and edits

- **`field_confirmation`** (`_show_field_confirmation`): "I understood {field} as {value}. Is that correct?" with `field_confirm_yes` "✅ Yes" / `field_confirm_change` "✏️ Change" (typed yes/haan/ok or no/change/edit also work; anything else → "Please reply Yes or Change."). Yes → that field's confidence set to 1.0 → `_advance_candidate_profile`; Change → the original question is asked again. Status `confirming`.
- **`confirm_profile`** (`_show_confirm`): `build_onboarding_summary` (role, desired role, experience, location, preferred location, skills, availability) + `confirm_yes` "✅ Correct" / `confirm_edit` "✏️ Change". Status `confirming`.
  - `confirm_yes` → `persist_onboarding_to_user(session)` → `_show_jobs`.
  - `confirm_edit` → `_open_change_menu(return_to="confirm_profile")`.
- **`change_field_selection`** (list) → `chg_*` → `edit_*` state (`answers.editing_field`). Each `edit_*` is an open question:

| Option | Edit state | Prompt |
|---|---|---|
| `chg_name` | `edit_name` | What is your full name? |
| `chg_role` | `edit_role` | What role are you currently working in? |
| `chg_experience` | `edit_experience` | How many years of experience do you have? |
| `chg_skills` | `edit_skills` | Which skills would you like to add or change? … To remove one, write: remove Java |
| `chg_location` | `edit_location` | What is your current location? |
| `chg_desired_role` | `edit_desired_role` | What kind of job are you looking for? |
| `chg_preferred_location` | `edit_preferred_location` | Where would you prefer to work? |
| `chg_availability` | `edit_availability` | When can you start? |
| `chg_other` | `edit_other` | What would you like to add to your profile? (`classify_other`) |
| `chg_done` | — | returns to the summary |

After an edit: `_sync_linked_profile` (only if `session.user` is linked) → `_return_after_edit` → back to
`confirm_profile` or `resume_confirm` with an "updated" summary. Voice notes on an edit step go to
`process_onboarding_edit_voice`.

### 7.4 What is written to the WapHire user (`persist_onboarding_to_user`)

- `profile_sync.upsert_worker_profile` → `WhatsAppWorkerProfile` (`is_confirmed` = `confirm_profile == "confirm_yes"`), plus `vehicle_type` from `vehicle_type_text`.
- `profile_sync.sync_to_candidate_user` → finds `User` by mobile variants or creates `wa_{mobile}@whatsapp.waphire.local`, adds the CAN role.
- Then sets on `User`: `curr_pos`, `job_function_id` (desired), `experience_yrs` (0 → `salary_type = "fresher_monthly"`), `name` (from `parsed_name`), skills attach/detach, current location, `preferred_locations`, and `onboarding_completed = True` when `complete`.
- Partial syncs (`complete=False`) happen during the no-resume flow once a name exists.
- Role-bank answers other than vehicle type (license, shift, cuisine, B2B/B2C, etc.) stay **only in `session.answers`** (and `WhatsAppWorkerProfile.answers_raw`).

---

## 8. Job matching and applying

**`_show_jobs`**: "Got it 👍 Finding matching jobs for you..." → `job_bridge.match_jobs_for_answers(answers, limit=5)`
→ saves `matched_job_ids`, `matched_job_count`, `current_question_id = "job_matching"`.

**Match gates** (`job_bridge`): open jobs only (`is_active`, `is_open`, `show_on_website`, not expired; newest first;
scans `max(limit*20, 100)`). A job must pass **all** of:
- `_role_matches` — desired/current role label vs. job title or job function (substring or token overlap);
- `_experience_matches` — within `experience_from`/`experience_to`;
- `_skills_match` — **every** primary skill on the job is covered by the candidate's skills;
- `_location_allowed` — home city, `preferred_cities`, or `location_any`.

**Results:**
- ≥1 → headline ("We found N matching jobs for you 🎯") + job card text (`format_job_card_text`) with `job_view_{id}` / `job_apply_{id}` buttons.
- 0 and not yet asked → `_ask_other_cities`: list of cities where everything except location matches (`city_{i}`), plus `city_any`, `city_none`. Choosing one re-runs matching.
- 0 otherwise → "No matching jobs found right now. We'll let you know when a suitable job becomes available."

**Apply** (`job_apply_{id}` → `apply_yes` confirm → `_do_apply`):
`apply_to_job(user, job_id)` → `CandidateJobApplication` (status `"applied"`); returns `created` / `exists` /
`job_closed` / `not_found` / `error`. Success → "Application submitted successfully! 🎉" (or "You've already
applied…") + buttons `track_apps`, `more_jobs`, `view_profile`; schedules an inactivity watch with 60 s.
`track_apps` → `list_applications_for_user`.

---

## 9. Background paths: voice, resume, inactivity, delivery status, errors

### 9.1 Voice answers (onboarding)

`handle_onboarding_voice`:
- On an `edit_*` step → "voice ack" + `process_onboarding_edit_voice.delay`.
- On `current_work_voice` or any open question → `process_onboarding_voice.delay(session_id, media_id, msg_id, locale)` (queue `whatsapp_voice`).
- On a button/list step → legacy `_handle_voice` → i18n `voice_not_needed`.
- Before queuing: "voice ack" message and status `processing_voice`.

`process_onboarding_voice`: audio (Azure copy on retry, else Meta download + Azure store) → `asr.transcribe_audio`
(faster-whisper, Hindi/English auto-detect, never translated; mock transcript when `WHATSAPP_ASR_MOCK`) → sends "I heard: *…*" → calls
`handle_onboarding_text(session, transcript)`, so a transcript follows exactly the same logic as typed text.
Failure → buttons `🎤 Try Again` / `💬 Type Instead`, status `awaiting_voice`, task retry (except for an empty transcript).

Voice tasks always use `.delay()`, **even when `WHATSAPP_WEBHOOK_SYNC=True`**, so a Celery worker consuming
`whatsapp_voice` is required for voice.

**LLM extraction** (`extraction.extract_experience_from_transcript`, Claude) is used by the legacy
`process_experience_voice` / `process_experience_transcript` path and as a fallback in `_parse_experience_answer`
when regex parsing of an experience answer fails.

### 9.2 Resume processing (`tasks.process_whatsapp_resume`, queue `whatsapp_voice`, `max_retries=2`, 8 s delay)

1. `download_whatsapp_media(media_id)`; reject if > `WHATSAPP_RESUME_MAX_BYTES` (8 MB).
2. `azure_media.store_inbound_media(purpose="resume")` → blob `whatsapp/{wa_id}/resume/{date}_{uuid}.{ext}`, `WhatsAppMediaAttachment` row.
3. `upsert_worker_profile` + `sync_to_candidate_user` (links `session.user`).
4. `store_resume_bytes` + `upsert_candidate_resume` (resume saved on the user).
5. `accounts.resume_parse.parse_resume_pdf_bytes` (requires `ANTHROPIC_API_KEY`; model `CLAUDE_RESUME_MODEL`).
6. Attachment `parsed_json` / `parse_status` updated → `candidate_flow.apply_resume_parse_result`:
   name, skills (≤12), experience (`max(parsed, experience_years_from_work)`; unknown → `experience_yrs_known = False`),
   location, current and desired role, JobFunction ids, `current_role_bank` via `role_bank_key`, `field_confidence`,
   `merge_resume_role_fields` → `_show_profile_review`.
7. Failure → "I couldn't read that resume…" → `_ask(session, "current_work")`.

### 9.3 Inactivity reminder (`inactivity.py`)

- `mark_user_activity` bumps `inactivity_token` on every inbound message.
- `schedule_inactivity_watch` (after each inbound) → `send_inactivity_reminder.apply_async(countdown=WHATSAPP_INACTIVITY_SECONDS)` (default 60 s). De-duplicated via `wa:inact:sched:{session_id}:{token}`. Skipped for `abandoned`, `processing_voice`, `handed_off`, or if already reminded for this idle period.
- `evaluate_inactivity_reminder` sends i18n `inactivity_reminder` only if the token is unchanged and enough time has passed; stores `inactivity_reminder_token`.
- A greeting after the reminder restarts the flow from scratch (Stage 8).

### 9.4 Delivery status
`process_status_update(status)` → `WhatsAppDeliveryStatus` row (sent / delivered / read / failed).

### 9.5 Errors and retries (summary)

| Where | Behaviour |
|---|---|
| Webhook exception | 503 → Meta redelivers |
| Bad signature | 403 |
| Conversation lock busy | `TimeoutError` → `process_inbound` retry (3×, 2 s) |
| Exception inside a handler | caught → "friendly_error" message; no Celery retry |
| Duplicate message id | skipped by `claim_message` |
| Graph API send | up to 4 attempts, backoff on 408/429/5xx |
| Resume task | 2 retries, then fallback message |
| Voice task | retries except for empty transcript; user gets Try Again / Type Instead |

---

## 10. Known gaps and quirks observed in code

These are statements about the current code, not recommendations.

1. **Technician/Warehouse/Helper banks are unreachable by typing.** They have no `role_aliases`. "Technician" resolves to the `electrician` bank; "Warehouse" / "Helper" resolve to no bank (they proceed without role questions only if a matching JobFunction exists, otherwise they are rejected). `role_questions_for_answers` passes `current_job_function_id` before `current_work`, so `role_option_id`'s `role_technician` value is not used for bank lookup.
2. **Optional role questions are never asked** (`required: false`): delivery platforms, security certification, cooking shift, tech secondary skills/notice period/CTC, support industry.
3. **Resume parse failure falls back to the legacy `current_work` list** (Driver/Technician/Warehouse/Helper/Construction/…) instead of `current_role`.
4. **Raw inbound messages are not persisted.** Only `ProcessedWhatsAppMessage.message_id` and `session.last_inbound_message_id` are stored. Because `claim_message` runs before processing, a crash after the claim followed by a Celery retry would be treated as a duplicate and skipped.
5. **Rate-limited messages are dropped silently** (no reply, no retry).
6. **Onboarding sessions never reach `status = completed`**; they remain at `job_matching`.
7. **Two summary builders:** `_show_confirm` uses `build_onboarding_summary`, but if a user sends "Hi" while on `confirm_profile`, `ask_question` renders `profile_sync.build_profile_summary` (legacy format). "Hi" on `resume_confirm` re-sends only "Does everything look correct?" without the summary.
8. **Extra message on list questions** in the no-resume path: every list question is followed by a separate "You can switch between voice and chat at any time." button message.
9. **Static skills list is driving-specific** (Driving, GPS, Customer Handling, Loading, Vehicle Maintenance); used for driver/delivery or when no bank is known and no catalog skills exist.
10. **Unused tasks:** `send_outbound_payload` and `reengage_session` (and therefore `reengage.py` / `send_template`) have no callers in the flow. `process_id_document_ocr` is a stub used only by legacy media handling.
11. **Signature verification is disabled** when `WHATSAPP_APP_SECRET` is empty.
12. **Role-bank answers are not mapped to `User` fields** except vehicle type (§7.4).
13. The comment in `_handle_inbound_locked` mentions a "30s reminder"; the default setting is 60 s.

---

## 11. Configuration reference

From `settings.py` (defaults in brackets):

| Setting | Use |
|---|---|
| `WHATSAPP_TOKEN`, `WHATSAPP_PHONE_NUMBER_ID` | Graph API credentials |
| `WHATSAPP_VERIFY_TOKEN` | Webhook GET verification |
| `WHATSAPP_APP_SECRET` [''] | Webhook signature (skipped if empty) |
| `WHATSAPP_WEBHOOK_SYNC` [False] | Process inbound inline in the web process |
| `WHATSAPP_DEFAULT_SOP_ID` [`candidate_onboarding_v1`] | SOP for new sessions |
| `WHATSAPP_CONVERSATION_LOCK_WAIT_SECONDS` [30] / `_TTL_SECONDS` [180] | Per-number lock |
| `WHATSAPP_INBOUND_RATE_PER_MIN` [30] | Per-number inbound limit |
| `WHATSAPP_OUTBOUND_RATE_PER_SEC` [80] | Global outbound limit |
| `WHATSAPP_SESSION_CACHE_TTL` [3600] | Session snapshot TTL |
| `WHATSAPP_INACTIVITY_SECONDS` [60] | Idle reminder delay |
| `WHATSAPP_RESUME_MAX_BYTES` [8 MB] | Resume size limit |
| `WHATSAPP_PROCESSED_RETENTION_DAYS` [90] / status retention [30] | Purge task |
| `WHATSAPP_PROMO_IMAGE_MEDIA_ID` / `WHATSAPP_PROMO_IMAGE_URL` | Welcome image source (else uploads `assets/waphire_promo.jpg`) |
| `WHATSAPP_ASR_MOCK` [False], `WHATSAPP_ASR_MOCK_TRANSCRIPT` | Mock ASR |
| `WHISPER_*` | faster-whisper model / device / languages |
| `ANTHROPIC_API_KEY`, `CLAUDE_RESUME_MODEL` | Resume parsing and transcript extraction |
| `WHATSAPP_REENGAGE_TEMPLATE_NAME` | Template for re-engagement (unused path) |
| `WHATSAPP_DEMO_API` | Enables `demo/inbound/` outside DEBUG |
| `CELERY_BROKER_URL` | RabbitMQ |
