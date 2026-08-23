# Evaluation Report — GST Q&A RAG Bot (Week 2 Project)

**System under test:** n8n RAG pipeline over official CBIC GST publications
**Date:** 23 August 2026
**Corpus:** 14 official CBIC PDFs (~110 pages): 9 GST Flyers (registration, ITC, returns, annual return, refunds, refund of unutilised tax, e-way bill, composition levy, reverse charge) + 5 chapters extracted from the CBIC "FAQ on GST" compilation, 2nd edition (registration, payment of tax, input tax credit, returns, refunds)
**Retrieval stack:** Pinecone (dense, `text-embedding-3-small`, 1536-dim, cosine) top-20 → Cohere Rerank top-6 → AI Agent (gpt-4o-mini) with multi-query prompting
**Chunking:** Recursive character splitter, 3,000 chars (~750 tokens), 400-char overlap → 100 chunks

---

## Methodology

15 questions across four categories, each run against the live chat endpoint:

| Category | Count | Purpose |
|---|---|---|
| Straightforward | 8 | Core retrieval quality on single-topic questions |
| Ambiguous | 3 | Real-world phrasing that doesn't name the legal concept ("Can I claim ITC on my car?") |
| Multi-document | 2 | Answers requiring synthesis across two corpus documents |
| Unanswerable | 2 | Out-of-corpus questions that MUST be refused, not guessed |

Each answer was scored on:
- **Faithfulness (1–5):** Is every claim grounded in the corpus? (5 = fully grounded, 1 = hallucinated)
- **Citations:** Does the answer name its source document and section/rule numbers?
- **Refusal correctness** (for unanswerable questions): Did it decline rather than guess?

The full 15-question run was executed **twice**: once with Cohere reranking enabled (top-20 → rerank → top-6) and once disabled (dense-only top-6), to measure reranking impact.

---

## Results — Reranker ON (production configuration)

| # | Question (abridged) | Category | Faithfulness | Citations | Verdict |
|---|---|---|---|---|---|
| 1 | Who must register for GST? | Straightforward | 5 | ✔ Flyer + FAQ | Pass |
| 2 | What is ITC and who can claim it? | Straightforward | 5 | ✔ FAQ-ITC, Sec 39 | Pass |
| 3 | Time limit for taking ITC? | Straightforward | 5 | ✔ FAQ-ITC, 180-day rule | Pass |
| 4 | Composition levy scheme + eligibility? | Straightforward | 5 | ✔ Flyer, CMP-02/GSTR-4 | Pass |
| 5 | What is an e-way bill, when required? | Straightforward | 5 | ✔ Flyer, ₹50k threshold | Pass |
| 6 | What is reverse charge? | Straightforward | 5 | ✔ Sec 9(3)/9(4), 5(3)/5(4) | Pass |
| 7 | Refund of unutilised ITC? | Straightforward | 5 | ✔ Sec 54(3), RFD-01A, Rule 91 | Pass |
| 8 | Which returns must a normal taxpayer file? | Straightforward | 4 | ✔ GSTR-1/3B/9 | Pass (see note A) |
| 9 | Can I claim ITC on my car? | Ambiguous | 5 | ✔ Sec 17(5), direct quote | Pass |
| 10 | Do I need to register if I sell online? | Ambiguous | 4 | ✔ Sec 24 | Pass (see note B) |
| 11 | What happens if I file late? | Ambiguous | 4 | ✔ late-fee amounts | Pass |
| 12 | Composition trader: ITC + annual return? | Multi-document | 5 | ✔ two documents | Pass |
| 13 | Cancel registration: ITC on stock + refund? | Multi-document | 5 | ✔ Sec 29(5) + Sec 54 | Pass |
| 14 | GST rate on cement? | Unanswerable | 5 | n/a | **Correct refusal** |
| 15 | Income tax slabs? | Unanswerable | 5 | n/a | **Correct refusal** |

