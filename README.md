# News Extractor — Company Relationship Graph

A pipeline that crawls Naver News, extracts relationships between companies, and maps them to listed stock tickers to build a queryable knowledge graph. The end goal is a web service where users can explore how publicly listed companies are connected through news coverage.

## Core Challenge: Entity Linking

NER and relation extraction are solved problems. The real bottleneck is **entity linking** — resolving an extracted company name (e.g. "삼성", "현대", "SK") to a unique, unambiguous stock ticker code (e.g. `005930`).

```
Article text
    │
    ▼  NER
"삼성전자", "현대", "SK하이닉스"   ← surface forms (ambiguous)
    │
    ▼  Entity Linking  ◀── bottleneck
005930,       ?,       000660         ← stock ticker codes (ground truth)
```

"현대" alone could refer to 현대자동차(005380), 현대건설(000720), 현대모비스(012330), and more. Disambiguation requires both a candidate lookup table and contextual reasoning.

## Pipeline

```
Naver News  →  [1] Crawler  →  [2] NER + RE  →  [3] Entity Linking  →  Graph DB  →  API  →  Web UI
```

### Stage 1 — Crawling (`src/collector/`)
- Naver News Search API: query by keyword, fetch article metadata
- Article fetcher: follow URLs, extract full body text

### Stage 2 — NER + Relation Extraction (`src/extractor/`)
- **NER**: extract company name surface forms from article text
  - Model: LLM (Claude API) for accuracy; optionally fine-tuned Korean NER (klue/bert-base) for scale
- **RE**: extract typed relationships between company pairs
  - Model: Claude API structured extraction → `(company_a, relation_type, company_b, evidence)`
  - Relationship types: `ACQUIRED`, `INVESTED_IN`, `PARTNERED_WITH`, `COMPETES_WITH`, `SUPPLIES_TO`, `SPIN_OFF`

### Stage 3 — Entity Linking (`src/linker/`) ← core bottleneck
- **Candidate generation**: look up surface form against a master ticker table (KRX listed companies)
- **Disambiguation**: when multiple candidates exist, use article context to pick the correct ticker
  - Rule-based: industry keywords, co-mentioned companies
  - LLM-based: ask Claude to pick from candidates given the sentence context
- **Master ticker table**: sourced from KRX (한국거래소) or DART; updated periodically
- **Alias dictionary**: maps surface forms and abbreviations to ticker codes (e.g. "삼성" → 005930 or disambiguation required)

## Architecture

```
news-extractor/
├── src/
│   ├── collector/       # Naver News crawling & article fetching
│   ├── extractor/       # NER + relation extraction
│   ├── linker/          # Entity linking: surface form → ticker code
│   ├── graph/           # Neo4j client and Cypher queries
│   └── api/             # FastAPI web service
├── docs/
│   ├── architecture.md
│   ├── relation-extraction.md
│   └── entity-linking.md
├── scripts/
├── tests/
└── pyproject.toml
```

## Tech Stack

| Layer | Technology |
|-------|-----------|
| Language | Python 3.12 |
| News collection | Naver News API + `httpx` + `BeautifulSoup4` |
| NER / RE | Claude API (`anthropic`) |
| Entity linking | KRX ticker table + Claude API disambiguation |
| Graph DB | Neo4j |
| API | FastAPI |
| Frontend | Next.js + D3.js / vis.js |

## Quickstart

```bash
git clone https://github.com/<your-username>/news-extractor.git
cd news-extractor
pip install -e ".[dev]"

cp .env.example .env
# Fill in: ANTHROPIC_API_KEY, NAVER_CLIENT_ID, NAVER_CLIENT_SECRET, NEO4J_URI, NEO4J_PASSWORD

python -m src.extractor.run --keyword "삼성전자" --limit 50
```

## Roadmap

- [ ] Naver News crawler
- [ ] NER + RE with Claude API
- [ ] KRX ticker master table ingestion
- [ ] Entity linking: alias lookup + LLM disambiguation
- [ ] Neo4j graph storage
- [ ] FastAPI query endpoints
- [ ] Frontend graph visualization
- [ ] Scheduled incremental crawling
- [ ] Web service deployment

## License

MIT
