# Module 9: Online Scoring

Evals are great for testing changes before you ship. But once your app is in production, you need to know how it's doing in real time — not just on a fixed dataset.

Online scoring is Braintrust's answer to that. Instead of running a scorer manually, you configure it once and Braintrust runs it automatically on every new log as it comes in. You get continuous quality signals without any extra code.

This module covers how to set it up — starting with single-turn scoring (simpler), then extending it to full multi-turn traces. Then we'll generate a batch of conversations and watch the scores appear automatically.

All the code is in [`module-09/`](https://github.com/braintrustdata/eval-course/tree/main/module-09) in the course repo.

---

## What online scoring does

When you configure online scoring, Braintrust watches your project logs and runs your scorer on each new span as it's logged. The score appears directly on the trace in the Logs tab, just like scores you'd see in an experiment.

You can configure it to score:

- **Individual turns** — each assistant response as it's logged. Good for response-level quality signals like Brand Alignment.
- **Root spans** — the full conversation trace, once it's complete. Good for trace-level signals like Conversation Quality.

You can run both at the same time.

---

## Creating the Conversation Quality scorer

You already have a Brand Alignment scorer from earlier modules. Now create the trace-level scorer.

Create a second scorer the same way. Go to **Scorers → + New scorer → LLM-as-a-judge**:

- **Name:** Conversation Quality
- **Model:** gpt-4o
- **Use chain-of-thought:** Yes

Paste this as the prompt:

```
Evaluate this customer support conversation.

{{input}}

Rate the overall quality:
A - Resolved: The customer's issue was fully resolved. The agent was consistent across all turns and didn't ask for the same information twice.
B - Partial: The issue was partially addressed, or resolved with unnecessary back-and-forth or minor inconsistencies.
C - Unresolved: The issue was not resolved, or the agent contradicted itself or gave incorrect information.

Answer A, B, or C.
```

Set the choice scores: A = 1, B = 0.5, C = 0.

---

## Setting up the Brand Alignment automation rule

Now that both scorers exist, create automation rules to run them on new logs automatically.

Go to **Settings → Automations → + Create rule**.

Configure the rule:

- **Rule name:** Brand Alignment
- **Functions:** Select **Brand Alignment** from "This project"
- **Apply to spans:** All spans (this scores every turn, not just root spans)
- **Scope:** Span
- **Sampling rate:** 100%

Save it. From now on, every new span logged to this project gets a Brand Alignment score automatically.

---

## Setting up the Conversation Quality automation rule

Go to **Settings → Automations → + Create rule** again.

Configure the rule:

- **Rule name:** Conversation Quality
- **Functions:** Select **Conversation Quality** from "This project"
- **Apply to spans:** Root spans only
- **Scope:** Trace
- **Sampling rate:** 100%

Two key settings here. **Root spans only** tells Braintrust to only trigger on the top-level conversation span. **Trace scope** means the scorer receives the full trace context — all spans and conversation history — not just one individual span.

Now you have two automation rules running in parallel:

1. Brand Alignment on every turn, span scope (is this response good?)
2. Conversation Quality on root spans, trace scope (did this conversation resolve the issue?)

`generate_conversations.py` creates 10 scripted customer support conversations and logs them to Braintrust:

- **5 single-turn conversations** — one customer message, one agent response. Shipping delay, damaged product, price match request, address change, and an app crash report.
- **5 multi-turn conversations** — 3 to 7 turns each. Wrong item exchange, late delivery with missing items, locked account, late return request, and a double charge on a credit card.

Each conversation uses the same system prompt and logging structure as the chat app from Module 6. The multi-turn conversations log nested turn spans under a root conversation span, just like a real chat session.

Run it:

```
python3 generate_conversations.py
```

Output looks like:

```
Generating 10 conversations...

[1/10] shipping_delay (single-turn)
    Turn 1: Where's my order #4412? It was supposed to arrive two da...

[2/10] damaged_product (single-turn)
    Turn 1: I just opened my package and the ceramic vase is cracke...

...

[8/10] late_return_request (6-turn)
    Turn 1: I need to return a jacket I bought. Order #1190....
    Turn 2: I bought it about 45 days ago. I know your policy is 30...
    Turn 3: It's completely unworn, tags still on. I just didn't ge...
    Turn 4: I'd be fine with store credit if a full refund isn't po...
    Turn 5: That's fair. How do I start the return?...
    Turn 6: Got it. Thanks for being flexible on this....

...

Done. Check the Logs tab to see online scores appear on each trace.
```

---

## What you'll see in the Logs tab

Open the **Logs** tab after the script finishes. Because online scoring is active, scores start appearing on the new traces automatically — no batch script needed.

For each trace:

- Every **turn span** has a **Brand Alignment** score.
- The **root span** has a **Conversation Quality** score.

---

## Sampling rate

Online scoring runs on 100% of logs by default. For high-traffic production apps, that gets expensive fast.

Braintrust lets you set a **sampling rate** per scorer — for example, 10% means the scorer only runs on 1 in 10 conversations. You still get a representative quality signal without scoring every single log.

For this course project, we left it at 100%. But in production, you'd tune this based on traffic volume and scoring cost.

---

## What we covered

In this module, we:

1. Created scorers and set up automation rules for Brand Alignment (per-turn, span scope) and Conversation Quality (root spans, trace scope).
2. Generated 10 conversations (single-turn and multi-turn) and watched online scoring apply automatically.
3. Covered sampling rate for controlling scoring costs in production.

The next module covers analyzing production logs at scale — exploring patterns across many conversations, not just individual traces.
