# GST Sahayak — a RAG Q&A Bot over Official CBIC Publications

**Week 2 Project · n8n (no-code) track · Bring-your-own use case**

---

## One-liner

> My RAG app helps small-business owners and accountants answer GST procedure and compliance questions (registration, ITC, returns, refunds — *not* current tax rates) from CBIC's official GST Flyers and FAQ chapters (~110 pages of PDFs) in an n8n chat interface with 95% faithfulness, refusing when the answer isn't in the corpus.

## Project overview

India's GST regime generates a constant stream of practical questions from small-business owners: Do I need to register? Can I claim credit on this purchase? Which return do I file, and when? The authoritative answers exist — in dense official CBIC publications most business owners never read.

This project builds a RAG-powered chat assistant over those official publications, built entirely in n8n (no-code track). The bot answers in plain language, cites the source document and section numbers for every claim, and — critically — **refuses to answer** when the information is not in its corpus rather than hallucinating. Tax misinformation is worse than no answer, so the refusal path was designed first.

## The RAG framework (as required by the handout)

| Field | Decision |
|---|---|
| **Use case** | GST procedure/compliance Q&A for small-business owners and accountants, in a chat interface |
| **Corpus** | 14 official CBIC PDFs (~110 pages): 9 topic-wise GST Flyers + 5 chapters extracted from CBIC's "FAQ on GST" (2nd ed.). Source of truth: CBIC (cbic.gov.in / gstcouncil.gov.in). English. |
| **Ingestion + cleaning** | PDFs downloaded from official sources; the 223-page FAQ compilation was split into per-chapter PDFs (registration, payment of tax, ITC, returns, refunds) before ingestion so every file covers exactly one topic |
| **Ingestion + freshness** | Manual one-shot ingestion workflow; re-run on corpus change. Corpus is dated (2017–2019) — the system prompt makes the bot flag change-prone topics and never answer current tax rates |
| **Chunking + embedding** | Recursive character splitter, 3,000 chars (~750 tokens) with 400 overlap → 100 chunks; OpenAI `text-embedding-3-small` (1536-dim). Chunk size and model capacity matched per the handout guidance |
| **Retrieve** | Pinecone serverless (dense, cosine) → top-20 → **Cohere Rerank** → top-6 → agent. Agent does **multi-query retrieval** (exact legal terms + plain-language paraphrase) to compensate for the lack of sparse/BM25 matching |

## Architecture

Two n8n workflows:

**1. Corpus Ingestion** (manual, run once)
```
Manual Trigger → Read Files from Disk (14 PDFs) → Default Data Loader (PDF)
              → Recursive Character Text Splitter (3000/400)
              → Embeddings OpenAI (text-embedding-3-small)
              → Pinecone Vector Store (insert)
```

**2. Q&A Chat** (active, public chat endpoint)
```
Chat Trigger → AI Agent (gpt-4o-mini + window memory)
                 ├─ tool: Pinecone Vector Store (retrieve-as-tool, top-20)
                 │        └─ Reranker Cohere (→ top-6)
                 └─ system prompt: multi-query, citations, refusal path,
                    no-rate-questions rule, dated-corpus caution
```

## Prompts / agent instructions

The full system prompt is in the workflow JSON (`02-gst-qa-chat.json`). Its five design rules:

1. **Grounding:** answer ONLY from retrieved passages; never fill gaps from general knowledge
2. **Multi-query:** search at least twice per question — once with the user's exact terms (section numbers, form names) and once with a plain-language paraphrase
3. **Citations:** name the source document and quote section/rule numbers
4. **Refusal:** if the passages don't contain the answer, say so and point to a CA/CBIC — never guess. Current tax rates are always refused (they change too often for a static corpus)
5. **Freshness caution:** on change-prone topics (return forms, late fees), append a verify-current-rules note

## Iterations & what I learned

1. **Corpus prep mattered more than any node setting.** Splitting the 223-page FAQ compilation into per-chapter PDFs (instead of ingesting it whole) meant every retrieved chunk arrives topic-pure. This was the single highest-leverage decision.
2. **Credential/config debugging is real work in no-code too.** Hit and fixed: n8n's secure-cookie block on Safari, a file-access restriction on the corpus folder (n8n blocks arbitrary disk reads by default — fixed with `N8N_RESTRICT_FILE_ACCESS_TO`), an OpenAI key with exhausted quota (429), and an index-name mismatch between the workflow and Pinecone.
3. **Reranking earned its place empirically.** Running the same 15 questions with the reranker off produced one materially wrong answer (ITC on stock after registration cancellation — the dense top-6 missed the Section 29(5) reversal rule; the reranker recovered it). Full comparison in the evaluation report.
4. **A corpus with mixed document vintages gives mixed answers.** The 2017 FAQ says ₹20 lakh registration threshold; the 2019-updated flyer says ₹40 lakh. The bot surfaced the newer figure — grounded, but inconsistent-by-luck. Production fix: effective-date metadata per document.
5. **Design the refusal first** (as the handout says). Both out-of-scope eval questions were cleanly refused; no answerable question was wrongly refused.

## Evaluation summary

15 questions (8 straightforward, 3 ambiguous, 2 multi-document, 2 unanswerable), run against the live bot, scored for faithfulness and citations — **15/15 pass, mean faithfulness 4.8/5, 2/2 correct refusals**, plus a rerank on/off comparison. Full report: `evaluation-report.md`.

## Known limitations / production roadmap

- **No true hybrid (BM25+dense):** n8n's Pinecone node is dense-only; hybrid would need direct sparse-dense API calls. Mitigated by multi-query + rerank; named as the first production upgrade.
- **Dated corpus:** 2017–2019 publications; mitigated by prompt-level cautions. Production: scheduled re-ingestion from CBIC with effective-date metadata.
- **Character-based chunking:** a legal-structure-aware splitter (split on section boundaries) would be more robust than the 3,000/400 recursive splitter.
- **In-memory chat sessions:** window-buffer memory resets with the instance; production would use persistent session storage.

## Tools & stack

n8n 2.35.7 (self-hosted, free) · Pinecone serverless (free tier) · OpenAI (`text-embedding-3-small`, `gpt-4o-mini`) · Cohere Rerank (trial) · Claude Code as build assistant (corpus prep, workflow JSON authoring, debugging, eval automation)

**Datasets:** all corpus documents are official public CBIC/GST Council publications (gstcouncil.gov.in e-flyers; cbic-gst.gov.in FAQ compilation).
