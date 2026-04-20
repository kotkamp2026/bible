# PHASE 4: DISCOVERY ENGINE & REPUTATION SCORING

## 4.1 THE BRAIN: MCP ARCHITECTURE
This phase employs a massive, generalized LLM operating via Model Context Protocol (MCP) physically interacting directly with the assembled Graph. It synthesizes sub-graphs mapping to identical infraction contexts into unified `Node_Discovery` entities.

## 4.2 `DiscoveryConfig` MATRIX (HIGH FIDELITY METRICS)
```json
{
  "DiscoveryConfig": {
    "industry_sentiment_modifiers": {
      "FINANCE": { "base_risk_multiplier": 1.3, "fraud_boost_modifier": 1.5, "keywords": ["wyłudzenie", "pranie pieniędzy"] },
      "CONSTRUCTION": { "base_risk_multiplier": 1.1, "corruption_boost_modifier": 1.4, "keywords": ["łapówka", "przetarg publiczny"] },
      "REAL_ESTATE": { "base_risk_multiplier": 1.2, "fraud_boost_modifier": 1.6, "keywords": ["oszustwo", "wywłaszczenie"] },
      "IT": { "base_risk_multiplier": 0.8, "fraud_boost_modifier": 1.0, "keywords": [] }
    },
    "source_veracity_weights": {
      "GOV_REGISTRY_PL": 1.0,
      "MINISTRY_OF_FINANCE": 1.0,
      "OFAC_SANCTIONS": 1.0,
      "TIER_1_NEWS_AGENCY": 0.9,
      "TIER_2_LOCAL_MEDIA": 0.6,
      "FRINGE_BLOG_SOCIAL": 0.2
    },
    "temporal_decay_factors": {
      "half_life_days": 1095,
      "minimum_asymptote": 0.15
    },
    "stock_correlation_flags": {
      "alert_on_drop_pct": 15.0,
      "max_lookback_window_days": 7,
      "correlation_decay_modifier": 0.05
    }
  }
}
```

## 4.3 MATHEMATICAL SCORING ALGORITHMS
### 4.3.1 Localized Event Severity ($S_{e,t}$)
For a given `Node_Clue` mapping to an infraction `Node_Discovery`, its temporal severity scales dynamically based on age and trust:
$$
S_{e,t} = \max\left( \text{minimum\_asymptote}, \left[ M_{ind} \times V_{src} \times I_{base} \right] \cdot e^{-\lambda(t - t_0)} \right)
$$
- $M_{ind}$: Industry risk multiplier from `DiscoveryConfig` (e.g., 1.3 for Finance).
- $V_{src}$: Trustworthiness of the URI originating the Clue ($0.2$ to $1.0$).
- $I_{base}$: Subjective infraction severity assigned by the MCP ($0.0$ to $1.0$).
- $\lambda$: Decay constant $\frac{\ln(2)}{1095}$, allowing the infraction's algorithm penalty to decay by half every 3 years.
- $t - t_0$: Days elapsed since publication.

### 4.3.2 Reputation Time-Series Vector ($R_{current}$)
To output the historical trend visualization requested, aggregate $S_{e,t}$ dynamically over historical intervals (e.g., $t_{-12\text{ months}}$ to $t_0$). 
$$
R_{\tau} = \max\left(0.0, 1.0 - \sum_{e \in E} S_{e,\tau}\right) \quad \forall \tau \in \text{Timeline}
$$
(Yields a visualization array trending downwards as timeline risk spikes, e.g.: `[1.0, 1.0, 0.85, 0.65, 0.60]`)

## 4.4 MCP SYSTEM PROMPT: THE SYNTHESIS INSTRUCTION
```text
[SYSTEM: DISCOVERY_ENGINE_ORCHESTRATOR]
INPUT_DATA: Sub-Graph payload mapping [Seeds] <-> [Mentions] <-> [Clues].

OPERATIONAL MATRIX:
1. TRUTH LINEAGE: You cannot invent facts. Every generated Discovery MUST map its `source_nodes` array to explicit `Node_Clue` IDs inside the payload.
2. POLISH LINGUISTIC UNIFICATION: Treat "wyłudził", "wyłudzenie", "wyłudzono" as the identical semantic vector `FRAUD`.
3. BASE SEVERITY EXTRACTION: Assign `I_base` (0.01 - 1.0) assessing absolute risk context. E.g., final court conviction = 1.0, unproven allegation = 0.5.
4. CORRELATION GENERATION: Cross-reference stock data timelines if active flags met.

OUTPUT_SCHEMA (STRICT JSON):
{
  "discoveries": [
    {
      "discovery_id": "UUID",
      "infraction_category": "CORRUPTION",
      "description": "Evidence from investigative article linking Board Member X to public tender bribe...",
      "base_severity_assigned": 0.8,
      "source_nodes": ["clue-uuid-1"]
    }
  ],
  "reputation_vector_computation": {
    "current_score": 0.65,
    "historical_trend": [1.0, 0.95, 0.80, 0.65]
  }
}
```
