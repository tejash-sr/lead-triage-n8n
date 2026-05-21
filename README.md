# Lead Inbox Triage Bot

A production-style n8n workflow that watches a shared test inbox, classifies every incoming email with an LLM into `LEAD / SUPPORT / SPAM / OTHER`, logs it to a Google Sheets CRM with idempotent appends, drafts a personalised reply for every lead (never sends), and posts a Slack card so a human can pick it up. A separate Error Trigger sub-workflow captures every failure into an `errors` tab and an alert channel.

---

## 1. Repo layout

```
.
├── README.md
├── docker-compose.yml
├── .env.example
├── .gitignore
├── workflows/
│   ├── v1-phase1.json                        ← Gmail trigger + seed setup
│   ├── v2-phase2.json                        ← AI classifier + JSON parsing
│   ├── v3-phase3.json                        ← Switch routing + Sheets CRM
│   ├── v4-phase4.json                        ← Draft reply + Gmail draft + Slack
│   ├── v5-phase5-final.json                  ← Hardened final workflow
│   ├── error-handler.json                    ← Error Trigger sub-workflow
│   ├── challenge-a-multi-tenant.json         ← Bonus A: tenant config via Sheets
│   ├── challenge-b-self-improving.json       ← Bonus B: webhook + feedback loop
│   ├── challenge-c-daily-digest.json         ← Bonus C: 9AM IST digest scheduler
│   └── uniqueness-high-throughput.json       ← ⭐ Advanced: scale-hardened variant
├── screenshot/
│   ├── v1-phase1.png
│   ├── v2-phase2.png
│   ├── v3-phase3.png
│   ├── v4-phase4.png
│   ├── v5-phase5.png
│   └── full workflow.png
├── prompts/
│   ├── classification_prompt.md
│   └── reply_draft_prompt.md
├── samples/
│   └── emails/                               ← 10 seed .eml files
├── sheets/
│   ├── leads.csv
│   ├── support.csv
│   ├── spam.csv
│   ├── other.csv
│   ├── errors.csv
│   ├── test_fixtures.csv
│   └── CRM_SHEET_TEMPLATE.md
├── scripts/
│   ├── check_no_send.py                      ← CI guard: fails if any Gmail "send" found
│   ├── validate_workflows.py                 ← Lint all workflow JSONs before push
│   └── seed_inbox.md                         ← Instructions to seed the test inbox
└── fixtures/
    └── expected_classifications.json
```

---

## 2. Architecture

```
Gmail Trigger (poll 1m)
        │
        ▼
Normalize email payload (Set)  →  Mark message as read (Gmail)
        │
        ▼
Prompts (Set — classifier + reply prompts, model, temperature)
        │
        ▼
Clean Email (Code — strip signatures, quoted replies, RFC noise)
        │
        ▼
Classify email (OpenAI/Groq, JSON mode, temperature 0)
        │
        ▼
Parse classification (Code — schema-coerce, never throws)
        │
        ▼
Route by category (Switch — LEAD / SUPPORT / SPAM / OTHER + fallback)
        │
        ├── LEAD ─── Check Existing Lead ─── Duplicate Lead? ──── Skip Lead Duplicate
        │                                            │
        │                                     Append Lead in sheet
        │                                     Append all messages in sheet
        │                                            │
        │                                     Lead Reply Prompts (Set)
        │                                            │
        │                                     Generate lead draft (AI)
        │                                            │
        │                                     Lead reply defaults (Set)
        │                                            │
        │                                     Create gmail draft (NEVER send)
        │                                            │
        │                                     Send a message (Slack card)
        │
        ├── SUPPORT ─ Check Existing Support ─ Duplicate? ─── Append Support in sheet
        ├── SPAM ──── Check Existing Spam ───── Duplicate? ─── Append spam in sheet
        ├── OTHER ─── Check Existing Others ─── Duplicate? ─── Append other in sheet
        └── fallback ─────────────────────────────────────────── Append FallBack Error

Sub-workflow (LeadInboxTriageBot_ErrorHandler):
  Error Trigger → Normalize Error Payload (Set) →
  Append row in sheet (errors tab) → Send a message (Slack alert)
```

Annotated canvas screenshots per phase: [`screenshot/`](screenshot/)

---

## 3. Quick start

