# Model Versioning and Migration: Pinning, Aliases, and Safe Upgrades

## What you will learn

- The difference between pinned model snapshot IDs and moving aliases, and when to use each
- Why model upgrades are a change to your system that needs testing, like any dependency bump
- How to run a safe migration: eval gate, shadow traffic, canary, rollback
- Which parts of a prompt and pipeline tend to break when the model changes
- How to design the app so the model is configuration, not code

## Explanation

### Models are dependencies

Your application's behavior depends on the model. A new model can follow instructions differently, write longer or shorter answers, call tools more or less often, or format output differently. Models are also **deprecated and retired** on a published schedule, so staying on one forever is not an option. Treat the model ID as a dependency you version, test, and upgrade deliberately.

### Pinned IDs vs. aliases

| Kind | Example | Behavior | Use for |
|---|---|---|---|
| Pinned snapshot ID | `claude-sonnet-5-5` style dated or versioned ID | Points to one fixed model | Production |
| Alias (a "latest" pointer) | an ID without a snapshot suffix | May move to a newer snapshot over time | Prototypes, local experiments |

In production, **pin**. An alias can change underneath you, so behavior can shift with no code change and no review. Pinning makes releases reproducible: the same commit gives the same model behavior, and a regression can be traced to a change you made.

Check the current model list and deprecation schedule in the docs. Do not memorize IDs for the exam; know the principle.

### Keep the model as configuration

```python
# config.py (one place, overridable per environment)
MODELS = {
    "router":   "<small-fast-model-id>",
    "worker":   "<mid-model-id>",
    "reviewer": "<strongest-model-id>",
}

resp = client.messages.create(
    model=MODELS["worker"],
    max_tokens=1024,
    messages=messages,
)
```

Benefits: one-line upgrades, per-role model choice (lesson 011), A/B tests, and instant rollback by reverting config. Avoid model IDs scattered through the code.

### A safe migration process

1. **Read the migration notes.** Look for renamed or removed parameters, changed defaults, new stop reasons, and changed limits.
2. **Run your eval suite** (lesson 007) against old and new models on the same test set. Compare pass rate, cost, latency, and tool-call counts. A "better" model that fails your cases is not an upgrade for you.
3. **Fix prompts, not just the code.** Old workarounds (heavy emphasis, repeated warnings, prefilled hacks) may no longer help and can hurt. Re-test after simplifying.
4. **Shadow traffic.** Send real requests to the new model in parallel, log results, and do not show them to users. Compare with the old model's output.
5. **Canary.** Route a small percentage of live traffic to the new model; watch quality metrics, error rates, and cost (lesson 021).
6. **Roll out gradually, keep rollback ready.** Rollback is a config change back to the old pinned ID, which only works while that model is still available. Plan upgrades before the retirement date, not on it.

### What commonly changes between models

- **Output length and style:** parsers or UI that assumed a format may break.
- **Tool use behavior:** how eagerly tools are called, parallel calls (lesson 026), argument formatting.
- **Token counts:** a new tokenizer can change token counts for the same text, which affects cost estimates, `max_tokens`, context budgets, and cache sizes.
- **Prompt caching:** caches are per model, so switching models starts with cold caches (lesson 003).
- **Refusals and safety behavior:** edge cases may shift; re-run your guardrail tests (lesson 008).
- **Feature support:** not every model supports every feature (for example extended thinking or certain tools). Check before switching.

### Mixed fleets

You do not need one model for everything. A common architecture: a small model routes and classifies, a mid model does the bulk of the work, and the strongest model handles hard cases or review. Each role can be upgraded independently, each with its own eval slice.

## Common exam traps

- **"Use the latest alias in production so you always get improvements."** Wrong. Pin for reproducibility; upgrade deliberately after evals.
- **"A newer model is always a drop-in replacement."** Wrong. Re-run evals; prompts and parsers may need adjustment.
- **"Swap the model and the cache keeps working."** Wrong. Caches do not carry across models.
- **"Evaluate on a few manual examples."** Wrong. Use a representative regression suite, plus shadow or canary traffic.
- **"Migrate when the old model is retired."** Wrong. Migrate well before the retirement date so rollback remains possible.
- **Hard-coding model IDs throughout the code** makes upgrades risky and slow.

## Practice questions

1. Your production service uses a moving alias. Support complaints rise one morning, and no deploy happened. What is the likely cause and the fix?
2. You are migrating to a newer model. List the order of steps from offline testing to full rollout.
3. After switching models, your costs rise 15% although traffic is unchanged and prices are the same. Name two plausible causes.

<details>
<summary>Answers</summary>

1. The alias was moved to a newer snapshot, changing behavior without any code change. Fix: pin a specific snapshot ID in configuration, and upgrade only after passing evals.
2. Read migration notes, run the eval suite on old vs. new, adjust prompts and parsers, shadow traffic, canary on a small share of live traffic, gradual rollout with rollback ready (all well before the old model's retirement date).
3. Any two of: the new tokenizer yields more tokens for the same text; the model writes longer outputs; it makes more tool calls or more turns; prompt caches are cold after the switch (or cache prefixes no longer hit).

</details>

## Key takeaways

- Pin model snapshot IDs in production; use aliases only for experiments.
- Keep model IDs in configuration, per role, so upgrades and rollbacks are one-line changes.
- Upgrade like any dependency: evals, shadow traffic, canary, gradual rollout, rollback plan.
- Expect changes in style, tool behavior, token counts, and caching; re-test prompts and guardrails.
- Plan migrations ahead of published deprecation dates.
