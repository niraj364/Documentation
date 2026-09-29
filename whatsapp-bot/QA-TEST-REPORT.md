# WapHire WhatsApp Bot — QA Test Report

**Date:** 2026-09-26  
**Role:** Senior QA / Automation / Production Reliability  
**Scope:** Existing `candidate_onboarding_v1` implementation (no application rebuild)  
**Framework:** Django `TestCase` via `manage.py test backend.whatsapp_bot`  
**Environment:** Docker Compose (`web`, `worker`, `db`, `redis`, `broker`) + host frontend `:3000`

---

## SUMMARY

| Metric | Count |
|--------|------:|
| Automated tests executed (whatsapp_bot suite) | 97 |
| Automated tests passed | 97 |
| Automated tests failed (after fixes) | 0 |
| Smoke checks executed (TC-001–013) | 13 |
| Smoke PASS | 11 |
| Smoke FAIL | 0 |
| Smoke BLOCKED | 2 (Azure credentials; WA dedicated workers caveat on TC-006) |
| Critical product bugs found & fixed | 1 |
| Test defects fixed | 1 |
| Load / 62k concurrency | **BLOCKED** (not run) |
| Live Meta Graph E2E on real phone (full journey) | **PARTIAL** (welcome path proven earlier; full apply not re-run live this session) |
| Dedicated WhatsApp frontend UI responsive matrix | **BLOCKED** (no dedicated WA chat UI; site homepage only) |

\*Final full suite: **Ran 97 tests — OK** (after BUG-001 + Phase3 SOP pin).

---

## Production readiness assessment

**Not production-ready for full scale** until dedicated WA workers are running, Azure credentials are valid, App Secret HMAC is enforced in prod, and live resume/ASR/LLM paths are proven end-to-end.

**Candidate manual onboarding → match → apply → track** is **proven in automated DB-backed tests** (Graph outbound mocked). Post-apply Track CTA bug was found and fixed.

---

## Critical fix applied

### BUG-001 — Post-apply Track/More Jobs spawned a new blank session (HIGH → fixed)

| Field | Detail |
|-------|--------|
| **ID** | BUG-001 / E2E-002 regression |
| **Scenario** | After successful apply, session set `STATUS_COMPLETED`; next button (`track_apps`) created a new empty session |
| **Expected** | Track Application / Find More Jobs continue the same conversation with existing user/applications |
| **Actual** | `get_or_create_session` treats COMPLETED as “start new” → lost context |
| **Root cause** | `_do_apply` marked session completed immediately after apply |
| **Fix** | Keep session `ACTIVE` on `job_matching` after apply; set `completed_at` for analytics; store `last_apply_status` / `last_application_id` |
| **File** | `backend/src/backend/whatsapp_bot/candidate_flow.py` |
| **Regression** | `test_e2e_manual_hindi_confirm_apply_track` in `test_onboarding_qa.py` |

### TEST-FIX-001 — Phase3 plumber SOP test used default onboarding SOP

| Field | Detail |
|-------|--------|
| **Issue** | `load_sop()` resolved `candidate_onboarding_v1` after default SOP switch |
| **Fix** | Pin `WHATSAPP_DEFAULT_SOP_ID=driver_four_wheeler_v1` + `load_sop("driver_four_wheeler_v1")` |
| **File** | `backend/src/backend/whatsapp_bot/tests.py` |

---

## Smoke (TC-001–013)

