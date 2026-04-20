# PHASE 2: TARGETED INVESTIGATION LOOP & EXTERNAL SCRAPERS

## 2.1 ARCHITECTURAL CONSTRAINT: ISOLATION
All raw scraping (HTML/PDF/Legal API polling) is restricted to an isolated external acquisition module. The AI Pipeline communicates strictly via asynchronous REST API contracts. This protects the orchestration core from DOM alterations and IP bans.

## 2.2 EXTERNAL SCRAPER HTTP API CONTRACTS (OpenAPI 3.1)
```yaml
openapi: 3.1.0
info:
  title: OSINT Scraper Protocol
paths:
  /api/v1/gather/osint:
    post:
      summary: Dispatch asynchronous wide-net NLP search.
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              properties:
                entities: { type: array, items: { type: string } }
                search_vectors: { type: array, items: { type: string } }
                keywords: { type: array, items: { type: string } }
                depth: { type: integer, default: 0 }
                regional_target: { type: string, default: "PL" }
      responses:
        '202':
          description: Accepted & Queued
          content: { application/json: { schema: { properties: { taskId: { type: string } } } } }

  /api/v1/gather/legal:
    post:
      summary: Deep registry lookup (e.g. KRS / Sanctions lists).
      requestBody:
        required: true
        content:
          application/json:
            schema:
              properties:
                primary_id: { type: string }
                lookup_type: { type: string, enum: ["SANCTIONS", "COURT", "REGISTRY"] }
      responses:
        '202': { description: Accepted & Queued }
```

## 2.3 `InvestigationConfig` SCHEMA
```json
{
  "InvestigationConfig": {
    "max_search_depth": 3,
    "potential_score_threshold": 0.65,
    "regional_weighting": {
      "PL": 1.0,
      "EU": 0.8,
      "OFFSHORE": 1.5,
      "RU": 2.0
    },
    "keyword_matrices": {
      "corruption": ["łapówka", "korupcja", "prywata", "nepotyzm"],
      "fraud": ["oszustwo", "wyłudzenie", "przekręt", "defraudacja"],
      "governance": ["zarzuty", "zarząd", "akt oskarżenia", "aresztowanie"],
      "sanctions": ["sankcje", "pranie pieniędzy", "zamrożenie aktywów"]
    },
    "inflection_handling": {
      "enabled": true,
      "engine": "nlp_lemmatizer",
      "synonym_expansion": true
    }
  }
}
```

## 2.4 HEURISTIC: `Potential_Score` EVALUATOR
When the webhook callback hits `POST /webhook/callback` returning raw HTML extracts (Clues), they are instantaneously assessed by an upstream, high-throughput logic router to calculate their value. High-value clues trigger exponential queue depth.

**System Prompt Matrix:**
> SYSTEM: Assess text `raw_extract` for AML/Due Diligence potential. Output strict JSON: `{"potential_score": X.X}`
>
> SCORING RULES:
> - Base score = 0.0
> - IF exact entity phrase found -> ADD 0.3
> - IF fuzzy NLP entity found (accounting for PL inflection cases: Nominative vs Dative, e.g. "Zarządzie") -> ADD 0.2
> - IF match against `InvestigationConfig.keyword_matrices` -> ADD 0.4
> - IF regional context implies offshore holding shell -> ADD 0.2
> - IF PR buzzword density > 60% OR unrelated industry -> SUBTRACT 0.6
> 
> Limit final metric output strictly to Range $[0.0, 1.0]$.
