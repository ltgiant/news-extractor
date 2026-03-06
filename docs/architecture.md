# Architecture

> Detailed design notes. See README.md for the overview.

## Pipeline Stages

### 1. Collection (`src/collector/`)

- **NaverNewsClient**: wraps Naver Search API (`/v1/search/news.json`) and handles pagination
- **ArticleFetcher**: follows news URLs, extracts full article body via `httpx` + `BeautifulSoup4`
- Articles are stored as `Article` Pydantic models before processing

### 2. Extraction (`src/extractor/`)

- **NERExtractor**: identifies company entities in article text
- **RelationExtractor**: produces `(company_a, relation_type, company_b, evidence)` triples
- **LLMExtractor**: calls Claude API with structured prompts; parses JSON responses
- **EntityNormalizer**: maps aliases / spelling variants to canonical company IDs

### 3. Graph Storage (`src/graph/`)

- **Neo4jClient**: thin wrapper around the official `neo4j` driver
- **GraphRepository**: upsert nodes and edges; merge duplicate relationships by weight
- Cypher queries are kept in `src/graph/queries.py`

### 4. API (`src/api/`)

- FastAPI app exposing:
  - `GET /companies/{id}/relationships` — neighbors of a company node
  - `GET /graph?companies=A,B,C&depth=2` — subgraph around given companies
  - `POST /pipeline/run` — trigger incremental crawl + extraction (admin)

## Data Flow

```
NaverNewsClient.search(keyword)
    → Article[]
    → ArticleFetcher.fetch_body(article)
    → NERExtractor.extract_companies(article.body)
    → RelationExtractor.extract(article, companies)
    → Triple[]
    → EntityNormalizer.normalize(triples)
    → GraphRepository.upsert(triples)
```
