# Module 2: What Actually Is an Eval?

Now that we've talked about **why evaluations matter**, the next question is:

What actually *is* an eval?

The simplest way I can put it is that evals answer very practical questions about your AI system.

Questions like:

- Which model is actually best for my use case?
- How does my system perform across different real-world inputs, like English versus Japanese or Typescript vs Python?
- Can we maintain high accuracy without driving costs through the roof?
- Do the responses reflect our company's tone, standards, and policies?
- And most importantly — how do we know when something breaks?

Without evals, the only way to answer these questions is by manually testing a few prompts and hoping everything works. With evals, you can answer them systematically and repeatedly.

---

At its core, an evaluation is actually very simple.

You run your AI system on a set of inputs,

measure the quality of the outputs,

and compute a score.

That's it. And every eval can be broken down into three core components:

1. Dataset
2. Task
3. Scorer

We're going to walk through each of these using three example datasets. By the end of this module, you'll see how the same dataset flows through the entire evaluation pipeline — from input, to task, to scoring.

---

# 1. Dataset

A dataset is simply a collection of inputs you want to test your system on.

These inputs represent the kinds of situations your AI will encounter in the real world.

We're going to use three running examples throughout this module, each designed to highlight a different aspect of evals.

### Example A: Customer support chatbot

You're building a support assistant for a SaaS product. Your dataset contains real customer messages:

| input |
| --- |
| "How do I reset my password?" |
| "My order never arrived." |
| "Can I get a refund?" |
| "Your app keeps crashing on iOS." |

### Example B: Factual Q&A

You're building a system that answers factual questions. Here, there *are* correct answers, so we include them in the dataset:

| input | expected_output |
| --- | --- |
| "What is 2 + 2?" | "4" |
| "Capital of Japan?" | "Tokyo" |
| "Who wrote Hamlet?" | "William Shakespeare" |
| "What year did the Berlin Wall fall?" | "1989" |

### Example C: AI music generation

You're building a system that generates short musical compositions from text prompts. Your dataset looks like:

| input | genre | mood |
| --- | --- | --- |
| "A 30-second intro for a tech podcast" | electronic | upbeat |
| "Background music for a meditation app" | ambient | calm |
| "A jingle for a coffee shop ad" | acoustic | cheerful |
| "An outro for a true crime podcast" | cinematic | suspenseful |

Notice how each dataset is shaped differently. Example A has just the inputs. Example B has explicit expected outputs. Example C has metadata (genre, mood) that guide creative generation but no single right answer.

---

# 2. Task

Once you have a dataset, the next step is defining the task.

The task describes what your AI system should do with the inputs in the dataset. Let's define the task for each of our three examples.

### Example A: Customer support chatbot

**Task:** Given a customer message, generate a helpful, empathetic support response that follows company policy.

