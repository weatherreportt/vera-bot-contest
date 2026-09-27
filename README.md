# magicpin AI Challenge — Submission

## Approach

Deterministic, template-based composer — no external LLM call in the hot
path. Every message is built by a per-`trigger.kind` function that only
ever inserts values actually present in the pushed Category/Merchant/
Trigger/CustomerContext. If a template's required fields aren't in the
context, it declines to compose rather than invent data (fabrication is
penalized at -2 and caps the whole dimension at 5/10 per the case-studies
rubric — not worth it for a v1).

Deepest, best-tested paths: `research_digest` (merchant-facing) and
`recall_due` (customer-facing) — the two flagship examples used throughout
the brief, the design doc, and the case studies. 14 other merchant-scope
kinds and 7 customer-scope kinds have dedicated templates; anything else
falls back to a grounded generic template that only fires when the trigger
payload has real (non-placeholder) fields.

`/v1/reply` is rule-based pattern matching, purpose-built for the three
replay scenarios in the testing brief:
- **Auto-reply streak** — tracked *per-merchant*, not per-conversation-id
  (the reference judge harness issues a new `conversation_id` every turn
  in this scenario, so per-conversation tracking would never catch it).
  1st match → wait 4h, 2nd → wait 24h, 3rd+ → end.
- **Hostile** — ends immediately and suppresses the merchant from all
  future ticks.
- **Intent transition** — explicit-commitment phrases short-circuit
  straight to an action-oriented reply, skipping any further qualifying
  question.

## What's NOT yet handled (tradeoffs for a fast, safe v1)

- No LLM in the loop, so phrasing is templated rather than freshly
  generated per case — trades some engagement-compulsion polish for
  100% determinism and zero fabrication risk.
- Off-topic/curveball replies and mid-conversation multi-ask follow-ups
  get a generic acknowledge-and-advance reply rather than a tailored one.
- The dataset generator's placeholder-payload triggers (`{"placeholder":
  true, ...}`) are intentionally declined — there's no real data to
  ground a message in, so the bot sends nothing rather than a hollow
  templated line.

## What would help most next

- Real customer-roster overlap data for `supply_alert` (currently states
  the chronic-Rx pool to check rather than a fabricated exact count).
- An LLM pass *behind* the deterministic scaffold for phrasing variety
  once the grounding/anti-fabrication logic is proven out.

## Running locally

```bash
pip install -r requirements.txt
uvicorn bot:app --host 0.0.0.0 --port 8080
```

Then push `dataset/categories/*.json`, `merchants_seed.json`,
`customers_seed.json`, `triggers_seed.json` via `/v1/context` and call
`/v1/tick`.

## Deploying to get a public URL

Any of these give a public `https://` URL in minutes:
- **Render / Railway / Fly.io**: connect this repo (Dockerfile included),
  deploy, they hand you a public URL.
- **ngrok** (fastest for local testing): run the bot locally, then
  `ngrok http 8080` for a temporary public tunnel.
