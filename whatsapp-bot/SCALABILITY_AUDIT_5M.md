# WapHire WhatsApp — Scalability & Architecture Audit (target: 5M concurrent conversations)

| | |
|---|---|
| Audited code | `backend/` branch `whatsapp-integration`, commit `e20446c8` (clean tree). `stash@{0}` holds an unfinished outbound rate-limit change that is **not** applied and not assumed here. |
| Method | Static reading of the actual code paths (file:line references throughout), `.env`/compose/infra files, plus the few measurements taken earlier on the developer laptop. **No code was modified.** |
| Number labels | **[M]** measured · **[E]** estimate (assumption shown) · **[U]** unknown — needs the benchmark named |

Paths are relative to `backend/src/backend/` unless they start with `backend/`.

---

## Executive Summary

1. **The current system cannot support 5M concurrent conversations, and several defects would lose messages at any scale.** The three most serious are: messages are marked "processed" *before* they are processed (`whatsapp_bot/orchestrator.py:305`) and Celery acknowledges tasks on receipt (no `acks_late` in `settings.py`), so any crash or late exception loses the message permanently; the local `.env` runs the whole bot inside the webhook request (`WHATSAPP_WEBHOOK_SYNC=true`); and an unauthenticated demo API that can drive any phone number's conversation is switched on (`WHATSAPP_DEMO_API=true`, `whatsapp_bot/demo_views.py:113-182`).
2. **The hard ceiling at multi-million scale is not Django — it is Meta, voice transcription, and PostgreSQL write volume.**
   - **Meta:** the code sends from **one** phone number (`WHATSAPP_PHONE_NUMBER_ID`, `whatsapp_bot/services.py:30-32`). Meta's Cloud API throughput per number is 80 msg/s by default and up to 1,000 msg/s on request (Meta docs; confirm for your account). At 1.5 replies per inbound message [E], **one number tops out at ≈ 667 inbound messages/s ≈ 40,000 users chatting once a minute.** 5M concurrent needs on the order of **125–250 numbers**, and a candidate's conversation is bound to the number they messaged — this is a business/Meta onboarding item, not a code switch.
   - **Voice:** CPU `faster-whisper small/int8` ran at roughly real-time-or-slower on the laptop [M]. Even modest voice usage at 5M users needs thousands of GPU-equivalents or a capped product design (§ Whisper).
   - **PostgreSQL:** one text turn issues ≈ 25–110 SQL statements with ≈ 5–15 full-row rewrites of `ConversationSession` [E, static count]. At 5M users × 1 msg/min that is ≈ 0.8–2.5M writes/s — impossible on one primary. Cutting writes per turn to 1–2 is mandatory; past ~500K concurrent, conversation state needs horizontal sharding by `wa_id`.
3. **The architecture direction is right** (webhook enqueues; separate chat/voice/outbound queues; DB is the source of truth; Redis is cache/locks only). The work is mostly *correctness, write-efficiency, ordering and workload isolation*, then infrastructure.
4. **Nothing in this report is load-tested.** The repo's own QA report says capacity is "Not measured" (`documentation/whatsapp-bot/QA-TEST-REPORT.md:178`), and `whatsapp_bot/capacity.py` (62K target) is not imported anywhere.

---

## Current Architecture

```
Meta ──HTTPS──► [ngrok locally | VM in dev/prod] ──► WAF (ModSecurity CRS) ──► nginx LB ──► gunicorn (3 sync workers/replica, 2 replicas)
                                                                                                 │
                                       WHATSAPP_WEBHOOK_SYNC=true (local .env) ──► whole turn runs HERE
                                       WHATSAPP_WEBHOOK_SYNC=false ──► Celery .delay() ──► RabbitMQ (single node, no volume)
                                                                                                 │
                         ┌────────────────────────────┬──────────────────────────────┬──────────┘
                    whatsapp_chat                 whatsapp_voice                 whatsapp_outbound (almost unused)
              worker-wa (prefork, conc 3–8)   worker-wa-voice (prefork, conc 1–2,   
              + `worker` also consumes chat    faster-whisper small/int8 CPU,        
                                               also resume parse + Claude)           
                         │                            │
          PostgreSQL 16 (single, CONN_MAX_AGE=0, no pooler) · Redis 7 (no auth/persistence; LocMem fallback if REDIS_URL unset)
          Azure Blob (sync SDK) · Meta Graph v21.0 (sync requests, in-worker retries) · Claude (sync requests, no session reuse)
```

- **Deployments.** Local: `backend/docker-compose.yml`. Dev and prod: one self-hosted VM each, systemd `akki-gunicorn`, `akki-celery-worker`, `akki-celery-beat` (`backend/.github/workflows/*-deployment.yml`).
  - **[U] Queues on the VM:** the unit files aren't in the repo, and the docs start the worker with no `-Q` (`docs/deployment.md:40`). If that's how it runs, **the WhatsApp queues are never consumed on the VM.**
  - **[U] Whisper on the VM:** no voice worker service exists there.
- **Versions.** Django 4.2.4 (`backend/requirements/pip.txt:23`), Celery with RabbitMQ, psycopg 3.
- **Read replica router.** It exists (`whatsapp_bot/db_router.py`) but is not enabled. If it were enabled, it would route `ConversationSession` *reads* to the replica, which gives stale state inside a turn. That router must not be enabled as written.

---

## Current Message Lifecycle

Legend: S = synchronous in that process, A = handed to queue; bound by CPU / IO / DB / EXT (external API) / REDIS.

### Text message

| # | Step | Code | S/A | Bound |
|---|---|---|---|---|
| 1 | Meta POSTs webhook | — | — | EXT |
| 2 | HMAC check — **skipped when `WHATSAPP_APP_SECRET` empty (it is unset)** | `whatsapp_bot/views.py:18-28` | S | CPU |
| 3 | Parse JSON, per message `process_inbound.delay()` (sync publish, no publisher confirms) — **or the whole turn inline if SYNC** | `views.py:56-96` | S | IO (broker) |
| 4 | Return 200 (503 if enqueue raised) | `views.py:88-97` | S | — |
| 5 | Worker: per-number lock, `cache.add` polled every 50 ms ≤ 30 s, TTL 180 s | `cache_store.py:149-207` | S | REDIS |
| 6 | Blocklist (Redis 60 s, DB on miss) | `blocklist.py:17-32` | S | REDIS/DB |
| 7 | Inbound rate limit 30/min fixed window — over limit = **silent drop after 200** | `cache_store.py:90-113` | S | REDIS |
| 8 | **Claim** `ProcessedWhatsAppMessage.get_or_create` (unique) — committed **before** processing | `orchestrator.py:50-58, 305` | S | DB |
| 9 | Load/create `ConversationSession`, save `last_inbound_message_id`, bump inactivity token | `orchestrator.py:61-83, 314-321`, `inactivity.py:30-40` | S | DB |
| 10 | SOP handler `handle_onboarding_text`: loads **all active Locations**, regex over **all active Skills**, `iexact` lookups on unindexed `name`, may call **Claude synchronously** (≤ 90 s default timeout) — up to 3× for one message | `candidate_flow.py:1419-1559, 1581-1809`, `extraction.py:32-59` | S | DB/CPU/EXT |
| 11 | Profile persistence: `answers` JSON rewritten on many saves; `WhatsAppWorkerProfile.update_or_create`; `sync_to_candidate_user` (atomic, full `user.save()`); possibly run **twice** per turn | `candidate_flow.py:399-468, 655-723`, `profile_sync.py:60-166` | S | DB |
| 12 | Ask next question: 1–2 Meta sends, each sync HTTPS (15 s timeout, ≤ 4 attempts with `time.sleep` ≤ 8 s or Retry-After ≤ 60 s) **while holding the conversation lock** | `orchestrator.py:128-212`, `services.py:304-397` | S | EXT |
| 13 | `finally`: refresh, **schedule one Celery countdown task (inactivity) per message**, write unused Redis snapshot, release lock (GET+DEL, non-atomic) | `orchestrator.py:352-363`, `inactivity.py:43-77` | S/A | DB/REDIS/IO |

Static estimate per text turn: **≈ 25–110 SQL**, **≈ 10–15 Redis round trips**, **1–2 Meta sends (up to 7 for a job list)**, 1 broker publish, 0–3 Claude calls [E].

### Button / list reply
Same steps 1–9 and 13. Step 10 is `handle_onboarding_interactive` (`candidate_flow.py:754-1096`):
- **Simple steps** (for example language): about 10–15 SQL and 2 sends.
- **`confirm_yes` / show jobs:** job matching runs. It reads up to **500 newest open jobs** with 4 prefetches, scores them in Python, and repeats the whole scan when there are zero matches (`job_bridge.py:151-165, 263-315`). Then up to 7 sends.
- **Apply:** a check-then-insert into `CandidateJobApplication`, which has **no unique (candidate, job) constraint** (`job_bridge.py:318-368`).

### Voice note

| # | Step | Code | S/A | Bound |
|---|---|---|---|---|
| 1–9 | as text | | | |
| 10 | Route; take an ordering ticket (Redis INCR); enqueue voice task; send "processing" ack | `candidate_flow.py:2340-2382`, `tasks.py:46-55` | S/A | REDIS/EXT |
| 11 | Voice worker: if an earlier note is unfinished, **re-publish itself every 2 s for ≤ 120 s** | `tasks.py:118-129` | A | IO |
| 12 | Meta media: 2 HTTP calls (URL 30 s, download 60 s), whole body in memory, no byte cap | `media.py:28-56` | S | EXT |
| 13 | Azure upload (sync SDK) **before** transcription | `azure_media.py:88-123` | S | IO |
| 14 | faster-whisper: decode with PyAV (in memory), language detection pass unless locale trusted, transcribe (beam 1, VAD) | `asr.py:142-302` | S | **CPU** |
| 15 | Re-take conversation lock, re-read session, Claude extraction (`CLAUDE_RESUME_MODEL`, 1,024 max tokens) **inside the lock**, DB writes, echo + replies | `tasks.py:157-173, 332-335`, `orchestrator.py:612-653` | S | EXT/DB |

