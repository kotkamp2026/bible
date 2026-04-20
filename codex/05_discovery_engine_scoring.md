# PHASE 4: DISCOVERY ENGINE & REPUTATION SCORING

This split contains discovery contracts, context-aware industry risk semantics, scoring formulas, and the MCP synthesis prompt.

## `DiscoveryConfig`

```json
{
  "$id": "DiscoveryConfig",
  "type": "object",
  "required": [
    "risk_taxonomy",
    "industry_sentiment_modifiers",
    "source_veracity_weights",
    "temporal_decay_factors",
    "claim_state_weights",
    "evidence_ladder",
    "linguistic_engine",
    "graph_reasoning",
    "aggregation",
    "timeline",
    "status_thresholds",
    "stock_correlation_flags"
  ],
  "properties": {
    "risk_taxonomy": {
      "type": "object",
      "properties": {
        "BRIBERY": { "type": "number", "default": 0.82 },
        "CHARGES": { "type": "number", "default": 0.78 },
        "BOARD": { "type": "number", "default": 0.68 },
        "CORRUPTION": { "type": "number", "default": 0.88 },
        "SANCTIONS": { "type": "number", "default": 0.94 },
        "MONEY_LAUNDERING": { "type": "number", "default": 0.96 },
        "FRAUD": { "type": "number", "default": 0.86 },
        "EXTORTION": { "type": "number", "default": 0.84 },
        "PROCUREMENT": { "type": "number", "default": 0.72 },
        "LITIGATION": { "type": "number", "default": 0.55 }
      }
    },
    "industry_sentiment_modifiers": {
      "type": "object",
      "properties": {
        "FINANCIAL_SERVICES": {
          "type": "object",
          "properties": {
            "MONEY_LAUNDERING": { "type": "number", "default": 1.35 },
            "SANCTIONS": { "type": "number", "default": 1.30 },
            "FRAUD": { "type": "number", "default": 1.15 },
            "CORRUPTION": { "type": "number", "default": 1.05 },
            "CHARGES": { "type": "number", "default": 1.12 }
          }
        },
        "CONSTRUCTION": {
          "type": "object",
          "properties": {
            "BRIBERY": { "type": "number", "default": 1.35 },
            "CORRUPTION": { "type": "number", "default": 1.30 },
            "PROCUREMENT": { "type": "number", "default": 1.25 },
            "FRAUD": { "type": "number", "default": 1.10 }
          }
        },
        "ENERGY": {
          "type": "object",
          "properties": {
            "SANCTIONS": { "type": "number", "default": 1.30 },
            "CORRUPTION": { "type": "number", "default": 1.20 },
            "BRIBERY": { "type": "number", "default": 1.15 }
          }
        },
        "DEFENSE": {
          "type": "object",
          "properties": {
            "SANCTIONS": { "type": "number", "default": 1.40 },
            "CORRUPTION": { "type": "number", "default": 1.25 },
            "BRIBERY": { "type": "number", "default": 1.15 }
          }
        },
        "HEALTHCARE": {
          "type": "object",
          "properties": {
            "FRAUD": { "type": "number", "default": 1.25 },
            "PROCUREMENT": { "type": "number", "default": 1.20 },
            "CORRUPTION": { "type": "number", "default": 1.15 }
          }
        },
        "GENERIC": {
          "type": "object",
          "properties": {
            "BRIBERY": { "type": "number", "default": 1.00 },
            "CHARGES": { "type": "number", "default": 1.00 },
            "BOARD": { "type": "number", "default": 1.00 },
            "CORRUPTION": { "type": "number", "default": 1.00 },
            "SANCTIONS": { "type": "number", "default": 1.00 },
            "MONEY_LAUNDERING": { "type": "number", "default": 1.00 },
            "FRAUD": { "type": "number", "default": 1.00 },
            "EXTORTION": { "type": "number", "default": 1.00 }
          }
        }
      }
    },
    "source_veracity_weights": {
      "type": "object",
      "properties": {
        "official_sanctions_list": { "type": "number", "default": 1.00 },
        "court_document": { "type": "number", "default": 1.00 },
        "official_registry": { "type": "number", "default": 0.98 },
        "listed_disclosure": { "type": "number", "default": 0.96 },
        "tier1_wire": { "type": "number", "default": 0.90 },
        "national_media": { "type": "number", "default": 0.82 },
        "trade_press": { "type": "number", "default": 0.76 },
        "local_media": { "type": "number", "default": 0.70 },
        "investigative_media": { "type": "number", "default": 0.78 },
        "company_statement": { "type": "number", "default": 0.55 },
        "blog": { "type": "number", "default": 0.20 },
        "forum_social": { "type": "number", "default": 0.15 }
      }
    },
    "temporal_decay_factors": {
      "type": "object",
      "properties": {
        "SANCTIONS": { "type": "number", "default": 0.002 },
        "MONEY_LAUNDERING": { "type": "number", "default": 0.003 },
        "CORRUPTION": { "type": "number", "default": 0.004 },
        "BRIBERY": { "type": "number", "default": 0.004 },
        "CHARGES": { "type": "number", "default": 0.005 },
        "FRAUD": { "type": "number", "default": 0.006 },
        "EXTORTION": { "type": "number", "default": 0.006 },
        "BOARD": { "type": "number", "default": 0.008 },
        "LITIGATION": { "type": "number", "default": 0.009 },
        "ALLEGATION_ONLY_MULTIPLIER": { "type": "number", "default": 1.40 }
      },
      "description": "Values are lambda/day in exp(-lambda * days_since_event)."
    },
    "claim_state_weights": {
      "type": "object",
      "properties": {
        "mention": { "type": "number", "default": 0.20 },
        "allegation": { "type": "number", "default": 0.55 },
        "investigation": { "type": "number", "default": 0.70 },
        "charge": { "type": "number", "default": 0.82 },
        "sanction": { "type": "number", "default": 0.98 },
        "conviction": { "type": "number", "default": 1.00 },
        "settlement": { "type": "number", "default": 0.72 },
        "denial": { "type": "number", "default": -0.20 },
        "acquittal": { "type": "number", "default": -0.75 }
      }
    },
    "role_factors": {
      "type": "object",
      "properties": {
        "seed_direct": { "type": "number", "default": 1.00 },
        "board_member": { "type": "number", "default": 0.88 },
        "ubo": { "type": "number", "default": 0.92 },
        "subsidiary": { "type": "number", "default": 0.78 },
        "affiliate": { "type": "number", "default": 0.60 },
        "bystander": { "type": "number", "default": 0.25 }
      }
    },
    "evidence_ladder": {
      "type": "object",
      "properties": {
        "L0_MENTION": { "type": "string", "default": "co-mention or weak semantic reference only" },
        "L1_SINGLE_SOURCE_ALLEGATION": { "type": "string", "default": "single non-official source with aligned actor-role" },
        "L2_CORROBORATED_ALLEGATION": { "type": "string", "default": ">=2 independent sources or 1 high-veracity source + corroborating media" },
        "L3_OFFICIAL_PROCEEDING": { "type": "string", "default": "official investigation, charge, sanction, regulator action" },
        "L4_FINALIZED_OUTCOME": { "type": "string", "default": "conviction, final sanction, final court-confirmed outcome" }
      }
    },
    "linguistic_engine": {
      "type": "object",
      "properties": {
        "polish_inflection": {
          "type": "object",
          "properties": {
            "enabled": { "type": "boolean", "default": true },
            "lemma_match_weight": { "type": "number", "default": 1.00 },
            "surface_form_match_weight": { "type": "number", "default": 0.92 },
            "diacritic_fold_match_weight": { "type": "number", "default": 0.88 }
          }
        },
        "synonym_mapping": {
          "type": "object",
          "properties": {
            "łapówka": { "type": "string", "default": "BRIBERY" },
            "zarzuty": { "type": "string", "default": "CHARGES" },
            "zarząd": { "type": "string", "default": "BOARD" },
            "korupcja": { "type": "string", "default": "CORRUPTION" },
            "sankcje": { "type": "string", "default": "SANCTIONS" },
            "pranie pieniędzy": { "type": "string", "default": "MONEY_LAUNDERING" },
            "oszustwo": { "type": "string", "default": "FRAUD" },
            "wyłudzenie": { "type": "string", "default": "EXTORTION" }
          }
        },
        "negation_markers": {
          "type": "array",
          "default": ["nie", "bez", "brak", "oddalono", "uniewinniono", "dismissed", "acquitted"]
        },
        "hedge_markers": {
          "type": "array",
          "default": ["rzekomo", "prawdopodobnie", "allegedly", "may have", "could have"]
        },
        "historical_reference_markers": {
          "type": "array",
          "default": ["w przeszłości", "historycznie", "former", "previously", "years ago"]
        },
        "role_patterns": {
          "type": "object",
          "properties": {
            "actor": { "type": "array", "default": ["spółka", "zarząd", "prezes", "ubo", "subsidiary", "company was charged"] },
            "target": { "type": "array", "default": ["wobec spółki", "against the company", "targeted entity"] },
            "bystander": { "type": "array", "default": ["mentioned alongside", "quoted", "commented on"] }
          }
        }
      }
    },
    "graph_reasoning": {
      "type": "object",
      "properties": {
        "latent_edge_threshold": { "type": "number", "default": 0.84 },
        "max_relation_distance_from_seed": { "type": "integer", "default": 2 },
        "require_provenance_for_asserted_edges": { "type": "boolean", "default": true },
        "independent_source_similarity_ceiling": { "type": "number", "default": 0.85 },
        "duplicate_document_similarity": { "type": "number", "default": 0.92 },
        "contradiction_penalty": { "type": "number", "default": 0.25 }
      }
    },
    "stock_correlation_flags": {
      "type": "object",
      "properties": {
        "enabled": { "type": "boolean", "default": true },
        "apply_only_if_listed_or_quote_available": { "type": "boolean", "default": true },
        "window_hours_before": { "type": "integer", "default": 24 },
        "window_hours_after": { "type": "integer", "default": 72 },
        "abnormal_return_formula": {
          "type": "string",
          "default": "AR_t = R_entity_t - (alpha + beta * R_market_t)"
        },
        "uplifts": {
          "type": "object",
          "properties": {
            "mild_drop_leq_-0.03": { "type": "number", "default": 0.03 },
            "moderate_drop_leq_-0.07": { "type": "number", "default": 0.07 },
            "severe_drop_leq_-0.12": { "type": "number", "default": 0.12 }
          }
        }
      }
    },
    "sanctions_correlation": {
      "type": "object",
      "properties": {
        "direct_entity_match_uplift": { "type": "number", "default": 0.20 },
        "ubo_or_board_match_uplift": { "type": "number", "default": 0.10 },
        "subsidiary_match_uplift": { "type": "number", "default": 0.08 },
        "minimum_match_confidence": { "type": "number", "default": 0.90 }
      }
    },
    "aggregation": {
      "type": "object",
      "properties": {
        "independence_boost_rho": { "type": "number", "default": 0.20 },
        "positive_evidence_cap": { "type": "number", "default": 0.99 },
        "mitigation_weight": { "type": "number", "default": 0.65 },
        "discovery_relevance_direct": { "type": "number", "default": 1.00 },
        "discovery_relevance_indirect": { "type": "number", "default": 0.75 },
        "company_aggregation_mode": { "type": "string", "enum": ["noisy_or"], "default": "noisy_or" }
      }
    },
    "timeline": {
      "type": "object",
      "properties": {
        "bucket_granularity": { "type": "string", "enum": ["day", "week", "month"], "default": "month" },
        "smoothing_ewma_rho": { "type": "number", "default": 0.65 },
        "use_event_date_over_publication_date": { "type": "boolean", "default": true }
      }
    },
    "status_thresholds": {
      "type": "object",
      "properties": {
        "probable_min": { "type": "number", "default": 0.55 },
        "confirmed_min": { "type": "number", "default": 0.80 },
        "min_independent_sources_for_probable": { "type": "integer", "default": 1 },
        "min_independent_sources_for_confirmed_without_official_doc": { "type": "integer", "default": 3 }
      }
    },
    "explainability": {
      "type": "object",
      "properties": {
        "store_top_drivers": { "type": "integer", "default": 5 },
        "store_per_bucket_components": { "type": "boolean", "default": true },
        "require_evidence_quote_spans": { "type": "boolean", "default": true }
      }
    }
  }
}
```

