# PHASE 2: TARGETED INVESTIGATION LOOP & EXTERNAL SCRAPERS

This split contains the isolated scraper API contract, investigation loop controls, and recursive clue triage heuristics.

# 3. DATA CONTRACTS & SCHEMAS

## External Scraper HTTP API Specs (OpenAPI-style)

```yaml
openapi: 3.1.0
info:
  title: External Scraper Contract
  version: "1.0"
servers:
  - url: /api/v1

paths:
  /gather/osint:
    post:
      summary: Gather public media, trade press, and open web evidence for the seed entity and search vectors.
      headers:
        X-Idempotency-Key: { schema: { type: string } }
      requestBody:
        required: true
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/OsintGatherRequest'
      responses:
        "202":
          description: Async task accepted
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/AsyncTaskAccepted'

  /gather/legal:
    post:
      summary: Gather legal, court, registry-adjacent, and official public document evidence.
      headers:
        X-Idempotency-Key: { schema: { type: string } }
      requestBody:
        required: true
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/LegalGatherRequest'
      responses:
        "202":
          description: Async task accepted
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/AsyncTaskAccepted'

  /gather/registry:
    post:
      summary: Gather public registry data for company aliases, officers, UBOs, subsidiaries, addresses, sector codes.
      headers:
        X-Idempotency-Key: { schema: { type: string } }
      requestBody:
        required: true
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/RegistryGatherRequest'
      responses:
        "202":
          description: Async task accepted
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/AsyncTaskAccepted'

  /gather/sanctions:
    post:
      summary: Resolve direct and adjacency-based sanctions hits against entity, board, UBO, subsidiaries.
      headers:
        X-Idempotency-Key: { schema: { type: string } }
      requestBody:
        required: true
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/SanctionsGatherRequest'
      responses:
        "202":
          description: Async task accepted
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/AsyncTaskAccepted'

  /task/{task_id}:
    get:
      summary: Poll async task state.
      parameters:
        - in: path
          name: task_id
          required: true
          schema: { type: string }
      responses:
        "200":
          description: Task status
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/TaskStatus'

  /task/{task_id}/results:
    get:
      summary: Fetch paginated normalized clue results for a completed or partial task.
      parameters:
        - in: path
          name: task_id
          required: true
          schema: { type: string }
        - in: query
          name: cursor
          required: false
          schema: { type: string }
        - in: query
          name: page_size
          required: false
          schema: { type: integer, minimum: 1, maximum: 500 }
      responses:
        "200":
          description: Page of normalized clues
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/CluePage'

components:
  schemas:
    AsyncTaskAccepted:
      type: object
      required: [task_id, status, submitted_at, poll_url]
      properties:
        task_id: { type: string }
        status: { type: string, enum: [queued] }
        submitted_at: { type: string, format: date-time }
        poll_url: { type: string }
        results_url: { type: string }
        callback_expected: { type: boolean }
        parent_task_id: { type: string }
        estimated_result_count: { type: integer }

    CallbackSpec:
      type: object
      required: [url, secret_ref, events]
      properties:
        url: { type: string, format: uri }
        secret_ref: { type: string }
        events:
          type: array
          items:
            type: string
            enum: [task.queued, task.running, task.partial, task.completed, task.failed]

    TaskStatus:
      type: object
      required: [task_id, status, submitted_at, progress]
      properties:
        task_id: { type: string }
        parent_task_id: { type: string }
        status:
          type: string
          enum: [queued, running, partial, completed, failed, cancelled]
        submitted_at: { type: string, format: date-time }
        started_at: { type: string, format: date-time }
        completed_at: { type: string, format: date-time }
        progress:
          type: object
          properties:
            percent: { type: number, minimum: 0, maximum: 100 }
            urls_seen: { type: integer }
            documents_parsed: { type: integer }
            clues_emitted: { type: integer }
            retries: { type: integer }
        error:
          type: object
          properties:
            code: { type: string }
            message: { type: string }
            retriable: { type: boolean }

    GatherRequestBase:
      type: object
      required: [case_id, seed_profile, scope, search_context, depth]
      properties:
        case_id: { type: string }
        depth: { type: integer, minimum: 0 }
        parent_clue_ids:
          type: array
          items: { type: string }
        seed_profile:
          $ref: '#/components/schemas/SeedProfileRef'
        scope:
          type: object
          required: [jurisdictions, languages, date_window]
          properties:
            jurisdictions:
              type: array
              items: { type: string }
            languages:
              type: array
              items: { type: string }
            date_window:
              type: object
              properties:
                from: { type: string, format: date-time }
                to: { type: string, format: date-time }
        search_context:
          type: object
          required: [vectors, keyword_matrix_ref]
          properties:
            vectors:
              type: array
              items:
                $ref: '#/components/schemas/SearchVector'
            keyword_matrix_ref: { type: string }
            regional_weighting_ref: { type: string }
        callback:
          $ref: '#/components/schemas/CallbackSpec'

    OsintGatherRequest:
      allOf:
        - $ref: '#/components/schemas/GatherRequestBase'
        - type: object
          properties:
            source_families:
              type: array
              items:
                type: string
                enum: [news, trade_press, investigative_media, corporate_site, blog]
            crawl_policy:
              type: object
              properties:
                max_urls: { type: integer }
                max_hops_from_seed_page: { type: integer }
                allow_query_expansion: { type: boolean }
                same_domain_bias: { type: number, minimum: 0, maximum: 1 }

    LegalGatherRequest:
      allOf:
        - $ref: '#/components/schemas/GatherRequestBase'
        - type: object
          properties:
            document_families:
              type: array
              items:
                type: string
                enum: [court_document, registry_filing, official_notice, procurement_record, regulator_notice]
            legal_terms:
              type: array
              items: { type: string }
            registry_ids:
              type: array
              items: { type: string }

    RegistryGatherRequest:
      allOf:
        - $ref: '#/components/schemas/GatherRequestBase'
        - type: object
          properties:
            enrichment_targets:
              type: array
              items:
                type: string
                enum: [aliases, officers, ubos, subsidiaries, addresses, sector_codes]
            include_historical_changes: { type: boolean }

    SanctionsGatherRequest:
      allOf:
        - $ref: '#/components/schemas/GatherRequestBase'
        - type: object
          properties:
            match_targets:
              type: array
              items:
                type: string
                enum: [entity, board_members, ubos, subsidiaries]
            sanctions_scopes:
              type: array
              items:
                type: string
                enum: [EU, PUBLIC_GLOBAL]
            fuzzy_name_matching: { type: boolean }
            transliteration_matching: { type: boolean }

    SearchVector:
      type: object
      required: [vector_id, label, priority]
      properties:
        vector_id: { type: string }
        label: { type: string }
        priority: { type: number, minimum: 0, maximum: 1 }
        query_terms:
          type: array
          items: { type: string }
        entity_anchor:
          type: string
          enum: [seed, alias, board_member, ubo, subsidiary]
        category_bias:
          type: array
          items: { type: string }

    SeedProfileRef:
      type: object
      required: [primary_id, official_name]
      properties:
        primary_id: { type: string }
        official_name: { type: string }
        aliases:
          type: array
          items: { type: string }
        board_members:
          type: array
          items: { type: string }
        ubos:
          type: array
          items: { type: string }
        subsidiaries:
          type: array
          items: { type: string }
        industry_code: { type: string }

    CluePage:
      type: object
      required: [task_id, clues]
      properties:
        task_id: { type: string }
        cursor: { type: string }
        next_cursor: { type: string }
        clues:
          type: array
          items:
            $ref: '#/components/schemas/Clue'

    Clue:
      type: object
      required:
        [clue_id, task_id, source, document, content, extracted_entities, extracted_events, language, timestamps]
      properties:
        clue_id: { type: string }
        task_id: { type: string }
        parent_clue_ids:
          type: array
          items: { type: string }
        source:
          type: object
          required: [source_type, source_name, veracity_prior]
          properties:
            source_type:
              type: string
              enum: [news, trade_press, official_registry, court_document, sanctions_list, company_statement, blog]
            source_name: { type: string }
            source_domain: { type: string }
            veracity_prior: { type: number, minimum: 0, maximum: 1 }
        document:
          type: object
          required: [document_id, canonical_url, content_hash]
          properties:
            document_id: { type: string }
            canonical_url: { type: string, format: uri }
            artifact_uri: { type: string }
            content_hash: { type: string }
            title: { type: string }
            publication_date: { type: string, format: date-time }
            event_date: { type: string, format: date-time }
            mime_type: { type: string }
        content:
          type: object
          required: [text, extraction_quality]
          properties:
            text: { type: string }
            extraction_quality: { type: number, minimum: 0, maximum: 1 }
            snippet_spans:
              type: array
              items:
                type: object
                properties:
                  start: { type: integer }
                  end: { type: integer }
                  text_hash: { type: string }
        extracted_entities:
          type: array
          items:
            type: object
            properties:
              mention: { type: string }
              mention_type: { type: string, enum: [company, person, location, identifier] }
              canonical_hint: { type: string }
              confidence: { type: number, minimum: 0, maximum: 1 }
              role_hint:
                type: string
                enum: [seed, alias, board_member, ubo, subsidiary, unrelated]
        extracted_events:
          type: array
          items:
            type: object
            properties:
              category:
                type: string
                enum: [BRIBERY, CHARGES, BOARD, CORRUPTION, SANCTIONS, MONEY_LAUNDERING, FRAUD, EXTORTION, PROCUREMENT, LITIGATION]
              claim_state:
                type: string
                enum: [mention, allegation, investigation, charge, sanction, conviction, denial, acquittal, settlement]
              actor_alignment: { type: number, minimum: 0, maximum: 1 }
              sentence_span:
                type: object
                properties:
                  start: { type: integer }
                  end: { type: integer }
              confidence: { type: number, minimum: 0, maximum: 1 }
        language:
          type: object
          properties:
            detected: { type: string }
            confidence: { type: number, minimum: 0, maximum: 1 }
        timestamps:
          type: object
          properties:
            retrieved_at: { type: string, format: date-time }
            parsed_at: { type: string, format: date-time }
```