The model receives the `input` (the customer's message) and produces a support response.

### Example B: Factual Q&A

**Task:** Answer the question in `{{input}}`.

Here, we're directly referring to the input column using curly brace syntax. The model's job is simple: produce the correct answer. We're not doing anything with `expected_output` at this stage — that comes in during scoring, which we'll cover in a second.

### Example C: AI music generation

**Task:** Generate a short musical composition that matches the given prompt, genre, and mood.

The model receives the text prompt along with the genre and mood metadata, and produces an audio output.

---

# 3. Scorer

The scorer is how we **measure whether the output is good or bad.**

This is where evaluations get interesting — and where our three examples really diverge.

Each dataset we introduced naturally lends itself to a different scoring approach. Let's walk through all three.

---

### Deterministic scoring

Sometimes scoring can be done with traditional code. These checks are deterministic — they either pass or fail.

This is the natural fit for **Example B (Factual Q&A)**, where we have ground-truth answers in the dataset.

For our Q&A dataset, a deterministic scorer could:

- **Exact match:** Check whether the model's output matches the `expected_output` exactly. If the model says "Tokyo" and the expected output is "Tokyo", it scores a 1. Anything else is a 0.
- **Contains match:** A more lenient version — check whether the expected value appears somewhere in the output. So if the model says "The capital of Japan is Tokyo", it still passes because it contains "Tokyo".
- **Normalized match:** Lowercase both strings, strip whitespace, and then compare. This way "tokyo" and " Tokyo " both match.

You could also use deterministic scoring for structural checks on other types of outputs — like verifying a generated JSON matches a schema, or that a SQL query executes without errors. But the core idea is the same: there's a clear right answer, and code can check it.

---

### LLM-as-a-judge

For more subjective tasks, we often use **LLMs as judges.**

This is the natural fit for **Example A (Customer support chatbot)**. There's no single correct answer to "My order never arrived" — but there are clearly better and worse responses.

For our customer support dataset, an LLM judge could evaluate:

- **Helpfulness:** Did the response actually address the customer's issue? A reply to "My order never arrived" should include steps to track or replace the order, not just "I'm sorry to hear that."
- **Tone:** Is the response empathetic and professional? For a billing question like "Can I get a refund?", the response shouldn't be defensive or dismissive.
- **Policy compliance:** Does the response follow company guidelines? For example, if company policy requires offering a replacement before a refund, the judge can check for that.

In practice, you write a prompt that describes your scoring criteria, and an LLM rates each output on a scale (e.g., 0–10). The judge model reads both the input and the output and produces a score with reasoning.

This approach works well because the criteria are well-defined, but the "correct" answer isn't a fixed string — it's a judgment call that requires understanding context and nuance. We'll build a full LLM-as-a-judge scorer for this customer support dataset in the next module.

---

### Human review

In some cases, the best evaluator is still a human.

This is the natural fit for **Example C (AI music generation)**. No deterministic check can tell you if a melody is catchy. No LLM can listen to audio and judge whether it *feels* right for a meditation app versus a coffee shop ad.

For our music generation dataset, human review might look like:

- A reviewer listens to each generated track and rates it on dimensions like **musical quality**, **genre fit**, and **mood alignment**.
- For the prompt "Background music for a meditation app" with genre `ambient` and mood `calm`, a reviewer could score whether the output is actually calming, whether it fits the ambient genre, and whether it would work in that specific context.
- Reviewers might also flag outputs that are technically competent but miss the brief — a track that's beautifully produced but way too energetic for a meditation app.

Human review is also commonly used when:

- Accuracy is extremely important (like medical or legal use cases)
- You're building a benchmark dataset and need high-quality ground-truth labels
- You want to calibrate your automated scorers against human judgment

The tradeoff is speed and cost — human review doesn't scale the way deterministic or LLM-based scoring does. But for creative, subjective, or high-stakes tasks, it's often the only way to get a reliable signal.

---

# Playground vs Experiments

Another concept that's important to understand is the difference between a **playground** and an **experiment.**

These two tools are often confused, but they serve very different purposes.

### Playgrounds

A playground is where you **manually try prompts and inspect outputs.** It's a scratchpad — you type in a prompt, see what the model says, tweak it, and try again. You can run it against your whole dataset, switch models, adjust parameters, and get a sense of what works.

But when you close the playground, those results are gone. Nothing is saved. If you spent an hour tweaking a prompt to get it just right, there's no record of the version that worked or how it scored. It's a sandbox for exploration — useful, but ephemeral.

---

### Experiments

An experiment takes a specific configuration — your prompt, model, and parameters — runs it against your dataset, scores the outputs, and **saves the entire thing as a snapshot.**

That snapshot is permanent. You can come back to it days later, compare it against other experiments, and see exactly what the model produced for every input.

This is where experiments start to compound in value. A few scenarios:

- **Comparing approaches:** Say you're testing two different system prompts for your customer support chatbot. You save each as its own experiment and compare scores side by side — no need to re-run anything.
- **Iterating on a prompt:** You tweak one of those prompts and save it as a new experiment. Now you can compare the updated version against the original. Did the change actually help?
- **Avoiding wasted time:** As your dataset grows, runs can take a long time. If an experiment takes 30 minutes to complete, you don't want to lose those results because you forgot to save them or because you closed a tab. Experiments are durable — once a run finishes, the results are there for good.

The playground is where you explore. Experiments are where you **commit to a configuration, measure it, and keep the receipts.**

In the next module, we'll walk through this exact workflow hands-on — testing different prompts in a playground, saving them as experiments, and comparing the results.

---

# Putting it all together

So to recap, every evaluation is built from three simple components:

- **Dataset** — the inputs you want to test
- **Task** — the operation your AI performs
- **Scorer** — how you measure output quality

And the type of dataset you have often determines the type of scorer you need:

| Dataset | Scoring approach | Why |
| --- | --- | --- |
| Factual Q&A (ground-truth answers) | Deterministic | Clear right/wrong answers that code can check |
| Customer support (subjective quality) | LLM-as-a-judge | Well-defined criteria, but no single correct answer |
| Music generation (creative output) | Human review | Subjective, multi-modal — requires human judgment |

Instead of relying on playground testing, evaluations let you run **experiments across many inputs**, measure performance, and make decisions based on real data.

In the next module, we're going to go hands-on and **build a simple eval together in Braintrust**, starting with setting up your account and running your first experiment.
