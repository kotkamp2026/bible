# PHASE 3: GRAPH ASSEMBLER & PROVENANCE

This split contains graph-native evidence structures, provenance-preserving nodes and edges, and latent-link logic.

## Clue, Node, Edge, and Discovery Graph Contracts

### `Clue`

```json
{
  "clue_id": "clu_01",
  "task_id": "task_legal_01",
  "parent_clue_ids": ["clu_parent_00"],
  "source": {
    "source_type": "court_document",
    "source_name": "Public Court Bulletin",
    "source_domain": "example.gov",
    "veracity_prior": 1.0
  },
  "document": {
    "document_id": "doc_01",
    "canonical_url": "https://example.gov/doc/01",
    "artifact_uri": "artifact://doc_01",
    "content_hash": "sha256:...",
    "title": "Postawienie zarzutów członkowi zarządu",
    "publication_date": "2026-03-01T10:00:00Z",
    "event_date": "2026-02-25T00:00:00Z",
    "mime_type": "application/pdf"
  },
  "content": {
    "text": "....",
    "extraction_quality": 0.96,
    "snippet_spans": [
      { "start": 180, "end": 260, "text_hash": "sha256:..." }
    ]
  },
  "extracted_entities": [
    {
      "mention": "Jan Kowalski",
      "mention_type": "person",
      "canonical_hint": "person_001",
      "confidence": 0.97,
      "role_hint": "board_member"
    }
  ],
  "extracted_events": [
    {
      "category": "CHARGES",
      "claim_state": "charge",
      "actor_alignment": 0.94,
      "sentence_span": { "start": 180, "end": 260 },
      "confidence": 0.95
    }
  ],
  "language": {
    "detected": "pl",
    "confidence": 0.99
  },
  "timestamps": {
    "retrieved_at": "2026-03-01T10:05:00Z",
    "parsed_at": "2026-03-01T10:05:07Z"
  }
}
```

### `Node`

```json
{
  "node_id": "node_event_01",
  "node_type": "EVENT",
  "subtype": "CRIMINAL_CHARGE",
  "canonical_ref": "seed_pl_nip_5250001001",
  "display_name": "Charge against board member linked to seed",
  "attributes": {
    "category": "CHARGES",
    "claim_state": "charge",
    "event_date": "2026-02-25T00:00:00Z",
    "actor_scope": "board_member",
    "industry_code": "PKD:41.20"
  },
  "confidence": 0.93,
  "embeddings_ref": "embed://node_event_01",
  "source_parents": [
    {
      "clue_id": "clu_01",
      "document_id": "doc_01",
      "source_url": "https://example.gov/doc/01",
      "content_hash": "sha256:...",
      "spans": [{ "start": 180, "end": 260 }]
    }
  ],
  "timestamps": {
    "observed_at": "2026-02-25T00:00:00Z",
    "ingested_at": "2026-03-01T10:06:00Z"
  }
}
```

### `Edge`

```json
{
  "edge_id": "edge_01",
  "from_node_id": "node_person_001",
  "to_node_id": "node_event_01",
  "edge_type": "INVOLVED_IN",
  "subtype": "BOARD_MEMBER_CHARGED",
  "directionality": "directed",
  "weight": 0.94,
  "state": "validated",
  "attributes": {
    "role": "actor",
    "relation_distance_to_seed": 1,
    "extraction_method": "typed_rule+entity_resolution"
  },
  "source_parents": [
    {
      "clue_id": "clu_01",
      "document_id": "doc_01",
      "source_url": "https://example.gov/doc/01",
      "content_hash": "sha256:...",
      "spans": [{ "start": 180, "end": 260 }]
    }
  ]
}
```

### `Discovery`

```json
{
  "discovery_id": "disc_01",
  "target_seed_id": "seed_pl_nip_5250001001",
  "status": "CONFIRMED",
  "category": "CHARGES",
  "severity_score_current": 0.84,
  "confidence": 0.91,
  "evidence_level": "L3_OFFICIAL_PROCEEDING",
  "event_window": {
    "start": "2026-02-25T00:00:00Z",
    "end": "2026-03-01T10:00:00Z"
  },
  "claim_summary": "Board-linked criminal charge with official-source corroboration.",
  "evidence_node_ids": ["node_event_01", "node_person_001", "node_doc_01"],
  "contradiction_node_ids": [],
  "score_components": {
    "base_severity": 0.78,
    "source_veracity": 1.0,
    "industry_modifier": 1.12,
    "role_factor": 0.88,
    "temporal_decay": 0.99,
    "independence_factor": 1.18,
    "stock_correlation_uplift": 0.0,
    "sanctions_uplift": 0.0
  },
  "historical_time_series": [
    {
      "bucket_start": "2026-02-01T00:00:00Z",
      "bucket_end": "2026-02-29T23:59:59Z",
      "raw_score": 0.71,
      "smoothed_score": 0.71,
      "delta_vs_prev": 0.71
    },
    {
      "bucket_start": "2026-03-01T00:00:00Z",
      "bucket_end": "2026-03-31T23:59:59Z",
      "raw_score": 0.84,
      "smoothed_score": 0.80,
      "delta_vs_prev": 0.09
    }
  ],
  "source_parents": [
    {
      "clue_id": "clu_01",
      "document_id": "doc_01",
      "source_url": "https://example.gov/doc/01",
      "content_hash": "sha256:...",
      "spans": [{ "start": 180, "end": 260 }]
    }
  ]
}
```

## Connector Agent Latent-Edge Rule

For nodes `i,j`:

```math
Latent(i,j)=\cos(z_i,z_j)\cdot TypeCompat(i,j)\cdot ProvenanceIndep(i,j)\cdot BridgeUtility(i,j)
```

Create `LATENT_ASSOCIATION` edge iff:

```math
Latent(i,j)\ge \theta_{latent}
```

with `validation_required=true`. Latent edges are never scored as confirmed evidence until validated by at least one provenance-backed rule or external corroboration.
