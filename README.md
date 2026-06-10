<div align="center">

# Elina

### *Fundamentos jurídicos verificáveis*
**AI-powered Portuguese legal research with source-anchored citations**

Elina answers questions about Portuguese law in plain language and grounds every
claim in the actual text of the statute or court decision it relies on — so the
reasoning can always be traced back to the source.

<img src="docs/images/welcome.png" alt="Elina welcome screen" width="620">

</div>

> [!NOTE]
> **This is a public showcase.** The application source lives in a private
> repository; this repo exists to explain what Elina is, what it can do, and how
> it is built. All figures below are drawn from the project's own technical
> documentation.

---

## Contents

- [Why Elina](#why-elina)
- [Capabilities](#capabilities)
- [The legal corpus](#the-legal-corpus)
- [How an answer is built](#how-an-answer-is-built)
- [Architecture](#architecture)
- [Technical build](#technical-build)
- [Privacy & compliance](#privacy--compliance)

---

## Why Elina

Portuguese legal research means working across dozens of codes (Código Civil,
Código do Trabalho, CIRS…), a constantly-updated body of legislation, and
hundreds of thousands of court decisions. General-purpose chatbots answer such
questions fluently but **without provenance** — you cannot tell which article or
ruling an assertion came from, or whether it exists at all.

Elina is built around the opposite principle: **the answer is only as good as
the source you can open and read.** Every cited norm is a clickable chip that
opens the exact diploma and scrolls to the precise article; every cited ruling
opens the full court decision. The grounding is enforced by the backend, not
left to the language model.

---

## Capabilities

### 🔎 Grounded legal Q&A

Ask a question in natural language (e.g. *"pesquisa lei freelance"*) and receive
a structured legal analysis. Each referenced norm appears inline as a citation
chip (`Art. 11.º CT`, `Art. 3.º CIRS`, `Ac. STA …`). The footer states the
grounding explicitly: *"Resposta gerada por IA, baseada nos textos legais."*

<div align="center">
<img src="docs/images/chat-citations.png" alt="Legal answer with inline verifiable citations" width="900">
</div>

### 📖 Click a citation → jump straight to the article

Clicking a citation opens the **legal document viewer** side-by-side and scrolls
to — and highlights — the exact article inside the diploma. The viewer features:

- **Infinite scroll** over full códigos/leis/decretos with neighbour-page prefetching
- **Article-anchored navigation** that resolves a citation to its precise article, even when several articles share a number across a publishing decree and its annex
- **Table-of-contents / índice navigation** (book → title → chapter → section → article)
- **In-document search** with fuzzy matching and keyboard navigation
- **Text sanitization** that strips the unrenderable glyphs ("tofu" squares, zero-width characters, replacement chars) left behind by PDF/HTML scraping

<div align="center">
<img src="docs/images/document-viewer.png" alt="Legal document viewer opened from a citation" width="900">
</div>

### ⚖️ Jurisprudence (acórdão) viewer

Citations to case law open a dedicated court-decision viewer that parses raw
DGSI-scraped text into structured sections — sticky header (tribunal, processo,
data), metadata grid, sumário, and full text — with in-decision search and
highlighting.

### 💬 Conversation experience

- **Conversation history** persisted per user, with a sidebar to revisit threads
- **Live updates** via Supabase realtime subscriptions on the message channel
- **Voice input** — dictate a query (European Portuguese, `pt-PT`) via Google Cloud Speech with a Web Speech API fallback and a live audio visualizer
- **Guest mode** — try it with no login (guest sessions are not persisted)

### 🛠️ Admin dashboard

A role-gated dashboard for managing the platform and its corpus:

<div align="center">
<img src="docs/images/dashboard.png" alt="Admin dashboard for documents and jurisprudence" width="900">
</div>

| Tab | What it does |
|---|---|
| **Documents** | Browse/search the legislation corpus, toggle publication, edit metadata, upload & delete books |
| **Articles** | Paginated, filterable table of parsed articles with rich enrichment fields (semantic decomposition, temporal/penal flags, cross-references) |
| **Acórdãos** | Paginated jurisprudence browser with tribunal/date/embedding filters and debounced search |
| **Feedback** | Triage user-submitted bug reports & suggestions with status and admin notes |
| **Billing** | Daily multi-cloud cost dashboard aggregating Vercel, Hetzner and GCP (BigQuery) spend |
| **Sandbox** | Interactive pipeline builder to compose and test multi-step LLM chains against sample questions, with step-by-step traces, timing and cost estimates |

### 🔐 GDPR self-service

Users manage their own data directly from settings: **export** (Art. 15 & 20 —
full JSON bundle), **erasure** (Art. 17 — account deletion), and **restrict
processing** (Art. 18 — blocks new chat with a `423 Locked` while preserving
existing data).

---

## The legal corpus

Elina's answers are grounded in a curated, embedded corpus of Portuguese law,
scraped and parsed from official sources (Diário da República / DRE, dgsi.pt,
juris.stj.pt, tribunalconstitucional.pt) and refreshed daily by scheduled jobs.

| Corpus | Scale |
|---|---:|
| Codes, laws & decrees | **1,249** |
| Individual legislation articles/norms (embedded) | **309,879** |
| Court decisions — *acórdãos* (embedded) | **292,258** |
| Source courts (STJ, TRL, TRP, TRC, TRE, TRG, TC, STA, TCA, TCN) | **10** |
| Article-citation links (rulings ↔ articles) | **6.17 M** |
| DGSI legal descriptors (embedded) | **61,173** |

Every article and ruling carries vector embeddings, so the corpus is searchable
**semantically** (meaning) as well as by keyword.

---

## How an answer is built

```
  Question ─▶ Retrieve ─▶ Ground ─▶ Synthesize ─▶ Cite
              hybrid       resolve     multi-agent    deterministic
              search       to source   reasoning      chip insertion
```

1. **Retrieve** — a single hybrid-search call combines semantic vector search
   (pgvector / HNSW) with keyword search (PostgreSQL trigram + full-text),
   fused via **Reciprocal Rank Fusion**, filtered by legal area, court and year.
2. **Ground** — candidate norms and rulings are resolved to their canonical
   source documents and validated against the database; unverifiable references
   are dropped before generation.
3. **Synthesize** — the language model composes the answer using only the
   validated material.
4. **Cite** — citation chips are inserted **deterministically by the backend**
   (not chosen by the model), each mapped to the exact article in the viewer.

Elina runs **two complementary pipelines**, both live:

- a **fast direct-answer pipeline** (the current default) — grounded search →
  source resolution → validated citations, with a sub-second precise-lookup
  short-circuit for direct "article *N* of code *X*" queries; and
- a **deeper multi-agent pipeline** (the *Elina / NYROCORE chain*) — 30+
  single-responsibility agents covering classification, retrieval, curation,
  synthesis, normative analysis, reconciliation and drafting, for complex
  questions that need multi-step grounding.

---

## Architecture

A Next.js frontend (Vercel) talks to a containerized FastAPI backend (Hetzner).
Heavy work — retrieval, generation, ingestion, scraping — runs asynchronously on
Celery workers behind a Redis queue, with an autoscaler and a Flower monitoring
dashboard. Persistence and vector search live in Supabase (PostgreSQL +
pgvector); embeddings and grounded generation use the Google Gemini API.

<div align="center">
<img src="docs/images/architecture.png" alt="Elina system architecture" width="640">
</div>

```
                          ┌──────────────────┐
   Next.js frontend ─────▶│  Nginx (proxy)   │─────▶ FastAPI ─────▶ Supabase
       (Vercel)           └──────────────────┘         │         (Postgres + pgvector)
                                                        │
                                                        ├─▶ Redis ──▶ Celery workers
                                                        │             ├─ chat
                                                        │             ├─ ingest
                                                        │             ├─ scrape
                                                        │             └─ beat (daily scrapes)
                                                        │
                                                        └─▶ Google Gemini API
                                                            (embeddings · generation ·
                                                             grounded search)
```

### The running stack

The backend ships as a Docker Compose stack — **Nginx**, the **FastAPI** API,
**Redis**, the **Celery** chat / ingest / scrape workers + beat scheduler, and
the **Flower** dashboard:

<div align="center">
<img src="docs/images/docker-stack.png" alt="Docker Compose stack running" width="900">
</div>

---

## Technical build

### Frontend

| Area | Technology |
|---|---|
| Framework | **Next.js 15** (App Router) · **React 19** · **TypeScript 5** |
| Styling / UI | **Tailwind CSS 4** · shadcn/ui · Radix UI · Lucide / Tabler icons |
| Data & auth | Supabase JS client · Supabase Auth (Google OAuth, PKCE) · realtime subscriptions |
| Content | `react-markdown` + `remark-gfm` · `recharts` (billing charts) · `pdfjs-dist` / `pdf-lib` |
| AI / voice | Google Generative AI client · Google Cloud Speech (voice input) |
| Hosting | **Vercel** (serverless) · Vercel Analytics |

The frontend's API routes proxy to the backend over an `X-API-Key`–authenticated
channel; chat responses stream back to the UI.

### Backend

| Area | Technology |
|---|---|
| API | **FastAPI** · **Python 3.13** · Uvicorn |
| Async | **Celery 5** (Redis broker) · Flower monitoring · custom queue-depth autoscaler |
| Data | **Supabase** — PostgreSQL 17 + **pgvector** (HNSW), `pg_trgm`, full-text search |
| Retrieval | Unified hybrid-search RPC: vector + keyword + **RRF fusion**, with area/court/year filters |
| AI | **Google Gemini** — 768-dim embeddings, 2.0 Flash generation, grounded search |
| Ingestion | Playwright (DRE PDFs) · PyMuPDF · BeautifulSoup/lxml · article-level chunking · LLM classification & enrichment |
| Infra | **Docker Compose** · **Nginx** · deployed on **Hetzner** |

---

## Privacy & compliance

Built for a real legal practice, Elina handles personal data under the GDPR/RGPD
with features that are implemented, not aspirational:

- **Data subject rights** — self-service export (Art. 15/20), erasure (Art. 17,
  with cascading deletion), and processing restriction (Art. 18).
- **Retention** — scheduled purge jobs remove aged chat data and orphaned records.
- **Data minimization** — only the message and conversation id are persisted;
  transient pipeline state lives in Redis with a TTL and never reaches the database.
- **Row-level security** across all user-scoped tables, plus request audit logging.
- **Authentication** — Supabase JWT (ES256 via JWKS) for users and `X-API-Key`
  for service-to-service calls, with an enforcement mode that hardens in production.

---

<div align="center">
<sub>© Elina — proprietary. This repository is a showcase and does not contain the application source code.</sub>
</div>