Measured on the laptop CPU before the latency work [M]:
- Download: 1.5–3 s.
- Language detection: 10–15 s.
- Transcription: 4–8 s.
- Profile update plus about 3 replies: 2.5–5.5 s.
- **Total: about 26–29 s per note.**
- Model RSS: about 490 MB (about 650 MB per process); load time 56–100 s.

**[U] Post-optimization numbers:** the two latency rounds trusted Hindi to skip detection and merged sends, but their end-to-end timings weren't captured. The `whisper_completed` log line already records real-time factor (RTF), so they can be measured from logs.

### Resume (WhatsApp document)
Code: `candidate_flow.py:2148-2193`, `tasks.py:456-566`, `accounts/resume_parse.py:359-455`, `accounts/resume_library.py:18-98`.
1. Download from Meta (2 calls), then an 8 MB size check **after** the full download.
2. `store_inbound_media` without `content=` **downloads from Meta a second time**, then uploads copy #1.
3. `store_resume_bytes` uploads copy #2 to temp. If Azure fails it **writes to local disk**, with `randint` names (`resume_parse.py:431-455`).
4. `upsert_candidate_resume` downloads, uploads copy #3 to `candidate_images/`, then deletes.
5. Claude parse:
   - Text mode: pypdf, up to 80,000 characters plus a schema of about 800 tokens, `max_tokens` 4,096.
   - PDF fallback: the whole file as base64.
   - Model is Haiku 4.5 in `.env`; the code default is Sonnet.
   - **Measured 3.6–4.2 s** with Haiku [M].
6. Map fields, then the missing-field logic (`candidate_flow.py:356-468`).
7. **Retry bug** (`tasks.py:547-566`): after telling the user it failed and moving them on, the task still retries. That repeats 2 downloads, 3 uploads and 2 Claude calls, re-sends the failure message, and can pull the user back to profile review.
8. **No content-hash dedupe on this branch.** The SHA-256 cache built earlier (`ResumeParseResult`) is on `anthropic-api-cost`, not here.
9. With SYNC on, this whole pipeline runs **on the webhook request thread** (`candidate_flow.py:2183-2185`).

---

## Current WhatsApp Architecture — what is already right

- **Fast-ack webhook.** In async mode the webhook does no DB or Redis work and returns as soon as it has enqueued (`views.py:56-97`).
- **Signature check done correctly.** When a secret is set, the HMAC runs on the raw body with `compare_digest`.
- **Separate queues** for chat, voice and outbound (`celery.py:22-33`).
- **Whisper model loaded once per process,** preloaded with `worker_process_init` (`asr.py:67-136`), and decoded in memory with no temp files.
- **PostgreSQL is the source of truth** for conversation state. The Redis snapshot is write-only (`orchestrator.py:355`), so a Redis flush loses no state.
- **Idempotency backed by a DB unique constraint** (`whatsapp_bot/models.py:105`).
- **Per-process keep-alive Graph session** (`http_client.py`).
- **Retention purges** for processed ids (90 days) and delivery statuses (30 days) (`tasks.py:274-301`).

---

## Current Bottlenecks (ranked by severity)

### P0 — must fix before any large-scale production

| # | Problem | Current implementation | Why it fails | Impact | Recommended solution | Effort |
|---|---|---|---|---|---|---|
| P0-1 | **Message loss after claim** | Claim committed before handling (`orchestrator.py:305`); handler exceptions swallowed (`:348-351`); retry of `process_inbound` (`tasks.py:226-229`) hits its own claim | Any exception outside the inner try, any broker hiccup in `finally`, or a worker crash permanently drops the message | Candidate silently stuck | Claim → `processing` with lease; mark `done` in the same transaction as the state change; reprocess expired leases; `acks_late` + `reject_on_worker_lost` | M |
| P0-2 | **Celery reliability settings never loaded** | `.env` has `CELERY_TASK_ACKS_LATE`, `..._PREFETCH_MULTIPLIER`, `..._REJECT_ON_WORKER_LOST`, but `settings.py:304-316` never reads them | Default ack-on-receive, prefetch 4; tasks lost on crash/OOM (voice workers hold ~650 MB models) | Silent loss | Read them in `settings.py`; per-task `acks_late=True` for idempotent tasks | S |
| P0-3 | **Synchronous webhook mode on** | `.env` `WHATSAPP_WEBHOOK_SYNC=true`; inline handling `views.py:82-85`, inline resume `candidate_flow.py:2183` | Turn (Meta sends, Claude, resume pipeline) runs on 3 sync gunicorn workers, 120 s timeout | A few concurrent users exhaust the web tier; timeouts lose claimed messages | `false` everywhere except one-off debugging; alert if true in non-local env | S |
| P0-4 | **Unauthenticated demo API on** | `.env` `WHATSAPP_DEMO_API=true`; `demo/inbound` accepts any `wa_id`; `demo/session` returns any user's session (PII); WAF body rules disabled for `/demo/` | Anyone can drive real conversations / Meta sends / applications and enumerate PII | Security incident | Off in every deployed env; restrict to demo prefix + staff auth | S |
| P0-5 | **Meta signature not enforced** | `views.py:18-20` returns True when secret empty; secret unset; webhook exempt from LB rate limit | Forged inbound events at unlimited rate | Spoofed conversations, queue flooding, cost | Require `WHATSAPP_APP_SECRET` in non-local envs (fail closed) | S |
| P0-6 | **VM may not run WhatsApp workers at all** | systemd unit not in repo; docs show worker without `-Q`; no voice worker; deploy never installs deps (`requirements.txt` vs `requirements/pip.txt`) | WhatsApp queues unconsumed; new packages (faster-whisper) missing | Bot dead in dev/prod | Version unit files in repo; dedicated chat/voice/outbound units; fix pip step | S |
| P0-7 | **LocMem fallback when `REDIS_URL` missing** | `settings.py:178-194` | Locks, rate limits, voice ordering become per-process | Concurrent corruption of a conversation | Fail fast at startup if `REDIS_URL` unset outside tests | S |
| P0-8 | **Global outbound cap drops replies** | Fixed 60 s window, one key; `None` returned → message lost (`cache_store.py:116-137`, `services.py:334-336`) | A burst consumes a minute's budget in 1 s, then replies vanish | Lost replies | Per-second token bucket per phone number; defer to outbound queue, never drop | S–M |
| P0-9 | **No backups beyond pre-deploy `pg_dump` on the same VM** | `*-deployment.yml` | RPO = time since last deploy; VM loss = total loss | Data loss | Managed PG with PITR; off-host backups; restore drill | M |

### P1 — required for 100K–1M concurrent

| # | Problem | Current | Why it doesn't scale | Solution | Effort |
|---|---|---|---|---|---|
| P1-1 | **Write amplification** | ≈ 5–15 `ConversationSession` UPDATEs/turn rewriting whole `answers` JSON; indexed `updated_at auto_now` prevents HOT updates; `_persist_no_resume_profile` twice; full `user.save()` copies answers into `professional_links` (`profile_sync.py:135-157`) | WAL, IO, vacuum and index churn scale with writes × row size | Accumulate changes in memory, **one** session write per turn (`update_fields`), User/profile writes only when fields change; drop `updated_at` index or index a coarser column | M |
| P1-2 | **Whole-table reads per text turn** | All active Locations (`candidate_flow.py:1446`), regex over all Skills (`:1535-1539`), all JobFunctions loop (`profile_edit.py:135`), `iexact` on unindexed names | O(catalog) CPU + IO on every message | Load catalogs into a per-process cache refreshed every N minutes (Aho–Corasick / dict lookup); `lower(name)` or trigram indexes | M |
| P1-3 | **Inactivity: one ETA task per message** | `inactivity.py:43-77`, countdown on `whatsapp_chat` | At 5M active users ≈ 5M unacked ETA messages held in worker RAM; stale tasks wake and query DB | Store `remind_at` on the session; sharded Redis ZSET (or DB index) + sweeper every 1–5 s | M |
| P1-4 | **Ordering by lock, not by lane** | Busy-wait lock (`cache_store.py:149-164`) blocks a worker slot ≤ 30 s; timeout re-enqueues at queue tail, dropped after 3 retries; TTL can expire mid-turn; release not atomic | Burns concurrency, reorders and drops messages under load | Partition by `hash(wa_id)` into ordered lanes (single active consumer per lane); keep lock as safety with fencing token + atomic Lua release | L |
| P1-5 | **External I/O inside the conversation lock** | Meta sends with sleeps, Claude (≤ 90 s), media download/upload under lock (`services.py:340-394`, `tasks.py:157-172`, `orchestrator.py:584-609`) | Lock hold time = slowest dependency; worker slots idle | Transactional **outbox**: turn writes outbound intents; outbound workers send. LLM/media work happens before taking the lock, result applied as an ordered event | L |
| P1-6 | **Prefork workers for I/O-bound chat** | `--concurrency 3–8`, prefork | Each turn waits on network 2–5 s; 1 process ≈ 0.2–0.5 turns/s [E] | gevent/eventlet pool (or async) for chat/outbound after removing `threading.local` assumptions; prefork kept for Whisper | M |
| P1-7 | **No DB pooling** | `CONN_MAX_AGE=0`, no pgbouncer | New connection per task/request; `max_connections` exhausted when scaled out | PgBouncer (transaction mode) + `CONN_MAX_AGE` | S |
| P1-8 | **Delivery statuses in PG on the chat queue** | 1 INSERT with raw JSON per status (`tasks.py:232-271`) ≈ 3 per outbound message; mixed changes drop statuses (`views.py:73-76`) | Largest write stream in the system, competes with chat | Separate low-priority queue; batch insert; store only failures + latest status; daily partitions, drop instead of DELETE | M |
| P1-9 | **Duplicate/unnecessary Claude calls** | Up to 3 identical calls per typed message (`candidate_flow.py:1593/1659/1675`); Sonnet default; no session reuse; no 429 handling | Cost × 3, rate-limit exhaustion | Memoize per turn; Haiku for small extraction; shared `requests.Session`; LLM queue with token-bucket and retry on 429/529 | S–M |
| P1-10 | **Resume pipeline triple I/O + retry bug** | 2 Meta downloads, 3 Azure writes, no content-hash dedupe, retries after user moved on, no time limit | 3× bandwidth/storage; duplicate messages; flow hijack | Pass bytes once; single blob write; SHA-256 dedupe (port from `anthropic-api-cost`); no task retry after user notified; time limit | M |
| P1-11 | **Missing indexes / constraints** | `WhatsAppMediaAttachment.whatsapp_media_id` (queried 4–5×/voice note), `User.mobile`, `Job(is_active,is_open,show_on_website,created_at)`; no unique active session per `wa_id`; no unique (candidate, job) application; duplicate indexes on `processed_at`/`created_at` | Seq scans grow with table size; duplicate rows under races | Add indexes `CONCURRENTLY`; partial unique index `(wa_id) WHERE status <> 'abandoned'`; unique application | S |
| P1-12 | **Slack ERROR handler is synchronous** | `requests.post(timeout=10)` per ERROR log (`slack_logger.py:146-163`) on worker threads | An outage that logs errors stalls every worker | Async/queued log shipping, rate-limited, PII-scrubbed | S |
| P1-13 | **Whisper shares slots with resume/LLM** | resume, transcript-only Claude, OCR all on `whatsapp_voice` | Expensive model slots idle on HTTP waits | Separate `whatsapp_resume`/LLM pool | S |

