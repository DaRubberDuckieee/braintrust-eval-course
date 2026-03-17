# Module 3: Build a Simple Eval in the Braintrust UI

We've talked about datasets, tasks, and scorers. We've talked about why experiments are more useful than playgrounds. Now let's actually build one.

We're going to use the customer support chatbot from Module 2 and build an eval entirely in the Braintrust UI — no code required. By the end of this module, you'll have:

- Set up a Braintrust account and API key
- Tested two different chatbot personalities in a playground
- Saved both as experiments and compared them side by side

All the assets for this module — including the customer complaints CSV — are available in the [course repo under `module-03/`](https://github.com/braintrustdata/eval-course/tree/main/module-03).

---

## Setting up Braintrust

### Creating your account

Head to [braintrust.dev](https://www.braintrust.dev) and create an account. The free tier is more than enough for everything we're doing in this course.

### Adding your OpenAI API key to Braintrust

The playground and online scoring run inside Braintrust, so Braintrust needs your model provider key to make LLM calls on your behalf. Go to **Settings → Secrets** and add your OpenAI API key.

We're using OpenAI in this course, but Braintrust supports many model providers — Anthropic, Google Gemini, Mistral, and others. You can also route requests through any provider using the Braintrust proxy. For now, OpenAI is all you need.

### Setting up for code (Module 4+)

You won't need these until the next module when we write code, but you can set them now so they're ready. Go to **Settings → API Keys** in Braintrust and create a new key, then set both as environment variables:

```bash
export BRAINTRUST_API_KEY="your-braintrust-api-key"
export OPENAI_API_KEY="your-openai-api-key"
```

---

## Testing in the playground

Let's start in the playground. The goal here is to figure out the right personality for our customer support chatbot before we commit to anything.

### Creating the project

First, create a new project in Braintrust. Click **New Project** and name it **"Customer Support Chatbot"**. This is the project we'll use for the rest of the course — every experiment, dataset, and production log will live here.

### Creating the dataset

Inside your new project, click **Datasets** and create one called **"Customer Support Messages"**.

You can download the customer complaints CSV from the [course repo (`module-03/customer_complaints.csv`)](https://github.com/braintrustdata/eval-course/tree/main/module-03) and upload it directly to Braintrust.

### Testing two personalities

Open a playground and connect it to your dataset. We're going to test two different system prompts to see how each personality handles these customer messages.

**Prompt A — The Polite Personality:**

> You are a warm, empathetic customer support agent. Always acknowledge the customer's feelings before addressing their issue. Use phrases like "I completely understand how frustrating that must be" and "I'm so sorry you're dealing with this." Be thorough in your response and make the customer feel heard.

Run this against the dataset. For an input like "Why did my package disappear after tracking showed it was delivered?", the model responds with something like:

> *"I'm really sorry you're dealing with this — it's frustrating and unnerving when a package shows 'Delivered' but you can't find it. I'll walk through the most common reasons this happens and give clear next steps you can take right now to try to locate it or start a claim..."*

The response goes on to list causes, suggest steps, and offer follow-ups — thorough, warm, and detailed.

Now switch the system prompt.

**Prompt B — The Concise Personality:**

> You are an efficient, no-nonsense customer support agent. Get straight to the point. Provide the necessary information and next steps without filler. Be polite but brief. Your response must be 3 sentences or fewer — no exceptions.

For the same input, the model responds with:

> *"Common causes: carrier scanned it as delivered early, left in an unusual spot, delivered to a neighbor, or misdelivered/stolen. Check around your property and with neighbors, review any delivery photos, and wait 24 hours; then contact the carrier with your tracking number to open an investigation or file a claim. If the carrier can't locate it, contact the seller for a replacement or refund."*

Factually complete. Actionable. But notice what's missing — there's no acknowledgment that this is a stressful situation. The scorer will flag that.

This is exactly what the playground is for. You're exploring, getting a feel for the options. But right now, these outputs aren't scored, and they'll disappear when you close the playground.

---

## Saving as experiments

Now let's make these results permanent.

### Adding a scorer

Before we save, we need to define how we're scoring. This is where we write the LLM-as-a-judge prompt we discussed conceptually in Module 2.

Create a scorer with a prompt like this:

> You are evaluating a customer support response.
>
> **Customer message:** {{input}}
>
> **Assistant response:** {{output}}
>
> Rate the overall quality of this support response, considering helpfulness, tone, and policy compliance.
>
> - **Helpfulness:** Does it directly address the issue with actionable next steps?
> - **Tone:** Is it empathetic and professional?
> - **Policy compliance:** Does it follow company support guidelines?
>
> Rate as:
> - **(A) Excellent** — helpful, appropriate tone, and policy-compliant
> - **(B) Acceptable** — partially addresses the issue or has minor tone/policy gaps
> - **(C) Poor** — unhelpful, inappropriate tone, or violates policy

Notice how the prompt references `{{input}}` and `{{output}}` — those get filled in from the dataset and the model's response for each row. The judge sees the full context and scores each dimension independently.

One important setting: make sure to turn on **chain-of-thought reasoning** for your scorer. This tells the judge LLM to explain its reasoning before giving a score. It tends to produce more accurate and consistent scores — and later, when you're debugging why something scored low, you'll be able to read the judge's reasoning instead of just seeing a number.

A quick caveat worth flagging: we're using an LLM to judge the output of another LLM. If you're using the same model for both (e.g., GPT-4o as the task model and GPT-4o as the judge), there can be bias — the judge might favor outputs that match its own style. This is a real limitation of LLM-as-a-judge, and we'll dig into it in a later module. For now, it's a useful enough approach to get started, just know it's not a perfect system.

### Running and saving the experiments

Now run the playground with Prompt A and the scorer attached. Once it finishes, save it as an experiment — call it something like **"Polite Personality"**.

Then switch to Prompt B, run it again with the same scorer, and save it as **"Concise Personality"**.

You now have two saved snapshots with scored outputs for every input in the dataset.

---

## Comparing experiments

This is where it gets interesting. Open the experiment comparison view in Braintrust and look at the two experiments side by side.

Here's what the results actually show:

- **Polite Personality** scored ~97% Brand Alignment. The only input it dropped points on was the gibberish message — the response was off-tone for a support channel. On emotionally charged inputs like the angry complaint and the unusual shoe-return request, it handled those warmly and scored Excellent.
- **Concise Personality** scored ~84% Brand Alignment. It lost points on 5 inputs: the missing package, gibberish, the single-shoe return, the broken camera refund request, and the color complaint. These are all inputs where the customer was frustrated, confused, or making an unusual ask — exactly where empathy matters and the 3-sentence cap forced the model to skip it.

On purely technical questions — app crashes, discount codes, refund timelines — both personas scored identically. The gap only appears when the customer is emotionally invested or the request is ambiguous.

The token count tells the other side of the story. Concise responses are 2–3 sentences, so completion tokens are significantly lower across the board. In production at scale, that's a real cost and latency difference.

Neither is clearly better. That's the whole point — without scoring across the full dataset, you'd just be picking whichever one *felt* better on the one or two examples you tried in the playground.

---

## What we just built

In this module, we:

1. Created a dataset of customer support messages.
2. Defined a task — generate a support response — with two different personalities.
3. Wrote an LLM-as-a-judge scorer that evaluates helpfulness, tone, and policy compliance.
4. Ran both personalities in the playground to explore, then saved them as experiments with scores.
5. Compared the experiments to see where each personality excels and where it falls short.

In the next module, we'll rebuild this same eval in code using the Braintrust SDK — which gives you version control, programmatic access, and the ability to integrate evals into your development workflow.
