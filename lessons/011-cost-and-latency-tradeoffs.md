# Cost and Latency Trade-offs: Model Choice, Batching, Streaming, and Output Control

## What you will learn

- How cost and latency are driven by input tokens, output tokens, and model choice
- When to use the Message Batches API instead of real-time calls
- How streaming improves *perceived* latency without changing total time
- Which levers cut cost and latency (caching, `max_tokens`, routing, shorter outputs)
- How to pick the right lever for a given scenario

## Explanation

### What you pay for and wait for

- **Cost** = input tokens x input price + output tokens x output price. Output tokens cost more per token than input tokens.
- **Latency** has two parts: time to first token (mostly input size and queueing) and generation time (mostly the number of output tokens).
- Larger, more capable models usually cost more and respond more slowly than smaller ones. Check current model names and prices in the official docs rather than memorizing numbers.

Rule of thumb: **output length is the biggest latency lever; repeated input is the biggest cost lever.**

### Lever 1: Choose the right model per task (routing)

Do not send everything to the biggest model. Route by difficulty:

```python
def pick_model(task_type: str) -> str:
    if task_type in ("classify", "extract", "route"):
        return SMALL_MODEL      # fast and cheap
    if task_type in ("summarize", "draft"):
        return MID_MODEL
    return LARGE_MODEL          # hard reasoning, planning, tricky code
```

A common architecture: a small model triages requests and a large model handles only the hard ones. In agents (lesson 009), subagents doing simple lookups can use a smaller model than the orchestrator. Validate the choice with an eval set (lesson 007), not by guessing.

### Lever 2: Prompt caching

Large stable prefixes (system prompt, tool definitions, documents) can be cached so repeat reads are billed at a reduced rate and start faster. See lesson 003. Keep stable content first and variable content last, or the cache will not hit.

### Lever 3: Message Batches API

For work that is **not interactive**, submit many requests as a batch:

- Typically a **discount (about 50%)** versus standard pricing.
- Results are asynchronous: usually done within hours, with a maximum window of 24 hours. No latency guarantee.
- Each request has a `custom_id`; results can come back in **any order**, so match by `custom_id`, never by position.

```python
batch = client.messages.batches.create(requests=[
    {"custom_id": f"doc-{i}",
     "params": {"model": MODEL, "max_tokens": 300,
                "messages": [{"role": "user", "content": doc}]}}
    for i, doc in enumerate(docs)
])
# later: poll batch status, then read results and join on custom_id
```

Good fits: nightly document classification, bulk evals, backfills, content generation. Bad fits: a chat UI, anything where a user is waiting, or workflows where the next step needs the answer immediately.

Batching and caching can be combined for extra savings on shared prefixes.

### Lever 4: Streaming

Streaming (`stream=True`) sends tokens as they are generated. Total generation time is about the same, but **time to first visible token** drops, so the app feels faster. It also helps avoid timeouts on long outputs. It does not reduce cost.

### Lever 5: Control output size

- Set `max_tokens` to a sensible cap. It is a ceiling, not a target, and a response cut off by it has `stop_reason: "max_tokens"`.
- Ask for concise formats: "Answer in one word", "Return JSON only", "At most 3 bullets".
- Do not request long reasoning where a label suffices. Extended thinking improves hard tasks but adds output tokens, so enable it only where evals show a benefit.
- Trim conversation history (lesson 010) so input does not grow every turn.

### Choosing a lever

| Situation | Best lever |
|---|---|
| Same long document asked about many times | Prompt caching |
| 100,000 documents to label overnight | Batch API (+ small model) |
| Chat feels slow before anything appears | Streaming |
| Simple classification on an expensive model | Model routing |
| Responses are long and slow | Tighter prompt and `max_tokens` |

## Common exam traps

- **Using Batch for interactive features.** Batch has no latency guarantee; it is wrong when a user is waiting.
- **Assuming batch results are ordered.** They are not; use `custom_id`.
- **Thinking streaming makes the request cheaper or finish sooner.** It improves perceived latency only.
- **Treating `max_tokens` as a target.** It only caps output; it does not make Claude write that much, and too low a value truncates answers.
- **Picking the largest model by default.** The exam favors the cheapest model that passes the evals.
- **Caching volatile content.** Putting a timestamp at the start of the prompt breaks prefix caching.

## Practice questions

<details>
<summary>Question 1: A company must classify 2 million support tickets overnight, and results are needed by morning. Which approach is the most cost-effective?</summary>

**Answer:** Use the Message Batches API with a small, cheap model (validated by an eval). The work is non-interactive and tolerates delay, so the batch discount applies, and classification does not need a large model.
</details>

<details>
<summary>Question 2: Users complain that a chatbot shows nothing for several seconds, although answers are good. Total response time is acceptable. What is the best fix?</summary>

**Answer:** Enable streaming so tokens appear as they are generated. This reduces time to first visible token. Batching would make it worse, and a bigger model would not help.
</details>

<details>
<summary>Question 3: A batch of 500 requests finishes. Your code pairs result i with input i by list position and some answers look mismatched. What went wrong and how do you fix it?</summary>

**Answer:** Batch results are not guaranteed to be in submission order. Give each request a unique `custom_id` and join results to inputs on that ID.
</details>

## Key takeaways

- Output tokens drive latency; repeated input drives cost.
- Route simple tasks to small models and verify with evals.
- Use caching for repeated prefixes, Batch for offline bulk work, streaming for perceived speed.
- `max_tokens` is a cap, not a target; match batch results by `custom_id`.
- Always choose the lever that fits the constraint: cost, throughput, or responsiveness.
