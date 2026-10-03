# WhatsApp @ ~1.2M users — WapHire Scale Architecture

**Status:** Living design for the EXISTING WapHire stack  
**Date:** 2026-09-26  
**Principle:** Evolve the monolith — do not rewrite, do not microservices-for-vanity.

This document adapts **reference** chat-system patterns (LB → many app servers →
Postgres / message store / Redis / object storage) to WapHire’s real stack.
It is **not** a claim about Meta/WhatsApp’s private internals.

Related ops docs: [`README.md`](./README.md) (current bot behaviour).

---

## 0. What we inspected (CURRENT system)

| Layer | Today |
|--------|--------|
| API | Django 4.2 + DRF, Gunicorn/WSGI |
| DB | PostgreSQL (`APP_DATABASE_URL`) |
| Auth | JWT (simplejwt) + Google + OTP |
| Async | **Celery 5 + RabbitMQ** (no Redis/Kafka/Channels yet) |
| Media | Azure Blob + SAS |
| Clients | Next.js candidate site + Vite employer portal |
| API prefix | `/waphire-api/v1/` |
| Outreach | Email inbox + IMAP poll (not WhatsApp) |
| WhatsApp bot | Restored Django app `backend.whatsapp_bot` (was missing source; docs + bytecode remained) |

**Reuse for WhatsApp:** Celery/RabbitMQ, User/`mobile`, Claude helpers, Azure Blob,
idempotency pattern (`ProcessedWhatsAppMessage`), SOP JSON, existing deploy (gunicorn + worker + beat).

**Do not break:** JWT REST APIs, jobs/candidates/outreach email, employer SPA, candidate site.

---

## 1. Capacity model (assumptions → numbers)

1.2M is **registered / cumulative** users, **not** simultaneous sockets.

**Product ask (this iteration):** ~**62,000 concurrent active WhatsApp conversations**.
That is **not** 62k messages at the same instant. See
`backend/whatsapp_bot/capacity.py` for the explicit symbols used in code/docs.

### 1.0 Concurrent-conversation envelope (62k)

| Symbol | Assumption | Rationale |
|--------|------------|-----------|
| `C` | 62,000 concurrent active conversations | Product target |
| `send_frac` | 1–3% of `C` send an inbound msg in a given second | Chatty onboarding bursts |
| `voice_frac` | 15–25% of sessions include ≥1 voice note | Grey-collar capture |
| `retry_overhead` | 1.15× | Meta webhook + outbound retries |

| Metric | Planning (mid) | Peak headroom | How derived |
|--------|----------------|---------------|-------------|
| Concurrent conversations | 62k | 62k | Given |
| Peak inbound msgs/sec | ~600–1,200 | **~1,400** | `C × send_frac × retry` |
| Webhook HTTP RPS | ~800–1,500 | **~1,800** | Inbound + statuses |
| Outbound Graph calls/sec | Cap to Meta | **≤100** (token bucket) | Fair queue on `whatsapp_outbound` |
| Celery chat concurrency | 32–64 | **64–128** | Chat turns ~100–500ms + Graph |
| Celery voice concurrency | 16–32 | **32–64** prefetch=1 | ASR+LLM 8–20s |
| Gunicorn replicas | 4–8 | **8–16** × 4 workers | Thin ACK only after Phase 1 |
| RabbitMQ throughput | ≥1k msg/s | **≥2k** headroom | Chat + voice + outbound |
| Postgres WA writes/sec | ~200–500 | **~800** | Session + idempotency; Redis in Phase 2 |

If the target is instead **62k DAU** (not concurrent), peak concurrent sessions fall to
~2k–4k and peak inbound to ~40–120 msg/s — reuse §1.2 below.

### 1.1 Assumptions (1.2M registered — maturity)

| Symbol | Assumption | Rationale |
|--------|------------|-----------|
| `U` | 1,200,000 registered users | Product target |
| `DAU_frac` | 5–8% of `U` touch WhatsApp in a day at maturity | Consumer chat apps often higher; hiring bots lower |
| `peak_frac` | 12–15% of DAU active in the peak hour | India evening peak |
| `msg_per_active_session` | 8–15 inbound msgs / completed onboarding | SOP length (language → role → voice → confirm) |
| `voice_frac` | 15–25% of sessions send ≥1 voice note | Grey-collar voice capture |
| `media_frac` | 5–10% sessions send image/doc | ID / resume stubs |
| `retry_overhead` | 1.15× | Meta webhook retries + our outbound retries |

Tune these after 2–4 weeks of production metrics; treat below as **planning envelope**.

### 1.2 Derived loads