### P2 — required for multi-million

| # | Problem | Solution |
|---|---|---|
| P2-1 | Single Meta phone number | Multi-number routing: conversations are pinned to the number the candidate messaged; per-number token buckets; acquisition (links/QR) spread across numbers |
| P2-2 | Single PostgreSQL primary | Shard conversation-path tables by `hash(wa_id)` (Citus or app-level); keep global catalogs/jobs replicated |
| P2-3 | RabbitMQ throughput & ordering at ≥ 100K msg/s | Kafka (or RabbitMQ Streams/super streams) partitioned by `wa_id` for the inbound lane |
| P2-4 | CPU Whisper | GPU workers with batching, or managed STT for overflow; voice caps per user/step |
| P2-5 | Job matching scans 500 jobs/request in Python | Precomputed candidate→job candidate sets (Postgres FTS/pgvector or a search index), cached per profile hash |
| P2-6 | Single VM per env | Container orchestration with queue-based autoscaling (AKS or Azure Container Apps) |

### P3 — optimizations
Unused Redis snapshot (remove or use as read-through cache), duplicate indexes, phone numbers in blob paths and INFO logs, `capacity.py` stale, promo media re-upload per process, `ALLOWED_HOSTS='*'`, `CORS_ORIGIN_ALLOW_ALL`, Swagger on, `MASTER_PASSWORD` claim.

---

## Defining "5 Million Concurrent"

| Metric | Value | Note |
|---|---|---|
| A. Registered users | ≥ 5M | storage/index size driver |
| B. Concurrent active conversations | 5M | sessions with activity in the current window |
| C. Messages/sec | depends on message interval | calculated below |

**Amplification factors** (static code reading [E]):
- Replies per inbound message ≈ **1.5** (1–2 typical, up to 7 for job lists).
- Delivery statuses per outbound ≈ **3** (sent, delivered, read).
- So webhook requests per inbound ≈ 1 + 1.5 × 3 = **5.5**.

| Scenario | Calculation | Inbound msg/s | Outbound to Meta (×1.5) | Webhook req/s (×5.5) |
|---|---|---|---|---|
| 1 — Normal (1 msg / 5 min) | 5,000,000 / 300 | **16,667** | 25,000 | 91,667 |
| 1b — Quiet (1 msg / 10 min) | 5,000,000 / 600 | 8,333 | 12,500 | 45,833 |
| 2 — High (1 msg / 60 s) | 5,000,000 / 60 | **83,333** | 125,000 | 458,333 |
| 3a — Burst: 1% in 10 s | 50,000 / 10 | 5,000 (on top of baseline) | 7,500 | 27,500 |
| 3b — Burst: 5% in 30 s | 250,000 / 30 | 8,333 | 12,500 | 45,833 |
| 3c — Burst: 10% in 60 s | 500,000 / 60 | 8,333 | 12,500 | 45,833 |
| 3d — Broadcast reply: 20% in 60 s | 1,000,000 / 60 | 16,667 | 25,000 | 91,667 |

**Design point used below:**
- **Normal load:** Scenario 1 plus a burst of type 3d, about **33,000 inbound messages/s**.
- **Stress load:** Scenario 2, about **83,000/s**.

Scenario 2 means everyone is typing every minute, which is unusual for an onboarding bot. Real per-user message intervals are **[U]**; they can be measured from the `elapsed_ms` and `last_inbound_at` data already logged and stored.

---

## Django / Webhook Scalability

| Question | Answer (code) |
|---|---|
| Stateless? | Async mode: **yes**. Sync mode: no — full turn on request thread |
| Heavy processing / media download / Claude / Whisper on request? | Async: none. Sync: all of them (resume pipeline inline) |
| DB / Redis on request? | Async: none. Sync: yes, including the lock |
| Waits for RabbitMQ? | Yes, `.delay()` publishes synchronously per message; no publisher confirms → a broker crash after TCP write can lose it |
| Time to 200? | **[U]** not measured; async path ≈ JSON parse + N publishes [E: 5–20 ms] |
| Meta retries? | Non-2xx → Meta retries (up to ~7 days per Meta docs). 503 on enqueue error retries the whole batch; dedupe absorbs repeats |
| Idempotency correct? | Dedupe yes; **loss-safety no** (P0-1). Rate limit runs before claim, so duplicates count toward the limit |

**Can you run 10 / 100 / 1,000 Django instances behind a load balancer?**
- **10: yes**, if `WHATSAPP_WEBHOOK_SYNC=false`, `REDIS_URL` is set everywhere, and PgBouncer is in front of Postgres.
- **100 or 1,000: not as-is.** The webhook would be fine, but:
  1. `CONN_MAX_AGE=0` with no pooler. 1,000 instances × 3 workers = 3,000+ connections and connection churn.
  2. The local-disk resume fallback (`resume_parse.py:449`) only works on one host.
  3. Per-process LocMem if `REDIS_URL` is missing.
  4. The deploy pipeline targets one VM and restarts everything at once.
  5. Sync DRF plus the WAF per request costs about 3–5 ms of CPU each [E]. At 458K req/s that is roughly **1,400–2,300 cores** just to accept webhooks.

  A minimal ack endpoint (plain Django view or a small Go/Rust ingress) that verifies HMAC, writes to the durable log and returns is about 10× cheaper [E].
- **Sticky sessions:** not needed. There's no server-side session for WhatsApp traffic.

---

## Database Scalability (PostgreSQL)

**Tables and growth**

| Table | Growth | Pressure |
|---|---|---|
| `whatsapp_bot_conversationsession` | 1 row/conversation, **5–15 UPDATEs/turn**, JSON `answers` rewritten each time, ~8 indexes incl. `updated_at` | **Hottest write table** |
| `whatsapp_bot_processedwhatsappmessage` | 1 INSERT/inbound; duplicate `processed_at` index | append-heavy |
| `whatsapp_bot_whatsappdeliverystatus` | ≈ 4.5 INSERTs/inbound with raw JSON; duplicate `created_at` index | **largest append stream** |
| `whatsapp_bot_whatsappmediaattachment` | per media; no index on `whatsapp_media_id` | read-amplified per voice note |
| `whatsapp_bot_whatsappworkerprofile`, `accounts_user`, profile, M2M skills/locations | rewritten on many turns; `User.mobile` unindexed | write + seq-scan risk |
| `jobs_candidatejobapplication` | per apply; no unique (candidate, job) | duplicates under races |
| `jobs_job`, `web_app_skills/location/jobfunction` | read-heavy; no useful indexes for hot filters | seq scans per turn |

**Load estimate** at the normal design point (33K inbound msg/s) [E]:

| Quantity | Current code (≈ 40 SQL, ≈ 10 writes/turn) | After P1-1/P1-2 (≈ 8 SQL, ≈ 2 writes/turn) |
|---|---|---|
| Queries/s | 1.3M | 264K |
| Writes/s (rows) | 330K + 150K status inserts | 66K (statuses moved off hot path) |
| WAL (≈ 4 KB/session write, ≈ 0.5 KB other) [E] | > 1 GB/s | ≈ 150–250 MB/s |
| Connections (without pooler) | thousands | PgBouncer: few hundred server connections per primary |

**Sizing a single primary.** A well-tuned large primary sustains very roughly 20–50K small write transactions/s, and sustained WAL above about 100–200 MB/s stresses replication and backups [E; benchmark with `pgbench` using this schema].

| Concurrent users (1 msg/min, 2 writes/turn) | Writes/s | Single primary? |
|---|---|---|
| 100K | 3.3K | ✔ easily |
| 500K | 16.7K | ✔ with tuning, close to comfortable limit |
| 1M | 33K | ✖ borderline → shard |
| 5M | 167K | ✖ needs ≈ 8–16 shards |

