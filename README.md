# GST Assistant — RAG Q&A Bot over Official CBIC Publications

A no-code RAG application built in **n8n** that answers Indian GST compliance questions from official CBIC publications — with source citations, a designed refusal path, and a measured reranking pipeline.

> My RAG app helps small-business owners and accountants answer GST procedure and compliance questions (registration, ITC, returns, refunds — *not* current tax rates) from CBIC's official GST Flyers and FAQ chapters (~110 pages of PDFs) in an n8n chat interface with 95% faithfulness, refusing when the answer isn't in the corpus.

## Architecture

**Workflow 1 — Corpus Ingestion** (`workflows/01-gst-corpus-ingestion.json`)

Manual Trigger → Read 14 PDFs → PDF Loader → Recursive Splitter (3000/400) → OpenAI Embeddings (`text-embedding-3-small`) → Pinecone (insert)

**Workflow 2 — Q&A Chat** (`workflows/02-gst-qa-chat.json`)

Chat Trigger → AI Agent (`gpt-4o-mini` + memory) with Pinecone as a retrieval tool: **dense top-20 → Cohere Rerank → top-6**, multi-query prompting, mandatory citations, refusal-first system prompt.

## Results

15-question evaluation (straightforward / ambiguous / multi-document / unanswerable): **15/15 pass · faithfulness 4.8/5 · 2/2 correct refusals**. Rerank on/off comparison showed reranking fixed one materially wrong answer (Section 29(5) ITC reversal on registration cancellation). Full analysis: `deliverables/evaluation-report.md`.

## Bonus: vibe-coded chat UI

`ui/index.html` is a standalone chat front-end (vanilla HTML/JS, single file, no build step)
that talks directly to the n8n chat webhook — suggested-question chips, typing indicator,
citation-friendly rendering, and a not-tax-advice disclaimer. With the chat workflow active,
serve it locally and open http://localhost:8080:

```
npx http-server ui -p 8080 -c-1
```

## Repository layout

```
corpus/              14 official CBIC PDFs (GST Flyers + FAQ chapters)
workflows/           n8n workflow JSON (import via n8n → Import from File)
ui/                  vibe-coded chat front-end (bonus add-on)
deliverables/
  project-documentation.md    full write-up (framework, iterations, learnings)
  evaluation-report.md        15-question eval + reranking impact analysis
  eval_questions.json         the evaluation set
  eval_answers_rerank_on.json raw answers, production config
  eval_answers_rerank_off.json raw answers, dense-only baseline
```

## Reproduce

1. Self-host n8n (`npx n8n start`), import the two workflow JSONs
2. Create credentials: OpenAI, Pinecone, Cohere
3. Create a Pinecone serverless index: **dense, 1536, cosine**
4. Point both Pinecone nodes at your index, update the corpus path in "Read Corpus PDFs", and allow the path via `N8N_RESTRICT_FILE_ACCESS_TO`
5. Execute Workflow 1 once, then activate Workflow 2 and chat

## Corpus sources (all official/public)

- GST Council e-version flyers: https://www.gstcouncil.gov.in/e-version-gst-flyers
- CBIC "FAQ on GST", 2nd edition: https://cbic-gst.gov.in/pdf/new-faq-on-gst-second-edition.pdf

Built on the n8n no-code track with Claude Code as build assistant.
