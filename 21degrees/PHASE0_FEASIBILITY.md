# 21DEGREES Voice Automation: Phase 0 Feasibility Report

Status: **source-code audit complete. Nothing in this report has been tested on a live call yet.**
Scope: System A (AI voice calling, CRM and email). System B (SEO/blog) has not been started, as the handoff requires.

Evidence labels used below:

| Label | Meaning |
|---|---|
| **CODE** | Seen in this repository's source code (path given) |
| **DOC** | Stated in vendor documentation only |
| **TESTED** | Run successfully by us. *Nothing has this label yet.* |
| **UNVERIFIED** | Not yet established |
| **ABSENT** | Searched for and not found in the code |

---

## 1. Decision summary

| Question | Answer |
|---|---|
| Base platform | **Dograh (this fork)**. Licence is BSD 2-Clause (`LICENSE`), so commercial use is allowed if the copyright notice is kept. |
| First calling channel | **Plivo** (the client already has a Plivo number). Dograh has a complete Plivo adapter with outbound dialling and two-way streaming audio (**CODE**). |
| MSG91 | **Deprioritised.** No adapter exists. It would need a new provider package, and its real-time two-way media support is **UNVERIFIED**. Reconsider only if Plivo's India pricing or compliance fails. |
| ChakraHQ | Use as **CRM only** for the MVP. Do not use it as the calling transport (see §5). |
| WhatsApp calling | Dograh already supports Meta's WhatsApp Business Calling API directly over WebRTC, but **inbound only** (**CODE**). Outbound WhatsApp is a later, separately quoted item. |
| Bolna | **Do not merge it.** It is MIT-licensed, but its telephony set (Exotel, Plivo, Vobiz, Twilio, SIP, FreeSWITCH) is a subset of Dograh's, and it has no MSG91 or ChakraHQ code. The only idea worth revisiting is `bolna/lid/` (language ID for Hinglish switching). |
| Fallback route | Dograh + **Exotel** or **Vobiz** (both are Indian carriers with existing adapters). If self-hosting fails entirely, use a managed platform (DoubleTick or Sarvam) as the price benchmark or fallback. |

---

## 2. Task A: Dograh source audit

### 2.1 Where calls start
- Provider contract: `api/services/telephony/base.py` (`initiate_call`, `get_call_status`, `get_call_cost`, `parse_status_callback`, `transfer_call`, …).
- Providers register themselves through `ProviderSpec` in `api/services/telephony/registry.py`. Rules for adding a provider: `api/services/telephony/providers/AGENTS.md`. A new provider needs one folder plus two lines elsewhere.
- Campaign dispatch: `api/services/campaign/campaign_call_dispatcher.py` → `dispatch_call()` resolves the provider (`get_provider_for_campaign`) and enforces the rate limit (`apply_rate_limit`) and concurrency (`acquire_concurrent_slot`).
- **Plivo outbound:** `api/services/telephony/providers/plivo/provider.py:53` `initiate_call` → `POST https://api.plivo.com/v1/Account/{auth_id}/Call/`, with `answer_url`, `hangup_url` and `ring_url` callbacks per workflow run.

### 2.2 How audio enters the engine and returns to the caller
- When Plivo fetches the answer URL, Dograh replies with XML:
  `<Stream bidirectional="true" keepCallAlive="true" contentType="audio/x-mulaw;rate=8000">wss://…</Stream>` (`plivo/provider.py:251`).
- The WebSocket is bound to a Pipecat `FastAPIWebsocketTransport` using `PlivoFrameSerializer` (`plivo/transport.py`, `plivo/serializers.py`, re-exported from the `pipecat` submodule). The same socket carries audio in both directions. That is all the live call needs.
- Pipeline entry: `api/services/pipecat/run_pipeline.py::run_pipeline_telephony`. Interruption, turn-taking and voicemail/answer detection are configured there (`_create_user_turn_*`, `_create_answer_supervisor`).
- Exotel works the same way, but with `StreamType=bidirectional` set when the call is dialled (`exotel/provider.py:158`).

### 2.3 Where model providers are configured
- `api/services/configuration/registry.py:90` `ServiceProviders`. Options relevant to India and low cost include **sarvam**, **soniox**, **deepgram**, **google / google_vertex** (Gemini), **groq**, **openrouter**, **smallest**, and **speaches** (a self-hosted, OpenAI-compatible server that can run faster-whisper STT and Kokoro/Piper TTS).
- Services are created in `api/services/pipecat/service_factory.py` (`create_stt_service`, …). Sarvam STT accepts Indian language codes. Deepgram Flux has a `hi` hint.