## Context-Aware Sentiment / Risk Formulation

Generic polarity is auxiliary only. Primary signal is `industry-aware risk sentiment`.

For sentence `s`, category `k`, company `c`, industry `g`:

\[
Lex_{k}(s)=\max_{u\in U_k} match(\operatorname{lemma}(s),u)
\]

where `U_k` contains Polish lemmas, inflected forms, synonyms, and cross-lingual equivalents.

\[
Ctx_{k}(s,c)=Lex_k(s)\cdot Role(s,c)\cdot Assert(s)\cdot Hist(s)\cdot \left(1-Neg(s)\right)\cdot \left(1-Hedge(s)\right)\cdot M_{g,k}
\]

Where:

- `Role(s,c) ∈ [0,1]`: subject/object dependency alignment of company/board/UBO/subsidiary to event
- `Assert(s) ∈ [0,1]`: degree of factual assertion vs speculation
- `Hist(s) ∈ [0,1]`: historical-reference attenuation
- `Neg(s) ∈ [0,1]`: negation/exoneration factor
- `Hedge(s) ∈ [0,1]`: uncertainty factor
- `M_{g,k}`: industry sentiment modifier from `DiscoveryConfig.industry_sentiment_modifiers`

Article-level category risk:

\[
AR_k(a,c)=1-\prod_{s\in S_a}(1-w_k\cdot Ctx_k(s,c))
\]