**Evolution path** (sharding is justified only by the numbers in the table above):
1. **Now:** fix writes and indexes, add PgBouncer, move delivery statuses off the hot path.
2. **100K:** one managed primary with HA standby and PITR, plus a read replica *for analytics and admin only*. Fix the router so turn reads always hit the primary.
3. **500K:** daily partitions on append-only tables; use `pg_partman` and drop partitions for retention instead of running `DELETE`.
4. **1M+:** shard conversation-path tables by `hash(wa_id)`, either Citus distributed tables or app-level routing. Jobs, catalogs and the employer side stay on a separate, non-sharded cluster, replicated as reference tables.

**Vacuum and indexes**
- Frequent JSON row rewrites cause bloat. After moving to one write per turn, set `fillfactor≈80` on sessions to allow HOT updates, and drop the `updated_at` index or replace it with a partial or coarser one.
- Tune autovacuum per table: lower `autovacuum_vacuum_scale_factor` on sessions.

## Database Partitioning

| Table | Strategy | Why |
|---|---|---|
| Delivery status | **Time (daily)**, retention 7–30 days by partition drop | append-only, queried by recency/wamid |
| Processed message ids | **Time (daily)**, 7-day retention (covers Meta retry window) | append-only; dedupe needs recent only |
| Webhook/raw event log (new, if added) | Time (hourly/daily) | replay/debug |
| Conversation events / message history (if added) | Time + shard by `wa_id` | queried per conversation recently |
| `ConversationSession` | **Shard by `wa_id` hash**, not time | point lookups per conversation, update-heavy |
| Candidate activity / application history | Time (monthly) | analytics |
| `CandidateJobApplication` | none until > 100M rows; then by candidate hash with the user | joins with jobs/users |

---

## Redis Scalability

| Key (`cache_store.py` unless noted) | Class | TTL | If Redis restarts / key lost |
|---|---|---|---|
| `wa:conversation:{wa_id}` | **Distributed lock** | 180 s | Concurrent turns possible until next acquire; lock may expire mid-turn (no extension, non-atomic release) |
| `wa:rate:in:{wa_id}` | Rate limit (fixed window) | 60 s | Counters reset (fail open) |
| `wa:rate:out:global` | Rate limit — **single hot key** | 60 s | Reset; at 125K sends/s one key = one shard CPU |
| `wa:voice:seq/done:{scope}` | Ordering state | 24 h | In-flight voice notes wait 120 s then proceed (possible reorder) |
| `wa:inact:sched:{id}:{token}` | Dedupe of ETA scheduling | ~65 s | Duplicate ETA tasks (sends still guarded by DB token) |
| `wa:session:{wa_id}` | Cache (write-only, unused) | 1 h | Nothing |
| `wa:block:{wa_id}` (`blocklist.py`) | Cache | 60 s | DB fallback |
| `wa:promo_media_id` | Cache | 7 d | Re-upload image |
| `wa:demo:*` (`demo_outbox.py`) | Demo state, up to 8 MB media in Redis | 1–24 h | Demo timeline lost |
| voice metrics (`voice_metrics.py`) | Counters | 14 d | Metrics gap |

- **Redis is not the source of truth for candidate state. Keep it that way.**
- These must stay in PostgreSQL: session step and answers, inactivity tokens, idempotency records, voice-applied status, and outbound intents (once the outbox is added).
- **Ops estimate at 33K msg/s** [E]: about 20 ops per turn, about 660K ops/s. Remove the lock busy-polling and the unused snapshot, and spread rate-limit keys across numbers, and that drops to about 200–300K ops/s. That load fits a 6–12 shard Redis Cluster (Azure Cache for Redis Premium/Enterprise) with replicas.
- **Algorithms:**
  - Per-number and per-conversation limits: fixed-window counters are acceptable.
  - Meta per-number throughput: use a token bucket (atomic Lua) so the limit is exact.
  - Locks: one atomic Lua release, plus a fencing token checked on write (for example compare `session.version`).

---

## RabbitMQ / Celery Scalability

**What exists now**
- **Topology:** auto-declared classic durable queues: `celery`, `whatsapp_chat`, `whatsapp_voice`, `whatsapp_outbound`. No dead-letter exchange, no max-length, no quorum queues, no publisher confirms.
- **Broker:** a single node with **no volume in compose**, so recreating the container loses queued messages.
- **Beat:** there is no Celery beat service in compose, so the retention purge never runs locally.
- **Workers:** all prefork. Chat concurrency 3 (`.env`), voice 1; voice has prefetch 1, everything else prefetch 4.

**Broker traffic per inbound message** [E]:

| Source | Messages |
|---|---|
| `process_inbound` | 1 |
| Inactivity ETA | 1 |
| Status tasks | 4.5 |
| Voice re-publishes | 0–60 per waiting note |
| **Total** | **≈ 6.5** |

At 33K msg/s that is about **215K publishes/s plus deliveries**. Moving statuses to batch insert and inactivity to a sweeper brings it to about 1.2 per message, roughly **40K/s**.

**Is RabbitMQ still right?**
- **Up to about 1M concurrent (≈ 6–16K msg/s after the fixes): yes.** Use a 3-node cluster with quorum queues for chat and outbound, a dead-letter exchange per queue, and publisher confirms. For per-conversation ordering, use a consistent-hash exchange into N chat queues with single-active-consumer (§ Ordering).
- **From about 50–100K msg/s with per-key ordering, Kafka (Azure Event Hubs Kafka API) starts to pay off.** Ordering per partition keyed by `wa_id` is built in, replay and lag metrics are first-class, and it handles hundreds of thousands of messages per second per cluster. RabbitMQ Streams/super streams is the lower-change alternative to evaluate first. Either is justified **only** for the inbound lane at the 1M→5M phases.

**Recommended queues** (each justified by a different resource profile):

| Queue | Why separate | Pool |
|---|---|---|
| `whatsapp_inbound` (ordered lanes) | per-conversation ordering, latency-critical | gevent/async, many |
| `whatsapp_voice` | GPU/CPU heavy, long | prefork, prefetch 1, GPU nodes |
| `whatsapp_llm` (resume parse, extraction) | external rate-limited, 3–30 s | gevent, concurrency = LLM budget |
| `whatsapp_outbound` | Meta per-number throughput, backpressure | gevent, token bucket per number |
| `whatsapp_status` | high volume, low priority, batchable | small pool, batch inserts |
| `whatsapp_reminders` | sweeper-driven, bursty | small pool |
| `maintenance` (purge, metrics) | isolation | 1 worker |

`whatsapp_matching` and `whatsapp_notifications` are **not** justified yet. Matching takes about 5 queries and is CPU-light once cached; notifications are outbound messages.

**Celery settings to add:**
- `task_acks_late=True` and `task_reject_on_worker_lost=True` (idempotent tasks only).
- `worker_prefetch_multiplier=1` for long tasks.
- Global `task_soft_time_limit` and `task_time_limit`, so chat tasks have one too.
- `broker_transport_options={'confirm_publish': True}`.
- A dead-letter exchange on every queue.
- `task_default_delivery_mode=persistent`.
- Result backend: keep it disabled.

---

## Conversation Ordering

**Current behaviour**
- Ordering relies on the conversation lock alone.
- Two messages for one user on two workers run in **lock-acquisition order, not arrival order**.
- A lock timeout re-enqueues the message at the tail of the queue (reordering it) and drops it after 3 retries.
- Meta's message `timestamp` is ignored.
- Text and voice are not ordered relative to each other. Voice notes are ordered among themselves only (Redis tickets, re-publish polling).

**Recommended design** (preserves business logic):

```
webhook ─► key = hash(wa_id) mod N ─► lane N (Kafka partition | RabbitMQ consistent-hash queue with single-active-consumer)
                                        │  one consumer per lane at a time; lanes processed in parallel
                                        ▼
                              per-lane worker: process events for a wa_id strictly in order
                              (async I/O lets one worker interleave *different* wa_ids, never the same one)
                                        │
            long work (Whisper/LLM/resume) ─► separate pools ─► result re-enters the SAME lane as an event
                                                              ("transcript_ready seq=7") and is applied in order
```

- **Sequence:** assign a per-conversation sequence at enqueue time. It can come from a Redis INCR, or from Meta's timestamp plus message id as a tie-break. The lane consumer applies events in sequence and buffers early arrivals for a short window.
- **Safety net:** keep the DB lock or version as a fencing check. Each session write uses `UPDATE … WHERE id=? AND version=?`.
- **N:** start with 64–256 lanes. Throughput scales with lanes × async concurrency, not with lock waits.

## Idempotency

**Target flow** (replaces `claim_message`):

```
1. INSERT inbound_event(message_id UNIQUE, wa_id, payload, status='received') ON CONFLICT DO NOTHING
   — done by the webhook or the first consumer; duplicate → ack and stop
2. Lane consumer: UPDATE status='processing', lease_until=now()+60s WHERE message_id=? AND (status='received' OR lease expired)
3. Handle turn → in ONE transaction: session changes + outbound intents (outbox rows) + status='done'
4. Outbound workers send outbox rows with idempotency key (message_id, n); Meta-side duplicates prevented by checking outbox status
```

| Failure | Behaviour |
|---|---|
| Meta retry | Unique insert rejects it |
| Worker crash or broker redelivery | Lease expires, then the event is reprocessed |
| Partial work | Rolled back with the transaction, so it isn't left half-applied |
| Duplicate replies | Prevented by the outbox |

- **Retention:** 7 days via daily partitions, which covers Meta's retry window.
- **Redis:** may add a fast pre-check (`SET NX` with a 24 h TTL), but the DB constraint stays the authority.

---

## Whisper Scalability

**Implementation facts**
- Model: `small`, device `cpu`, `int8`, 4 threads, beam 1, VAD on (`settings.py:391-420`, `.env`).
- Loaded once per process via `worker_process_init` ✔.
- Language detection pass for non-trusted locales.
- Full decode before the 180 s trim, with no byte cap.