### 2.4 Structured extraction
- Per-node variable extraction: `api/services/workflow/pipecat_engine_variable_extractor.py`.
- End-of-call extraction, with a time budget and retries: `api/services/workflow/final_extraction.py`.
- Call outcome codes: `api/services/workflow/disposition_extraction.py` keeps `call_status` (how the call ended) separate from `call_disposition` (the business outcome). This maps directly to Part 10 of the brief.

### 2.5 Where results are stored and how webhooks work
- Results are stored on the workflow run (`gathered_context`, `usage_info`; models in `api/db/models.py`).
- The workflow **Webhook node** (`api/services/workflow/dto.py:836` `WebhookNodeData`) fires after the call from `api/tasks/run_integrations.py`.
- Delivery is **durable**: `api/tasks/webhook_delivery.py` stores a `WebhookDeliveryModel` row, retries with exponential backoff, dead-letters failed deliveries, sweeps on restart, and redacts PII in logs. This already covers brief items 13 and 14 (duplicate or delayed webhooks) on the sending side.

### 2.6 Campaign features that already exist (do not rebuild them)
- CSV import: `services/campaign/sources/csv.py`. This is the **only** source type (`source_sync_factory.py`).
- Retries on busy, no answer or voicemail, with a maximum count and a delay: `campaign_retry.py`, plus `retry_config` in `db/models.py`.
- Calling-hour windows by day of week and timezone: `campaign_orchestrator.py::_is_within_schedule`.
- A circuit breaker that pauses a campaign on provider errors: `circuit_breaker.py`.
- Pause, resume and status: `runner.py`. Recovery of calls interrupted mid-dispatch: `recover_stale_dispatches`.
- An admin UI for campaigns, runs, transcripts and recordings (`ui/`).

### 2.7 Gaps we must build (small, and kept outside Dograh's core where possible)

| # | Gap | Evidence | Plan |
|---|---|---|---|
| G1 | No do-not-contact or consent check before dialling | **ABSENT**: searching `services/campaign` for `do_not_call`, `dnc`, `suppression` or `opt_out` finds nothing | Add a pre-dispatch suppression check in `campaign_call_dispatcher.dispatch_call`, backed by a suppression list |
| G2 | No ChakraHQ CRM write-back | **ABSENT** | Small integration service that receives the Webhook-node payload, uses an idempotency key of `workflow_run_id`, validates the extraction, then updates ChakraHQ |
| G3 | No follow-up email | **ABSENT**: no SMTP, Gmail or SendGrid code | Same service: template-based email per outcome, with a send log keyed on `(workflow_run_id, template)` so each email goes out only once |
| G4 | Telephony cost is never recorded | `get_call_cost` is implemented for Plivo (`plivo/provider.py:254`) but has **no callers** | Call it after hang-up and store the result in `usage_info`. Note that the code labels Plivo's `total_amount` as USD; an Indian Plivo account may bill in INR, so check the currency |
| G5 | The schedule fails open: an invalid timezone **allows** calling | `campaign_orchestrator.py:233-238` | For India, change it to fail closed and enforce the permitted calling hours |
| G6 | No ChakraHQ source for campaigns | Only `csv` exists | MVP: export a CSV from ChakraHQ. Later: add a `chakrahq` sync source |

---

## 3. Task B: Provider capability matrix

| Capability | Plivo (chosen) | MSG91 | ChakraHQ / WhatsApp |
|---|---|---|---|
| Starting a call | **CODE** (`initiate_call`) | DOC only | Dograh: **inbound only** (`supports_outbound=False`, `providers/whatsapp/__init__.py`) |
| Live audio in | **CODE** (bidirectional `<Stream>`) | **UNVERIFIED** | **CODE** for inbound (WebRTC SDP, `whatsapp/routes.py`) |
| Live audio out | **CODE** (same socket) | **UNVERIFIED** | **CODE** for inbound |
| Interruption handling | **CODE** (Pipecat turn strategies) | n/a | **CODE** (same pipeline) |
| Call status webhooks | **CODE** (hangup/ring callbacks, `parse_status_callback`) | DOC | **CODE** (Meta `calls` webhook) |
| Authentication / signature checks | **CODE** (`verify_webhook_signature`, `verify_inbound_signature`) | UNVERIFIED | **CODE** (`signature_matches`, app secret) |
| Call cost API | **CODE** (`get_call_cost`, unused) | UNVERIFIED | UNVERIFIED |
| India outbound-to-mobile price | **UNVERIFIED**: read it from the client's Plivo console | UNVERIFIED | UNVERIFIED |
| Tested on a real call | **No** | No | No |

