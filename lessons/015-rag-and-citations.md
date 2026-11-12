# Retrieval-Augmented Generation (RAG): Grounding Claude in Your Own Data

## What you will learn

- What RAG is and when to choose it over fine-tuning or stuffing everything into the prompt
- The pipeline: chunk, embed, retrieve, (rerank), generate
- How to place retrieved context in the prompt and ask for grounded answers
- How citations reduce hallucination and make answers verifiable
- How to evaluate retrieval and generation separately

## Why RAG?

Claude only knows its training data and what is in the context window. Your private, fresh, or very large knowledge (wiki, tickets, manuals) is not there. RAG fixes this: at question time, **retrieve** the few relevant pieces and **put them in the prompt**.

| Approach | Good for | Weakness |
|---|---|---|
| Put all documents in the prompt | Small corpora (fits the window; combine with prompt caching, lesson 003) | Cost, latency, and quality drop as context grows (lesson 010) |
| RAG | Large or frequently changing corpora | Answer quality depends on retrieval quality |
| Fine-tuning | Style or format changes | Poor for adding facts; slow to update; not a way to keep knowledge current |

Exam rule of thumb: **new or changing facts -> retrieval. Behavior or style -> prompting (or tuning).**

## The pipeline

1. **Ingest (offline):** split documents into chunks (often a few hundred tokens, with some overlap, ideally along natural boundaries such as headings). Compute an embedding vector for each chunk and store it in an index, with metadata (source, title, date).
2. **Retrieve (online):** embed the user query, find the nearest chunks. Hybrid search (vector + keyword such as BM25) usually beats either alone, because keywords catch exact IDs and names that embeddings miss.
3. **Rerank (optional):** retrieve broadly (say 50), then use a reranker model to keep the best 5 to 10.
4. **Generate:** give Claude the chunks plus the question.

Claude does not produce embeddings itself; use a separate embedding model or provider for steps 1 and 2.

## Putting context in the prompt

Put long documents first and the question last. Wrap each chunk in tags with metadata, and tell Claude to answer only from them (see lesson 012).

```python
docs = "\n".join(
    f'<document index="{i}"><source>{c.source}</source>'
    f"<content>{c.text}</content></document>"
    for i, c in enumerate(chunks, 1)
)
prompt = f"""<documents>
{docs}
</documents>

Answer using only the documents above. Cite the document index for each claim.
If the documents do not contain the answer, say "I don't know".

Question: {question}"""
```

Two instructions matter most: **use only the provided documents** and **say so when the answer is missing**. Without the second, Claude may fall back on guesses.

## Citations

The Messages API supports a **citations** feature: you pass documents as content blocks with citations enabled, and the response text blocks come back with structured references to the exact passages used.

```python
{"type": "document",
 "source": {"type": "text", "media_type": "text/plain", "data": policy_text},
 "title": "Refund policy",
 "citations": {"enabled": True}}
```

Benefits over asking for citations in a prompt:
- Citations point to real passages in the supplied documents, so they cannot be invented the way a free-text "[doc 3]" can.
- The quoted cited text is not billed as output tokens.
- Your UI can show the source next to each claim.

Even with citations, verify important claims; a citation shows where the model looked, not that its reading is right.

## Retrieval as a tool

Instead of retrieving once before the call, expose `search_docs(query)` as a tool (lesson 002) and let Claude run the agentic loop (lesson 004). Claude can rewrite the query, search again, or search several topics. This costs more calls but handles multi-step questions. A good design: simple one-shot retrieval for FAQ-style bots; retrieval-as-tool for research-style agents. MCP (lesson 005) is a common way to expose such a search service.

## Security

Retrieved text is **untrusted input**. A poisoned document can contain "ignore previous instructions" (lesson 008). Mark retrieved content as data, do not give the model powerful tools it does not need, and apply access control at retrieval time so users only get chunks they are allowed to see. Filtering after generation is too late.

## Evaluating RAG

Test the two halves separately (lesson 007):
- **Retrieval:** is the right chunk in the top-k? (recall@k, MRR)
- **Generation:** given the right chunks, is the answer faithful (supported by the context) and complete?

If answers are wrong, first check whether the right chunk was retrieved. Improving the prompt will not fix missing context.

## Common exam traps

- **Choosing fine-tuning to add up-to-date facts.** Retrieval is the answer.
- **Thinking a bigger context window removes the need for RAG.** Cost, latency, and attention dilution still apply.
- **Chunks that are too big or too small.** Too big wastes context and buries the answer; too small loses meaning. Overlap and boundary-aware splitting help.
- **Vector-only search for exact identifiers** (error codes, SKUs). Add keyword search.
- **No "I don't know" path**, which invites hallucination.
- **Enforcing permissions after generation** instead of at retrieval.
- **Blaming the model** for bad answers when retrieval returned the wrong chunks.

## Practice questions

1. A support bot must answer from a product manual that is updated weekly. Fine-tune or RAG?
2. Users search for error code `E-40412` and the vector search returns unrelated chunks. What do you change?
3. Your legal team wants each claim in an answer linked to the exact source passage. What feature fits?

<details>
<summary>Answers</summary>

1. RAG. Facts change weekly; re-index the changed pages rather than retrain. Fine-tuning is slow and poor at injecting facts.
2. Add keyword search (hybrid, e.g. BM25 plus vectors), optionally with a reranker. Embeddings are weak on exact identifiers.
3. The citations feature on document content blocks. It returns structured references to the cited passages, which the UI can display.

</details>

## Key takeaways

- RAG = retrieve relevant chunks at query time, then generate from them; best for large or changing knowledge.
- Quality depends on chunking, hybrid search, and reranking.
- Tell Claude to answer only from the documents and to admit when the answer is absent.
- Use the citations feature for verifiable, passage-level sources.
- Treat retrieved text as untrusted and enforce access control at retrieval.
- Evaluate retrieval and generation separately.