| Metric | Conservative | Peak planning | How derived |
|--------|--------------|---------------|-------------|
| Registered users | 1.2M | 1.2M | Given |
| Daily active WA users (DAU) | 60k (5%) | 96k (8%) | `U × DAU_frac` |
| Peak concurrent sessions | ~1.2k–2.5k | ~3.5k–4k | ~(DAU × peak_frac) / hours_in_peak_window ≈ /3–4 |
| Peak inbound msgs/sec | ~25–40 | **80–120** | Peak sessions × msg rate / 3600 × retry |
| Peak voice notes/min | ~8–15 | **25–40** | Voice sessions in peak / ASR latency budget |
| Peak media uploads/min | ~3–8 | **15** | Media sessions in peak |
| Webhook HTTP RPS | ~30–50 | **150** | Inbound + status callbacks |
| Outbound Graph calls/sec | ~20–40 | **100** | Replies, buttons, lists (Meta rate limits apply) |
| Celery chat tasks/sec | ~25–40 | **100** | Prefer enqueue-all inbound |
| Celery voice workers (concurrent) | 4–8 | **16–32** | Voice notes × ASR+LLM latency (~8–20s) |
| Postgres writes/sec (hot) | ~40–80 | **200** | Session touch + idempotency + profile |
| Redis ops/sec (when added) | — | **5k–20k** | Session cache, rate limit, media-id cache |
| Queue throughput | RabbitMQ today | **≥500 msg/s** headroom | Chat + voice + outbound queues |
| Object storage | Azure | Grow with voice/ID media | ~50–200 KB voice × voice volume |

### 1.3 Worker / DB sizing (starting point)

| Component | Phase A (0–100k WA users) | Phase B (~1.2M registered) |
|-----------|---------------------------|----------------------------|
| Gunicorn API | 2–3 VMs / 4–8 workers each | 4–8 VMs behind LB |
| Celery `whatsapp_chat` | 2–4 concurrency | 16–32 |
| Celery `whatsapp_voice` | 2–4 (prefetch=1) | 16–32 dedicated nodes |
| Celery `whatsapp_outbound` | 2 | 8–16 (respect Meta TPS) |
| PostgreSQL | Primary | Primary + **read replica** for reports |
| Redis | Optional | Required (session cache + rate limits) |
| Message history store | Postgres JSON / tables | Postgres partitioned **or** Cassandra/Scylla if historical chat volume demands |
| Media | Azure Blob | Blob + CDN fronting public assets only |

---

## 2. Target architecture (adapted to WapHire)

```
                 ~1.2M REGISTERED USERS
                          │
                          ▼
               Meta WhatsApp Cloud API
                          │
                          ▼
              WAF / API Gateway / LB
                          │
            ┌─────────────┼─────────────┐
            ▼             ▼             ▼
        Gunicorn      Gunicorn      Gunicorn
        (Django)      (Django)      (Django)
            │             │             │
            └─────────────┼─────────────┘
                          │
                   Webhook Layer
            (verify + idempotency + enqueue)
                          │
                     RabbitMQ
                          │
         ┌────────────────┼─────────────────┐
         ▼                ▼                 ▼
   whatsapp_chat    whatsapp_voice    whatsapp_outbound
         │                │                 │
         ▼                ▼                 ▼
     SOP Engine         ASR             Graph API
         │        (faster-whisper)       (retry)
         ▼                ▼
   Conversation         Claude extract
         │                │
         └───────► Profile sync → User / WhatsAppWorkerProfile
                          │
                          ▼
                     PostgreSQL
                          │
              ┌───────────┼───────────┐
              ▼           ▼           ▼
            Redis      (optional)   Azure Blob
           cache     Cassandra/      media
                     Scylla for
                     high-volume
                     message log
```

### Mapping reference concepts → WapHire

| Reference idea | WapHire choice |
|----------------|----------------|
| Multiple chat servers | Multiple Gunicorn replicas (same Django app) |
| WebSocket presence | **Not required** — Meta owns the client connection |
| Postgres users | Existing `User` + org models |
| Cassandra messages | **Defer** until message volume justifies; start with Postgres + retention |
| Redis | Add for session hot path, Graph media-id cache, rate limits |
| S3/CDN | Keep **Azure Blob**; CDN only for public promo assets |
| Consistent hashing | Optional later for sticky session caches; Meta `wa_id` is the shard key |
| Rate limiting | Redis token bucket **per wa_id** + global outbound TPS |

---

## 3. Phased implementation (no big-bang rewrite)

### Phase 0 — Restore & wire (DONE in this iteration)
- Restore `backend.whatsapp_bot` from VCS
- Register in `INSTALLED_APPS`
- Mount `/waphire-api/v1/whatsapp/` (+ `/api/v1/whatsapp/`)
- Settings + `.env.example` for `WHATSAPP_*` / `WHISPER_*`
- Keep SOP voice/inactivity Celery tasks

### Phase 1 — Production hardening (IMPLEMENTED in codebase)
1. **Fast ACK:** webhook validates → enqueues `process_inbound` → HTTP 200  
2. Dedicated Celery queues: `whatsapp_chat`, `whatsapp_voice`, `whatsapp_outbound`  
3. Outbound Graph send with exponential backoff + Meta error taxonomy (429/5xx retry; 401/403 fatal)  
4. Delivery/read: `process_status_update` logs statuses (table/partition = Phase 2)  
5. Optional App Secret HMAC (`WHATSAPP_APP_SECRET` / `X-Hub-Signature-256`)  
6. Compose: default worker listens to all WA queues; `--profile wa-scale` for dedicated workers  
7. Observability: structured logs (queue lag / metrics exporters = follow-up)

