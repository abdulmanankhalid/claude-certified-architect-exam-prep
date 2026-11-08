# Evaluating Claude Applications: Test Sets, Graders, and Regression Checks

## What you will learn

- Why "it looked good in my demo" is not evidence, and what an eval is
- How to build a small, representative test set
- The three grader types: code-based, LLM-as-judge, and human
- How to use evals to compare prompts, models, and cost/latency trade-offs
- How to evaluate agents and tool use, not only single answers

## Explanation

An **eval** is a repeatable test: a set of inputs, a way to score the outputs, and a number you can compare over time. Prompts, models, and tools change. Without evals you cannot tell whether a change helped or broke something.

### 1. Build a test set

Each case has an input and either an expected answer or grading criteria.

```json
{"id": "refund-01", "input": "I want my money back for order 1234", "expected_label": "refund_request"}
{"id": "edge-07",  "input": "refund... just kidding, where is my parcel?", "expected_label": "order_status"}
```

Good test sets:
- Mirror real traffic (sample from production logs where allowed).
- Include **edge cases** and known failures, not only easy cases.
- Are versioned in the repo, and a held-out part is never used to tune the prompt.
- Start small (20 to 50 cases) and grow whenever a bug is found: every bug becomes a new case.

### 2. Choose a grader

| Grader | Use when | Pros | Cons |
|---|---|---|---|
| Code-based (exact match, regex, JSON schema, unit tests) | Output is checkable | Fast, cheap, deterministic | Cannot judge tone or free text |
| LLM-as-judge | Open-ended output | Scales, handles nuance | Needs its own validation, has biases |
| Human review | High stakes, calibration | Most trusted | Slow, expensive |

Prefer the cheapest grader that works. Code first, LLM judge when needed, humans to calibrate the judge.

### 3. LLM-as-judge done well

```python
JUDGE = """Grade the answer against the rubric.
<rubric>Answer must cite the refund policy and not promise a date.</rubric>
<answer>{answer}</answer>
Think step by step in <reasoning>, then output <grade>pass</grade> or <grade>fail</grade>."""
```

- Use a **specific rubric** and a **binary or small-scale** score, not "rate 1 to 10".
- Let the judge reason before the grade.
- Use a different or stronger model than the one being tested when possible, to reduce self-preference bias.
- Check the judge against a sample of human labels.

### 4. Run it in a loop

```python
def run_eval(cases, prompt):
    passed = 0
    for c in cases:
        out = call_claude(prompt, c["input"])
        passed += grade(out, c)
    return passed / len(cases)
```

Track accuracy **plus** cost (tokens) and latency. A prompt that gains 1% accuracy but triples cost may be a bad trade. Run the eval in CI so every prompt or model change gets a score (see `ci-notes.md`).

### 5. Evaluating agents and tool use

For agents (lesson 004), final-answer checks are not enough. Also grade:
- **Tool selection**: did it call the right tool with correct arguments?
- **Trajectory**: number of steps, wasted calls, loops.
- **Outcome**: is the end state correct (file changed, ticket created)? Check state, not the transcript.
- **Safety**: did it avoid forbidden actions?

Run each case several times, since agent behavior varies; report a pass rate.

## Common exam traps

- **Judging by a few hand-picked examples.** That is a demo, not an eval.
- **Tuning the prompt on the test set.** This overfits; keep a held-out set.
- **Using an LLM judge with a vague rubric** or one never validated against humans.
- **Only measuring accuracy.** Cost and latency must be weighed too.
- **Grading an agent only on its final text** when the real effect is in the system state.
- **Assuming a model upgrade is safe.** Re-run evals on every model or prompt change.

## Practice questions

1. Your classifier outputs one of five labels. Which grader do you choose first?
2. Your LLM judge passes almost everything. What steps do you take?
3. A new prompt raises accuracy from 90% to 91% but doubles tokens per request. How do you decide?

<details>
<summary>Answers</summary>

1. A code-based grader (exact match on the label). It is cheap and deterministic; no judge is needed.
2. Tighten the rubric to specific, checkable criteria with a binary result, and compare the judge's grades to a human-labeled sample to calibrate it. A different or stronger judge model also helps.
3. Weigh the gain against cost and latency requirements using the eval's accuracy, token, and latency numbers. One point of accuracy rarely justifies double the cost unless errors are very expensive; consider caching or a smaller model as alternatives.

</details>

## Key takeaways

- An eval = test set + grader + tracked score; run it on every change.
- Grow the test set from real failures and keep some data held out.
- Prefer code graders; use LLM judges with specific rubrics, validated by humans.
- Measure accuracy, cost, and latency together.
- For agents, grade tool choice, trajectory, and final state.
