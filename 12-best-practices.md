# Module 12: Best Practices

You've now built evals, shipped them to production, and used the improvement loop to fix real problems. Before we wrap up, let's cover the practical habits that separate evals that actually work from evals that give you a false sense of confidence.

---

## Run experiments multiple times

This is the single most important practice and the one most people skip.

LLMs are nondeterministic. Even with the same prompt and the same input, the model samples differently on every call. The judge LLM scoring your outputs is *also* nondeterministic. A response that scored A in one run might score B in the next. Stack those two sources of variance and your experiment scores can swing by 5–10% between runs — sometimes more.

This means a single experiment run doesn't tell you much. If you run Prompt A once and get 0.82, then run Prompt B once and get 0.87, you don't know if B is better. You might just be looking at noise.

**What to do instead:** Run each experiment 3–5 times and look at the range. If Prompt A scores 0.78–0.85 across five runs and Prompt B scores 0.84–0.91, you can be more confident B is actually better. If the ranges overlap, the difference probably isn't real.

This is especially important when:

- You're comparing two prompts or models and the score difference is small (< 5%).
- Your dataset is small (fewer than 20 inputs). Smaller datasets amplify variance.
- Your scorer uses chain-of-thought reasoning, which adds another layer of nondeterminism.
- You're using `temperature=1.0` (which you should, for realistic outputs — but it increases variance).

The extra runs cost money and time. It's worth it. A wrong decision based on a single noisy run costs more.

---

## Start with a small dataset, then grow it

Don't try to build the perfect 200-row dataset on day one. Start with 5–10 inputs that cover the most common cases. Run experiments. Find the gaps. Add inputs that target those gaps.

The dataset from Module 4 started with 4 inputs. By Module 11, it had grown with real production conversations. Each addition made the eval more meaningful — not because it was bigger, but because it covered a failure mode that was previously invisible.

A small, well-chosen dataset beats a large, random one. Your goal is coverage of the behaviors you care about, not raw count.

---

## Check your scorer before trusting it

A scorer that gives everything an A is useless. So is a scorer that gives everything a C.

Before you start comparing experiments, spot-check your scorer:

1. **Read the chain-of-thought.** Click into individual traces and read the judge's reasoning. Is it evaluating what you think it's evaluating? Or is it latching onto superficial features like response length?
2. **Test with obvious cases.** Feed your scorer a clearly good response and a clearly bad one. If it doesn't score them differently, the scoring criteria need work.
3. **Look for bias.** LLM judges tend to prefer longer responses, formal language, and responses that match their own style. If your "concise" persona consistently scores lower, check whether the scorer is penalizing brevity itself rather than actual quality.

Scorers are code. They need debugging just like any other code.

---

## Use metadata to slice your results

Don't just look at the average score. The average hides everything interesting.

Tag your dataset inputs with categories — billing, shipping, refunds, emotional, ambiguous, whatever dimensions matter for your use case. Then filter and sort by those categories after running experiments.

A chatbot scoring 0.85 overall might be scoring 0.95 on billing questions and 0.40 on refunds. The average looks fine. The refund experience is terrible. You'd never know without slicing.

---

## Don't change everything at once

When an experiment score drops, you need to know why. If you changed the system prompt, switched models, and updated the scorer between runs, you have no idea which change caused the regression.

Change one thing per experiment:

- Prompt change? Same model, same scorer, same dataset.
- New model? Same prompt, same scorer, same dataset.
- Updated scorer? Run it against the *same* experiment outputs first, before running new ones.

This sounds slow. It's faster than debugging a mystery regression across three variables.

---

## Keep a baseline experiment

Always know what "current production" looks like in your eval. Before making changes, run the existing system against your latest dataset and label it as the baseline.

Every subsequent experiment gets compared against this baseline. If you skip this step, you end up comparing new experiments against stale baselines from weeks ago, and the dataset may have grown since then. The comparison becomes meaningless.

When you ship a change, the new experiment becomes your baseline. Update it explicitly.

---

## What we covered

In this module, we covered the practices that make evals reliable:

1. Run experiments multiple times to account for nondeterminism.
2. Start with a small, targeted dataset and grow it from real failures.
3. Validate your scorer before trusting its output.
4. Use metadata to slice results by category instead of relying on averages.
5. Change one variable at a time so you can isolate what moved the score.
6. Maintain an explicit baseline experiment for comparisons.

None of this is glamorous. It's the difference between evals that inform real decisions and evals that generate numbers you can't act on.
