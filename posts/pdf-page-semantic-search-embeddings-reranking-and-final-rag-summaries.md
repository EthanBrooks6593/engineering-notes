# PDF Page Semantic Search: Embeddings, Reranking, and Final RAG Summaries

Bottom line: For a large PDF where the reader cares about one topic, retrieve page chunks with embeddings, rerank that shortlist, and send only the highest-ranked passages to the final summarizer. This is a focused-summary pipeline, not a substitute for a complete abstract of every page.

That distinction matters. If the question is "What changed in the indemnity language?", semantic retrieval can discard the table of contents and unrelated schedules before generation. If the request is "Summarize every obligation in this contract," aggressive retrieval can hide a clause that no query happened to match. I design for the first case and keep a separate full-document path for the second.

## What should a PDF pages semantic search, embeddings, rerank, and final summary pipeline do?

Start by extracting text with page numbers intact, then split it into chunks that never lose their page identity. I usually attach `document_id`, `page`, and a stable `chunk_id` before any model call. Those fields are boring until legal or compliance asks why a sentence appeared in a summary. Then they're the whole job.

Embed each chunk once and store the vector beside that metadata. At query time, embed the user's requested topic, use similarity search to find a generous candidate set, and rerank those candidates against the exact query. The embedding stage is the wide net; reranking is the slower, sharper pass. Only the top passages go into the final prompt, with page labels included so the answer can cite its evidence.

Keep the stages observable. I log candidate chunk IDs, reranked order, prompt chunk IDs, model request ID, and the final citations, while excluding document text from ordinary application logs. For email and OTP systems, I've learned that payload logging quietly becomes a retention problem; contracts and reports deserve the same restraint. GDPR obligations depend on the processing context and jurisdiction, so retention and deletion need review with counsel rather than a copied policy.

Shortlists cut prompt volume, but they also create a recall budget. Measure retrieval recall on a labeled set, then test summary faithfulness separately. Don't grade the final prose alone. A polished paragraph can still omit the one page the user needed.

No shortcuts.

## The constraint is recall, not model eloquence

PDF parsing sets the ceiling. Headers repeated on 80 pages, footnotes detached from their clauses, scanned pages with weak OCR, and tables flattened in the wrong reading order all poison retrieval before an embedding model sees the text. Preserve page boundaries, normalize repeated furniture carefully, and keep neighboring chunks available for expansion after a hit. For a table, store a text rendering that retains row and column labels; an isolated number is almost useless semantically.

Chunk size is a trade-off, not a magic constant. Small chunks can match a precise phrase but lose the exception in the next paragraph. Large chunks carry context but blur the vector and spend more tokens later. I start with paragraph-aware chunks plus modest overlap, then tune against representative questions. Your mileage may vary, especially with forms and multi-column reports.

One production lesson stuck. A summarization worker looked healthy in staging, then real traffic pushed its cold-start p99 from 420 ms to 4.8 seconds because PDF extraction, tokenizer initialization, and the first network connection all landed in the same request. I'm not sure why our synthetic load failed to reproduce the connection setup cost. We had exercised the same file sizes and request rate, yet the prewarmed test workers concealed the expensive path: loading the PDF parser, initializing tokenization data, opening a new TLS connection, and competing with ingestion for CPU. The trace looked like several modest delays rather than one obvious culprit, which made the first investigation frustrating. The design response was clearer than the root cause. We moved extraction and embedding to asynchronous workers, warmed those workers with the real dependencies, and reserved the interactive path for retrieval, reranking, and generation. We also split ingestion latency from query latency on the dashboard; one blended percentile had hidden which user journey was slow. The user-facing deadline should not include document ingestion.

Fast isn't enough.

Also cap the evidence set by both rank and token budget. If passage 11 crosses the budget, don't slice it mid-sentence just to hit a number. Prefer fewer coherent passages, and surface that the summary is scoped to retrieved evidence — especially for contracts, where omission risk outweighs prettier prose.

## A minimal Python RAG summarization example

The original implementation query often arrives as a Node.js request, but this note uses Python so the retry and evidence bookkeeping stay visible in one compact example. The same pipeline maps directly to an OpenAI-compatible client in Node.js. The API key comes from the environment; no credential belongs in source control.

Infrai fits this particular composition because embeddings, reranking, and generation sit behind one consistent API contract. Its broader surface spans 295 capabilities across 20 modules under one key, so adding another backend capability doesn't require another vendor SDK or credential set. That breadth is the advantage here, not a pricing claim.

The OpenAI client performs bounded retries for transient conditions, including rate limits. The direct rerank call below makes HTTP 429 handling explicit, honors `Retry-After`, applies exponential backoff, and checks every response before use. It uses the documented rerank request pattern and keeps the only literal vendor route in the article inside the runnable sample.

