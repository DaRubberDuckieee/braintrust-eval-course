# Module 7: Analyzing Results

You've run experiments in Modules 3–4, built a multi-turn chat app in Module 6, and now you've got data. Experiments with scores, production logs with traces. The question is: what do you do with all of it?

This module covers the tools Braintrust gives you for making sense of your results — from basic experiment comparisons to natural language queries to automated pattern detection.

---

## Comparing experiments

The most common thing you'll do after running an experiment is compare it to a previous one. Did the scores go up? Did anything regress?

In the Braintrust UI, go to your **Customer Support Chatbot** project and open the **Experiments** tab. Select two experiments to compare — say, `module_3_polite_persona` from Module 3 and `module_4_polite_persona` from Module 4.

The comparison view shows you:

- **Aggregate score changes.** The overall Brand Alignment score across all inputs — did it go up or down, and by how much?
- **Per-input diffs.** Which specific inputs improved? Which regressed? This is the most useful part — click into any row to see exactly what changed in the model's response.
- **New and removed inputs.** If your dataset grew between experiments (which it will, once you start sampling production logs), you'll see which inputs are new.

For the customer support chatbot, comparing `module_3_polite_persona` to `module_4_polite_persona` surfaces a real example of this. The input `"big."` — a single ambiguous word — [scored B in module 4](https://www.braintrust.dev/app/Jess%20Wang/p/Customer%20Support%20Chatbot/experiments/module_4_polite_persona?r=1ecf4e9d-2201-4b2e-9158-e0dcab5f4fc2&s=0c1896e0-3e38-4cbd-a65c-64ef95803c28) but A in module 3. Click into the span and you'll see why: the bot responded with "I completely understand how it feels when something isn't quite right or doesn't fit how you expected." Polite — but useless. It never asked what the user actually meant. That's exactly the kind of regression comparisons surface.

---

## Filtering and sorting results

Not every experiment row deserves your attention. In a 50-input dataset, maybe 5 rows are actually interesting.

Use the Braintrust UI to focus:

- **Sort by score, ascending.** The worst responses float to the top. Click into the trace, read the chain-of-thought from the scorer, and understand why it scored low.
- **Filter by score range.** Show only rows where Brand Alignment < 0.5 to focus on failures.
- **Filter by metadata.** If your dataset includes category tags (billing, shipping, returns), filter to see how the bot handles each category independently.

The goal isn't to look at every row. It's to find patterns in where the bot fails and use those patterns to inform your next prompt iteration.

---

## Using Loop to query your data

Loop is Braintrust's natural language interface for asking questions about your experiments and logs. Instead of manually building filters, you can ask questions directly.

Open Loop from the Braintrust sidebar and try queries like:

- "Which inputs in my latest experiment scored below 0.5 on Brand Alignment?"
- "What's the average Brand Alignment score for refund-related questions across all experiments?"
- "Show me the inputs where `module_3_polite_persona` and `module_4_polite_persona` disagreed the most."

Loop translates your questions into queries over your experiment and log data and returns the results. It's faster than building filters manually, especially when you're exploring and don't know exactly what you're looking for.

---

## Using the Braintrust MCP server

If you have the Braintrust MCP server configured in your development environment, you can ask it the same questions you'd ask in Loop — without leaving your editor. For example:

- "Which inputs in my latest experiment scored below 0.5 on Brand Alignment?"
- "Summarize my experiment for me."
- "Show me where `module_3_polite_persona` and `module_4_polite_persona` disagreed the most."

It's the same data as Loop, accessible from wherever you're working.

---

## What we covered

In this module, we:

1. Compared experiments side by side to find improvements and regressions.
2. Used filtering and sorting to focus on the most interesting results.
3. Used Loop to ask natural language questions about experiment and log data.
4. Used the Braintrust MCP server to query results programmatically.

The next module covers trace-level scoring — evaluating full multi-turn conversations instead of individual turns.