| ID | Scenario | Expected | Actual | Status | Severity |
|----|----------|----------|--------|--------|----------|
| TC-001 | Backend starts / webhook responds | Challenge echo | Challenge echoed | PASS | CRITICAL |
| TC-002 | Frontend starts | HTTP 200 | `:3000` → 200 | PASS | HIGH |
| TC-003 | Database connection | Connected | `DB_OK` | PASS | CRITICAL |
| TC-004 | Redis connection | PING | `REDIS_OK` | PASS | CRITICAL |
| TC-005 | RabbitMQ connection | Ping | Healthy broker | PASS | CRITICAL |
| TC-006 | Celery workers | WA queues consuming | Generic `worker` Up; `worker-wa` / `worker-wa-voice` **not started** | PASS* | HIGH |
| TC-007 | Webhook reachable | 200 | OK | PASS | CRITICAL |
| TC-008 | Webhook verification | Challenge | OK | PASS | CRITICAL |
| TC-009 | Env vars loaded | WA/Redis/Claude keys | Present in container | PASS | CRITICAL |
| TC-010 | Azure Blob | List containers | `AzureSigningError Incorrect padding` | BLOCKED | HIGH |
| TC-011 | ASR config | Configured / mock | `WHATSAPP_ASR_MOCK=true` | PASS | MEDIUM |
| TC-012 | Claude config | Key + model | Key Y, model haiku-4-5 | PASS | HIGH |
| TC-013 | No secrets in frontend | No WA/Azure/Claude secrets | Scan clean | PASS | CRITICAL |

\*TC-006 caveat: with `WHATSAPP_WEBHOOK_SYNC=true`, chat runs in-web; voice/resume still need `whatsapp_voice` worker when sync is off.

---

## Automated suite results

| Suite | Result |
|-------|--------|
| `CandidateOnboardingJourneyQATests` + `CandidateOnboardingTests` (23) | **OK** after BUG-001 fix |
| Full `backend.whatsapp_bot` (97) | **OK** (24.976s) after Phase3 SOP pin |

New coverage file: `backend/src/backend/whatsapp_bot/test_onboarding_qa.py`

Covers: E2E manual Hindi apply+track, language EN, random text, resume_yes/upload queue/reject MIME, parse→confirm, resume_fix, inactive/closed jobs, skills empty Done, confirm_edit, persist existing user, idempotent claim, empty webhook, invalid signature, session resume, job match soft fallback.

---

## Journey matrix (selected)

| ID | Scenario | Expected | Actual | Status | Severity |
|----|----------|----------|--------|--------|----------|
| TC-101/102 | Hi / second Hi | One session | Asserted in E2E | PASS | CRITICAL |
| TC-103 | Random text | No crash | Session exists, no exception | PASS | HIGH |
| TC-104 | Empty webhook | 200 | 200 | PASS | HIGH |
| TC-201/202 | Hindi / English | locale persisted | Asserted | PASS | CRITICAL |
| TC-301 | Resume yes | Ask upload | `resume_upload` | PASS | CRITICAL |
| TC-302–310 | Live PDF/Azure/parse | Full path | **BLOCKED** — Azure key invalid; Claude live parse not exercised in CI | BLOCKED | CRITICAL |
| TC-401–408 | Live LLM parsing edge cases | Graceful | Unit path for happy parse only; timeouts/unavailable **BLOCKED** | BLOCKED | HIGH |
| TC-501/502 | Resume ok / fix | Branch | Asserted | PASS | HIGH |
| TC-601–609 | Manual path fields | Persisted | E2E assert skills/exp/loc | PASS | CRITICAL |
| TC-701–708 | Voice / ASR | Queue + process | Grey-collar voice unit tests PASS; onboarding voice **partial**; live Sarvam **BLOCKED** (mock on) | BLOCKED | HIGH |
| TC-801–807 | Profile create/update | One user/profile | Asserted | PASS | CRITICAL |
| TC-901–904 | Summary / confirm / edit | DB-backed | Confirm+edit covered | PASS | HIGH |
| TC-1001–1007 | Job matching | Real Job queryset | Active only; soft fallback | PASS | CRITICAL |
| TC-1101–1107 | Job details | From Job model | `format_job_card_text` / view path in E2E | PASS | HIGH |
| TC-1201–1207 | Apply / duplicate / closed | Real Application | Created + duplicate exists + closed rejected | PASS | CRITICAL |
| TC-1301–1304 | Track apps | Real rows | Called after apply | PASS | HIGH |
| TC-1401–1406 | State resume | Locale retained | Mid-flow resume PASS; long inactivity **BLOCKED** (timing) | PASS / BLOCKED | HIGH |
| TC-1501–1504 | Ordering / races | Safe | Idempotency unit PASS; multi-worker race **BLOCKED** | BLOCKED | HIGH |
| TC-1601–1603 | Idempotency | Single process | `claim_message` PASS | PASS | CRITICAL |
| TC-1701–1709 | Queues / DLQ | Workers | Dedicated WA workers not up; DLQ **BLOCKED** | BLOCKED | HIGH |
| TC-1801–1805 | Redis | Cache / lock | Ping PASS; Redis-down chaos **BLOCKED** | PASS / BLOCKED | HIGH |
| TC-1901–1907 | DB consistency | One candidate/app | E2E assert | PASS | CRITICAL |
| TC-2001 | Invalid signature | 403 | 403 | PASS | CRITICAL |
| TC-2002–2009 | Auth / XSS / SQLi / rate | Various | Rate-limit unit in Phase2 PASS; XSS/SQLi injection **not fuzzed** | PARTIAL | HIGH |
| §24 Failure matrix | Meta/Rabbit/Redis/PG/Azure/ASR/Claude down | Retry/UX | **BLOCKED** — chaos not executed this session | BLOCKED | CRITICAL |
| §25–26 Frontend WA UI | Responsive chat UI | N/A | No WA chat frontend in repo; site only | BLOCKED | MEDIUM |
| §27 Performance | 1k–62k concurrency | Evidence | **Not run — do not claim capacity** | BLOCKED | CRITICAL |