**Summary: 15/15 pass · mean faithfulness 4.8/5 · both unanswerable questions correctly refused · every substantive answer carried document + section citations.**

### Notes on imperfect scores
- **Note A (Q8):** The answer correctly stated GSTR-1/3B/9 and even flagged that GSTR-2/GSTR-3 are suspended — but return filing rules have changed repeatedly since the corpus was published. The bot correctly appended its dated-corpus caution, which is the designed mitigation.
- **Note B (Q10):** Mostly grounded in Section 24, but blended the goods/services thresholds slightly across corpus editions (the corpus itself mixes 2017 FAQ and 2019 flyer content — see Freshness finding below).
- **Q11:** Late-fee figures matched the flyers, but this is the most change-prone topic in the corpus; the freshness caution fired correctly.

### Threshold-consistency finding (corpus-internal, not a hallucination)
Q1 answered "₹40 lakh" for goods registration. The 2017 FAQ chapter says ₹20 lakh; the *updated* Registration flyer in the corpus reflects the 2019 threshold change to ₹40 lakh. The bot surfaced the newer document's figure — technically grounded, but it demonstrates a real production issue: **a corpus mixing document vintages can give inconsistent answers depending on which chunk wins retrieval.** Production fix: stamp each document with an effective-date metadata field and instruct the agent to prefer, and cite, the most recent.

---

## Reranking impact (ON vs OFF, same 15 questions)

| Metric | Rerank OFF (dense top-6) | Rerank ON (top-20 → Cohere → top-6) |
|---|---|---|
| Refusals on unanswerable | 2/2 | 2/2 |
| Materially wrong/incomplete answers | **1 (Q13)** | **0** |
| Section-number citation density | Lower | Higher |

**Key finding — Q13 (cancellation of registration):** With reranking OFF, the answer claimed ITC on stock "can typically still be claimed" and framed the question purely as a refund scenario — missing the central rule. With reranking ON, the answer correctly led with **Section 29(5): ITC on stock must be reversed (pay credit or output tax, whichever is higher)**. This is exactly the failure mode reranking exists to fix: the dense top-6 retrieved refund-adjacent chunks, and the reranker pulled the cancellation-specific chunk back into the context window.

Secondary observation: rerank-ON answers cited specific sections more consistently (e.g., Q3's "October 20 upper limit" precision vs the off-run's vaguer "September" framing).

**Conclusion:** On a 15-question set, reranking turned one materially-wrong answer into a correct one (~7% of questions) and improved citation precision broadly — at a cost of one extra API call per retrieval.

---

## Where retrieval succeeds, where it fails

**Succeeds:**
- Topic-level questions land on the right document nearly every time — the corpus's one-topic-per-file structure (flyers + split FAQ chapters) makes dense retrieval effective.
- Multi-query prompting (exact terms + paraphrase) reliably bridged colloquial phrasing to legal concepts ("my car" → Section 17(5) blocked credits).
- The refusal path held under both test questions and never fired on answerable ones.

**Fails / limitations:**
1. **No true hybrid (BM25 + dense).** n8n's Pinecone node is dense-only. Multi-query + reranking compensated well in this eval (no exact-term question failed), but a larger corpus with many similar section numbers would expose the gap. Production upgrade: sparse-dense hybrid via Pinecone's API.
2. **Document-vintage inconsistency** (₹20 vs ₹40 lakh) — see finding above.
3. **Dated corpus (2017–2019).** Return forms, late fees, and e-invoicing have all changed since. Mitigated by the system-prompt caution, which fired appropriately on Q2, Q4, Q8, Q11, Q12.
4. **Chunking is character-based, not section-aware.** The 400-char overlap prevented visible mid-rule truncation in this eval, but a legal-structure-aware splitter would be more robust.

---

## Reproduction

- Questions: `eval_questions.json`
- Raw answers: `eval_answers_rerank_on.json`, `eval_answers_rerank_off.json`
- Both runs executed against the live n8n chat webhook, sessions isolated per question.
