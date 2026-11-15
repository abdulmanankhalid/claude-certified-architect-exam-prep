# Workflow Patterns vs. Agents: Chaining, Routing, Parallelization, and Evaluator-Optimizer

## What you will learn

- The difference between a **workflow** (code-defined path) and an **agent** (model-defined path)
- Four core workflow patterns: prompt chaining, routing, parallelization, and evaluator-optimizer
- The orchestrator-workers pattern and how it differs from fixed parallelization
- How to choose the simplest pattern that meets the requirement
- The cost, latency, and reliability trade-offs of each choice

## Explanation

### Workflows vs. agents

- A **workflow** orchestrates LLM calls through a path that **your code** decides in advance.
- An **agent** (see lesson 004) lets **the model** decide the next step and which tools to call, in a loop.

Workflows are more predictable, cheaper, and easier to test. Agents are more flexible but cost more, are slower, and can compound errors. **Rule of thumb: start with a single well-prompted call, move to a workflow when the task has clear steps, and use an agent only when the steps cannot be known in advance.**

### Pattern 1: Prompt chaining

Split a task into fixed sequential steps. Each call's output feeds the next. You can add a programmatic **gate** between steps to check quality.

```python
outline = call("Write an outline for: {topic}")
if not passes_check(outline):          # gate in code
    raise ValueError("Bad outline")
draft = call(f"Write the article from this outline:\n{outline}")
final = call(f"Translate to French:\n{draft}")
```

Use when: the task decomposes cleanly and you trade latency for higher accuracy, because each call is simpler.

### Pattern 2: Routing

Classify the input first, then send it to a specialized prompt, tool set, or model.

```python
label = call_small_model("Classify as: refund | technical | other\n" + msg)
handler = {"refund": refund_prompt, "technical": tech_prompt}.get(label, general_prompt)
answer = call(handler, msg)
```

Use when: inputs fall into distinct categories. It is also a cost lever: easy queries go to a small model (e.g. Haiku), hard ones to a larger one.

### Pattern 3: Parallelization

Run independent calls at the same time. Two variants:

- **Sectioning**: split a task into independent subtasks (e.g. one call checks for policy violations while another answers the user).
- **Voting**: run the *same* task several times and aggregate (e.g. three reviewers of the same code for vulnerabilities).

Latency is roughly that of the slowest call, but token cost adds up. Subtasks must truly be independent.

### Pattern 4: Orchestrator-workers

A central model **dynamically** breaks the task into subtasks, delegates them to worker calls, and synthesizes the results. It looks like parallelization, but the subtasks are **not predefined**. Example: a coding change that touches an unknown number of files. This is the workflow closest to an agent (see lesson 009).

### Pattern 5: Evaluator-optimizer

One call generates; another critiques against explicit criteria; loop until accepted or a **maximum iteration count** is hit.

```python
draft = generate(task)
for _ in range(3):                      # always cap the loop
    review = evaluate(draft, rubric)
    if review.passed:
        break
    draft = generate(task, feedback=review.comments)
```

Use when: you have clear evaluation criteria and iteration measurably improves output (translation nuance, code that must pass tests). It connects to lesson 007 on graders.

### Choosing a pattern

| Situation | Pattern |
|---|---|
| Fixed steps, each easier alone | Prompt chaining |
| Distinct input categories | Routing |
| Independent subtasks or need for confidence | Parallelization |
| Unknown subtasks, decided at runtime | Orchestrator-workers |
| Clear quality criteria, refinement helps | Evaluator-optimizer |
| Open-ended, unpredictable steps | Agent |

Patterns compose: a router can send a request to a chain, and a chain step can be an agent.

## Common exam traps

- **"Use an agent for everything."** Wrong. If the steps are known, a workflow is cheaper and more predictable.
- **Confusing parallelization with orchestrator-workers.** Fixed, predefined subtasks = parallelization. Subtasks chosen by the model at runtime = orchestrator-workers.
- **Unbounded evaluator-optimizer loops.** Always set a maximum iteration count and a stop condition.
- **Parallelizing dependent steps.** If step B needs A's output, that is chaining, not parallelization.
- **Forgetting that voting multiplies cost.** N votes means roughly N times the tokens.
- **Routing without a fallback.** Unknown or malformed labels need a default route.

## Practice questions

1. A support system receives emails that are billing questions, bug reports, or sales inquiries. Each needs a different prompt and some can be answered by a cheaper model. Which pattern fits?
2. You must review a pull request for security, style, and performance issues, and the three checks do not depend on each other. You want low latency. Which pattern?
3. A code-generation task must be refined until the unit tests pass. What pattern, and what safeguard is essential?

<details>
<summary>Answers</summary>

1. **Routing.** Classify the email (with a small model), then dispatch to a specialized prompt or model, with a default route for unrecognized labels.
2. **Parallelization (sectioning).** Run the three independent checks concurrently and merge the results; latency is about that of the slowest check.
3. **Evaluator-optimizer** (generate, run tests/critique, regenerate with feedback). Essential safeguard: a **maximum iteration count** (plus a clear pass criterion) to avoid infinite loops and runaway cost.

</details>

## Key takeaways

- Workflows follow code-defined paths; agents let the model choose the path.
- Start simple: single call, then workflow, then agent only if needed.
- Chaining = sequential steps; routing = classify then dispatch; parallelization = independent calls (sectioning or voting).
- Orchestrator-workers = model-decided subtasks; evaluator-optimizer = generate/critique loop with a cap.
- Patterns can be combined, and each adds cost, latency, or complexity you must justify.
