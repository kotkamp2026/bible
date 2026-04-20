# 1. [EXPANDED_CONTRACTS]: EXTERNAL SCRAPER WEBHOOK PAYLOADS

The external acquisition worker communicates back to the Orchestrator exclusively via the `POST /webhook/callback` endpoint. This schema mandates rigorous error states for handling immediate rate-limit feedback logic.

```yaml
openapi: 3.1.0
paths:
  /webhook/callback:
    post:
      summary: "Ingest asynchronous payload from external Scraper worker."
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              required: [task_id, target_entity, source_url, http_status, timestamp]
              properties:
                task_id:
                  type: string
                  format: uuid
                  description: "Correlates to Phase 2 Dispatch ID."
                target_entity:
                  type: string
                  description: "The primary Node_Seed string driving this scrape."
                source_url:
                  type: string
                  format: uri
                http_status:
                  type: integer
                  description: "Status returned by the target host (e.g. 200, 429, 403)."
                error_state:
                  type: object
                  nullable: true
                  properties:
                    code:
                      type: string
                      enum: [RATE_LIMITED, CLOUDFLARE_BLOCK, TIMEOUT, CAPTCHA_HIT, NONE]
                    retry_after_sec:
                      type: integer
                detected_language:
                  type: string
                  enum: [PL, EN, RU, UK, UNKNOWN]
                timestamp:
                  type: string
                  format: date-time
                raw_html_extract:
                  type: string
                  nullable: true
                  description: "Sanitized text block containing relevant entity proximity. Null if error_state != NONE."
                metadata:
                  type: object
                  properties:
                    author: { type: string }
                    publication_date: { type: string, format: date-time }
```

# 2. [EXPANDED_MCP]: DISCOVERY ENGINE TOOL DEFINITIONS

The Model Context Protocol (MCP) server grants the synthesis engine direct read/query privileges over the environment.

```json
{
  "tools": [
    {
      "name": "query_graph_topology",
      "description": "Traverses the Directed Property Graph from a starting Node_Seed to a maximum depth to retrieve adjacent Clues and Mentions.",
      "input_schema": {
        "type": "object",
        "properties": {
          "node_id": { "type": "string", "format": "uuid" },
          "max_depth": { "type": "integer", "default": 2 }
        },
        "required": ["node_id"]
      },
      "returns": "ARRAY of Sub-Graph Objects containing adjacent Node_Clues and Edge_Mentions weights."
    },
    {
      "name": "verify_registry_status",
      "description": "Synchronous call to PL KRS/REGON API to cross-reference entity legitimacy and active board members.",
      "input_schema": {
        "type": "object",
        "properties": {
          "nip": { "type": "string", "pattern": "^[0-9]{10}$" }
        },
        "required": ["nip"]
      },
      "returns": "{ 'registry_status': 'ACTIVE|SUSPENDED|BANKRUPT', 'board_active': ['string'] }"
    },
    {
      "name": "fetch_stock_variance",
      "description": "Retrieves % variance in stock price for the specified ticker over the window_days prior to today. Correlates market reaction to Clue publication.",
      "input_schema": {
        "type": "object",
        "properties": {
          "ticker": { "type": "string", "description": "GPW (Warsaw Stock Exchange) Ticker e.g. 'PKN'" },
          "window_days": { "type": "integer", "maximum": 30 }
        },
        "required": ["ticker", "window_days"]
      }
    }
  ]
}
```

# 3. [EXPANDED_HEURISTICS]: CONFLICT RESOLUTION & BAYESIAN UPDATING

**Scenario:** 
- `Node_Clue_A` (News Article) claims "CEO arrested for bribery" (Severity 0.9). 
- `Node_Clue_B` (Court Document) claims "Charges dismissed" (Severity 0.0).

**Mathematical Heuristic: Temporal Overwrite Matrix:**
To resolve topological conflicts without hallucinatory guesswork, the system forces $S_{override}$ logic based strictly on $V_{src}$ (Veracity) and temporal precedence $t$.

**Conflict Engine Logic:**
1. Identify intersecting Clue Nodes linked to the same Incident Vector (e.g. `FRAUD`).
2. **Rule 1 (Absolute Veracity Dominance):** If `Clue_B.source_veracity == 1.0` (Court/Registry) AND `Clue_A.source_veracity < 1.0`, `Clue_B` overrides `Clue_A` completely. $S_{\text{final}} = S_{Clue\_B}$.
3. **Rule 2 (Temporal Precedence within Tier):** If `Clue_A.timestamp < Clue_B.timestamp` AND `Clue_B.source_veracity >= Clue_A.source_veracity`:
   - Newer & Equal/Higher Veracity overwrites older state. 
   - $S_{\text{final}} = S_{Clue\_B}$.
4. **Rule 3 (Suspicious Later Claims):** If `Clue_A.timestamp < Clue_B.timestamp` BUT `Clue_B.source_veracity < Clue_A.source_veracity` (e.g. High Veracity arrest followed by Low Veracity PR spin):
   - High Veracity node anchors context.
   - Apply Bayesian penalty constraint: $S_{\text{final}} = S_{Clue\_A} - (0.1 \times S_{Clue\_B})$.

**Discovery Engine System Prompt Ingestion:**
> "CONFLICT PROTOCOL: If graph traversal yields contradictory claims spanning different `source_nodes`, you MUST resolve by Veracity Weight `V_src`. `V_src = 1.0` is an absolute anchor. If `V_src` is identical, the newest `timestamp_published` asserts the `base_severity_assigned`. State the resolution math in your debug payload."

# 4. [EXPANDED_NLP]: ADVANCED POLISH LINGUISTIC EDGE CASES

Polish grammar modifies names dynamically (Mianownik: Jan Kowalski $\rightarrow$ Dopełniacz: Jana Kowalskiego). Doing Raw-Regex guarantees false negatives.

**Pipeline Constraint: Pre-Vectorization Normalization Hook**
All `raw_html_extract` text MUST pass through an NLP Lemmatization pipeline before edge creation algorithms fire.

1. **Pipeline Execution:** `Tokenizer` $\rightarrow$ `Lemmatizer`.
2. Example transformation executed instantly prior to graph indexing: 
   - RAW: *"Zatrzymano Jana Kowalskiego za łapówkę w spółce z o.o."*
   - NORMALIZED: *"zatrzymać Jan Kowalski za łapówka w spółka z ograniczoną odpowiedzialnością"*

**Structural Alias Resolution Schema:**
```json
{
  "entity_normalization_map": {
    "Sp. z o.o.": "spółka z ograniczoną odpowiedzialnością",
    "Sp. z ograniczoną odp.": "spółka z ograniczoną odpowiedzialnością",
    "Spółka z o.o.": "spółka z ograniczoną odpowiedzialnością",
    "S.A.": "spółka akcyjna",
    "P.S.A.": "prosta spółka akcyjna"
  },
  "lemma_overrides_to_keywords": {
    "skorumpowanie": "korupcja",
    "wyłudził": "wyłudzenie",
    "wyłudzonego": "wyłudzenie",
    "defraudacja": "wyłudzenie"
  }
}
```
