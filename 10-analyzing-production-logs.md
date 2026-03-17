# Module 9: Discovering Patterns with Topics

You have scores on 10 production conversations from the last module. Every turn has a Brand Alignment grade. Every trace has a Conversation Quality grade. That's useful when you're looking at individual logs.

But what happens when you have thousands of conversations per day? You can't click into each trace. You need a way to find the big patterns: what are customers actually asking about, where is the bot struggling, and which problems show up over and over?

That's what Topics does. It automatically clusters your logs into named categories — no manual labeling, no regex rules. You get a map of your production traffic.

All the code is in [`module-10/`](https://github.com/braintrustdata/eval-course/tree/main/module-10) in the course repo.

---

## Generating enough data

Topics requires at least **200 traces** to generate topic clusters. The clustering algorithm needs enough data to find meaningful groups.

`generate_logs.py` creates 200 unique single-turn conversations across 5 topic areas: shipping delays, refund requests, product questions, account issues, and order tracking (40 per topic). It uses `gpt-4o-mini` to keep costs low.

```
python3 generate_logs.py
```

The messages are shuffled so topics are interleaved. The script prints progress every 20 conversations.

In production, this volume accumulates naturally. The script just gets you to the threshold so you can see Topics in action.

---

## What Topics does

Topics analyzes your traces through a multi-stage pipeline:

1. **Preprocessing.** Each trace is converted into a readable narrative — user messages, assistant responses, tool calls — so the AI can analyze it.
2. **Summary extraction.** An AI prompt reads each trace and extracts a concise summary (e.g., "Customer asked about delayed shipping for order placed 10 days ago").
3. **Clustering.** Once enough summaries are collected (100+), a clustering algorithm groups similar summaries together and names each cluster.
4. **Classification.** New traces are automatically tagged with the best-matching topic.

The result: every trace in your project gets a label like "Delivery and tracking issues" or "Refunds and returns" — without you writing any rules.

---

## Setting up topic maps

Go to **Topics** in your project sidebar. Braintrust provides three built-in topic maps:

- **Task** — clusters by user intent or goal
- **Sentiment** — classifies emotional tone
- **Issues** — identifies agent problems

Select the ones you want and click **Create topic maps**. Braintrust starts processing your traces in the background, extracting summaries at the configured sampling rate (100% by default).

Each topic map has an **Edit** tab where you can see the prompt used for summary extraction, the preprocessor config, and the scope (Trace vs Span). You can click **Test** on a sample log to verify the extraction looks right before committing.

---

## Generating topics

Once a topic map has extracted enough summaries (at least 100), you can generate topics.

1. Select a topic map from the sidebar (e.g. **Task**).
2. Click **Generate**.
3. Braintrust clusters similar summaries and displays the results with auto-generated names and descriptions.

With our data, the Task map generates 2 topics:

- **Delivery and tracking issues** (63.6%, 28 items) — "User wants to resolve problems related to order delivery delays, tracking, or shipping status."
- **Refunds and returns** (36.4%, 16 items) — "User wants to process refunds, returns, or resolve duplicate charge issues for purchased items."

Notice that we generated 5 categories of messages, but Topics only found 2 clusters. The algorithm decides what's distinct enough to be its own topic — shipping delays and order tracking got merged into one cluster, and the other categories (product questions, account issues) may not have clustered tightly enough at this data volume.

You can toggle between **List** and **Scatterplot** views using the dropdown. The scatterplot shows how traces are distributed in embedding space — tightly grouped dots mean the topic is well-defined.

Review the results:

- Check how many traces don't match any topic (shown in "Noise/Unclustered").
- Look for topics with unclear or overlapping names.
- Click into sample summaries in each topic to verify they're semantically similar.

If the clusters aren't right, click **Generate > Advanced clustering options** to adjust the minimum cluster size (larger = broader topics, smaller = more granular), try different clustering algorithms, or change the time range.

When you're happy, click **Save topics**. You can optionally enable:

- **Process existing traces** — retroactively classify your logs.
- **Classify incoming traces** — automatically tag new logs going forward.

---

## Using your topics

### Topic distributions

Go to **Logs** and select **Display > Layout > Topics**. Each topic map shows up as a card with the distribution of topics and trace counts. Click any topic to filter the table to just those traces.

### Monitoring trends

Topics automatically adds a chart to the **Monitor** page showing classified log volume over time, broken down by topic. This is where you spot shifts — a sudden spike in "Refunds and returns" means something changed upstream.

### Building datasets from topics

This is the real payoff. Filter your logs to a specific topic:

```sql
classifications.Task.label = 'Refunds and returns'
```

Or use **Filter > Classifications** in the UI. Select the matching traces and click **+ Dataset** to create a focused eval dataset.

Now you have a dataset of real refund conversations pulled directly from production — the exact input you need to test whether your refund handling actually works. In the next module, we use this to close the improvement loop.

---

## What we covered

In this module, we:

1. Generated 200 production logs to meet the Topics threshold.
2. Set up topic maps (Task, Sentiment, Issues) to automatically categorize traces.
3. Generated and reviewed topic clusters.
4. Used topic classifications to filter logs, monitor trends, and build targeted eval datasets.

The next module closes the loop: using what you found here to make a measurable improvement to the bot.
