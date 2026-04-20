# PHASE 1: SEED GENESIS & NLP UNIFICATION

This split contains the canonical seed entity contract, seed initialization controls, and deterministic/fuzzy entity-resolution logic.

## Seed Entity Contract

### `SeedProfile`

```json
{
  "case_id": "case_2026_0001",
  "seed_id": "seed_pl_nip_5250001001",
  "structured": {
    "primary_id": "5250001001",
    "official_name": "ACME Sp. z o.o.",
    "aliases": [
      {
        "value": "ACME",
        "alias_type": "short_name",
        "confidence": 0.98
      },
      {
        "value": "ACME Polska",
        "alias_type": "market_name",
        "confidence": 0.86
      }
    ],
    "board_members": [
      {
        "person_id": "person_001",
        "full_name": "Jan Kowalski",
        "aliases": ["J. Kowalski"],
        "role": "prezes",
        "confidence": 0.93
      }
    ],
    "UBOs": [
      {
        "person_id": "person_101",
        "full_name": "Anna Nowak",
        "ownership_share": 0.62,
        "confidence": 0.88
      }
    ],
    "subsidiaries": [
      {
        "entity_id": "entity_sub_01",
        "official_name": "ACME Trading",
        "relationship_type": "subsidiary",
        "confidence": 0.91
      }
    ],
    "industry_code": "PKD:41.20",
    "jurisdictions": ["PL", "EU"]
  },
  "unstructured": [
    {
      "key": "alias_anomaly",
      "value": "Historical press frequently omits legal suffix.",
      "confidence": 0.82,
      "source_parents": []
    }
  ],
  "resolution": {
    "match_score": 0.97,
    "runner_up_score": 0.71,
    "ambiguity_state": "resolved"
  }
}
```

# 4. PIPELINE CONFIGURATION OBJECTS

## `SeedConfig`

```json
{
  "$id": "SeedConfig",
  "type": "object",
  "required": [
    "resolution_mode",
    "jurisdiction_scope",
    "alias_depth",
    "fuzzy_thresholds",
    "ambiguity_policy",
    "normalization",
    "registry_enrichment"
  ],
  "properties": {
    "resolution_mode": {
      "type": "string",
      "enum": ["nip", "name", "mixed"],
      "default": "mixed"
    },
    "jurisdiction_scope": {
      "type": "object",
      "required": ["legal", "media"],
      "properties": {
        "legal": {
          "type": "array",
          "items": { "type": "string", "enum": ["PL", "EU"] },
          "default": ["PL", "EU"]
        },
        "media": {
          "type": "array",
          "items": { "type": "string" },
          "default": ["GLOBAL"]
        },
        "sanctions": {
          "type": "array",
          "items": { "type": "string", "enum": ["EU", "PUBLIC_GLOBAL"] },
          "default": ["EU"]
        }
      }
    },
    "alias_depth": {
      "type": "object",
      "properties": {
        "company_alias_hops": { "type": "integer", "minimum": 0, "maximum": 5, "default": 2 },
        "board_alias_hops": { "type": "integer", "minimum": 0, "maximum": 3, "default": 1 },
        "subsidiary_alias_hops": { "type": "integer", "minimum": 0, "maximum": 3, "default": 1 }
      }
    },
    "fuzzy_thresholds": {
      "type": "object",
      "properties": {
        "accept": { "type": "number", "minimum": 0, "maximum": 1, "default": 0.92 },
        "review": { "type": "number", "minimum": 0, "maximum": 1, "default": 0.78 },
        "exact_name_bonus": { "type": "number", "default": 0.12 },
        "alias_bonus": { "type": "number", "default": 0.06 },
        "acronym_bonus": { "type": "number", "default": 0.04 },
        "nip_bonus": { "type": "number", "default": 0.45 },
        "homonym_penalty": { "type": "number", "default": 0.20 },
        "industry_mismatch_penalty": { "type": "number", "default": 0.08 },
        "jurisdiction_mismatch_penalty": { "type": "number", "default": 0.06 }
      }
    },
    "ambiguity_policy": {
      "type": "object",
      "properties": {
        "top_k_candidates": { "type": "integer", "default": 5 },
        "min_margin_over_runner_up": { "type": "number", "default": 0.08 },
        "require_manual_confirmation_below": { "type": "number", "default": 0.92 },
        "block_on_competing_nip": { "type": "boolean", "default": true }
      }
    },
    "normalization": {
      "type": "object",
      "properties": {
        "strip_legal_suffixes": { "type": "boolean", "default": true },
        "preserve_diacritics": { "type": "boolean", "default": true },
        "diacritic_fold_index": { "type": "boolean", "default": true },
        "case_fold": { "type": "boolean", "default": true },
        "punctuation_fold": { "type": "boolean", "default": true },
        "whitespace_fold": { "type": "boolean", "default": true },
        "transliteration_matching": { "type": "boolean", "default": true },
        "acronym_generation": { "type": "boolean", "default": true },
        "nip_checksum_required": { "type": "boolean", "default": true }
      }
    },
    "registry_enrichment": {
      "type": "object",
      "properties": {
        "include_board_members": { "type": "boolean", "default": true },
        "include_ubos": { "type": "boolean", "default": true },
        "include_subsidiaries": { "type": "boolean", "default": true },
        "include_historical_names": { "type": "boolean", "default": true },
        "include_historical_officers": { "type": "boolean", "default": true }
      }
    },
    "unstructured_capture": {
      "type": "object",
      "properties": {
        "max_pairs": { "type": "integer", "default": 50 },
        "store_resolution_anomalies": { "type": "boolean", "default": true },
        "store_name_collision_notes": { "type": "boolean", "default": true }
      }
    }
  }
}
```

## Entity Resolution Decision Function

Let `q` be user input or article mention and `e` a candidate company record.

```math
ER(q,e)=0.45N+0.20X+0.10A+0.05C+0.06B+0.04U+0.04I+0.06J-0.20H
```

Where:

- `N`: exact valid NIP match
- `X`: normalized official-name exactness
- `A`: alias similarity
- `C`: acronym similarity
- `B`: board-member relational support
- `U`: UBO relational support
- `I`: industry compatibility
- `J`: jurisdiction compatibility
- `H`: homonym risk penalty

Acceptance:

```math
resolved \iff ER(q,e^\*)\ge T_{accept}\land ER(q,e^\*)-ER(q,e_2)\ge T_{margin}
```

Otherwise:

- `AMBIGUOUS` if `ER(q,e*) >= review`
- `REJECTED` if all candidates `< review`