**Measured [M]** on the laptop CPU, model under memory pressure:
- Transcription took 4–8 s for 5–10 s notes, so **RTF ≈ 0.8**.
- One observation showed RTF ≈ 3 when the machine was swapping.
- Detection added 10–15 s per note before it was skipped for Hindi.
- Throughput on server CPUs or GPUs is **[U]**.

**Load scenarios.** Assumption [E]: each voice user sends one 15 s note within a 10-minute onboarding window.

| Voice users | Notes/s (= users ÷ 600) | Audio-seconds per second (×15) | CPU workers at RTF 0.8 (4 threads each) | CPU cores | RAM at 0.65 GB per worker | GPUs at 100 audio-s/s per GPU [E, **unbenchmarked**] |
|---|---|---|---|---|---|---|
| 5% = 250K | 417 | 6,250 | 5,000 | 20,000 | 3.3 TB | ≈ 63 |
| 10% = 500K | 833 | 12,500 | 10,000 | 40,000 | 6.5 TB | ≈ 125 |
| 20% = 1M | 1,667 | 25,000 | 20,000 | 80,000 | 13 TB | ≈ 250 |

Scenario 2 with 10% of messages as voice is far heavier: 8,333 notes/s × 15 s = 125,000 audio-s/s, which is about 1,250 GPUs at the assumed rate. That traffic shape needs product limits as well as hardware.

**Conclusions**
- **CPU faster-whisper is realistic only to roughly the low thousands of concurrent voice users.**
- Beyond that you need either **GPU workers** (faster-whisper with batched inference, `float16`, one model per GPU process) or **managed speech-to-text for overflow**. Managed STT costs roughly $0.006–0.024 per audio minute at list prices [E]: 12,500 audio-s/s is about $75–300 per minute.
- **Product limits** (preserve logic, cap cost):
  - Voice only on the steps that need it.
  - Maximum 30–60 s per note, rejected before download using Meta's `file_size`.
  - A per-user voice-minute budget.
- **Required benchmark:** RTF and throughput per instance type (for example Azure `D8s v5` CPU vs `NC A10/T4` GPU), with 1/4/8/16 parallel streams, Hindi and English, measured with the existing `whisper_completed` logs.

**Target design:**

```
whatsapp_voice queue (prefetch 1)
  └► voice workers: download (size-capped) ─► transcribe ─► emit transcript event to the conversation lane
       • GPU pool (primary) · CPU pool (overflow/dev)
       • Azure upload moved AFTER transcription, async, optional (WHATSAPP_STORE_VOICE_AUDIO)
       • queue-depth autoscaling; reject/ask-to-type when backlog > SLO
```

## Claude / LLM Scalability

**Call sites** (all synchronous `requests.post` with no session reuse and no 429/529 retry, `accounts/resume_parse.py:333-342`):

| Call | Trigger | Tokens [E] | Latency |
|---|---|---|---|
| Resume parse, text mode | each resume | ≈ 800 schema + 1–5K resume in, ≤ 4K out | **3.6–4.2 s [M]** (Haiku 4.5) |
| Resume parse, PDF fallback | text extraction failed or HTTP ≥ 400 | whole PDF | [U] |
| Experience extraction | experience voice; typed text where regex fails (`candidate_flow.py:1340`) — **up to 3× per message** | ≈ 200 + ≤ 2K in, ≤ 1K out | [U] |

**Load estimate.** Assumption [E]: 20% of 5M users upload a resume within 1 hour of onboarding, and 5% of turns need extraction.
- **Resumes:** 1,000,000 / 3,600 = **278 req/s**. At about 4 s each, that's about 1,100 in flight.
  - At Haiku 4.5 list prices ($1 per million input tokens, $5 per million output, verify current pricing): 278 × (4K × $1 + 1.5K × $5) / 1M = **≈ $3.2/s ≈ $11.5K/hour** during that peak hour.
- **Extraction:** 33K msg/s × 5% = **1,650 req/s** before dedupe. With up to 3 calls per message today, the worst case is about 5K req/s.
- **Rate limits:** **[U]** for your account. Standard Anthropic tiers are far below 100K requests per minute, so this needs enterprise limits or provisioned throughput (Anthropic, AWS Bedrock or GCP Vertex), plus a client-side token bucket.

**Changes (all justified):**
1. Dedupe calls within a turn.
2. Use regex/rules first and call the LLM only on misses. This exists today but is called repeatedly.
3. Use Haiku for small extraction instead of Sonnet as the code default.
4. Content-hash cache for resumes (already built on `anthropic-api-cost`).
5. Put LLM work on its own queue, outside the conversation lock, with a concurrency budget and retry/backoff on 429/529.
6. Reuse a shared HTTP session.
7. Use structured output (tool use or a JSON schema) to cut retries.
8. Batching (the Message Batches API) only for non-interactive backfills, never for the live chat.

---

## WhatsApp / Meta API Scalability

**Current outbound path**
- Sends go synchronously from inside the chat and voice workers (`services.py:304-397`): Graph `v21.0`, one phone number id, a static token.
- **Retries:** 408/429/5xx with `time.sleep` inside the worker. 400/401/403 count as fatal.
- **Global cap:** fixed window; anything over it is dropped.
- **Ordering:** implicit, because sends happen in sequence inside the turn.
- **The `whatsapp_outbound` queue exists but is almost unused,** and `send_outbound_payload` has no callers.

**External limits** (Meta documentation; confirm in Business Manager):
- **Throughput:** 80 msg/s per number by default, up to 1,000 msg/s per number on request.
- **Messaging tiers** limit **business-initiated** conversations (templates, reminders after 24 h). They don't limit replies inside the 24 h customer-service window.
- **Reminders sent after 24 h need approved templates.** This affects inactivity and re-engagement design.

**Numbers required** at 1,000 msg/s per number [E]:

| Concurrent (1 msg/5 min / 1 msg/min) | Outbound msg/s | Numbers |
|---|---|---|
| 10K | 50 / 250 | 1 / 1 |
| 100K | 500 / 2,500 | 1 / 3 |
| 500K | 2,500 / 12,500 | 3 / 13 |
| 1M | 5,000 / 25,000 | 5 / 25 |
| 5M | 25,000 / 125,000 | **25 / 125** (+ burst headroom) |

**Target design:**

```
turn transaction ─► outbox rows (to, payload, number_id, seq, idempotency_key)
   └► outbound workers (gevent), partitioned by number_id
        • per-number token bucket (Lua) sized below Meta limit (e.g. 90%)
        • per-conversation order preserved: one in-flight send per wa_id
        • 429/5xx → requeue with jittered backoff (never sleep in the turn); honour Retry-After
        • 400/401/403 permanent → mark failed, alert (token expiry = page on-call)
        • queue depth > threshold → backpressure: slow inbound lanes, defer non-essential messages (job cards) first
```

## External Dependencies

| Service | Synchronous? | Rate limit | Retry | Timeout | Bottleneck at target |
|---|---|---|---|---|---|
| Meta Graph send | Yes, in turn & under lock | 80→1,000 msg/s/number; tiers for business-initiated | 4 attempts in-process with sleep; global cap drops | 15 s | **Yes — numbers** |
| Meta media download | Yes (2 calls) | [U] | Task retry (re-downloads) | 30 s + 60 s | Bandwidth; no size cap |
| faster-whisper | Yes, in voice worker | Hardware | 2 task retries | soft 120 s | **Yes — compute** |
| Claude | Yes, in chat/voice workers (and webhook if SYNC) | Account tier [U] | **None** for 429/529 | 35 s (`.env`) / 90 s default | **Yes — limits & cost** |
| Azure Blob | Yes, sync SDK, new client per call | Storage account ≈ 20K req/s/account (Azure docs) [E] | SDK defaults [U] | SDK defaults [U] | Medium — use several accounts/containers at multi-M; resume triple writes |
| OCR | Stub (disabled) | — | — | — | No |
| Slack logging | Yes, on ERROR | Slack webhook limits | No | 10 s | **Yes — stalls workers during incidents** |

---

## Storage Scalability

**Current**
- **Voice and ID documents** go to `whatsapp/{wa_id}/{purpose}/…` in the main container (`azure_media.py:59-63`). They are **kept forever**, and the blob path contains the phone number.
- **Resumes** are stored as 3 copies.
- **No lifecycle rule exists in code.** The cleanup cron only handles `temp/`, `law-reports/` and Excel files (`crons.py:66-69`).
- **`WhatsAppMediaAttachment` rows are never purged.**

**Estimates [E]:**

| Item | Calculation | Volume |
|---|---|---|
| Voice note (Opus ≈ 16 kbps) | 15 s ≈ 30 KB | — |
| 5M users × 2 notes/day | 10M × 30 KB | **300 GB/day ≈ 110 TB/year** if kept forever |
| With 30-day lifecycle | 300 GB × 30 | ≈ 9 TB steady state |
| Resumes, 1M × 200 KB | ×3 copies today / ×1 after fix | 600 GB / 200 GB |
| Delivery-status rows if kept as raw JSON (≈ 1 KB) | 25K outbound/s × 3 × 3,600 | ≈ 270 GB per peak hour — **do not store raw** |

**Recommended**
- **Lifecycle:** delete or archive voice audio after 30 days (or immediately once the transcript is applied, if not legally required). Move ID documents to the cool tier and delete per the retention policy. Keep resumes while attached to a profile.
- **Paths:** use opaque ids in blob paths instead of phone numbers.
- **Access:** short-lived SAS links for staff access.
- **CDN:** not needed for inbound media.
- **Partitioning:** spread across storage accounts by hash at multi-million scale.

**Network [E], Scenario 2:**
- Webhook ingress: about 458K × 1.5 KB ≈ 690 MB/s ≈ **5.5 Gbps**.
- Graph egress: about 125K × 1 KB ≈ **1 Gbps**.
- Voice at 10% of messages: 8.3K × 30 KB × 2 (download plus upload) ≈ **4 Gbps**.

## Memory / In-process State

