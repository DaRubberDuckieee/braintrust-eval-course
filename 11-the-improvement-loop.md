# Module 11: The Improvement Loop

You've run experiments, built a chat app, scored multi-turn traces, and found patterns in production logs. Now we close the loop: take a real problem from production, fix it, and verify the fix works without breaking anything else.

This is the core of how you use evals in practice.

---

## The loop

<!-- TODO: Outline the full loop: log → score → find patterns → sample into dataset → run baseline → fix → verify → ship. Brief description of each step with back-references to earlier modules. -->

---

## Finding the problem

<!-- TODO: Walk through sorting production logs by score to find a pattern (e.g., refund conversations scoring poorly). Show how to identify an actionable gap — the system prompt says nothing about refunds. -->

---

## Sampling logs into the dataset

<!-- TODO: Show how to take low-scoring production conversations and add them to the eval dataset. The dataset grows from real failures, not synthetic examples. Filter by category, select the bad ones, add to dataset. -->

---

## Running a baseline eval

<!-- TODO: Run the current system prompt against the expanded dataset to establish what "before" looks like. The original inputs score as expected; the new refund inputs score poorly. This is the baseline. -->

---

## Making the fix

<!-- TODO: Update the system prompt with explicit refund-handling instructions (ask for order number, confirm item/date, state timeline, mention 30-day window). Run a new experiment with the updated prompt. -->

---

## Verifying the fix

<!-- TODO: Compare baseline and fix experiments side by side. Check that refund scores improved AND that original inputs didn't regress. If both hold, the fix is clean. -->

---

## What good iteration looks like

<!-- TODO: Key takeaways — dataset grew from real failures (not synthetic), you tested the fix AND checked for regressions, the dataset keeps growing each cycle making the eval harder to pass over time. -->
