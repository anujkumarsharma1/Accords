# ACCORD

**Checking that an AI agent's checkout cart actually came from what the human approved — not just that the numbers work out.**

Solo build for the Razorpay AI Buildathon 2026, Track 02 (AI Risk Manager).

---

## Why this exists

Agentic checkout runs on one upfront approval — something like *"weekly groceries, under ₹10,000"* — and then the agent fills in the actual cart later, often from data an attacker can shape, like a poisoned product listing.

By the time a payment reaches the PSP, there's no reasoning trail left to check. Razorpay, or any merchant, only ever sees two things: the intent that got approved, and the cart that showed up. UPI Reserve Pay and the BigBasket/ChatGPT commerce pilot are about to push real money through exactly this gap.

The question everyone wants to ask — "is this agent compromised?" — can't be answered from outside the agent. The question that can be answered is: **does this cart actually descend from what the human approved?** That's what ACCORD checks.

## The diagram

![ACCORD architecture](assets/architecture.svg)

Mandate and cart both feed three independent checks. None of them gets to decide alone — the verdict logic combines all three into a per-item call.

## How it works

**Path 1 — deterministic constraints.** Spend limit, category allow-list, merchant allow-list. Plain rules, no model. This alone kills the obvious cases.

**Path 2 — structural conformance.** Each line item is checked against the mandate's declared intent using embedding similarity between the item description and the mandate category, plus a price-per-unit sanity check against typical ranges for that category. No LLM involved here either — this is the layer that catches "technically compliant, semantically wrong."

**Path 3 — scoped LLM check.** For the cases Paths 1 and 2 can't resolve with confidence, one contained LLM call answers a single question: *does this line item plausibly belong to this approved intent?* Everything sourced from the cart or product listings is passed in as data, never as instruction, so a poisoned description can't redirect the model. The call is scoped to one item at a time — no open-ended reasoning over the whole cart, no tool access, no memory between calls. And its output is one signal into the verdict, never the verdict itself.

**Verdict logic.** When confidence is high across the board, the cart settles. When it isn't — say the three paths disagree on one item — ACCORD doesn't reject the whole order. It settles the part it's confident about and holds the rest.

A ₹1,840 grocery order shouldn't get killed because an ₹8,007 gift card rode along with it in the same cart. ACCORD settles the ₹1,840 and holds the ₹8,007 for review.

## Evaluation

Tested against a sealed held-out corpus of mandate/cart pairs, including adversarial cases: prompt-injected product descriptions, carts that are numerically compliant but semantically mismatched, and boundary-case spend amounts the corpus was built to catch off-by-one errors on.

| Metric | Result |
|---|---|
| Conformance detection accuracy | — |
| False-reject rate on legitimate carts | — |
| Partial-settlement accuracy | — |
| Latency per verdict | — |

*(Numbers go here once the eval run against the sealed corpus is complete — don't publish estimates for these.)*

## Project layout

```
accord/
├── src/
│   ├── constraints/       # Path 1
│   ├── structural/        # Path 2
│   ├── llm_conformance/   # Path 3, with input sandboxing
│   └── verdict/           # settle/hold logic
├── eval/
│   └── held_out_corpus/   # sealed test set, untouched during dev
├── docs/
│   └── architecture.md
├── assets/
│   └── architecture.svg
└── README.md
```

## Running it

```bash
git clone https://github.com/<your-username>/accord.git
cd accord
pip install -r requirements.txt
cp .env.example .env   # add your LLM API key
```

```python
from accord import ConformanceEngine

mandate = {"intent": "weekly groceries", "max_total": 10000, "currency": "INR"}

cart = [
    {"item": "vegetables, bread, milk", "amount": 1840},
    {"item": "gift card", "amount": 8007},
]

engine = ConformanceEngine(mandate)
verdict = engine.evaluate(cart)

print(verdict)
# {
#   "settle": [{"item": "vegetables, bread, milk", "amount": 1840}],
#   "hold":   [{"item": "gift card", "amount": 8007, "reason": "low mandate-conformance confidence"}]
# }
```

## Design principles

1. The only question that matters is whether the cart traces back to the approval — not whether the agent is trustworthy, which can't be answered from outside it.
2. Attacker-controlled content is always data, never instruction.
3. The LLM is one signal among three. It never gets sole authority.
4. Fail partially, not totally — one bad line item shouldn't block the legitimate ones next to it.

## Submission artifacts

- This repo
- 5-minute pitch video
- [`docs/architecture.md`](docs/architecture.md)

## License

MIT