Article-level contextual risk sentiment:

\[
RiskSentiment(a,c)=\sigma\left(\sum_{k\in K}\gamma_k AR_k(a,c)-\sum_{m\in Mitigations}\delta_m Mit_m(a,c)\right)
\]

Interpretation:

- `0.00-0.24`: low-risk/irrelevant mention
- `0.25-0.54`: weak concern
- `0.55-0.79`: material risk
- `0.80-1.00`: severe risk

This is industry-aware because identical phrases receive different multipliers by sector; e.g. `SANCTIONS` in `FINANCIAL_SERVICES` > `SANCTIONS` in `GENERIC`, while `BRIBERY` in `CONSTRUCTION` > `BRIBERY` in `GENERIC`.

## Reputation Score: Current Value + Historical Time-Series

For evidence item `e` in category `k` at time `t`:

\[
Decay_e(t)=e^{-\lambda_k\cdot \Delta days(t,\tau_e)}
\]

where `τ_e = event_date` if present, else `publication_date`.

Independent-source boost:

\[
Indep(d)=1+\rho\left(1-e^{-n_{indep}(d)}\right)
\]

Evidence contribution:

\[
\chi_e(t)=Base_k \cdot Veracity_e \cdot Conf_e \cdot ClaimState_e \cdot Role_e \cdot Industry_{g,k} \cdot Decay_e(t) \cdot Indep(d) \cdot (1+Stock_e+Sanction_e)
\]

