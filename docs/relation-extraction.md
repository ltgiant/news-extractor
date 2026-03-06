# Relation Extraction Design

## Problem Statement

Given a Korean news article body, extract all company-to-company relationships as structured triples:

```
(Samsung Electronics, ACQUIRED, Harman International, "삼성전자가 하만을 인수했다")
```

## Approach Comparison

### Option A — Co-occurrence (baseline)

- Count how many articles mention company A and company B together
- No relationship typing; only "relatedness" as edge weight
- O(n²) pairs; use as a fallback or for discovery

### Option B — Dependency-pattern rules

- Parse sentences with spaCy `ko_core_news_lg`
- Match patterns like: `SUBJ --[nsubj]--> VERB <--[obj]-- OBJ`
- Pros: fast, deterministic, no API cost
- Cons: brittle for Korean morphology; high maintenance

### Option C — LLM extraction (primary)

Prompt Claude with a structured instruction:

```
You are an information extraction system. Given the Korean news article below,
extract ALL relationships between companies. For each relationship return:
{
  "company_a": "<canonical Korean company name>",
  "relation": "<one of: ACQUIRED|INVESTED_IN|PARTNERED_WITH|COMPETES_WITH|SUPPLIES_TO|SPIN_OFF|CO_MENTIONED>",
  "company_b": "<canonical Korean company name>",
  "evidence": "<verbatim sentence from the article supporting this relationship>",
  "confidence": <0.0–1.0>
}
Return a JSON array. If no relationships are found, return [].
```

- Handles complex and implicit relationships
- Language-agnostic; works well for Korean
- Cost: ~$0.001–0.005 per article at current Claude pricing
- Add caching to avoid re-processing identical article bodies

### Option D — Fine-tuned RE model (future)

- Use extracted LLM triples as silver-label training data
- Fine-tune KLUE-BERT or similar on the silver corpus
- Reduces API cost at scale; requires ~1k+ labeled examples

## Relationship Taxonomy

| Type | Korean trigger words |
|------|---------------------|
| `ACQUIRED` | 인수, 합병, 인수합병, 買收 |
| `INVESTED_IN` | 투자, 출자, 지분 취득, 펀딩 |
| `PARTNERED_WITH` | 협약, MOU, 제휴, 파트너십, 협력 |
| `COMPETES_WITH` | 경쟁, 맞대결, 시장 쟁탈 |
| `SUPPLIES_TO` | 납품, 공급, 부품 제공 |
| `SPIN_OFF` | 분사, 물적분할, 자회사 설립 |
| `CO_MENTIONED` | (any co-occurrence without explicit type) |

## Entity Normalization

Companies appear under many surface forms in Korean news:
- `삼성전자`, `삼성`, `Samsung Electronics`, `SEC`
- Build a normalization map (alias → canonical ID)
- Bootstrap with Wikipedia / Wikidata redirect links
- LLM can also be prompted to return canonical names
