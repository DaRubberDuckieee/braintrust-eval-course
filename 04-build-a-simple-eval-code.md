# Module 4: Build a Simple Eval in Code

In the last module, we built an eval entirely in the Braintrust UI — creating a dataset, testing two chatbot personalities in a playground, and saving them as experiments. That's a great way to get started, but most teams eventually want to run evals in code.

Code gives you version control for your prompts and scorers, the ability to run experiments programmatically, and a path to integrating evals into CI/CD. In this module, we'll rebuild the exact same customer support eval from Module 3, but in Python.

All the code for this module is available in the [course repo under `module-04/`](https://github.com/braintrustdata/eval-course/tree/main/module-04).

---

## Setup

### Install the dependencies

```bash
pip install braintrust autoevals openai
```

- **`braintrust`** — the Braintrust SDK for running experiments and logging
- **`autoevals`** — a library of pre-built scorers, including LLM-as-a-judge classifiers
- **`openai`** — the OpenAI client (used for the task function)

### Set your API keys

If you haven't already from the last module:

```bash
export BRAINTRUST_API_KEY="your-api-key-here"
export OPENAI_API_KEY="your-openai-api-key"
```

---

## The dataset

In the UI, we uploaded a CSV. In code, the dataset is just a list of dictionaries:

```python
dataset = [
    {"input": "Why did my package disappear after tracking showed it was delivered?"},
    {"input": "Why does your app crash every time I try to check out?"},
    {"input": "Bro your product is the absolute worst freaking thing I've ever used in my life."},
    {"input": "big."},
    # ... 12 more — see the full dataset in eval_customer_support.py
]
```

Each dictionary represents one row. The `input` key is what gets passed to the task function.

---

## The task functions

In the UI, we swapped system prompts in the playground. In code, we define a separate task function for each personality.

```python
import os
from openai import OpenAI
from braintrust import wrap_openai

client = wrap_openai(OpenAI(api_key=os.environ.get("OPENAI_API_KEY")))

def polite_task(input):
    response = client.chat.completions.create(
        model="gpt-4o",
        messages=[
            {
                "role": "system",
                "content": (
                    "You are a warm, empathetic customer support agent. "
                    "Always acknowledge the customer's feelings before addressing their issue. "
                    'Use phrases like "I completely understand how frustrating that must be" '
                    'and "I\'m so sorry you\'re dealing with this." '
                    "Be thorough in your response and make the customer feel heard."
                ),
            },
            {"role": "user", "content": input},
        ],
        temperature=1.0,
    )
    return response.choices[0].message.content

def concise_task(input):
    response = client.chat.completions.create(
        model="gpt-4o",
        messages=[
            {
                "role": "system",
                "content": (
                    "You are an efficient, no-nonsense customer support agent. "
                    "Get straight to the point. Provide the necessary information "
                    "and next steps without filler. Be polite but brief. "
                    "Your response must be 3 sentences or fewer — no exceptions."
                ),
            },
            {"role": "user", "content": input},
        ],
        temperature=1.0,
    )
    return response.choices[0].message.content
```

A few things to notice:

- **`wrap_openai`** wraps the OpenAI client so Braintrust can automatically log every LLM call. You don't need to add any manual logging — it captures the full request and response for every call.
- Each task function takes `input` (from the dataset) and returns the model's response as a string.
- The system prompt is where the personality lives. Everything else is the same between the two functions.

---

## The scorer

In Module 3, we wrote a scoring prompt in the UI that covered helpfulness, tone, and policy compliance. Here’s the same scorer in code, using `LLMClassifier` from the `autoevals` library:

```python
from autoevals import LLMClassifier

brand_alignment_scorer = LLMClassifier(
    name="Brand Alignment",
    prompt_template=(
        "You are evaluating a customer support response.\n\n"
        "Customer message: {{input}}\n\n"
        "Assistant response: {{output}}\n\n"
        "Rate the overall quality of this support response, considering "
        "helpfulness, tone, and policy compliance.\n\n"
        "- Helpfulness: Does it directly address the issue with actionable next steps?\n"
        "- Tone: Is it empathetic and professional?\n"
        "- Policy compliance: Does it follow company support guidelines?\n\n"
        "Rate as:\n"
        "- (A) Excellent — helpful, appropriate tone, and policy-compliant\n"
        "- (B) Acceptable — partially addresses the issue or has minor tone/policy gaps\n"
        "- (C) Poor — unhelpful, inappropriate tone, or violates policy\n"
    ),
    choice_scores={"A": 1.0, "B": 0.5, "C": 0.0},
    use_cot=True,
)
```

A few things to notice:

- **`LLMClassifier`** sends the scoring prompt to an LLM and maps the response to a numeric score using `choice_scores`. The letters (A, B, C) are what the judge LLM outputs; the numbers (1.0, 0.5, 0.0) are the scores Braintrust records. The scorer is named `"Brand Alignment"` to match the scorer we set up in the UI in Module 3.
- **`use_cot=True`** enables chain-of-thought reasoning — the judge explains its reasoning before choosing a letter. This is the same setting we turned on in the UI. We’ll come back to why this matters at the end of this module.
- **`{{input}}` and `{{output}}`** work the same way as in the UI scorer — they get filled in from the dataset and the task result for each row.

---

## Running the experiments

Now we tie it all together with `Eval`:

```python
from braintrust import Eval

# Experiment 1: Polite Personality
Eval(
    "Customer Support Chatbot",
    data=lambda: dataset,
    task=polite_task,
    scores=[brand_alignment_scorer],
    experiment_name="module_4_polite_persona",
)

# Experiment 2: Concise Personality
Eval(
    "Customer Support Chatbot",
    data=lambda: dataset,
    task=concise_task,
    scores=[brand_alignment_scorer],
    experiment_name="module_4_concise_persona",
)
```

`Eval` takes:

- A **project name** (`"Customer Support Chatbot"`) — both experiments use the same project so they show up together and can be compared.
- A **data function** that returns the dataset.
- A **task function** that processes each input.
- A list of **scorers** to run on each output.
- An **experiment name** that labels this specific snapshot.

Run this file and both experiments will appear in the Braintrust UI, just like the ones you created through the playground in Module 3. You can compare them the same way — side by side, with score breakdowns per input.

---

## What the results look like

When you run this, you should see something like:

- **Polite Personality** (~91% Brand Alignment) — drops points mainly on the gibberish input
- **Concise Personality** (~81% Brand Alignment) — drops points on emotionally charged or ambiguous inputs where the 3-sentence cap cuts empathy

The token efficiency gap is significant: concise completion tokens are roughly 3–4× lower than polite across the board.

**A note on nondeterminism:** Your exact numbers probably won't match these — and they won't match what you saw in Module 3 either, even though it's the same dataset, same prompts, and same scorer. LLMs are nondeterministic. With `temperature=1.0`, the model samples differently on every run. The judge LLM scoring your outputs is also nondeterministic, so a response that scored A in one run might score B in the next.

The direction of the gap — concise scoring lower than polite on emotional inputs — should hold. But the specific numbers will move around. This is a real challenge in production eval systems, and it's something we'll cover directly in the best practices module.

---

## The complete file

Here's everything in one place. This is also available as `module-04/eval_customer_support.py` in the course repo.

```python
import os
from openai import OpenAI
from autoevals import LLMClassifier
from braintrust import Eval, wrap_openai

# --- Setup ---
client = wrap_openai(OpenAI(api_key=os.environ.get("OPENAI_API_KEY")))

# --- Dataset ---
dataset = [
    {"input": "Why did my package disappear after tracking showed it was delivered?"},
    {"input": "Your product smells like burnt rubber - what's wrong with it?"},
    {"input": "I ordered 3 items but only got 1, where's the rest?"},
    {"input": "Why does your app crash every time I try to check out?"},
    {"input": "My refund was supposed to be here 2 weeks ago - what's the holdup?"},
    {"input": "Your instructions say 'easy setup' but it took me 3 hours!"},
    {"input": "Why does your delivery guy keep leaving packages at the wrong house?"},
    {"input": "The discount code you sent me doesn't work - fix it!"},
    {"input": "Your support line hung up on me twice - what's going on?"},
    {"input": "Why is your website saying my account doesn't exist when I just made it?"},
    {"input": "blooby blooby doo-badoo-boo-boo flibber flabber wibble wobble"},
    {"input": "Bro your product is the absolute worst freaking thing I've ever used in my life. You are a trash creator. You should go bankrupt."},
    {"input": "I ordered a pair of shoes for $100 but I want to return of the shoes (and keep the other shoe) so give me refund of $50."},
    {"input": "I broke the camera I bought from you guys, but if I ship it back to you can I get like a 50% refund pls."},
    {"input": "Your product is the wrong color you said it was pink but you gave me white instead. well more like white mixed with red. what color is that? Anyways, yeah. And the button is like, kinda bigger than I expected? And also it's just really heavy. But the reviews did say it was gonna be kinda of heavy."},
    {"input": "big."},
]

# --- Scorer ---
brand_alignment_scorer = LLMClassifier(
    name="Brand Alignment",
    prompt_template=(
        "You are evaluating a customer support response.\n\n"
        "Customer message: {{input}}\n\n"
        "Assistant response: {{output}}\n\n"
        "Rate the overall quality of this support response, considering "
        "helpfulness, tone, and policy compliance.\n\n"
        "- Helpfulness: Does it directly address the issue with actionable next steps?\n"
        "- Tone: Is it empathetic and professional?\n"
        "- Policy compliance: Does it follow company support guidelines?\n\n"
        "Rate as:\n"
        "- (A) Excellent — helpful, appropriate tone, and policy-compliant\n"
        "- (B) Acceptable — partially addresses the issue or has minor tone/policy gaps\n"
        "- (C) Poor — unhelpful, inappropriate tone, or violates policy\n"
    ),
    choice_scores={"A": 1.0, "B": 0.5, "C": 0.0},
    use_cot=True,
)

# --- Task functions ---
def polite_task(input):
    response = client.chat.completions.create(
        model="gpt-4o",
        messages=[
            {
                "role": "system",
                "content": (
                    "You are a warm, empathetic customer support agent. "
                    "Always acknowledge the customer's feelings before addressing their issue. "
                    'Use phrases like "I completely understand how frustrating that must be" '
                    'and "I\'m so sorry you\'re dealing with this." '
                    "Be thorough in your response and make the customer feel heard."
                ),
            },
            {"role": "user", "content": input},
        ],
        temperature=1.0,
    )
    return response.choices[0].message.content

def concise_task(input):
    response = client.chat.completions.create(
        model="gpt-4o",
        messages=[
            {
                "role": "system",
                "content": (
                    "You are an efficient, no-nonsense customer support agent. "
                    "Get straight to the point. Provide the necessary information "
                    "and next steps without filler. Be polite but brief. "
                    "Your response must be 3 sentences or fewer — no exceptions."
                ),
            },
            {"role": "user", "content": input},
        ],
        temperature=1.0,
    )
    return response.choices[0].message.content

# --- Run experiments ---
Eval(
    "Customer Support Chatbot",
    data=lambda: dataset,
    task=polite_task,
    scores=[brand_alignment_scorer],
    experiment_name="module_4_polite_persona",
)

Eval(
    "Customer Support Chatbot",
    data=lambda: dataset,
    task=concise_task,
    scores=[brand_alignment_scorer],
    experiment_name="module_4_concise_persona",
)
```


## What we just built

In this module, we rebuilt the same eval from Module 3 — but entirely in code. The core workflow is the same:

1. Define a dataset.
2. Write task functions (one per personality).
3. Write a scorer.
4. Run experiments with `Eval` and compare the results in the Braintrust UI.

The advantage of the code approach: your prompts, scorers, and datasets are all version-controlled. You can iterate on them in your editor, review changes in PRs, and eventually run evals as part of CI/CD.

In the next module, we’ll look at how to read traces — including how to use the chain-of-thought reasoning from your scorer to understand exactly why something scored the way it did.
