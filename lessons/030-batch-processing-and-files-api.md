# Message Batches and the Files API: Bulk Workloads and Reusable Inputs

## What you will learn

- When the Message Batches API is the right tool, and when it is not
- How to submit a batch, poll it, and match results back to requests with `custom_id`
- How batches combine with prompt caching and what that does to cost
- How the Files API avoids re-uploading the same large document in every request
- Which exam traps separate "cheap and slow" from "fast and interactive"

## Explanation

### Message Batches: asynchronous work at a discount

The Message Batches API takes many Messages API requests in one call and processes them **asynchronously**. In return you get a **discount (about 50%)** on token prices. Most batches finish within an hour, but the guarantee is **up to 24 hours**, so never use batches for anything a user is waiting on.

Good fits: nightly document classification, bulk summarization, running a large evaluation set, backfilling embeddings-style labels, content moderation of an archive.

Poor fits: chat, agents that loop with tool results, anything with a latency target.

Key facts:

- Each request in the batch is a normal Messages request (model, `max_tokens`, `messages`, tools, system prompt) plus a **`custom_id`** you choose.
- Batches have size limits (on the order of 100,000 requests or a few hundred MB per batch). Split larger jobs.
- Results are **not guaranteed to be in input order**. Always match on `custom_id`.
- Each request succeeds or fails **independently**. A batch can finish with a mix of results.
- Results stay available for a limited time (about 29 days), so download and store them.

### Submit, poll, collect

```python
batch = client.messages.batches.create(requests=[
    {
        "custom_id": f"doc-{doc.id}",
        "params": {
            "model": "claude-sonnet-5-5",
            "max_tokens": 500,
            "system": CLASSIFIER_PROMPT,
            "messages": [{"role": "user", "content": doc.text}],
        },
    }
    for doc in docs
])

# Poll until processing_status == "ended"
while client.messages.batches.retrieve(batch.id).processing_status != "ended":
    time.sleep(60)

for entry in client.messages.batches.results(batch.id):
    if entry.result.type == "succeeded":
        save(entry.custom_id, entry.result.message)
    elif entry.result.type == "errored":
        retry_later.append(entry.custom_id)
    # other types: "canceled", "expired"
```

Result types per request: `succeeded`, `errored`, `canceled`, `expired`. A robust job collects the non-succeeded IDs and resubmits only those in a new batch.

### Batches plus prompt caching

Batch requests can use prompt caching (lesson 003). If every request shares a long system prompt or reference document, mark it with `cache_control`. Savings stack: the batch discount applies, and cache reads are cheaper still. Because batch requests are processed in no fixed order and concurrently, cache hits are **best effort**, not guaranteed. Keep the shared prefix identical and byte-for-byte the same across requests to maximize hits.

### Designing a batch pipeline

1. Build requests with stable `custom_id`s (for example the database primary key), so reruns are idempotent.
2. Validate a small sample **synchronously** first. A bad prompt in a 50,000-request batch wastes a whole run.
3. Submit, store the batch ID, and poll on a schedule (or from a cron job), not in a blocking loop in a web request.
4. Handle partial failure by retrying only failed IDs.
5. Record token usage per result for cost reporting.

### The Files API: upload once, reference many times

Without it, each request must carry the full document as base64 or text, which costs upload bandwidth and request size every time. With the **Files API** you upload a file once, receive a `file_id`, and reference that ID in later requests (for PDFs, images, and text documents).

```python
uploaded = client.beta.files.upload(file=open("contract.pdf", "rb"))

client.beta.messages.create(
    model="claude-sonnet-5-5",
    max_tokens=1000,
    messages=[{"role": "user", "content": [
        {"type": "document", "source": {"type": "file", "file_id": uploaded.id}},
        {"type": "text", "text": "List the termination clauses."},
    ]}],
    betas=["files-api-2025-04-14"],
)
```

Points to remember:

- The Files API was a **beta feature** that needs a beta header or SDK `betas` parameter. Check current docs for the exact flag.
- A file ID saves **upload size**, not **tokens**. The document's content still counts as input tokens each time it is used. To cut token cost, combine it with prompt caching.
- Files belong to your workspace, so treat IDs as access-controlled resources and delete files you no longer need (data governance, lesson 024).
- It is useful for multi-turn document Q&A and for batches that reference the same large file.

### Choosing the right mechanism

| Need | Use |
| --- | --- |
| Many independent requests, no rush, lower cost | Message Batches |
| Same long prefix across many calls, fast responses | Prompt caching |
| Same large file across many calls | Files API (plus caching) |
| Interactive, low latency | Normal or streaming Messages calls |
| Multi-step tool loop | Synchronous agent loop, not a batch |

## Common exam traps

- **"Batches are always faster."** No. They trade latency for cost. Completion can take up to 24 hours.
- **Assuming results come back in order.** They do not. Use `custom_id`.
- **Treating a finished batch as all successes.** `processing_status: ended` means processing is over, not that every request succeeded.
- **Using batches for an agent loop.** Each tool round trip needs the previous result, so the loop cannot be one batch. (You could batch independent first steps only.)
- **Thinking the Files API reduces token usage.** It reduces payload size only. Tokens are still billed.
- **Expecting guaranteed cache hits inside a batch.** They are best effort.
- **Reusing duplicate `custom_id`s.** They must be unique within a batch.

## Practice questions

1. A team must tag 200,000 support tickets overnight, and the report is due next morning. Cost matters most. Which approach fits?

2. After a batch ends, 3% of the entries have type `errored`. What should the pipeline do?

3. An app asks 40 different questions about the same 300-page PDF over one session, with users waiting for each answer. Which combination is best?

<details>
<summary>Answers</summary>

1. Use the Message Batches API, split into multiple batches if the size limit requires it. The work is asynchronous and tolerant of delay, and the discount is about 50%. Add prompt caching for the shared instructions, and test a small sample synchronously first.

2. Collect the `custom_id`s of the failed entries, check the error types (invalid request errors need a fix, while transient or overloaded errors can be retried), then resubmit only those requests in a new batch. Do not rerun the whole batch.

3. Use synchronous Messages calls (users are waiting, so no batch). Upload the PDF once through the Files API (or include it once) and mark the document with prompt caching so repeated questions read the cached prefix at lower cost and latency. The Files API alone would still bill the document tokens each time.

</details>

## Key takeaways

- Batches = asynchronous, about 50% cheaper, up to 24 hours, never for interactive or looping work.
- Match results with `custom_id`; handle `succeeded`, `errored`, `canceled`, and `expired` separately, and retry only failures.
- Batch discounts stack with caching, but cache hits in batches are best effort.
- The Files API saves upload size and re-sending, not tokens. Pair it with caching to reduce cost.
- Pick the mechanism from the latency requirement first, then optimize cost.
