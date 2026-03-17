# Module 5: How to Read a Trace

Now that you've run your first experiment, you're going to want to understand what actually happened during execution. That's where traces come in.

We're going to start with the customer support eval from Modules 3 and 4. It's simple enough that we can define all the key concepts — traces, spans, span types, metrics — without getting lost in complexity. Then in the next module, we'll look at a much more complex agent trace to see how these same concepts scale.

---

## What is a trace?

A trace is the complete record of a single evaluation run, from the moment the input is handed to your task function to the moment the final scores are computed.

Every row in your experiment results table corresponds to one trace. When you click into a row, you're opening that trace.

For the customer support eval, each trace represents one customer message going through the full pipeline: the message gets passed to the task function, the LLM generates a response, and the scorer evaluates the output. That's one trace.

---

## What is a span?

A span is a single unit of work inside a trace. Every LLM call is a span. Every scorer execution is a span. Spans can be nested — a parent span might contain multiple child spans inside it.

Braintrust recognizes several span types:

- **`eval`** — The root span. Every trace has exactly one. It holds the top-level input, output, expected output, and final scores.
- **`llm`** — A single LLM call. Captures the model name, input messages, output, and token counts.
- **`score`** — A scorer execution. Records the scorer's name, the score it produced, and (if chain-of-thought is enabled) the reasoning behind the score.
- **`function`** — A named block of code in your eval harness. When you wrap a Python function with Braintrust's `@traced` decorator (or manually open a span), it shows up as a function span. These are structural — they tell you how the eval code is organized.
- **`task`** — Represents a unit of work that produces a meaningful result. The distinction from `function` is subtle: function spans tend to be infrastructure (setup, orchestration, wrappers), while task spans represent the actual work being evaluated. In practice, both show up as collapsible blocks in the UI, and you'll sometimes see them used interchangeably depending on how the eval was instrumented.
- **`tool`** — Anything the LLM invoked that runs outside the model itself. Our customer support eval doesn't have tool spans because the LLM just generates text — it doesn't call any external tools. We'll see plenty of these in Module 6.

Each span records its own start/end timestamps, so you can see exactly how long each operation took.

---

## Walking through the customer support trace

Let's open one of the traces from the `module_4_concise_persona` experiment. Click into the row for "Why did my package disappear after tracking showed it was delivered?"

### The root: eval span

The outermost span is the `eval` span. It shows:

- **Input:** `"Why did my package disappear after tracking showed it was delivered?"`
- **Output:** The concise personality's response — something like *"Common causes: carrier scanned it as delivered early, left in an unusual spot, delivered to a neighbor, or misdelivered/stolen..."*
- **Scores:** The `Brand Alignment` score from our `LLMClassifier` — `0.5` (Acceptable) for this input.

This is the bird's-eye view. You can see what went in, what came out, and how it scored, all in one place.

### The LLM span

Expand the eval span and you'll see a single `llm` span inside it. This is the GPT-4o call that generated the response. It captures:

- **Model:** `gpt-4o`
- **Input messages:** The system prompt (the concise personality instructions) and the user message.
- **Output:** The model's full response.
- **Metrics:** Token counts — prompt tokens, completion tokens, total tokens.

Because we used `wrap_openai` in Module 4, this span was created automatically. Braintrust intercepted the OpenAI API call and logged everything without us writing any tracing code.

### The score span

Next to the LLM span (not nested inside it), you'll see a `score` span for the `Brand Alignment` scorer. This is where the LLM judge evaluated the response.

Because we set `use_cot=True` in Module 4, the score span includes the judge's reasoning before the final grade. For this input, the reasoning looks something like:

> *The response lists common causes and gives clear next steps, which is actionable and helpful. However, the tone is professional but lacks an explicit empathetic phrase — there's no acknowledgment that this is a stressful situation for the customer. Rating: B.*

This is one of the most useful things about traces. When a score seems wrong — too high or too low — you can open the score span and read exactly what the judge was thinking. Two things you can do with this:

1. **Debug scores:** You don't have to guess why something scored low. The judge explains itself. A response that scores B isn't just "not good enough" — you can see whether it was a tone issue, a helpfulness gap, or a policy problem.
2. **Calibrate your scorer:** If the judge's reasoning doesn't match how you'd evaluate the response, that's a signal your scoring prompt needs adjustment. Maybe the criteria are too strict, too vague, or weighting the wrong things. Seeing the reasoning is the only way to catch this.

---

## The full picture

The customer support trace is about as simple as a trace gets:

```text path=null start=null
eval (root)
├── llm: gpt-4o
└── score: Brand Alignment
```

Input goes in, one LLM call happens, one scorer runs, done. Three spans total.

Now compare the two experiments. Open a trace from `module_4_polite_persona` and a trace from `module_4_concise_persona` for the same input. The structure is identical — same three span types — but the LLM span's output is different (different system prompt, different response), and the score span might produce a different rating.

This is the power of traces at the simplest level: you can see exactly what the model was told, what it said, and how the judge scored it, for every single row.

---

## Key metrics

Even on a simple trace, the metrics are worth glancing at:

- **Token counts** on the LLM span tell you how verbose the model's response was. The polite personality probably uses more completion tokens than the concise one.
- **Latency** (start/end timestamps) tells you how long the LLM call took. For a single GPT-4o call, this is usually 1–3 seconds.
- **Cost** can be computed from token counts. For a single call with a short input, it's fractions of a cent — but this adds up when you're running across a larger dataset.

On the score span, the key metric is the score itself — but as we saw above, the chain-of-thought reasoning text is often more valuable than the number.

---

## What we covered

In this module, we:

1. Defined traces and spans — the building blocks of understanding what happened during an eval.
2. Walked through a customer support trace with three spans: eval, llm, and score.
3. Identified the span types you'll encounter in Braintrust: eval, llm, score, function, task, and tool.
4. Used chain-of-thought reasoning in the score span to understand exactly why a response scored B — and how to use that for both debugging scores and calibrating your scorer.

This trace was simple — one LLM call, one scorer. In the next module, we'll evolve the customer support bot into a multi-turn chat application and see how traces get more interesting when conversations have multiple turns.