```bash
# 1. Boot n8n locally
cp .env.example .env          # fill in credentials
docker compose up -d
open http://localhost:5678

# 2. Inside n8n — add credentials
#    Gmail OAuth2, Google Sheets OAuth2, OpenAI (or Groq), Slack OAuth2

# 3. Import workflows
#    Workflows → Import from File
#    Import: workflows/v5-phase5-final.json
#    Import: workflows/error-handler.json

# 4. Bind credentials on every red node
# 5. Main workflow → Settings → Error Workflow → LeadInboxTriageBot_ErrorHandler
# 6. Activate both workflows

# 7. Seed the inbox
#    See scripts/seed_inbox.md — forward the 10 .eml files in samples/emails/

# 8. Validate
python3 scripts/validate_workflows.py
python3 scripts/check_no_send.py
```

---

## 4. Phase-by-phase build

| Phase | Branch | Tag | Deliverable | Workflow JSON |
|------:|--------|----:|-------------|---------------|
| 0 | `main` | — | Scaffold (README, folders, docker, scripts) | — |
| 1 | `phase-1` | 0.1 | Gmail Trigger + credentials + 10 seed emails | `v1-phase1.json` |
| 2 | `phase-2` | 0.2 | Clean email + AI classifier + safety net | `v2-phase2.json` |
| 3 | `phase-3` | 0.3 | Switch routing + Sheets CRM + idempotency | `v3-phase3.json` |
| 4 | `phase-4` | 0.4 | Draft reply + Gmail draft (never send) + Slack | `v4-phase4.json` |
| 5 | `phase-5` | 0.5 | Error handler + retries + regression run | `v5-phase5-final.json`, `error-handler.json` |
| A | `challenge-a` | 0.6 | Multi-tenant prompt templates (sheet-driven) | `challenge-a-multi-tenant.json` |
| B | `challenge-b` | 0.7 | Self-improving classifier (webhook + feedback) | `challenge-b-self-improving.json` |
| C | `challenge-c` | 0.8 | Daily digest scheduler (9:00 IST cron) | `challenge-c-daily-digest.json` |
| ⭐ | `uniqueness` | 0.9 | High-throughput hardened variant | `uniqueness-high-throughput.json` |

Branches are never deleted. `develop` is the integration branch; `main` is release.

---

## 5. Classification prompt — why each instruction exists

Full prompt: [`prompts/classification_prompt.md`](prompts/classification_prompt.md)

- **Four flat categories, no nesting.** `LEAD / SUPPORT / SPAM / OTHER`. Anything richer (priority, tags) is added as separate fields. Keeps the Switch node trivial and the schema future-proof.
- **One-sentence category definitions.** The model needs an unambiguous boundary between SUPPORT (existing customer with a problem) and LEAD (a stranger asking about us). Without that, customer bug reports keep getting filed as LEAD.
- **Two few-shot examples — one LEAD, one SPAM.** The two trickiest categories on the seed set. SPAM gets a list-unsubscribe newsletter; LEAD gets an RFP with team size and timeline. Adding more examples did not move accuracy but pushed token count up.
- **`temperature = 0`, `response_format = json_object`.** Determinism plus machine-parsable output. The Parse classification Code node is a defensive second line — if the model returns prose, we coerce to schema, set `parse_failed = true`, and route to OTHER.
- **`suggested_priority` only when LEAD.** Other categories get `null`. Downstream nodes can rely on this without a separate branch.
- **Free webmail bias rule.** If sender_domain is gmail/yahoo/outlook and the email lacks specifics, priority caps at MEDIUM. Empirically the strongest proxy for lead quality.
- **Unsubscribe = SPAM rule.** Without it, marketing emails with relevant topics sneak into LEAD ~15% of the time.

---

## 6. Reply prompt — why each instruction exists

Full prompt: [`prompts/reply_draft_prompt.md`](prompts/reply_draft_prompt.md)

- **80–150 word hard cap.** Long drafts feel automated; under 80 feels dismissive. This band copy-edits well.
- **First name from `from` field.** Personalisation that needs no extra lookup. Falls back to "Hi there," if only an email address is available.
- **Propose a 20-minute discovery call + ask for 2–3 time slots.** Concrete next step beats "let me know." We ask for slots rather than proposing times — we don't have the prospect's calendar.
- **No pricing, no commitments, no fabrication.** If the email is thin, the draft asks 1–2 clarifying questions instead of inventing details. This is the rule that keeps the bot deployable on a real `sales@` inbox.
- **Fixed signature from `Lead reply defaults` Set node.** One place to edit when the rep changes.
- **`temperature = 0.2`.** Enough warmth for a natural read, not enough randomness to invent facts.
- **Plain text output.** Gmail HTML drafts render inconsistently when a human pastes their own edits in.