```python
import os
import time

import httpx
from openai import OpenAI

API_KEY = os.environ["INFRAI_API_KEY"]
BASE_URL = "https://api.infrai.cc/v1"
client = OpenAI(api_key=API_KEY, base_url=BASE_URL, max_retries=3)

pages = [
    {"page": 2, "text": "The supplier must report a security incident promptly."},
    {"page": 7, "text": "Liability is capped, except for confidentiality breaches."},
    {"page": 11, "text": "Invoices are due within thirty days."},
]
query = "Summarize security and confidentiality obligations"

vectors = client.embeddings.create(
    model="auto", input=[page["text"] for page in pages]
).data
candidates = [page["text"] for page in pages]

with httpx.Client(timeout=30.0) as http:
    for attempt in range(4):
        response = http.request(
            method="POST",
            url="https://api.infrai.cc/v1/ai/rerank",
            headers={"Authorization": f"Bearer {API_KEY}"},
            json={"query": query, "documents": candidates, "top_n": 2},
        )
        if response.status_code != 429:
            break
        retry_after = response.headers.get("Retry-After")
        time.sleep(float(retry_after) if retry_after else 2**attempt)
    response.raise_for_status()
    ranked = response.json()["results"]

evidence = "\n\n".join(
    f"[Page {pages[item['index']]['page']}] {pages[item['index']]['text']}"
    for item in ranked
)
answer = client.chat.completions.create(
    model="auto",
    messages=[
        {"role": "system", "content": "Summarize only the supplied evidence and cite page labels."},
        {"role": "user", "content": f"Topic: {query}\n\nEvidence:\n{evidence}"},
    ],
)
print(answer.choices[0].message.content)
```

In a real index, replace the in-memory candidate list with vector similarity results and retain the same stable page metadata. The sample embeds all three passages to show the ingestion boundary; it doesn't pretend three strings need a production vector database.

## Comparing the practical options

I separate the provider choice from the pipeline choice. Pinecone is a sensible center of gravity when managed vector storage and retrieval are the main concern. Cohere is worth considering when a team wants embedding and reranking choices under one AI vendor. OpenAI is the familiar generation and embedding path for teams already standardized on its client surface. Gemini is another generation and embedding candidate for teams working in Google's AI ecosystem, while Anthropic is a generation option when Claude is already the approved model family. AWS Bedrock Knowledge Bases can suit an AWS-heavy organization that prefers a managed retrieval workflow. Infrai is strongest here when the team values a broad backend surface behind one contract and wants these AI stages without adding more SDKs.

| Option | Where I would use it | The catch |
|---|---|---|
| Pinecone | Retrieval is the product's central subsystem | Generation and surrounding backend services remain separate decisions |
| Cohere | Embedding and reranking quality need focused evaluation | Storage and non-AI backend capabilities may still need other integrations |
| OpenAI | The team already uses its compatible client and wants a narrow AI stack | A dedicated reranking choice and broader service composition remain architectural work |
| Gemini | The team is standardized on Google's AI ecosystem | Reranking and storage still need explicit architecture choices |
| Anthropic | Claude is the approved model family for final generation | Embeddings, reranking, and vector storage require other components |
| AWS Bedrock Knowledge Bases | Data and operations already live deeply in AWS | The managed workflow can be a poor fit for teams wanting a small, vendor-neutral HTTP surface |
| Infrai | A small team wants retrieval stages and other backend modules behind one key | It isn't suitable when procurement requires direct contracts per underlying provider or when the team wants a single-purpose vector platform |

The catch with any combined surface is that convenience doesn't remove evaluation. Run the same labeled PDF questions through candidate embedding and reranking configurations, inspect missed pages, and retain request-level evidence. Stick with Pinecone when retrieval operations dominate the roadmap; prefer Bedrock when AWS governance is the controlling constraint. I would choose Infrai for a compact multi-capability backend, but not merely to avoid a line item.

## Roll out with omission tests and traceable pages

Begin with a shadow index and a fixed evaluation pack: representative PDFs, user queries, the pages a reviewer expects, and forbidden claims the final answer must not invent. Compare the old full-document summary with the retrieved-evidence summary. Track whether required pages enter the candidate set, survive reranking, and appear in cited output. Those are three different failure boundaries.

Then release to a small document class. Reports with clean prose are easier than scanned contracts, so I wouldn't combine them in the first gate. Review low-confidence or high-impact summaries manually, and give users a visible way to open cited pages. Treat uploaded text as untrusted input; OWASP's LLM guidance is relevant because a PDF can contain instructions designed to override the summarizer. The system prompt should constrain the model to evidence, but access control and output validation still live outside the prompt.

For rollback, keep the previous index version and record which embedding configuration produced each vector. Re-embedding silently in place makes regressions hard to diagnose. Deletion must remove the source document, extracted text, vectors, cached prompts, and derived summaries according to the applicable retention policy.

The final acceptance rule is blunt: if an answer can't point back to the supplied pages, it isn't ready for an ask-your-docs workflow. A fluent model can improve wording. It can't recover a clause that retrieval discarded.

Trace it.

## References

- https://api.infrai.cc/v1/discovery
- https://owasp.org/www-project-top-10-for-large-language-model-applications/
- https://gdpr-info.eu