Where:

- `Base_k`: base severity from `risk_taxonomy`
- `Veracity_e`: source weight
- `Conf_e`: extraction + classification + resolution confidence
- `ClaimState_e`: state multiplier from `claim_state_weights`
- `Role_e`: direct/board/UBO/subsidiary factor
- `Industry_{g,k}`: sector modifier
- `Stock_e`: abnormal-return uplift
- `Sanction_e`: sanctions adjacency/direct-match uplift

Positive and mitigating evidence are separated:

\[
P_d(t)=1-\prod_{e\in E_d^+}\left(1-\min(0.99,\chi_e(t))\right)
\]

\[
M_d(t)=1-\prod_{e\in E_d^-}\left(1-\min(0.99,|\chi_e(t)|)\right)
\]

Discovery severity:

\[
Severity_d(t)=clamp_{[0,1]}\left(P_d(t)-\omega_m M_d(t)-\omega_c Contradiction_d(t)\right)
\]

Discovery status:

- `CONFIRMED` if `Severity_d(t) >= confirmed_min` and (`L3+` evidence ladder or `>=3` independent non-duplicate sources)
- `PROBABLE` if `Severity_d(t) >= probable_min`
- otherwise `UNRESOLVED`

Company-level current reputation score via noisy-or aggregation:

\[
Reputation(t)=1-\prod_{d\in D}\left(1-Relevance_d\cdot Severity_d(t)\right)
\]

Where `Relevance_d = 1.00` for direct-company discoveries, `0.75-0.92` for board/UBO/subsidiary-linked discoveries based on relation strength.

Historical time series over buckets `b`:

\[
R_b = Reputation(t=b_{end})
\]

Optional smoothing for visualization:

\[
\widetilde{R}_b = \rho_{ewma}R_b + (1-\rho_{ewma})\widetilde{R}_{b-1}
\]

Delta:

\[
\Delta_b=\widetilde{R}_b-\widetilde{R}_{b-1}
\]

Output timeline point schema:

```json
{
  "bucket_start": "YYYY-MM-DDTHH:MM:SSZ",
  "bucket_end": "YYYY-MM-DDTHH:MM:SSZ",
  "raw_score": 0.0,
  "smoothed_score": 0.0,
  "delta_vs_prev": 0.0,
  "dominant_categories": ["SANCTIONS", "MONEY_LAUNDERING"],
  "top_drivers": [
    {
      "discovery_id": "disc_01",
      "contribution": 0.18
    }
  ]
}
```

**Stock-correlation uplift**

\[
AR_t = R_{entity,t}-(\alpha+\beta R_{market,t})
\]