---

## 7. Known limitations & v2 ideas

1. **Google Sheets as a single-writer store.** The Lookup → IF → Append idempotency pattern is cooperative: two parallel executions on the same `message_id` can both pass the lookup and produce a duplicate. In practice the 1-minute poll and Gmail's own dedup make this nearly impossible, but for any production port the right move is Postgres with a unique index on `message_id`. The `uniqueness` branch demonstrates this approach with a claim-token pattern.

2. **Gmail Trigger has no in-node retry.** It is a polling node. If Gmail OAuth flakes, the next poll a minute later recovers. The on-call signal is a gap longer than 5 minutes in the executions log.

3. **PII in `raw_payload`.** The error handler stores the first 1,000 characters of the failing payload. For a real customer inbox a Presidio pass before the Sheets append would be required.

4. **Single-tenant prompts.** Tone, signature, and Slack channel are baked into Set nodes. The `challenge-a` branch shows the multi-tenant fix (driven by a `config` Sheet tab, one row per domain).

5. **No feedback loop.** Misclassifications are silent. The `challenge-b` branch adds a webhook and a feedback tab that becomes a few-shot reservoir once 20+ corrections accumulate.

6. **No daily summary.** Reps want a once-a-day summary. The `challenge-c` branch is that scheduler (9:00 IST weekdays).

---

## 8. LLM cost

Model: `llama-3.3-70b-versatile` via Groq (OpenAI-compatible endpoint).

| Phase | Emails | In / out tokens | Cost (USD) |
|------:|-------:|-----------------|------------|
| 1 | 10 | — (no LLM) | $0.00 |
| 2 | 10 | ~7k / ~0.8k | ~$0.002 |
| 3 | 10 | — (re-run, idempotent) | $0.00 |
| 4 | 10 | ~13k / ~3k | ~$0.009 |
| 5 | 12 | ~27k / ~6k | ~$0.019 |
| **Total** | | ~54k / ~11k | **~$0.030** |

Well within the $5 sprint cap.

---

## 9. Safety rules

1. **Gmail node is `resource: draft` only.** The CI guard `scripts/check_no_send.py` fails any commit that introduces a `send` operation.
2. **Test inbox only.** The Gmail Trigger is never pointed at a production `sales@` mailbox.
3. **No secrets committed.** Workflow JSONs carry only credential references (id + name), never values. `scripts/validate_workflows.py` greps for accidental leaks.
4. **Idempotent appends.** Re-running on the same `message_id` is a no-op.
5. **Error handler never re-triggers the failing workflow.** Logging and alerting only.

---

## 10. Uniqueness branch — what it adds

The `uniqueness` branch (`workflows/uniqueness-high-throughput.json`, tag `0.9`) answers the questions a senior reviewer will ask:

| Question | Answer implemented |
|----------|-------------------|
| What breaks first at 10,000 emails/day? | Sheets append quota (60 writes/min). Mitigation: early dedup gate writes one row before classification, eliminating duplicate LLM spend. |
| Why polling vs. push? | Gmail Push needs Pub/Sub + public webhook URL; polling is fine under 500 emails/day. The uniqueness branch keeps polling but adds an early-claim pattern. |
| Race conditions? | Lookup→Append is racy. The branch writes a PENDING claim token to `all_messages` *before* calling the LLM — only one execution can write it; the second hits the early dedup gate and stops. |
| Hardcodings? | All model names, temperatures, Slack channel, sheet ID, confidence threshold live in a single `Dynamic Config` Set node at the top. Change once, applies everywhere. |
| Low-confidence classifications? | A Confidence Gate (IF node, threshold configurable in Dynamic Config) routes anything below 0.4 to a `manual_review` tab instead of silently routing to OTHER. |
| Lead prioritisation? | A Lead Scoring node computes `score = confidence × priority_weight × 100` and adds a HOT/WARM/COLD label to the Slack card and the leads tab row. |
