# Module 8: Analyzing Multi-Turn Traces

The `Brand Alignment` scorer from Module 4 evaluates individual turns: one customer message, one agent response, one score. That works for single-turn evals.

Multi-turn conversations have failures that per-turn scoring can't see. A conversation where every individual response is polite and on-brand can still fail: the bot promised a 3–5 day refund on turn 3 then said refunds aren't available on turn 6. It asked for the order number on turn 2, got it, then asked again on turn 5. It gave seven helpful-sounding responses and never actually resolved the customer's issue.

This module covers trace-level scoring: evaluating the full conversation as a unit.

All the code is in [`module-08/`](https://github.com/braintrustdata/eval-course/tree/main/module-08) in the course repo.

---

## What the scorer receives

In Module 6, the root `conversation` span is logged with the full `conversation_history` array as its `input` — system prompt, every user message, every assistant response, in order.

A trace-level scorer receives this full array. The entire conversation is available for evaluation, not just one turn.

---

## Writing the scorer

The scorer looks at the full conversation and asks: did this interaction actually succeed?

```python
from autoevals import LLMClassifier

conversation_quality = LLMClassifier(
    name="Conversation Quality",
    prompt_template=(
        "Evaluate this customer support conversation.\n\n"
        "{{{input}}}\n\n"
        "Rate the overall quality:\n"
        "A - Resolved: The customer's issue was fully resolved. The agent was "
        "consistent across all turns and didn't ask for the same information twice.\n"
        "B - Partial: The issue was partially addressed, or resolved with unnecessary "
        "back-and-forth or minor inconsistencies.\n"
        "C - Unresolved: The issue was not resolved, or the agent contradicted itself "
        "or gave incorrect information.\n\n"
        "Answer A, B, or C."
    ),
    choice_scores={"A": 1.0, "B": 0.5, "C": 0.0},
    use_cot=True,
)
```

Note `{{{input}}}` (triple braces) — this inserts the conversation text as a raw block rather than escaping it. The CoT reasoning will tell you exactly what the scorer found wrong, which is useful for debugging.

---

## Running both scorers in a batch script

`score_traces.py` fetches all spans from your project logs, groups them by trace, and runs both scorers in one pass:

- **Per-turn (`Brand Alignment`):** Identifies child spans where `input` and `output` are both strings — one user message, one assistant response. Scores each independently.
- **Trace-level (`Conversation Quality`):** Finds the root span for each trace (where `span_id == root_span_id`), formats its full `conversation_history` input, and scores the whole thing.

```
python3 score_traces.py
```

The script prints results as it goes:

```
Found 1 conversations across 16 spans.

  Trace 780f7f26... (5 turns)
    turn b05596ed...  Brand Alignment: A (1.0)
    turn c915d5ca...  Brand Alignment: A (1.0)
    turn 7ea8f867...  Brand Alignment: A (1.0)
    turn cddb06fe...  Brand Alignment: A (1.0)
    turn 9ffe1478...  Brand Alignment: A (1.0)
    trace...          Conversation Quality: B (0.5)
```

Every turn scored A for Brand Alignment — each response was polite and on-brand. But the trace scored B for Conversation Quality. The CoT rationale (written as metadata on the root span) explains why: the agent promised to send a return label via email, then asked for the customer's email, then the customer pointed out the agent should already have it from the order. Unnecessary back-and-forth.

After it runs, open the **Logs** tab. Each `turn_N` span has a `Brand Alignment` score. The root `conversation` span has a `Conversation Quality` score. Both include the scorer's chain-of-thought rationale in the span's metadata.

---

## What the two scores tell you

Two levels of signal on every trace:

- **Turn-level (`Brand Alignment`):** Did each individual response hold up?
- **Conversation-level (`Conversation Quality`):** Did the full interaction succeed?

They're complementary — and they can disagree. In our example, every turn scored A but the trace scored B. Each response in isolation was helpful and professional, but the conversation had avoidable friction: the agent should have pulled the customer's email from the order instead of asking for it. That's the kind of failure that only shows up at the trace level.

The trace-level score surfaces what per-turn scoring structurally can't: did this conversation, as a whole, work?

---

## What we covered

In this module, we:

1. Explained why per-turn scoring misses conversation-level failures.
2. Wrote a `Brand Alignment` scorer for individual turns and a `Conversation Quality` scorer for full traces.
3. Ran both in a single batch script that groups spans by trace, writes scores back at the right level, and includes the CoT rationale as metadata for debugging.

The next module covers online scoring — running these same scorers automatically on new logs as they come in, without a batch script.
