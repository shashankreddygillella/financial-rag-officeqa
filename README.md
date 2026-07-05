# Financial RAG Challenge — OfficeQA

## Project Overview

This project builds a Retrieval-Augmented Generation (RAG) system over the Databricks OfficeQA
U.S. Treasury Bulletin corpus. It compares a Baseline RAG system against an Engineered RAG system
to measure how chunking strategy and metadata filtering actually affect retrieval and answer
quality — rather than assuming an "engineered" system is automatically better.

## Dataset

- Data Source: Databricks OfficeQA (Hugging Face)
- Format Used: `.txt` (Markdown) Treasury Bulletin files
- Years Used: 2022, 2023, 2024, 2025
- Answer Key: `officeqa_full.csv`, filtered to the chosen years via the `source_files` column
- Evaluation Cutoff: K = 5

## Technical Stack

- Vector Database: FAISS (`IndexFlatIP`, cosine similarity via normalized embeddings)
- Embedding Model: `sentence-transformers/all-MiniLM-L6-v2`
- Metadata: Year and Month tags parsed directly from filenames (`treasury_bulletin_{YEAR}_{MONTH}.txt`)
- Baseline Chunking: 1,200-character chunks, 100-character overlap, no metadata tags
- Engineered Chunking: 1,800-character section-aware chunks (split on headers/tables first), 250-character overlap, tagged with Year/Month
- Answer Generation: rule-based numeric extraction — pulls the number from retrieved chunks whose surrounding text has the highest keyword overlap with the question (no external LLM API required)
- Evaluation Metrics: Hit Rate@5, MRR, Groundedness, Factual Accuracy, Hallucination Rate

## Repository Structure

```text
financial-rag-officeqa/
├── financial_rag_officeqa_colab.ipynb
├── results/
│   ├── baseline_results.csv
│   ├── engineered_results.csv
│   └── scorecard.csv
└── README.md
```

## Architecture

```text
OfficeQA .txt Bulletins (Hugging Face)
        ↓
Filter officeqa_full.csv by source_files -> chosen years
        ↓
Baseline chunks (naive, no metadata)   Engineered chunks (section-aware, Year/Month tagged)
        ↓                                       ↓
Embed with MiniLM -> FAISS index       Embed with MiniLM -> FAISS index (filtered by metadata)
        ↓                                       ↓
Retrieve top-5 chunks per question
        ↓
Rule-based number extraction -> predicted answer
        ↓
Compare against officeqa_full.csv answer key
        ↓
Compute Baseline vs. Engineered metrics
```

## Scorecard

| Metric | Baseline (Simple) | Engineered (Improved) |
|---|---:|---:|
| Hit Rate@5 | 62.5% | 12.5% |
| MRR | 0.27 | 0.06 |
| Groundedness | 100.0% | 100.0% |
| Factual Accuracy | 0.0% | 0.0% |
| Hallucination Rate | 0.0% | 0.0% |

## Engineering Reflection

### 1. The Bottleneck

The main bottleneck turned out to be the Generator, not just the Retriever. Baseline retrieval
was reasonably strong (62.5% Hit Rate@5), meaning the correct bulletin was usually surfaced in
the top 5 chunks. Yet Factual Accuracy stayed at 0% for both systems. That gap between a decent
Hit Rate and a 0% Factual Accuracy shows that finding the right document isn't enough — my
rule-based number-extraction generator struggles to identify *which specific number* in a dense
financial table actually answers the question, even when the right context is right in front of
it.

### 2. The Metadata Fix

Adding Year/Month metadata filtering made retrieval noticeably worse rather than better: Hit
Rate@5 dropped from 62.5% to 12.5%, and MRR fell from 0.27 to 0.06. The most likely cause is a
mismatch between how years/months are phrased in the question text versus how they're encoded in
the metadata tags — a strict filter can silently shrink the search pool to the wrong (or a much
smaller) set of chunks than the unfiltered baseline had access to. This was the most useful
lesson of the project: metadata filtering is only as good as the extraction logic behind it, and
an "engineered" system needs to be measured, not assumed, to be an improvement.

### 3. Scaling Insight

The current engineered pipeline rebuilds a FAISS index on the fly for every single question
(filter chunks by metadata → re-embed the filtered subset → build a new index → search). That's
workable across 4 years of data, but it would be the first component to break scaling up to the
full 1939–2025 archive — re-embedding and re-indexing per query doesn't scale to a much larger
corpus. A production version would need a single persistent index built once, with metadata
filtering applied natively at query time (e.g. through a vector database's built-in metadata
filters) instead of rebuilding indexes per request.

## Key Learning

The biggest takeaway from this project is that a Baseline is what makes an "Engineered" claim
meaningful. Without measuring both systems side by side, it would have been easy to assume that
adding metadata filtering and better chunking automatically improves results. Here, it did the
opposite — and that negative result is itself a useful engineering signal: it points directly at
where the metadata extraction logic needs to be fixed before it can help rather than hurt.

## Results Files

- `results/baseline_results.csv`
- `results/engineered_results.csv`
- `results/scorecard.csv`

## How to Run

Open `financial_rag_officeqa_colab.ipynb` in Google Colab (no local install required) and run all
cells top to bottom. The first code cell installs dependencies; a later cell will prompt for a
Hugging Face Read token (via Colab Secrets, not hardcoded) to download the dataset. The notebook
downloads the OfficeQA data, builds both RAG systems, evaluates them, and saves results into the
`results/` folder.