| Location | Safe for many instances? |
|---|---|
| `sop_loader.load_sop` `@lru_cache` (`sop_loader.py:50-63`) | Yes (read-only; needs coordinated restart on SOP change) |
| `role_catalog.load_catalog` `@lru_cache` | Yes |
| `asr._model` global (`asr.py:67-68`) | Yes (intended, one per process) |
| `http_client._session` per PID | Yes |
| `welcome_image._cached_media_id` | Yes (duplicate uploads only) |
| `cache_store._held_locks` `threading.local` | Yes under prefork; **must be revisited for gevent** (greenlet-local) |
| LocMem cache fallback (`settings.py:188-194`) | **No** |
| Local-disk resume fallback `MEDIA_ROOT/temp` (`resume_parse.py:449`) | **No** |
| Demo media in Redis | Works but wrong place for blobs |

## Celery Pools

| Task | Bound | Pool |
|---|---|---|
| `process_inbound` | IO (DB, Meta) | Inbound lanes, gevent |
| `process_status_update` | DB | Status pool, batched |
| `send_inactivity_reminder` | DB + Meta | Reminder pool (sweeper-driven) |
| `process_*_voice` | **CPU/GPU** | Voice pool, prefork, prefetch 1 |
| `process_experience_transcript`, `process_whatsapp_resume` | EXT (Claude), IO | LLM pool, gevent |
| `send_outbound_payload` | EXT (Meta) | Outbound pool, per-number buckets |
| `purge_processed_messages` | DB | Maintenance |

## Inactivity Reminder Scalability

**Current**
- Every inbound message bumps a token and publishes a 60 s countdown task (`inactivity.py:43-77`) to `whatsapp_chat`.
- Stale tasks wake up and run 2 SELECTs before doing nothing.
- At 5M active users that means **about 5M countdown messages held unacked in worker memory** at any moment. Their prefetch isn't bounded, they occupy chat workers, and they compete with live turns.

**Recommended** (fits the existing token logic):
- On each turn, write `remind_at = now + N` on the session; the inactivity token already exists.
- Mirror it into a Redis ZSET, sharded by `hash(wa_id)`, with the score set to `remind_at`.
- A sweeper per shard runs every 1–5 s: `ZRANGEBYSCORE … LIMIT 1000`, then claims each entry (`ZREM` succeeds only for one sweeper), enqueues to `whatsapp_reminders`, and the worker re-checks the DB token before sending.
- **Redis loss:** rebuild the ZSET from the indexed DB column `remind_at` (partial index `WHERE remind_at IS NOT NULL`).
- **Templates:** reminders after the 24 h window must use templates.

This gives O(due reminders) work instead of O(messages).

## Rate Limiting (layers)

| Layer | Limit | Where |
|---|---|---|
| IP | Webhook: allow only Meta IP ranges + valid HMAC (no IP rate limit, Meta retries); APIs: existing nginx zones | WAF/LB |
| Signature | Mandatory HMAC (P0-5) | Webhook |
| Phone number (`wa_id`) | 30 msg/min (exists) → move **after** dedupe, reply "slow down" once instead of silent drop | Inbound lane |
| Conversation | Max 1 in-flight turn (lane), max N queued events; drop/merge rapid duplicates | Lane |
| Meta API | Token bucket per business number at 90% of Meta limit | Outbound |
| LLM | Global token bucket (RPM and TPM), per-user daily cap | LLM pool |
| Whisper | Per-user voice-minutes/day, max note length, queue-depth gate | Voice pool |
| Loop guard | Max bot messages per conversation per minute (prevents SOP loops) | Outbound |

## Load Balancing
`Internet → Azure Front Door/App Gateway (WAF) → L7 LB → webhook pool` (stateless, no sticky sessions).
- **Keep ModSecurity off the webhook path.** It's already excluded for the body; HMAC is the right control.
- **Health checks** must check the broker as well as the DB. Today `/health/` checks only the DB (`health.py:6-13`), and the deploy check accepts HTTP 500.

## High Availability

| Component | Current | Failure mode | Impact | Required redundancy |
|---|---|---|---|---|
| Webhook/Django | 1 VM (dev/prod); 2 replicas locally | VM down | All inbound lost until Meta retries | ≥ 3 instances across zones behind LB |
| PostgreSQL | Single, no PITR | Disk/VM loss | Total data loss since last deploy | Managed HA (zone-redundant standby) + PITR |
| Redis | Single, no auth/persistence | Restart | Locks/rate limits reset (state safe) | Replicated tier, AOF optional |
| RabbitMQ | Single node, no volume (compose) | Restart | Queued messages lost | 3-node quorum queues, persistent |
| Whisper | 1 worker, no VM service | Crash | Voice stalls; task lost (no acks_late) | Pool across nodes, acks_late |
| Claude | Single provider | Outage/429 | Resume/extraction fail | Retry queue, fallback model/provider, degrade to manual questions |
| Meta | External, 1 number | Number restricted/quality drop | Bot silent | Multiple numbers, quality monitoring |
| Azure Blob | Single account (LRS?) [U] | Region issue | Media store fails; ID flow stops | ZRS/GRS; flows must continue without blob |
| LB/WAF | Containers on same host | Host down | Total | Managed, zone-redundant |

## Disaster Recovery

- **Current:**
  - Backups are `pg_dump` at deploy time only, kept on the same VM. RPO is days, and RTO is several hours with manual work.
  - Redis and RabbitMQ have no persistence.
  - The secrets live in `.env` on the VM. Key Vault is used on dev according to earlier work, but this branch's `settings.py` has no Key Vault code [U].
- **Targets:**
  - **PostgreSQL:** RPO ≤ 5 min with PITR, RTO ≤ 1 h.
  - **Queues:** RPO 0 for accepted webhooks, via a durable log plus Meta's retries.
  - **Redis:** best effort, since it is rebuildable.
  - **Blob:** GRS or ZRS.
  - **Configuration:** infrastructure-as-code (Bicep/Terraform) and Key Vault, with a quarterly restore drill.

## Observability

**Missing:** metrics, tracing, structured logs and correlation ids.
- Logging is plain text plus a synchronous Slack handler.
- The only telemetry is the `elapsed_ms` log per turn and the voice metrics Redis counters.

**Add:**
- **Metrics** (Prometheus/Azure Monitor):
  - webhook req/s and p50/p95/p99;
  - enqueue failures;
  - queue depth and age per queue or lane;
  - task duration per task;
  - lock wait time;
  - DB latency and connections;
  - Redis latency;
  - Whisper RTF and queue wait;
  - LLM latency, tokens, 429s and cost;
  - Meta send latency, error codes and per-number throughput;
  - outbound msg/s;
  - duplicates skipped;
  - messages dropped by rate limits (must be visible, never silent).
- **Logs:** JSON with `trace_id`, `wa_message_id`, `conversation_id` (session id), `candidate_id`, `task_id` and `lane`. Use a hashed `wa_id` instead of the raw phone number.
- **Tracing:** OpenTelemetry across webhook, enqueue (propagate context in task headers), worker, DB, LLM/Whisper and Meta.
- **Alerts:** queue age over the SLO, webhook 5xx, Meta 401 (token), drop counters, and DB replication lag.

## Security

| Finding | Location | Severity |
|---|---|---|
| Signature check skipped (secret unset) | `views.py:18-20` | P0 |
| Unauthenticated demo API acting on any `wa_id`; session PII exposed | `demo_views.py:113-182`, `.env` | P0 |
| ModSecurity body rules disabled for `/whatsapp/demo/` | `deploy/waf/REQUEST-900…conf:14-17` | P0 (with above) |
| Employers/staff can list **all** WhatsApp sessions | `viewsets.py`, `permissions.py:15-16` | P1 |
| DRF default permission AllowAny; no throttling | `settings.py:113-122` | P1 |
| `ALLOWED_HOSTS='*'`, `CORS_ORIGIN_ALLOW_ALL`, Swagger on, `MASTER_PASSWORD` token claim | `settings.py:31,159`, `urls.py:117`, `accounts/services.py:119` | P1 |
| Phone numbers in INFO logs on every message/status; blob paths contain phone; Slack payloads include request POST | `orchestrator.py:298-372`, `tasks.py:266-271`, `azure_media.py:60`, `slack_logger.py:108-116` | P1 (PII/DPDP) |
| Redis without auth; RabbitMQ `guest`; management port published | compose | P1 |
| Verify-token compare not constant-time | `views.py:49` | P3 |

---

## Capacity Model

All at the **normal design point**, Scenario 1 plus a 3d burst (33K inbound msg/s). Scenario 2 is shown in brackets. **[M]** measured, **[E]** estimate, **[U]** unknown.

| Component | Requirement | Per-unit capacity | Units | Basis |
|---|---|---|---|---|
| Webhook | 183K req/s [458K] | Minimal ack: ≈ 2K req/s per core [E] | ≈ 90 cores [230] | [E]; benchmark the ack endpoint |
| Inbound/chat | 33K turns/s [83K] | After fixes, ≈ 10–20 ms CPU/turn [E]; I/O-bound 2–5 s wall [M Graph 0.6–1.5 s/send] | 330–660 cores; in-flight = 33K × 3 s = **100K concurrent turns** → gevent ≈ 500/process ⇒ ≈ 200 processes [E] | Little's law |
| Voice | 6.3K–25K audio-s/s (5–20% voice users) | RTF 0.8 CPU/4 threads [M laptop]; GPU [U] | See Whisper table | Benchmark required |
| LLM | 278 resume/s + ≈ 1.6K extract/s | 3.6–4.2 s per resume [M] | ≈ 1.1K + ≈ 3K in-flight | Account limits [U] |
| PostgreSQL | 66K writes/s, 264K reads/s (after fixes) | ≈ 20–50K writes/s per primary [E] | **2–4 shards** [8–16] | pgbench [U] |
| Redis | 200–300K ops/s (after fixes) | ≈ 50–100K ops/s per shard [E] | 6–12 shards | [E] |
| Broker | ≈ 40K msg/s (after fixes) | RabbitMQ quorum ≈ 10–30K msg/s per queue/node [E] | Cluster + hash lanes; Kafka at [83K] | [E] |
| Meta outbound | 50K msg/s [125K] | 1,000 msg/s per number (Meta) | **50 numbers** [125] | Meta docs |
| Storage | ≈ 300 GB/day voice (2 notes/user/day) | — | 9 TB with 30-day lifecycle | [E] |
| Network | ≈ 2–3 Gbps ingress [5.5], 0.4 Gbps Graph egress [1], 1.5 Gbps voice [4] | — | — | [E] |