\[
Stock_e=
\begin{cases}
0.12 & \text{if } AR_t \le -0.12 \\
0.07 & \text{if } AR_t \le -0.07 \\
0.03 & \text{if } AR_t \le -0.03 \\
0 & \text{otherwise}
\end{cases}
\]

**Sanctions uplift**

\[
Sanction_e=
\begin{cases}
0.20 & \text{direct entity match} \\
0.10 & \text{UBO/board match} \\
0.08 & \text{subsidiary match} \\
0 & \text{no match}
\end{cases}
\]

## MCP System Prompt Architecture for `DISCOVERY_ENGINE`

```text
SYSTEM: Discovery_Engine

Mission:
Transform a provenance-complete seed subgraph into evidence-backed AML/due-diligence discoveries and a defendable reputation score explanation.

Inputs:
1. SeedProfile
2. DiscoveryConfig
3. InvestigationConfig
4. GraphSubgraph {nodes, edges, latent_edges, source_parents}
5. OfficialEvidence {court, registry, sanctions}
6. MarketSignals {abnormal returns, event windows}

Non-negotiable rules:
1. Use only supplied graph facts, official enrichments, and quoted source spans.
2. Never assert guilt from co-mention alone.
3. Distinguish states exactly: mention, allegation, investigation, charge, sanction, conviction, denial, acquittal, settlement.
4. Prefer official/legal evidence over media when they conflict; preserve contradiction explicitly.
5. Treat board/UBO/subsidiary evidence as indirect unless relation confidence passes config threshold.
6. Normalize Polish inflections and synonyms before categorization.
7. If a claim lacks actor alignment to the seed or linked role, mark it irrelevant or low-confidence.
8. Latent edges are hints, not proof.
9. If evidence is insufficient, output unresolved instead of guessing.
10. Output structured JSON only.

Reasoning procedure:
A. Canonicalize lexemes:
   - łapówka->BRIBERY
   - zarzuty->CHARGES
   - zarząd->BOARD
   - korupcja->CORRUPTION
   - sankcje->SANCTIONS
   - pranie pieniędzy->MONEY_LAUNDERING
   - oszustwo->FRAUD
   - wyłudzenie->EXTORTION
B. Cluster evidence by {target entity, category, event window, claim state}.
C. Build evidence table with fields:
   {source_type, source_veracity, actor_alignment, claim_state, role_scope, event_date, contradiction_flag, provenance_chain}
D. Assign evidence ladder level:
   L0 mention
   L1 single-source allegation
   L2 corroborated allegation
   L3 official proceeding
   L4 finalized outcome
E. Apply industry modifiers from DiscoveryConfig.industry_sentiment_modifiers using SeedProfile.industry_code.
F. Compute discovery-level severity using the supplied formulas; do not invent alternative scoring logic.
G. Mark discovery status:
   CONFIRMED if threshold satisfied and evidence ladder permits.
   PROBABLE if threshold satisfied but official certainty is lower.
   UNRESOLVED otherwise.
H. Produce contradiction notes, mitigation notes, and top score drivers.
I. Emit historical score contributions per bucket using event_date-preferred temporal logic.

Required JSON output:
{
  "discoveries": [
    {
      "discovery_id": "...",
      "category": "SANCTIONS|MONEY_LAUNDERING|FRAUD|...",
      "status": "PROBABLE|CONFIRMED|UNRESOLVED",
      "evidence_level": "L0|L1|L2|L3|L4",
      "severity_score_current": 0.0-1.0,
      "confidence": 0.0-1.0,
      "claim_summary": "...",
      "target_scope": "seed_direct|board_member|ubo|subsidiary",
      "evidence_node_ids": ["..."],
      "contradiction_node_ids": ["..."],
      "score_components": {
        "base_severity": 0.0-1.0,
        "veracity": 0.0-1.0,
        "claim_state_weight": -1.0-1.0,
        "role_factor": 0.0-1.0,
        "industry_modifier": 0.0-2.0,
        "temporal_decay": 0.0-1.0,
        "independence_factor": 1.0-2.0,
        "stock_uplift": 0.0-0.2,
        "sanctions_uplift": 0.0-0.2
      },
      "provenance": [
        {
          "clue_id": "...",
          "document_id": "...",
          "source_url": "...",
          "span_start": 0,
          "span_end": 0
        }
      ]
    }
  ],
  "company_score": {
    "current": 0.0-1.0,
    "historical_time_series": [],
    "dominant_risk_categories": ["..."],
    "top_drivers": ["..."]
  }
}
```
