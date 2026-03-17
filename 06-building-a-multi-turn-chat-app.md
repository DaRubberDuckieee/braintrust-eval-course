# Module 6: Building a Multi-Turn Chat App

So far, our customer support eval has been single-turn: one message in, one response out. That's useful for testing, but it's not how real customer support works. A customer writes in, the bot responds, the customer clarifies, the bot follows up — it's a conversation.

In this module, we're going to evolve the customer support bot from Modules 3–4 into an actual multi-turn chat application. We'll instrument it with Braintrust logging so every conversation gets traced automatically. Then we'll look at what those traces look like — and see how multi-turn conversations produce richer, more interesting traces than the single-turn version.

All the code for this module is available in the [course repo under `module-06/`](https://github.com/braintrustdata/eval-course/tree/main/module-06).

---

## The chat app

We're building a simple CLI chat interface. The user types a message, the bot responds, and the conversation continues until the user quits. Nothing fancy — just enough to simulate a real support interaction.

The full code is in [`module-06/chat_app.py`](https://github.com/braintrustdata/eval-course/tree/main/module-06/chat_app.py) in the course repo. Here's the core of it:

A few things to notice:

- **`init_logger`** replaces `Eval` here. We're not running an experiment — we're logging production conversations. The logger sends each conversation to the Braintrust project as a single production log entry.
- **`wrap_openai`** still wraps the OpenAI client, so every LLM call is automatically captured as an `llm` span.
- **`@traced`** on the `chat` function creates a function span each time it's called. This groups the LLM call inside a named span.
- **`logger.start_span(name="conversation")`** wraps the entire chat session in a single root span. Each turn gets its own child span nested inside it, so the full conversation shows up as one log entry with a detailed trace.
- **`GROUP_AS_CONVERSATION`** is a flag at the top of the file you can flip to see the difference. Set it to `False` and each turn becomes its own top-level log entry — you'll see 5 separate rows in the logs table instead of one. Try both to get a feel for which structure makes sense for your use case.

This is the key difference from Module 4: instead of `Eval` running a batch of inputs through a task function, we're logging a live conversation as it happens.

---

## What the trace looks like

Run the chat app and have a conversation. Here's a real 5-turn session about a wrong-size t-shirt:

> **You:** I ordered a tshirt and it come a size too big
> **Agent:** I'm sorry to hear that the t-shirt isn't the right size. I'd be happy to help you sort this out. Can you please provide your order number? Also, could you confirm if you'd like to exchange it for a different size or if you prefer a refund?
> **You:** Yes, my order number is 6767 and the size I ordered was S and I got a M
> **Agent:** Thank you for providing your order number. I apologize for the mix-up in sizes. To make this right, I can assist you with an exchange for the correct size. I'll initiate the exchange process so that you can receive the correct size S t-shirt. We'll send you a prepaid shipping label via email for the return.
> **You:** I want to exchange for correct size
> **Agent:** Great, we'll proceed with the exchange for the correct size. You will receive an email shortly with the prepaid shipping label to return the incorrect size M t-shirt. Once we receive the return, we'll immediately ship out the size S t-shirt to you.
> **You:** But you don't know what my email is
> **Agent:** You're right, I apologize for that oversight. Could you please provide the email address associated with your order so we can send you the prepaid shipping label?
> **You:** johndoe@gmail.com
> **Agent:** Thank you for providing your email address. I will arrange for the prepaid shipping label to be sent to johndoe@gmail.com. Please let me know if there's anything else you need help with, and thank you for your patience.

Now open the Braintrust production logs for the Customer Support Chatbot project. You'll see a single log entry for the entire 5-turn conversation. Click into it and the trace looks like this:

```text
conversation (root)
├── turn_1
│   └── chat (function)
│       └── Chat Completion (llm: gpt-4o)
├── turn_2
│   └── chat (function)
│       └── Chat Completion (llm: gpt-4o)
├── turn_3
│   └── chat (function)
│       └── Chat Completion (llm: gpt-4o)
├── turn_4
│   └── chat (function)
│       └── Chat Completion (llm: gpt-4o)
└── turn_5
    └── chat (function)
        └── Chat Completion (llm: gpt-4o)
```

The entire multi-turn conversation is captured in one trace. Each turn is a child span, and within each turn you can see the `chat` function span and the underlying LLM call. Click into the LLM spans to see how the conversation history grows — turn 1 has just the system prompt and one user message, while turn 5 has the system prompt plus all 9 previous messages.

This shows up in the metrics too:

- **Turn 1:** 75 prompt tokens, 129 total (system prompt + 1 user message)
- **Turn 3:** 280 prompt tokens, 356 total (system prompt + 5 messages)
- **Turn 5:** 416 prompt tokens, 463 total (system prompt + 9 messages)

Prompt tokens roughly 5x'd over the course of the conversation because the full history gets sent each time. For a 5-turn conversation, the cost is manageable. For a 20-turn conversation, it adds up fast — and the trace metrics make that visible immediately.

---

## Evaluating multi-turn conversations

Single-turn evals are straightforward: one input, one output, score the output. Multi-turn evals introduce a question: what exactly are you scoring?

There are a few levels to think about:

- **Per-turn scoring.** Run a scorer on each turn's input/output pair. This catches obvious per-response issues — a rude reply, incorrect information, a hallucinated policy. But it misses conversation-level problems, like the bot contradicting itself across turns or asking the same clarifying question twice.

- **Trace-level scoring.** Score the entire conversation as a single unit. Braintrust supports this — you can configure online scoring to apply to the root span of a trace, which means the scorer sees the full conversation (all turns, all context) and evaluates the interaction as a whole. This is how you catch coherence issues, resolution quality, and whether the conversation actually solved the customer's problem.

- **Both.** In practice, you'll often want per-turn scores to flag individual bad responses *and* trace-level scores to evaluate the full interaction. They answer different questions.

We'll cover trace-level scoring in Module 7, and set it up on production logs in Module 8. For now, the important thing is that the multi-turn traces we're generating here give us the structure to support both approaches — each turn is its own span, and the whole conversation is a trace.

---

## What we covered

In this module, we:

1. Built a multi-turn CLI chat app from the customer support bot.
2. Instrumented it with `init_logger`, `wrap_openai`, and `@traced` so each conversation is captured as a single production log entry in Braintrust.
3. Saw how multi-turn traces differ from single-turn — growing conversation history, increasing prompt tokens per turn.
4. Introduced the concept of multi-turn scoring — per-turn, trace-level, or both.

Now that we have a working chat app with logging, the next module covers how to analyze all the data you're collecting — comparing experiments, finding patterns, and understanding where your system performs well and where it falls short.
