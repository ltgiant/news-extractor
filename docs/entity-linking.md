# Entity Linking Design

## Problem

NER extracts surface forms from text. Entity linking resolves them to canonical stock ticker codes.

```
"삼성"        → 005930 (삼성전자) ?  or ambiguous
"현대"        → 005380 / 000720 / 012330 / ... (disambiguation required)
"SK하이닉스"  → 000660 (unambiguous)
```

## Step 1 — Master Ticker Table

Build a lookup table of all KRX-listed companies:

| ticker | official_name | market | sector |
|--------|--------------|--------|--------|
| 005930 | 삼성전자 | KOSPI | 반도체 |
| 000660 | SK하이닉스 | KOSPI | 반도체 |
| 005380 | 현대자동차 | KOSPI | 자동차 |
| ... | ... | ... | ... |

**Data sources:**
- `pykrx` library: programmatic access to KRX data
- KRX 상장법인목록 CSV (https://kind.krx.co.kr)
- DART OpenAPI: additional company metadata

This table is updated periodically (weekly or on IPO events).

## Step 2 — Alias Dictionary

Map known surface forms and abbreviations to ticker candidates:

```python
{
  "삼성전자": ["005930"],
  "삼성":     ["005930", "028260", ...],  # ambiguous → needs disambiguation
  "현대차":   ["005380"],
  "현대":     ["005380", "000720", "012330", ...],  # ambiguous
  "SK하닉":   ["000660"],
}
```

Bootstrap sources:
- Wikipedia / Wikidata redirect titles
- DART company name variants
- Manual curation for top ~500 companies

## Step 3 — Disambiguation

When a surface form maps to multiple ticker candidates, use context to select the correct one.

### Rule-based signals
- Co-mentioned companies in the same article (industry clustering)
- Sector keywords in the sentence ("반도체" → 삼성전자 over 삼성물산)
- Article category (경제/산업 section tags from Naver)

### LLM disambiguation
When rules are insufficient, call Claude:

```
Given this Korean news sentence:
"현대는 올해 전기차 라인업을 대폭 확대할 계획이다."

Which company does "현대" refer to?
Candidates:
- 현대자동차 (005380): 자동차 제조
- 현대건설 (000720): 건설
- 현대모비스 (012330): 자동차 부품

Return the ticker code and a one-sentence justification.
```

### Fallback
- If confidence is below threshold: store as unresolved, flag for manual review
- Never silently assign the wrong ticker

## Step 4 — Confidence & Audit

Every entity link stores:
```python
{
  "surface_form": "현대",
  "ticker": "005380",
  "method": "llm_disambiguation",  # or "alias_exact" | "rule_based"
  "confidence": 0.92,
  "evidence": "전기차 라인업을 대폭 확대",
}
```

Low-confidence links can be reviewed and corrected to improve the alias dictionary over time.
