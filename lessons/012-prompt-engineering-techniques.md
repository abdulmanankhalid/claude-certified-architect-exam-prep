# Prompt Engineering Techniques: Clear Instructions, XML Structure, Examples, and Structured Output

## What you will learn

- How to write clear, specific prompts and why context and the "why" improve results
- How to separate instructions from data using XML-style tags
- When and how to use few-shot examples
- How to get reliable structured output (and why tool use is the strongest option)
- How to give Claude room to reason before answering, and how to iterate on prompts with evals

## Explanation

### 1. Be clear, specific, and give context

Treat Claude like a smart new colleague who has no background on your task. Vague prompts give vague results.

Weak:

```
Summarize this ticket.
```

Better:

```
You are helping a support lead triage tickets. Summarize the ticket below in
2 sentences for a busy manager. State the customer's problem first, then the
requested action. Do not include greetings or customer names.
```

Good prompts state: the task, the audience, the output format, the constraints, and *why* a rule exists ("Do not use markdown, because the output is read aloud by a text-to-speech engine"). Giving the reason lets Claude handle edge cases you did not list.

Prefer positive instructions ("write in plain prose") over only negative ones ("don't use bullets"), and keep instructions free of contradictions.

### 2. Use the system prompt for role and stable rules

The `system` parameter (see lesson 001) holds the role, rules, and tone that stay constant. The `user` turn holds the per-request task and data. Putting stable text in the system prompt also makes it a good prompt-caching prefix (lesson 003).

### 3. Separate instructions from data with XML tags

When a prompt mixes instructions, documents, and examples, wrap each part in descriptive tags. This removes ambiguity about where the data starts and ends, and it helps against prompt injection (lesson 008), though it is not a complete defense.

```
<instructions>
Extract every action item from the meeting notes. Return one per line.
</instructions>

<meeting_notes>
{{notes}}
</meeting_notes>
```

Tips:
- There are no "magic" tag names. Pick consistent, meaningful names (`<document>`, `<policy>`, `<example>`).
- Put long documents **first** and the question/instructions **after** them; this usually improves quality on long inputs.
- Refer to tags by name in the instructions ("using the text in `<policy>`...").

### 4. Few-shot examples

Examples are one of the most reliable ways to control format, tone, and edge-case handling. Show 3 to 5 that are:

- **Relevant**: they look like real inputs.
- **Diverse**: they cover edge cases, so Claude does not copy one narrow pattern.
- **Clearly marked**: wrap them in `<example>` tags so they are not mistaken for the real input.

```
<examples>
<example>
<input>Refund me, the app crashed twice!</input>
<output>{"category": "billing", "urgency": "high"}</output>
</example>
<example>
<input>How do I change my avatar?</input>
<output>{"category": "how_to", "urgency": "low"}</output>
</example>
</examples>
```

Trade-off: examples cost input tokens on every call. Cache them if they are large and stable.

### 5. Getting structured output

Options, from weakest to strongest guarantee:

1. **Ask in the prompt** ("Return only JSON with keys a, b, c") and show an example. Works often, but you must still validate and handle bad output.
2. **Tags to delimit the answer** (`<answer>...</answer>`), then parse with a regex. Easy to extract from text that also contains reasoning.
3. **Tool use with a JSON schema** (lesson 002). Define a tool whose `input_schema` is the shape you want and force it with `tool_choice: {"type": "tool", "name": "record_ticket"}`. The `input` of the returned `tool_use` block is your structured data. This is the standard architect answer for "reliable structured extraction".
4. Any approach: **validate** the result in code and retry with the error message on failure.

Note: pre-filling the start of the assistant turn was an older trick for forcing format; it is not supported on all newer models, so prefer tool use or tags for portability.

### 6. Let Claude think before answering

For multi-step reasoning, give Claude space to work: ask it to reason step by step in `<thinking>` tags before the final `<answer>`, or enable extended thinking via the API. Reasoning that never gets generated cannot improve the answer. The trade-off is more output tokens and latency (lesson 011), so use it for hard tasks, not simple lookups.

### 7. Chain prompts and iterate with evals

Break a complex job into steps (extract, then analyze, then write), each with a focused prompt, when a single prompt is unreliable. Chaining costs more calls but is easier to debug.

Never judge a prompt change on one example. Keep a test set and compare before/after (lesson 007). Change one thing at a time.

## Common exam traps

- **"Add more capital letters and urgent words (CRITICAL, NEVER)"**: shouting is usually a worse fix than explaining the rule and its reason; it can cause over-application on newer models.
- **Relying on "return only JSON" for production parsing**: the exam answer is a forced tool call with a schema plus code-side validation.
- **Putting variable data in the system prompt** when it should be in the user turn; it also breaks caching.
- **One example only**: Claude may over-copy it. Use diverse examples.
- **Believing XML tags are required syntax**: they are a convention for clarity, not an API feature.
- **Using prompt changes to fix what code should enforce**: hard requirements (valid JSON, permissions, limits) need schema, validation, or hooks, not just wording.
- **Judging prompts by vibes**: the correct process is a test set and measured comparison.

## Practice questions

1. Your pipeline parses Claude's JSON reply, but about 2% of responses include a sentence of prose before the JSON and break the parser. What is the most robust fix?
2. A prompt contains a 20,000-token contract and a question. Where should each go, and how should they be marked up?
3. Claude keeps classifying sarcastic reviews as positive even though your instructions say "classify sentiment accurately". Name two prompt-level improvements.

<details>
<summary>Answers</summary>

1. Define a tool whose `input_schema` matches the JSON and force it with `tool_choice` naming that tool; read the `tool_use` block's `input`. Also validate in code and retry on failure. Merely adding "return only JSON" is less reliable.
2. Put the contract first, wrapped in a tag such as `<contract>`, and put the instructions and question after it (optionally in `<instructions>` / `<question>` tags). Long data first, query last, generally gives better results.
3. Add diverse few-shot examples that include sarcastic reviews labeled correctly (in `<example>` tags), and explain the rule and why ("sarcasm expresses the opposite of the literal words, so judge the intended sentiment"). Optionally let Claude reason before labeling. Then check the change against a test set.

</details>

## Key takeaways

- Be specific: task, audience, format, constraints, and reasons.
- Stable role and rules go in the system prompt; per-request data goes in the user turn.
- Use descriptive XML tags to separate instructions, documents, and examples; put long documents before the question.
- 3 to 5 relevant, diverse examples beat long descriptions of format.
- For reliable structured output, use forced tool use with a schema, and validate in code.
- Let Claude reason before answering on hard tasks, at a token and latency cost.
- Measure every prompt change against an eval set.
