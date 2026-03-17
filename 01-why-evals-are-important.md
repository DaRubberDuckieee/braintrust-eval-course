# Module 1: Why Evals Are Important

If you've ever built, or are building, an AI system, you've probably run into one or more of these problems before.

1. Your code works well in testing, you deploy it, and then it starts hallucinating or producing inconsistent results.
2. The models your system relies on get upgraded, and suddenly your app starts performing differently.
3. You push changes that improve one part of your app, but cause regressions somewhere else.
4. You want to choose a model that gives accurate results, but won't bankrupt you on cost.
5. You want to tweak prompts to improve your AI's output, but you're not sure how to actually measure that improvement.
6. You don't know how to communicate to your team whether your AI feature is ready to ship using real numbers and benchmarks.

The reason we run into these issues is that we're used to traditional software, which behaves deterministically, whereas AI systems are inherently unpredictable.

AI can hallucinate facts, produce inconsistent outputs, degrade when models get upgraded, and behave differently depending on prompt wording.

So when something breaks, it's not always obvious why.

And when you want to improve the system, it's not always obvious how.

Without evaluation infrastructure, those questions become almost impossible to answer systematically.

Let me give you a real-world example.

In April 2025, OpenAI had to roll back one of its models.

They had updated the model to make it more helpful and responsive.

But the change actually made the model too agreeable, and therefore less truthful.

Instead of pushing back on incorrect assumptions, it started affirming them.

That's a perfect example of how subtle changes can introduce unintended behavior.

Evaluations help you:

1. **Measure system quality** — understand things like accuracy, cost, and latency.
2. **Track improvements** — see whether you're actually improving accuracy, reducing cost, or speeding up response time.
3. **Catch regressions** — so nuanced failures like what happened to OpenAI don't happen to you.
4. **Ship with confidence** — because you have real benchmarks for the important parts of your app, and you can monitor them in production.