## `InvestigationConfig`

```json
{
  "$id": "InvestigationConfig",
  "type": "object",
  "required": [
    "threshold",
    "max_search_depth",
    "search_vectors",
    "keyword_matrices",
    "regional_weighting",
    "language_policy",
    "requeue_policy",
    "time_scope",
    "clue_ranking_weights"
  ],
  "properties": {
    "threshold": {
      "type": "number",
      "minimum": 0,
      "maximum": 1,
      "default": 0.65
    },
    "max_search_depth": {
      "type": "integer",
      "minimum": 0,
      "maximum": 6,
      "default": 3
    },
    "search_vectors": {
      "type": "array",
      "items": {
        "type": "object",
        "required": ["vector_id", "label", "priority", "query_templates"],
        "properties": {
          "vector_id": { "type": "string" },
          "label": {
            "type": "string",
            "enum": [
              "Russian Nexus",
              "Financial Fraud",
              "Board Charges",
              "Corruption & Bribery",
              "Sanctions Exposure",
              "Public Procurement",
              "Money Laundering",
              "Extortion & Fraud"
            ]
          },
          "priority": { "type": "number", "minimum": 0, "maximum": 1 },
          "query_templates": {
            "type": "array",
            "items": { "type": "string" }
          },
          "anchor_scope": {
            "type": "array",
            "items": {
              "type": "string",
              "enum": ["seed", "alias", "board_member", "ubo", "subsidiary"]
            }
          },
          "expansion_rules": {
            "type": "array",
            "items": {
              "type": "string",
              "enum": [
                "append_risk_lemmas",
                "append_role_titles",
                "append_jurisdiction_terms",
                "append_sanctions_terms",
                "append_procurement_terms"
              ]
            }
          }
        }
      },
      "default": [
        {
          "vector_id": "v1",
          "label": "Financial Fraud",
          "priority": 0.95,
          "query_templates": [
            "{alias} oszustwo",
            "{alias} wyłudzenie",
            "{alias} fraud",
            "{alias} financial irregularities"
          ],
          "anchor_scope": ["seed", "alias", "subsidiary"],
          "expansion_rules": ["append_risk_lemmas", "append_jurisdiction_terms"]
        },
        {
          "vector_id": "v2",
          "label": "Board Charges",
          "priority": 0.98,
          "query_templates": [
            "{board_member} zarzuty",
            "{board_member} prokuratura",
            "{alias} zarząd zarzuty",
            "{alias} management charges"
          ],
          "anchor_scope": ["board_member", "seed"],
          "expansion_rules": ["append_role_titles", "append_risk_lemmas"]
        },
        {
          "vector_id": "v3",
          "label": "Russian Nexus",
          "priority": 0.70,
          "query_templates": [
            "{alias} Rosja sankcje",
            "{alias} Russia sanctions",
            "{ubo} Belarus sanctions",
            "{subsidiary} export controls"
          ],
          "anchor_scope": ["seed", "ubo", "subsidiary"],
          "expansion_rules": ["append_sanctions_terms", "append_jurisdiction_terms"]
        }
      ]
    },
    "keyword_matrices": {
      "type": "object",
      "properties": {
        "BRIBERY": {
          "type": "object",
          "properties": {
            "lemmas": {
              "type": "array",
              "default": ["łapówka", "przekupstwo", "korzyść majątkowa", "bribe"]
            },
            "surface_forms": {
              "type": "array",
              "default": ["łapówki", "łapówkę", "łapówek", "wręczenie łapówki", "bribery"]
            },
            "negative_context": {
              "type": "array",
              "default": ["bez zarzutów łapownictwa", "nie stwierdzono korzyści majątkowej"]
            }
          }
        },
        "CHARGES": {
          "type": "object",
          "properties": {
            "lemmas": {
              "type": "array",
              "default": ["zarzut", "zarzuty", "postawienie zarzutów", "charge", "indictment"]
            },
            "surface_forms": {
              "type": "array",
              "default": ["usłyszał zarzuty", "postawiono zarzuty", "act of indictment"]
            },
            "negative_context": {
              "type": "array",
              "default": ["oddalono zarzuty", "brak zarzutów"]
            }
          }
        },
        "BOARD": {
          "type": "object",
          "properties": {
            "lemmas": {
              "type": "array",
              "default": ["zarząd", "prezes", "wiceprezes", "członek zarządu", "board", "management"]
            }
          }
        },
        "CORRUPTION": {
          "type": "object",
          "properties": {
            "lemmas": {
              "type": "array",
              "default": ["korupcja", "skorumpowany", "corruption", "corrupt practice"]
            }
          }
        },
        "SANCTIONS": {
          "type": "object",
          "properties": {
            "lemmas": {
              "type": "array",
              "default": ["sankcje", "lista sankcyjna", "asset freeze", "sanctions", "export ban"]
            }
          }
        },
        "MONEY_LAUNDERING": {
          "type": "object",
          "properties": {
            "lemmas": {
              "type": "array",
              "default": ["pranie pieniędzy", "prać pieniądze", "AML", "money laundering"]
            },
            "surface_forms": {
              "type": "array",
              "default": ["praniu pieniędzy", "zarzuty prania pieniędzy", "laundering proceeds"]
            }
          }
        },
        "FRAUD": {
          "type": "object",
          "properties": {
            "lemmas": {
              "type": "array",
              "default": ["oszustwo", "defraudacja", "fraud", "fałszerstwo"]
            },
            "surface_forms": {
              "type": "array",
              "default": ["oszustwa", "oszustwem", "fraudulent", "misrepresentation"]
            }
          }
        },
        "EXTORTION": {
          "type": "object",
          "properties": {
            "lemmas": {
              "type": "array",
              "default": ["wyłudzenie", "wymuszenie", "extortion", "fraudulent extraction"]
            },
            "surface_forms": {
              "type": "array",
              "default": ["wyłudzenia", "wyłudził", "wymusili", "extorted"]
            }
          }
        }
      }
    },
    "regional_weighting": {
      "type": "object",
      "properties": {
        "PL": { "type": "number", "default": 1.0 },
        "EU": { "type": "number", "default": 0.85 },
        "GLOBAL": { "type": "number", "default": 0.60 },
        "HIGH_RISK_TRANSBORDER": { "type": "number", "default": 0.95 }
      }
    },
    "language_policy": {
      "type": "object",
      "properties": {
        "supported_languages": {
          "type": "array",
          "default": ["pl", "en", "de", "uk", "ru"]
        },
        "polish_inflection_handling": { "type": "boolean", "default": true },
        "cross_lingual_synonym_expansion": { "type": "boolean", "default": true },
        "minimum_language_confidence": { "type": "number", "default": 0.70 }
      }
    },
    "time_scope": {
      "type": "object",
      "properties": {
        "mode": { "type": "string", "enum": ["rolling", "fixed"] },
        "lookback_days": { "type": "integer", "default": 365 },
        "include_historical_backfill": { "type": "boolean", "default": true }
      }
    },
    "clue_ranking_weights": {
      "type": "object",
      "properties": {
        "entity_match": { "type": "number", "default": 1.30 },
        "risk_keyword_density": { "type": "number", "default": 1.10 },
        "actor_alignment": { "type": "number", "default": 0.90 },
        "source_veracity": { "type": "number", "default": 0.70 },
        "jurisdiction_relevance": { "type": "number", "default": 0.60 },
        "temporal_relevance": { "type": "number", "default": 0.60 },
        "novelty": { "type": "number", "default": 0.80 },
        "graph_bridge_potential": { "type": "number", "default": 0.80 },
        "extraction_quality": { "type": "number", "default": 0.50 },
        "noise_penalty": { "type": "number", "default": 1.00 }
      }
    },
    "requeue_policy": {
      "type": "object",
      "properties": {
        "max_children_per_clue": { "type": "integer", "default": 5 },
        "dedupe_similarity_threshold": { "type": "number", "default": 0.92 },
        "require_new_anchor_or_new_category": { "type": "boolean", "default": true }
      }
    }
  }
}
```