**Not yet (after Phase 1):** Redis session cache, block list, human handoff inbox, partitioned idempotency table — see Phase 2 (now implemented).

### Phase 2 — Scale envelope (IMPLEMENTED in codebase)
1. **Redis** via `REDIS_URL` + Django cache (`cache_store.py`: session snapshot, promo media id, inbound/outbound rate limits, inactivity schedule coalesce)  
2. **Indexes + retention** on `ProcessedWhatsAppMessage` / delivery statuses; daily `purge_processed_messages` beat task; ops note for native monthly PARTITION  
3. **Optional read replica** (`APP_DATABASE_REPLICA_URL` + `WhatsAppReplicaRouter`)  
4. **Worker isolation:** `worker` = `celery` only (email/blog); `worker-wa` = chat+outbound; `worker-wa-voice` = voice prefetch=1  
5. **Delivery status table** `WhatsAppDeliveryStatus`  
6. **Block list** `WhatsAppBlockList` + outbound/inbound guards  
7. **Human handoff** session status `handed_off` (+ admin action)

**Still deferred:** Cassandra/Scylla message log (only if measured Postgres QPS demands it).

### Phase 3 — Product expansion (IMPLEMENTED in codebase)
1. **More SOP roles** — plumber / painter banks in `driver_four_wheeler_v1.json`  
2. **ID media** — image/document → Azure Blob via `azure_media.py` + `WhatsAppMediaAttachment`  
3. **Optional OCR** — `process_id_document_ocr` when `WHATSAPP_OCR_ENABLED=true`  
4. **24h re-engage** — `reengage.send_reengage_message` / `reengage_session` task (free-form inside window, Meta template outside via `WHATSAPP_REENGAGE_TEMPLATE_NAME`)  
5. **Employer conversation viewer** — read-only `GET /waphire-api/v1/whatsapp/sessions/` (+ detail)

---

## 4. Data ownership

| Data | Store | Retention |
|------|--------|-----------|
| Candidate/employer accounts | Postgres `User` | Permanent |
| Conversation SOP state | `ConversationSession` | Active + 90d archive |
| Idempotency | `ProcessedWhatsAppMessage` | 30–90d |
| Worker capture | `WhatsAppWorkerProfile` | Permanent / GDPR delete with User |
| Voice / media blobs | Azure | 90d unless attached to profile |
| Delivery status | New table (Phase 1) | 30d |
| Full chat transcript | Postgres first; migrate if needed | Product policy |

---

## 5. Failure handling

| Failure | Behaviour |
|---------|-----------|
| Duplicate webhook | `ProcessedWhatsAppMessage` unique `message_id` |
| ASR down | Retry Celery; fallback typed transcript prompt |
| Claude low confidence | Re-ask voice (existing min confidence setting) |
| Graph 401 | Alert; stop outbound until token rotated |
| Graph 429 | Backoff on outbound queue |
| User blocked | Persist block; no outbound |
| Worker crash mid-voice | `STATUS_PROCESSING_VOICE` + Celery acks_late redelivery |

---

## 6. Security

- Webhook: no JWT; protect with HTTPS + `WHATSAPP_VERIFY_TOKEN` + optional App Secret HMAC (`X-Hub-Signature-256`) — add in Phase 1  
- Never log full tokens or raw ID images in plaintext logs  
- PII: phone = `wa_id`; align delete path (already cleans WhatsApp tables on user delete)

---

## 7. What we are NOT doing

- Replacing Django with a Node “WhatsApp microservice” by default  
- Introducing Kafka/Cassandra on day one  
- Claiming this mirrors Meta’s internal WhatsApp architecture  
- Changing candidate/employer website UX as part of bot restore  

---

## 8. Immediate operator checklist

1. Set `WHATSAPP_TOKEN`, `WHATSAPP_PHONE_NUMBER_ID`, `WHATSAPP_VERIFY_TOKEN` in env  
2. `migrate` (apps `0001`, `0002`)  
3. Run `web` + **Celery worker** (voice + inactivity need worker)  
4. Point Meta callback to `https://<api>/waphire-api/v1/whatsapp/webhook/`  
5. Subscribe to `messages` (and later `message_status`)  
6. Prefer **permanent** system-user tokens over temporary Graph tokens  

---

## 9. Success metrics

- Webhook p99 ACK &lt; 200 ms (after Phase 1 enqueue)  
- Voice end-to-end p95 &lt; 25 s  
- Duplicate menu rate ≈ 0 (idempotency)  
- Outbound success ≥ 99% excluding user blocks  
- Queue lag &lt; 5 s for chat; &lt; 60 s for voice at peak  

---

*Owner: Backend platform. Update numbers after first production load test.*
