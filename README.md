# News Extractor — Company Relationship Graph

A pipeline that crawls Naver News articles, extracts relationships between companies using NLP and LLMs, and visualizes them as an interactive knowledge graph. The end goal is a web service where users can explore how companies are connected through news coverage.

## Overview

```
Naver News  →  Crawler  →  NLP Extractor  →  Graph DB  →  API  →  Web UI
```

Given a set of news articles, the system identifies company entities and the relationships between them (e.g., *Samsung acquired Harman*, *Kakao partnered with Naver*, *Hyundai competes with Kia*), then stores them as a weighted, typed graph that can be queried and visualized.

## Relation Extraction Approach

Extracting **typed, directed relationships** from free text is the core challenge. This project uses a hybrid strategy:

### 1. Named Entity Recognition (NER)
Identify company names within article text using:
- **Fine-tuned Korean NER model** (e.g., klue/bert-base fine-tuned on KLUE-NER) for production
- **LLM zero-shot NER** (Claude API) for bootstrapping and edge cases

### 2. Relation Extraction (RE)

| Method | Signal | Use case |
|--------|--------|----------|
| **Co-occurrence** | Two companies in the same article/paragraph | Baseline; scalable; weak typing |
| **LLM structured extraction** | Claude extracts `(subject, relation, object)` triples | High accuracy; handles complex sentences |
| **Dependency-pattern rules** | spaCy dependency trees + regex patterns | Fast; interpretable; handles common patterns |

**Relationship taxonomy** (initial set, extensible):

| Type | Examples |
|------|---------|
| `ACQUIRED` | "A acquired B", "A took over B" |
| `INVESTED_IN` | "A invested in B", "A led B's Series C" |
| `PARTNERED_WITH` | "A and B signed MOU", "A collaborates with B" |
| `COMPETES_WITH` | "A and B are rivals", "A challenges B's market" |
| `SUPPLIES_TO` | "A supplies components to B" |
| `SPIN_OFF` | "B was spun off from A" |
| `CO_MENTIONED` | Generic co-occurrence (weak signal) |

### 3. LLM Extraction Pipeline (primary method)

```
Article text
    │
    ▼
Chunking (by paragraph / sliding window)
    │
    ▼
Claude API prompt:
  "Extract all (company_A, relationship_type, company_B, evidence_sentence)
   from the following text. Return JSON."
    │
    ▼
Structured triples + confidence scores
    │
    ▼
Deduplication & merging (same pair, different articles)
    │
    ▼
Graph DB upsert
```

### 4. Graph Schema

**Nodes** — `Company`
```
id: string (normalized company name)
aliases: string[]
sector: string
```

**Edges** — `Relationship`
```
type: RelationshipType
weight: float         # number of supporting articles
first_seen: date
last_seen: date
articles: string[]    # source article URLs
evidence: string[]    # supporting sentences
```

## Architecture

```
news-extractor/
├── src/
│   ├── collector/       # Naver News crawling & RSS ingestion
│   ├── extractor/       # NER + relation extraction (NLP + LLM)
│   ├── graph/           # Graph DB client (Neo4j)
│   └── api/             # FastAPI web service
├── docs/
│   ├── architecture.md
│   └── relation-extraction.md
├── scripts/             # One-off scripts and experiments
├── tests/
└── pyproject.toml
```

## Tech Stack

| Layer | Technology |
|-------|-----------|
| Language | Python 3.12 |
| News collection | Naver News API / `httpx` + `BeautifulSoup4` |
| NLP | `spaCy` (ko_core_news_lg), KLUE-BERT |
| LLM | Claude API (`anthropic`) |
| Graph DB | Neo4j |
| API | FastAPI |
| Frontend | Next.js + D3.js / vis.js |
| Queue | Celery + Redis (async crawling) |

## Quickstart

```bash
# Clone and install
git clone https://github.com/<your-username>/news-extractor.git
cd news-extractor
pip install -e ".[dev]"

# Set environment variables
cp .env.example .env
# Fill in: ANTHROPIC_API_KEY, NEO4J_URI, NEO4J_PASSWORD, NAVER_CLIENT_ID, NAVER_CLIENT_SECRET

# Run the extraction pipeline on a keyword
python -m src.extractor.run --keyword "삼성전자" --limit 50
```

## Roadmap

- [ ] Naver News crawler (RSS + Search API)
- [ ] Company NER (LLM zero-shot baseline)
- [ ] Relation extraction with Claude API
- [ ] Neo4j graph storage
- [ ] FastAPI query endpoints
- [ ] Frontend graph visualization
- [ ] Korean NER fine-tuning
- [ ] Scheduled incremental crawling
- [ ] Web service deployment

## License

MIT