# 5. HEURISTICS & ALGORITHMIC PROMPTS

## Potential_Score Evaluator

**Feature vector**

For clue `c` at depth `d`:

- `E(c)`: entity match confidence to seed/alias/board/UBO/subsidiary
- `K(c)`: normalized risk-keyword density after Polish lemma/synonym expansion
- `A(c)`: actor-role alignment score
- `S(c)`: source veracity prior
- `J(c)`: jurisdiction relevance weight
- `T(c)`: temporal relevance weight
- `N(c)`: novelty vs existing clue set
- `B(c)`: graph bridge potential
- `Q(c)`: extraction quality
- `X(c)`: noise/duplication penalty

\[
Potential\_Score(c)=\sigma\left(\beta_0+w_EE+w_KK+w_AA+w_SS+w_JJ+w_TT+w_NN+w_BB+w_QQ-w_XX\right)
\]

Default weights from `InvestigationConfig.clue_ranking_weights`:

\[
\beta_0=-1.0,\;
(w_E,w_K,w_A,w_S,w_J,w_T,w_N,w_B,w_Q,w_X)=
(1.30,1.10,0.90,0.70,0.60,0.60,0.80,0.80,0.50,1.00)
\]

Branching rule:

\[
expand(c)\iff Potential\_Score(c)\ge \theta \land depth(c)<D_{max}
\]

