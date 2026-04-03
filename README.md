# india-equity-rag 🇮🇳📊

> Multi-document financial research assistant over Indian corporate Annual Reports  
> Reliance Industries · HDFC Bank · TCS

[![Python](https://img.shields.io/badge/Python-3.11+-blue.svg)](https://python.org)
[![AWS Bedrock](https://img.shields.io/badge/AWS-Bedrock-orange.svg)](https://aws.amazon.com/bedrock/)
[![OpenSearch](https://img.shields.io/badge/OpenSearch-Serverless-purple.svg)](https://aws.amazon.com/opensearch-service/)
[![RAGAS](https://img.shields.io/badge/Eval-RAGAS-red.svg)](https://docs.ragas.io)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)

---

## The Problem

Indian equity research analysts spend hours manually searching through 400+ page Annual Reports to find specific financial metrics, management commentary, and risk factors. A query like *"How did HDFC Bank's GNPA ratio trend after the HDFC merger?"* requires cross-referencing multiple sections across multiple filings — work that takes 30 minutes manually.

This system answers such queries in under 3 seconds, with citations pointing to the exact page and section in the source filing.

---

## Why This Is Technically Hard

These are not three similar documents. They are three structurally different financial document types:

| Company | Type | Key Complexity |
|---------|------|---------------|
| **Reliance Industries** | Conglomerate | Segment-level EBITDA (O2C · Jio · Retail · E&P · New Energy), inter-segment revenues, standalone vs consolidated distinction |
| **HDFC Bank** | Private Sector Bank | Completely different statements — NII, NIM, GNPA, NNPA, CASA ratio, PCR. Not standard P&L metrics. |
| **TCS** | IT Services | Constant currency revenue, deal TCV, geography/vertical breakdowns, headcount and attrition metrics |

A system that handles all three accurately — routing numerical queries to structured SQL and narrative queries to hybrid semantic search — is genuinely non-trivial.

---

## Architecture

```
BSE/NSE Annual Reports (PDFs) — Reliance · HDFC Bank · TCS · FY2021–FY2023
          ↓
     S3 (Raw Filing Store)
          ↓
  Lambda Ingestion Pipeline
    ├── PDF Text Extractor (pdfplumber)
    │     └── Section Detector: MD&A · Risk Factors · Segment Results · Notes
    ├── Table Extractor
    │     └── Financial Table Classifier → Canonical Metric Mapper
    │     └── Number Normaliser: ₹ Crores / Lakhs / Billions → canonical Crores
    │     └── SQLite Structured Store
    ├── Semantic Chunker (512 tokens, 100 overlap)
    └── Bedrock Titan Embeddings V2 (dim=1024)
          ↓
   OpenSearch Serverless (BM25 + kNN) + Local FAISS fallback
          ↓
     Query Router (FastAPI)
    ├── Numerical/Comparison → Text-to-SQL → SQLite
    └── Narrative/Conceptual → Hybrid Search (BM25 + Dense + RRF fusion)
                                      ↓
                             Cross-Encoder Reranker
          ↓
     Bedrock Claude 3 Sonnet ← Merged Context
          ↓
   Answer + Citations (company · filing type · FY · page · section)
          ↓
     Async FastAPI Gateway (SSE Streaming)
          ↓
     React Web Portal
          ↓
   RAGAS + DeepEval + Custom NumericalAccuracyMetric
   + Regression Gate (Lambda + SNS + SSM)
```

---

## Key Technical Decisions

**Why query routing instead of pure RAG?**
Numerical questions like *"What was HDFC Bank's GNPA ratio in FY2023?"* have a single correct answer: 1.12%. Retrieving text chunks and asking an LLM to extract this is slow and introduces hallucination risk. Direct SQL against a structured store is faster, deterministic, and verifiable.

**Why RRF instead of score averaging for hybrid search?**
BM25 scores and cosine similarity scores live on incompatible scales. Averaging them directly produces meaningless rankings. Reciprocal Rank Fusion fuses ranked lists position-by-position, making it scale-invariant and empirically superior across retrieval benchmarks.

**Why pdfplumber for Indian Annual Reports?**
Indian Annual Reports mix dense financial tables with narrative text on the same page. pdfplumber provides spatial positioning data that allows table boundary detection — essential for separating a segment EBITDA table from the surrounding MD&A text on the same page.

**Why SQLite for the structured store?**
Three companies, 9 filings, ~500 financial metrics. SQLite handles this trivially with zero operational overhead. The query patterns are simple joins and filters, not distributed aggregations. DynamoDB or RDS would add cost and complexity without benefit at this scale.

**Why ap-south-1 (Mumbai)?**
Indian financial data. Indian users. Data residency alignment. Bedrock model availability confirmed in ap-south-1.

---

## Sample Queries

| Query | Route | What It Demonstrates |
|-------|-------|---------------------|
| *"What was HDFC Bank's GNPA ratio in FY2023?"* | SQL | Exact numerical retrieval |
| *"Compare Jio and Reliance Retail EBITDA contribution to consolidated EBITDA FY2023"* | SQL | Segment analysis, cross-entity comparison |
| *"What risks did Reliance highlight for its Green Energy business?"* | RAG | Narrative retrieval with section citation |
| *"Why did TCS attrition spike in FY2022 and what did management say about it?"* | HYBRID | Number + narrative combination |
| *"TCS constant currency vs rupee growth FY2023 — explain the difference"* | HYBRID | Financial domain depth |

---

## Evaluation Results

| Metric | Score |
|--------|-------|
| context_precision | — |
| answer_relevancy | — |
| NumericalAccuracyMetric (SQL route) | — |
| HallucinationMetric (DeepEval) | — |
| p95 latency | — |
| Query routing accuracy | — |

*Metrics populated after evaluation pipeline completion (Day 7)*

---

## Stack

| Layer | Technology |
|-------|-----------|
| LLM | AWS Bedrock Claude 3 Sonnet |
| Embeddings | AWS Bedrock Titan Embeddings V2 (dim=1024) |
| Vector Store | OpenSearch Serverless + FAISS (local dev) |
| Structured Store | SQLite |
| PDF Extraction | pdfplumber |
| Chunking | LangChain RecursiveCharacterTextSplitter |
| Hybrid Search | BM25 + Dense + RRF (manual implementation) |
| Reranking | cross-encoder/ms-marco-MiniLM-L-6-v2 |
| Query Routing | Bedrock Claude 3 Haiku (structured output) |
| API | FastAPI async + SSE streaming |
| Evaluation | RAGAS + DeepEval + custom NumericalAccuracyMetric |
| Observability | CloudWatch + structured JSON logging |
| Infrastructure | AWS Lambda · S3 · ap-south-1 |
| Frontend | React + Tailwind CSS |

---

## Project Status

🔄 **Active development** — building daily, committing daily.

| Component | Status |
|-----------|--------|
| Domain research + document collection | ✅ Complete |
| Repo scaffold + AWS bootstrap | ✅ Complete |
| PDF extraction + section detection | 🔄 In Progress |
| Financial table extraction + SQLite | ⬜ Planned |
| Hybrid search (BM25 + Dense + RRF) | ⬜ Planned |
| Query router | ⬜ Planned |
| SQL retriever | ⬜ Planned |
| FastAPI gateway + SSE streaming | ⬜ Planned |
| RAGAS evaluation pipeline | ⬜ Planned |
| Regression gate | ⬜ Planned |
| React web portal | ⬜ Planned |

---

## Setup

```bash
git clone https://github.com/piyush-genai/india-equity-rag
cd india-equity-rag
pip install -r requirements.txt
cp .env.example .env
# Configure AWS credentials, Bedrock region (ap-south-1), OpenSearch endpoint
```

See `docs/setup.md` for full AWS infrastructure setup instructions.

---

## Domain Context

**Indian Financial Year:** April 1 – March 31. FY2023 = April 2022 – March 2023.

**Number formats handled:** ₹ Crores · ₹ Lakhs · ₹ Lakh Crores · USD Millions → all normalised to canonical ₹ Crores internally.

**Key metrics tracked:**
- Reliance: Segment EBITDA (O2C · Jio · Retail · E&P · New Energy), Jio ARPU, Retail store count
- HDFC Bank: NII, NIM, GNPA%, NNPA%, CASA ratio, CAR/CRAR, PCR
- TCS: Revenue (₹ + USD), constant currency growth, EBIT margin, headcount, attrition, deal TCV

---

*Built on AWS ap-south-1 · Indian financial domain · Production-grade evaluation pipeline*