**Questions that still need answers about the client's Plivo account (blocking Phase 3):**
1. Is the number an Indian (+91) number, and is its KYC/compliance complete for **outbound** calls to Indian mobiles?
2. What is the billed rate per minute for outbound calls to Indian mobiles, and what is the billing increment (e.g. 60s or per-second)? In which currency?
3. Is Audio Streaming (`<Stream bidirectional>`) enabled on the account?
4. Auth ID, Auth Token and Application ID. These must go into Dograh's UI (Settings → Telephony), **not** into chat or git.

**Question for ChakraHQ:** can ChakraHQ itself bridge a business-initiated WhatsApp call to an external SIP/WebRTC endpoint? Or should we use Meta's Cloud API directly, which Dograh already speaks for inbound calls? Ask for a demo of an outbound call carrying our own audio stream before any work is quoted.

---

## 4. Task C: Comparison of routes

| Route | Effort | Hindi/Hinglish | Provider fit | Infrastructure | Verdict |
|---|---|---|---|---|---|
| 1. Dograh only (+ Plivo) | **Lowest**: G1–G6 only | Good: Sarvam, Soniox or Deepgram for STT; Sarvam or Smallest for TTS | Native Plivo, Exotel, Vobiz | Postgres, Redis, MinIO, API, UI (existing docker-compose) | **Primary** |
| 2. Dograh + Bolna code | Higher, little gain | Same | No new carriers | Same | Not justified now |
| 3. Pipecat + LiveKit | High: rebuild campaigns, UI, retries | Same models | LiveKit SIP would need a SIP trunk | LiveKit server as well | Not justified |
| 4. VoicEra | Medium: re-audit needed | Unknown | Unknown | Similar | Second platform only if Dograh fails |
| 5. Hosted (DoubleTick / Sarvam ≈ ₹3.90/min) | Lowest code | Good | Managed | None | **Fallback / price benchmark** |

**Fallback inside route 1:** if Plivo India is blocked, change the telephony configuration to Exotel or Vobiz. These are configuration changes, not code.

### Cost model (to measure, not promised)
Per connected minute = Plivo rate + STT + LLM + TTS + a share of hosting. Every term is **UNVERIFIED** until a pilot runs. Telephony cost will come from G4. LLM token cost is already recorded in `usage_info["llm"]` (`tasks/run_integrations.py:165`). The target of under ₹1.50 per minute stays a target.

---

## 5. Architecture for the MVP

```
ChakraHQ (CRM) --CSV export--> Dograh campaign --(suppression check G1)--> Plivo outbound call
                                     |                                         |
                                     |<========= bidirectional mu-law WS =======|
                                     v
                     Pipecat pipeline (STT -> LLM -> TTS), workflow + extraction
                                     v
                       Webhook node (durable, retrying)
                                     v
               21degrees integration service (new, small; G2/G3)
                  - idempotency on workflow_run_id
                  - validate extraction
                  - update ChakraHQ contact
                  - send one templated email
                  - write opt-outs to suppression list
```

---

## 6. Next steps (Task D, mock-first)

1. Initialise the `pipecat` submodule. Run the stack with `docker-compose-local.yaml` (see `docs/contribution/setup.mdx`).
2. Build the approved sample conversation in the workflow builder and run it in the **browser (WebRTC)** with fake contact data.
3. Pick a low-cost model set for the first test: Sarvam or Deepgram for STT, a Gemini Flash-class model for the LLM, Sarvam for TTS. Benchmark Hindi/Hinglish.
4. Build G2/G3 against a mock CRM and a mock mailer, with tests for duplicate webhooks.
5. Build G1 and G4.
6. Only then configure Plivo and place **one** call to a consenting test number.

### Inputs needed from the client
- Answers to the Plivo questions in §3, and credentials entered through the UI.
- The call script: its purpose, questions, approved facts and escalation rules (Part 18, questions 1, 11–13).
- Consent status of the contact list, and the permitted calling hours.
- ChakraHQ API access (plan, API key, custom fields).
- Email sending account.
