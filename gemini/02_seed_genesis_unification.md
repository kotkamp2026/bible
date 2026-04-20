# PHASE 1: SEED GENESIS & NLP UNIFICATION

## 1.1 TRIGGER & ACTION
- **Trigger:** System receives bare inputs (`NIP`, fuzzy `company name`, or `board_member`).
- **Action:** The Unification engine operates via NLP to traverse aliases, legal name variants, and homonyms. It outputs a normalized base node to begin mapping.

## 1.2 NODE_SEED SCHEMA (DATA STRUCTURE)
```json
{
  "Entity_Name_Variant": "Node_Seed",
  "Schema": {
    "node_id": "UUID v4",
    "primary_id": "STRING(NIP) INDEXED",
    "official_name": "STRING",
    "aliases": ["STRING"],
    "registry_status": "ENUM(ACTIVE, SUSPENDED, BANKRUPT)",
    "board_members": [{"name": "STRING", "role": "STRING", "start_date": "ISO8601"}],
    "ubos": ["STRING"],
    "subsidiaries": ["UUID"],
    "industry_code": "STRING(e.g., SIC/NAICS/PKD)"
  }
}
```

## 1.3 SEED CONFIGURATION SCHEMA (`SeedConfig`)
This JSON controls the operational thresholds of the NLP unification module to prevent mismatched NIP assignments or falsely merging two different entities sharing a generic name.
```json
{
  "SeedConfig": {
    "fuzzy_match_thresholds": {
      "exact": 1.0,
      "levenshtein_min": 0.85,
      "phonetic_min": 0.80,
      "jaccard_token_min": 0.75
    },
    "alias_resolution_depth": 2,
    "jurisdictional_boundaries": {
      "legal_registries": ["EU", "PL"],
      "media_osint": ["GLOBAL"]
    },
    "pl_nlp_normalization": {
      "enable_diacritic_folding": true, 
      "handle_legal_abbreviations": true,  
      "legal_abbreviation_map": {
        "Sp. z o.o.": ["spolka z ograniczona odpowiedzialnoscia", "sp z oo", "sp. z ograniczoną odp."],
        "S.A.": ["spółka akcyjna", "spolka akcyjna"],
        "S.K.A.": ["spółka komandytowo-akcyjna"]
      }
    }
  }
}
```

## 1.4 ALGORITHMIC NLP UNIFICATION
To correctly combine mentions, the engine first stripes Polish diacritics and maps abbreviation sets. Two potential entities, $N_A$ and $N_B$, are merged into a unified seed if:
```math
\max(Levenshtein(N_A.name, N_B.name), Phonetic(N_A.name, N_B.name)) > SeedConfig.fuzzy_match_thresholds.levenshtein_min 
```
_AND_ the resulting entities share at least one hard structural match (e.g., matching NIP, identical registry address, or overlapping board members).