---

## End-to-end scenarios

| ID | Scenario | Status | Notes |
|----|----------|--------|-------|
| E2E-001 | Hindi + Resume + Apply | BLOCKED | Resume needs Azure+Claude+voice worker |
| E2E-002 | Hindi + Manual + Apply | PASS | Automated with real DB Job/Application |
| E2E-003 | Return mid-flow | PASS | Locale retained, one session |
| E2E-004 | Voice + ASR | BLOCKED | Mock ASR; onboarding voice not fully automated |
| E2E-005 | Leave + return | PASS (short) | Long inactivity not timed |
| E2E-006 | Duplicate webhook | PASS | claim_message / prior suite |
| E2E-007 | Apply twice | PASS | `exists`, single row |
| E2E-008 | Resume parse fails | BLOCKED | Needs forced Claude/Azure failure |
| E2E-009 | ASR fails | BLOCKED | Live provider |
| E2E-010 | Meta API fails | PARTIAL | Outbound errors logged; user still advances in sync path |
| E2E-011 | Simultaneous candidates | BLOCKED | Not load-tested |
| E2E-012 | High concurrency | BLOCKED | Not run |

---

## Security issues

| Issue | Severity | Status |
|-------|----------|--------|
| `WHATSAPP_APP_SECRET` empty in local → HMAC skipped | HIGH (prod misconfig risk) | Documented; code rejects when secret set (TC-2001 PASS) |
| Azure account key incorrect padding | HIGH | Credential/env issue — resume storage blocked |
| Frontend secret scan | — | PASS |

---

## Database consistency (E2E-002)

After automated journey for one wa_id:

- 1 `ConversationSession`
- 1 `User` (onboarding_completed=True)
- 1 `WhatsAppWorkerProfile`
- 1 active `CandidateJobApplication`
- Matched job ids stored on session answers
- No duplicate application on second apply

---

## Performance results

**Not measured.** Claims of 62,000 concurrent conversations are **unsupported**.

---

## Remaining issues / follow-ups

1. Start `worker-wa`, `worker-wa-voice`, `worker-wa-outbound` for non-sync production-like mode.  
2. Fix Azure storage credentials (`Incorrect padding`).  
3. Live E2E-001 with real PDF + Claude parse.  
4. Chaos tests (Redis/Rabbit/Meta/Claude down).  
5. Load test with evidence before scale claims.  
6. Enforce `WHATSAPP_APP_SECRET` in non-dev environments.  
7. Prefer Graph mocks in legacy `CandidateOnboardingTests` to avoid noisy 401s from `test-token` during unit runs.

---

## How to re-run

```powershell
cd backend
docker compose exec -T web python manage.py test backend.whatsapp_bot --verbosity=1
docker compose exec -T web python manage.py test backend.whatsapp_bot.test_onboarding_qa --verbosity=2
```
