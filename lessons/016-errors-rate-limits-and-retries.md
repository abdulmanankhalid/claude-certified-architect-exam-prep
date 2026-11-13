# Reliability: API Errors, Rate Limits, Retries, and Graceful Degradation

## What you will learn

- The main HTTP error classes of the Claude API and which are worth retrying
- How rate limits work and how to respond to a 429
- How to build retries with exponential backoff and jitter
- How to handle non-HTTP failures: `stop_reason` of `max_tokens`, truncated streams, and invalid output
- Architecture patterns for degrading gracefully (fallback models, queues, idempotency)

## Explanation

Any production app built on Claude will see failures. A good architect decides **which errors to retry, which to fix, and which to surface**.

### Error classes

| Status | Meaning | Retry? |
|---|---|---|
| 400 `invalid_request_error` | Malformed request (bad params, bad message order) | No. Fix the request. |
| 401 `authentication_error` | Bad or missing API key | No. |
| 403 `permission_error` | Key lacks access to the resource | No. |
| 404 `not_found_error` | Unknown model or resource | No. |
| 413 `request_too_large` | Request exceeds size limit | No. Shrink the input. |
| 429 `rate_limit_error` | You hit a rate limit | Yes, after waiting. |
| 500 `api_error` | Unexpected server error | Yes. |
| 529 `overloaded_error` | API temporarily overloaded | Yes, with backoff. |

Rule of thumb: **4xx (except 429) means your request is wrong, so retrying the same request will fail again. 429 and 5xx are transient, so retry.**

### Rate limits

Limits are measured in requests per minute (RPM), input tokens per minute (ITPM), and output tokens per minute (OTPM), and they depend on your usage tier. A 429 response includes a `retry-after` header saying how many seconds to wait. Honor it before retrying. Cached input tokens (see lesson 003) generally count less against input-token limits, so caching helps throughput as well as cost.

Ways to stay under limits:
- Smooth traffic with a queue and a concurrency cap instead of bursts.
- Use the Batch API for non-urgent work (lesson 011).
- Cut tokens: trim context (lesson 010), cache prefixes, pick a smaller model where quality allows.

### Retries with exponential backoff and jitter

```python
import random, time
import anthropic

client = anthropic.Anthropic(max_retries=0)  # we handle retries ourselves here

RETRYABLE = (anthropic.RateLimitError, anthropic.InternalServerError,
             anthropic.APIConnectionError, anthropic.APITimeoutError)

def call_with_retry(max_attempts=5, **kwargs):
    for attempt in range(max_attempts):
        try:
            return client.messages.create(**kwargs)
        except RETRYABLE as e:
            if attempt == max_attempts - 1:
                raise
            delay = min(30, 2 ** attempt) + random.uniform(0, 1)  # backoff + jitter
            retry_after = getattr(getattr(e, "response", None), "headers", {}).get("retry-after")
            if retry_after:
                delay = max(delay, float(retry_after))
            time.sleep(delay)
```

- **Exponential backoff** (1s, 2s, 4s, ...) gives a struggling service room to recover.
- **Jitter** (random extra delay) stops many clients from retrying at the same instant (the "thundering herd").
- **Cap** both the delay and the number of attempts.
- The official SDKs already retry some errors by default (connection errors, 408, 409, 429, 5xx; default 2 retries). You can tune this with `max_retries`. Do not stack your own retry loop on top without disabling or accounting for the SDK's.

### Timeouts and streaming

Long, non-streamed requests can hit network timeouts. For large `max_tokens`, use **streaming** (lesson 011). An error can also arrive *mid-stream* (for example an `overloaded_error` event after a 200 response started). Your code must handle that: discard or resume the partial output, then retry.

### Failures that are not HTTP errors

A 200 response can still be a problem. Always check `stop_reason`:

| `stop_reason` | What to do |
|---|---|
| `end_turn` | Normal. |
| `max_tokens` | Output was cut off. Raise `max_tokens`, ask for shorter output, or continue the turn. Never parse truncated JSON. |
| `tool_use` | Run the tool and continue the loop (lesson 004). |
| `refusal` | Claude declined for safety reasons. Do not blindly retry. Surface it or change the request. |
| `pause_turn` | A long server-tool turn paused. Send the conversation back to continue. |

For structured output, **validate** it (JSON schema or Pydantic). On failure, retry once with the validation error appended as feedback, then fall back. Do not loop forever.

### Graceful degradation patterns

- **Fallback model**: on repeated 529s, try a different model tier. Make sure prompts and evals (lesson 007) work on the fallback.
- **Queue and async processing**: put work on a queue so spikes become delay rather than failure.
- **Idempotency**: a retry may repeat a side effect. If a tool sends an email or charges a card, use an idempotency key so a repeated tool call is safe. Note that retrying the *API call* is safe, but re-running *tools* is your responsibility.
- **Circuit breaker**: after many consecutive failures, stop calling for a while and serve a cached or static response.
- **Observability**: log the `request-id` response header, status, latency, and token usage. Anthropic support uses the request id to investigate problems.

## Common exam traps

1. **Retrying 400 errors.** A malformed request will never succeed. Only 429, 5xx (including 529), and connection errors are retryable.
2. **Backoff without jitter.** Synchronized retries can overload the service again. The best answer includes jitter.
3. **Ignoring `retry-after`.** On a 429, the server tells you how long to wait.
4. **Treating HTTP 200 as success.** `max_tokens` truncation and refusals arrive with status 200.
5. **Retrying forever.** Always cap attempts and surface the error or degrade.
6. **Assuming retries are free of side effects.** Retrying the model call is safe, but tool side effects need idempotency.
7. **Fixing rate limits by adding retries only.** Retries do not raise capacity. Also reduce tokens, smooth traffic, batch, or request a higher tier.

## Practice questions

1. Your service gets `invalid_request_error` (400) on about 2% of calls. A teammate proposes adding five retries with backoff. Is that a good fix?
2. Many workers hit a 429 at the same time and all sleep exactly 2 seconds before retrying, and they get 429 again. What two improvements apply?
3. A response has HTTP 200 and `stop_reason: "max_tokens"`, and your JSON parser throws. What is the root cause and what is the fix?

<details>
<summary>Answers</summary>

1. No. 400 means the request itself is invalid, so retries will fail identically. Log the error message, find the bad inputs (e.g. wrong message ordering, invalid parameter), and fix or validate the request before sending.
2. Add **jitter** so workers do not retry in lockstep, and use **exponential backoff** while honoring the `retry-after` header. Also consider a shared queue or concurrency cap to smooth traffic.
3. The output was truncated at the token limit, so the JSON is incomplete. Check `stop_reason` before parsing; raise `max_tokens`, request more concise output, or continue the generation. Use streaming for large outputs.

</details>

## Key takeaways

- Retry 429, 5xx, 529, and connection errors. Do not retry other 4xx errors.
- Use exponential backoff with jitter, honor `retry-after`, and cap attempts.
- SDKs retry by default. Avoid duplicating that logic.
- Check `stop_reason` and validate output, because success status does not mean a usable answer.
- Design for degradation: queues, fallback models, circuit breakers, idempotent tools, and request-id logging.