**Phase 0 (current) capacity, estimated** [E]:
- **Local compose:** 3 chat processes × about 0.3 turns/s ≈ **1 turn/s**. That's about **60 users** messaging once a minute, or about 300 messaging every 5 minutes.
- **Voice:** 1 process at about 26–29 s per note [M], so about **2 notes/minute**.
- **SYNC mode:** 6 gunicorn workers, about the same ceiling.
- **Dev/prod VM:** [U]. It may be zero if the WhatsApp queues are unconsumed.

---

## 5 Million Concurrent User Architecture

```
                        Meta (125–250 business numbers, conversations pinned per number)
                                         │ HTTPS + HMAC
                         Azure Front Door / App Gateway (WAF; webhook path = HMAC + Meta IPs)
                                         │
                    Stateless ack service (verify HMAC → append to durable log → 200 in < 50 ms)
                                         │
                Kafka / Event Hubs (partition key = wa_id; ~256–1024 partitions)   ← RabbitMQ until ~1M
                                         │
        ┌──────────────────┬─────────────┴──────────┬─────────────────────┬────────────────────┐
  Conversation lane     Voice pool (GPU         LLM pool (rate-       Outbound senders      Status/batch +
  workers (gevent,      node pool, prefetch 1,  limited, retry        (per-number token     reminder sweepers
  in-order per wa_id,   size-capped download)   queue, cache)         bucket, outbox)       (Redis ZSET shards)
  SOP logic unchanged)        │                        │                     │
        │                     └── results re-enter the conversation lane ────┘
        │
  PostgreSQL: conversation shards by hash(wa_id) (Citus/app-level) · core cluster (users, jobs, employers) + replicas
  Redis Cluster (locks w/ fencing, rate limits, ZSET timers, caches) · Azure Blob (lifecycle, multiple accounts)
  OpenTelemetry · Prometheus/Azure Monitor · structured logs · KEDA autoscaling on lag
```

## Kubernetes / Orchestration
- **Not needed for Phase 1–2.** Use a few VMs, or **Azure Container Apps** (built-in KEDA scaling on queue depth), with separate apps for webhook, chat, voice, outbound and sweeper.
- **Needed from about 500K concurrent.** By then there are 6+ workload types scaling independently on different signals:
  - webhook: req/s;
  - lanes: consumer lag;
  - voice: queue depth;
  - outbound: per-number backlog;
  - LLM: in-flight budget.
- At that point you also need **GPU node pools**, spot capacity for the CPU overflow voice pool, and rolling deploys. **AKS with KEDA** fits:
  - Horizontal Pod Autoscaler on CPU for the webhook.
  - KEDA on Kafka lag or RabbitMQ queue length for workers.
  - A separate GPU node pool with taints for Whisper.

---

## Required Code Changes (business logic preserved)

1. **Idempotency.** Replace `claim_message` with an inbound event record carrying a status and lease, committed atomically with session changes.
2. **Celery.** Load `acks_late`, `reject_on_worker_lost`, prefetch, time limits and publisher confirms. Add dead-letter exchanges.
3. **Outbox.** Persist outbound intents in the turn transaction and send them from outbound workers. Remove sleeps from the turn.
4. **Outbound rate limiting.** A per-number token bucket that defers sends instead of dropping them. Multi-number support with conversations pinned to a number.
5. **Writes.** One `ConversationSession` write per turn. Persist User/profile changes only when they change. Remove the duplicate `_persist_no_resume_profile`.
6. **Catalogs.** Cache Skills, Locations and JobFunctions in memory. Index the lookups.
7. **Inactivity.** Replace per-message countdown tasks with `remind_at` plus a sharded ZSET sweeper.
8. **Ordering.** Ordered lanes by `hash(wa_id)`. Atomic lock release with a fencing version check. Voice and LLM results re-enter the lane.
9. **LLM.** Per-turn memo, Haiku for extraction, shared HTTP session, 429/529 retry queue, run outside the lock. Port the resume SHA-256 cache.
10. **Resume task.** Pass the downloaded bytes through, write one blob, no retry after the user has been notified, add a time limit.
11. **Voice.** Size cap from Meta's `file_size`, upload after transcription, separate LLM work from the Whisper pool.
12. **Webhook.** Fail closed without an app secret. Handle mixed status and message changes. Minimal ack path.
13. **Statuses.** Separate queue, batched writes, store failures and the latest status only.
14. **Indexes and constraints.** `whatsapp_media_id`, `User.mobile`, the Job filters, a unique active session per `wa_id`, a unique application. Drop the duplicate indexes.
15. **Config.** Fail fast without `REDIS_URL`. Remove the local-disk resume fallback. Lock down the demo API.
16. **Logging.** Structured logs with trace ids, PII scrubbing, async Slack.

## Required Infrastructure Changes

| Phase | Infrastructure |
|---|---|
| 1 | Managed PostgreSQL (HA, PITR) + PgBouncer · managed Redis · 3-node RabbitMQ (quorum, DLX) · versioned systemd/compose units per pool · Key Vault · monitoring stack |
| 2 | Container Apps/AKS · multiple webhook instances across zones · GPU benchmark · Meta throughput upgrade + 3–5 numbers |
| 3–4 | Conversation DB sharding · Redis Cluster · Kafka/Event Hubs for lanes · GPU node pool · multi-storage-account media |
| 5 | 125–250 numbers · 8–16 PG shards · 256–1024 partitions · multi-zone everything; consider multi-region active/passive |

---

## Load Testing Strategy

- **Harness:** replay signed Meta webhook payloads (k6 or Locust) against the real ingress.
- **Stubs:**
  - Meta Graph: a fake server with configurable latency (0.6–1.5 s [M]), 429/5xx rates and per-number limits.
  - Claude: a stub with 3–5 s latency and 429s.
  - Whisper: run real models on a dedicated benchmark first, then stub at the measured rate.
- **Traffic model:** realistic SOP scripts per synthetic user: text, buttons, 10% voice, 20% resume, at think time T.

| Stage (concurrent users) | 1K | 5K | 10K | 25K | 50K | 100K | 250K | 500K | 1M | 2.5M | 5M |
|---|---|---|---|---|---|---|---|---|---|---|---|
| Inbound msg/s at 1/min | 17 | 83 | 167 | 417 | 833 | 1.7K | 4.2K | 8.3K | 16.7K | 41.7K | 83.3K |
| Prerequisite phase | 0→1 | 1 | 1 | 2 | 2 | 2 | 3 | 3 | 4 | 5 | 5 |

**Measure at each stage:**
- p50/p95/p99 webhook ack time and end-to-end reply latency;
- throughput, error rate, and drops (must be 0 silent drops);
- queue depth and age;
- DB CPU, connections, writes/s, WAL MB/s and replication lag;
- Redis CPU and ops;
- broker depth;
- worker CPU and memory;
- Whisper RTF and queue wait;
- LLM latency and 429s;
- outbound latency and per-number utilization.

**Pass criteria (proposed):**
- Webhook p99 < 200 ms.
- Text reply p95 < 3 s, measured from Meta in to the first reply sent.
- Voice reply p95 < 25 s.
- 0 lost messages, verified by reconciling inbound ids against processed ids.
- Duplicate replies < 0.01%.
- DB CPU < 60% and queue age < 5 s at the stage target.
- Sustain each stage for 30 minutes and pass a 2× burst test.

**Fail:** any silent drop, unbounded queue growth, or a p95 breach for more than 5 minutes.

## Failure Testing Strategy

| Inject | Expected (after fixes) | Verify |
|---|---|---|
| Kill webhook instance | LB drains; Meta retries any non-acked | No lost ids |
| Kill worker mid-turn | Lease expiry → reprocess; outbox prevents double reply | Reconciliation |
| RabbitMQ node down | Quorum queue failover; publisher confirms retry | No lost tasks |
| Redis restart | Locks/timers rebuilt from DB; no state loss | Reminders still fire |
| PG replica down | Turn path unaffected (primary only) | — |
| PG primary failover | Short write outage; webhook still acks into log; lanes resume | RPO/RTO met |
| Meta timeout / 429 / 5xx | Outbound backoff, no worker stall, no drop | Outbox drains |
| Claude timeout / 429 | LLM retry queue; flow degrades to manual question | No stuck sessions |
| Whisper crash | Task redelivered (acks_late), user notified after N failures | — |
| Azure failure | Voice/resume continue without blob; ID flow retries later | — |
| Duplicate webhook | Unique insert → skipped | 1 reply |
| Network partition (worker↔Redis) | Fencing version prevents stale writes | No corrupted session |

---

## Scaling Roadmap