where `θ = InvestigationConfig.threshold`.

**System prompt**

```text
SYSTEM: Potential_Score_Evaluator

Objective:
Score whether a normalized clue justifies further investigative expansion for AML/due-diligence.

Inputs:
1. SeedProfile
2. InvestigationConfig
3. Clue
4. ExistingCaseSummary {known entities, discovered categories, seen domains, seen hashes}

Mandatory reasoning:
1. Normalize Polish inflections to canonical lemmas.
2. Map synonyms and cross-lingual variants to canonical categories:
   łapówka->BRIBERY, zarzuty->CHARGES, zarząd->BOARD, korupcja->CORRUPTION,
   sankcje->SANCTIONS, pranie pieniędzy->MONEY_LAUNDERING, oszustwo->FRAUD, wyłudzenie->EXTORTION.
3. Score actor alignment: direct company > board/UBO > subsidiary > affiliate > bystander.
4. Penalize duplicate or syndicated content using content_hash/similarity/domain repetition.
5. Reward clues that bridge unresolved graph gaps: new board alias, new subsidiary, official-case reference, sanctions adjacency, market-event alignment.
6. Do not treat co-mention alone as high value.
7. Prefer event_date over publication_date when both exist.
8. Output only structured JSON.

Output JSON:
{
  "clue_id": "...",
  "potential_score": 0.0-1.0,
  "expand": true|false,
  "recommended_search_vectors": ["..."],
  "feature_scores": {
    "entity_match": 0.0-1.0,
    "risk_keyword_density": 0.0-1.0,
    "actor_alignment": 0.0-1.0,
    "source_veracity": 0.0-1.0,
    "jurisdiction_relevance": 0.0-1.0,
    "temporal_relevance": 0.0-1.0,
    "novelty": 0.0-1.0,
    "graph_bridge_potential": 0.0-1.0,
    "extraction_quality": 0.0-1.0,
    "noise_penalty": 0.0-1.0
  },
  "reason_codes": ["DIRECT_BOARD_CHARGE", "OFFICIAL_SOURCE", "NEW_ALIAS", "..."]
}
```