| Phase | Target | Infrastructure | Database | Redis | Broker | Workers | Whisper | LLM | Monitoring | Load test | Expected bottleneck |
|---|---|---|---|---|---|---|---|---|---|---|---|
| **0 Baseline** | ~60–300 active [E] | 1 VM / laptop compose, SYNC on | Single PG, no pool | Single, optional | Single node, no volume | 3 chat prefork, 1 voice | CPU small | sync, dup calls | none | none | Everything; message loss |
| **1 — 10K** | 167 msg/s @1/min | ≥ 2 app VMs, managed PG/Redis, 3-node RabbitMQ | PITR, PgBouncer, indexes, 1 write/turn | Managed, auth | quorum + DLX, confirms | gevent chat 2–4 × 200, outbound pool | CPU pool 10–40 workers; benchmark GPU | dedupe, Haiku, retry queue | metrics + JSON logs + alerts | 1K→10K | Meta 1 number at 250 msg/s ✔; voice CPU |
| **2 — 100K** | 1.7K msg/s | Container Apps / small AKS, KEDA | Large primary + HA + analytics replica; partition logs | Primary+replica | RabbitMQ hash lanes (64) | lanes 10–20 procs; status batcher; reminder sweeper | GPU pool (≈ 5–15 GPUs [E]) or STT overflow | enterprise limits | OTel tracing | 25K→100K | Meta: 3 numbers; PG writes ≈ 3–7K/s |
| **3 — 500K** | 8.3K msg/s | AKS + GPU node pool | Single primary near limit → plan shards; FTS/vector for matching | Cluster 3–6 shards | RabbitMQ cluster or Streams; evaluate Kafka | lanes 50–100 procs | GPU ≈ 30–60 [E] | provisioned throughput | SLO dashboards | 250K→500K | PG write ceiling; Meta 13 numbers |
| **4 — 1M** | 16.7K msg/s | Multi-zone AKS | **Shard conversation tables (4–8)** | Cluster 6–12 | **Kafka/Event Hubs for lanes** | 100–200 procs | GPU ≈ 60–125 [E] | multi-provider fallback | capacity forecasting | 1M | Meta 25 numbers; LLM limits |
| **5 — 5M** | 83K msg/s peak | Multi-zone, consider multi-region | 8–16 shards + core cluster | 12–24 shards | Kafka 256–1024 partitions | 500+ procs | GPU ≈ 125–1,250 depending on voice policy [E] | dedicated capacity | full tracing, chaos drills | 2.5M→5M | Meta numbers (125–250), voice cost |

---

## Priority Matrix

| Priority | Items |
|---|---|
| **P0** (now) | P0-1 loss-safe idempotency · P0-2 Celery acks/prefetch · P0-3 SYNC off · P0-4 demo API off · P0-5 enforce HMAC · P0-6 VM worker units + dependency install · P0-7 require Redis · P0-8 never drop outbound · P0-9 backups/PITR |
| **P1** (100K–1M) | write reduction · catalog caching + indexes · inactivity sweeper · ordered lanes · outbox (I/O out of lock) · gevent pools · PgBouncer · status pipeline · LLM dedupe/limits · resume pipeline fix · constraints · async logging · voice/LLM pool split |
| **P2** (multi-M) | multiple Meta numbers · PG sharding · Kafka lanes · GPU Whisper · matching index · orchestration |
| **P3** | snapshot removal, duplicate indexes, PII in paths/logs, config hardening, `capacity.py` refresh |

## Final Recommendations

1. **Fix correctness before scale.** Most P0 items are configuration changes or small code changes, and they currently cause silent loss or open security holes.
2. **Measure before buying.** The three missing measurements that drive cost:
   - Real message intervals per user, from production logs.
   - Whisper throughput on a candidate GPU versus CPU SKU.
   - Your Anthropic and Meta throughput limits.
3. **Reduce work per message by 5–10×** (writes, catalog scans, duplicate LLM calls, status rows, ETA tasks). That pushes the first database scaling step from about 50K to about 500K concurrent users.
4. **Treat Meta numbers and voice policy as product and business decisions.** No code change lifts the per-number limit or the per-minute cost of speech-to-text.
5. **Adopt Kafka, sharding and Kubernetes only at the phase where the numbers above require them.**

---

## Most Important Output

| Area | Current State | Problem | Required Change | Priority | Scale Impact |
|---|---|---|---|---|---|
| Webhook | Fast enqueue in async mode; **SYNC on in `.env`**; HMAC skipped; mixed status/message changes drop statuses | Web tier runs whole turns; forged events possible | SYNC off, enforce HMAC, minimal ack → durable log | P0 | Unblocks horizontal scale of ingress |
| Django | 3 sync gunicorn workers, `CONN_MAX_AGE=0`, LocMem fallback, local-disk fallback | Connection storms; per-process state | PgBouncer, require Redis, remove disk fallback, separate ack service | P0/P1 | 10 → 1,000 instances safely |
| PostgreSQL | Single, 25–110 SQL & 5–15 session rewrites/turn, missing indexes, no PITR | Write/WAL ceiling ≈ 50K concurrent today [E] | 1 write/turn, indexes, partitions, PITR → shard by `wa_id` at ≥ 1M | P0/P1/P2 | Largest internal ceiling |
| Redis | Locks (busy-wait, non-atomic release), fixed-window limits, one hot global key | Hot key; lock races | Lua token buckets per number, fencing, ZSET timers, cluster | P1 | Removes hot spot, enables timers |
| RabbitMQ | Single node, classic queues, no DLX/confirms, ~6.5 msgs per inbound | Loss on restart; ETA/status flood | Quorum + DLX + confirms; remove ETA/status traffic; Kafka lanes at ≥ 1M | P0/P1/P2 | 5× fewer broker msgs |
| Celery | acks on receipt, prefetch 4, prefork chat, no time limits on chat | Loss on crash; idle slots | acks_late, prefetch 1, gevent chat/outbound, pool split | P0/P1 | 50–100× in-flight capacity per host |
| Whisper | CPU small/int8, 1–2 procs, RTF ≈ 0.8 [M], shares queue with LLM/resume | Compute ceiling ≈ hundreds of notes/min | GPU pool (benchmark), size caps, voice limits, separate pool | P1/P2 | Determines voice cost/feasibility |
| Claude | Sync, inside lock, up to 3 dup calls, no retry, Sonnet default | Cost ×3, rate limits, lock stalls | Dedupe, Haiku, LLM queue with budget, resume hash cache | P1 | 3× cost cut, no lock stalls |
| Azure Blob | Sync uploads, 3 resume copies, voice kept forever, phone in path | Storage/bandwidth growth, PII | Single write, lifecycle 30 d, opaque paths | P1 | ≈ 110 TB/yr → ≈ 9 TB steady |
| Meta API | **1 phone number**, Graph v21.0, static token | 1,000 msg/s max ⇒ ≈ 40K users @1 msg/min | Throughput upgrade, multi-number pinning, template reminders | P2 (plan now) | **Hard external ceiling** |
| Outbound | Sync sends in turn with sleeps; global cap **drops** | Lost replies, stalled workers | Outbox + per-number token bucket + backpressure | P0/P1 | Reliable delivery at any scale |
| Inactivity | 1 countdown task per message on chat queue | ≈ 5M held ETA tasks at target | `remind_at` + sharded ZSET sweeper | P1 | O(due) instead of O(messages) |
| Monitoring | Plain logs + sync Slack; no metrics/tracing | Blind to loss and lag; Slack stalls workers | Metrics, OTel, JSON logs with ids, async alerts | P0/P1 | Required to run any load test |
| Security | Open demo API, HMAC off, `ALLOWED_HOSTS='*'`, PII in logs, Redis no auth | Abuse, PII exposure | Lock down demo, HMAC, scope session API, scrub PII, auth on infra | P0/P1 | Prevents catastrophic incident |
| Deployment | Single VM per env, unit files not versioned, deps never installed, same-host backups | Workers may not run; no HA | Versioned units per pool → Container Apps → AKS + KEDA | P0→P2 | Enables independent scaling |

## TOP 10 CHANGES WE SHOULD MAKE FIRST

1. **Turn off `WHATSAPP_WEBHOOK_SYNC` and `WHATSAPP_DEMO_API`, and set and enforce `WHATSAPP_APP_SECRET`** (`views.py:18-20`, `demo_views.py`). These are config-level, fix exposure and web-tier exhaustion, and take hours.
2. **Make message handling loss-safe.** Replace the pre-processing claim (`orchestrator.py:305`) with a leased status committed with the state change, and load `acks_late`, `reject_on_worker_lost` and prefetch=1 in `settings.py`. 1–3 days.
3. **Version and fix the VM deployment.** Systemd units for dedicated chat, voice and outbound workers, Celery beat, the `requirements/pip.txt` install, and a real `/health/` check that includes the broker and Redis. 1–2 days.
4. **Never drop an outbound message.** A per-number token bucket that defers to `whatsapp_outbound`, then an outbox so no Meta send happens inside the conversation lock (`services.py:304-397`). 3–5 days.
5. **Cut database work per turn.** One session write per turn, no duplicate profile persist, cached catalogs instead of whole-table scans (`candidate_flow.py:1446, 1535`), and missing indexes and unique constraints. 1–2 weeks.
6. **Replace per-message inactivity countdown tasks** with `remind_at` plus a ZSET sweeper (`inactivity.py:43-77`). 3–5 days.
7. **Fix the LLM path.** Dedupe repeated `_parse_experience_answer` calls, use Haiku for extraction, run outside the lock on an LLM queue with 429 retry, and port the resume SHA-256 cache. Also fix the resume task's triple I/O and retry bug (`tasks.py:456-566`). 1 week.
8. **Production data safety.** Managed PostgreSQL with PITR and HA, PgBouncer, managed Redis with auth, and a 3-node RabbitMQ with quorum queues and dead-letter exchanges. 1–2 weeks (infrastructure).
9. **Observability before load testing.** JSON logs with trace and message ids, metrics for queue age, drops, lock wait, Meta, LLM and Whisper, and Slack logging moved off the hot path. 1 week.
10. **Run the three missing measurements, then the staged load test** (1K → 10K → 100K): Whisper GPU vs CPU benchmark, real per-user message intervals, and Meta and Anthropic throughput limits. At the same time, start the business process for Meta throughput upgrades and additional phone numbers, which is the longest-lead item for multi-million scale.
