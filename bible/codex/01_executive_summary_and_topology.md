# EXECUTIVE ARCHITECTURE SUMMARY & TOPOLOGY

Codex split of the full architecture bible sectioning executive doctrine, topology, and async workflow.

# 1. EXECUTIVE ARCHITECTURE SUMMARY

| Phase | Purpose | Primary Input | Primary Output | Hard Constraint |
| --- | --- | --- | --- | --- |
| `SEED_GENESIS` | Resolve the investigated company into a canonical seed entity | `NIP` or fuzzy company name | `SeedProfile` + ranked candidate set + ambiguity state | deterministic ID match first, fuzzy only after normalization/alias expansion |
| `TARGETED_INVESTIGATION_LOOP` | Collect scope-bound OSINT/legal/sanctions/registry clues and recursively expand only high-value branches | `SeedProfile`, `InvestigationConfig` | normalized `Clue[]` | AI pipeline never touches raw HTML/PDF; only external Scraper Module does |
| `GRAPH_ASSEMBLER` | Convert clues into a provenance-preserving directed property graph | `Clue[]` | typed `Node[]`, `Edge[]`, `source_parents[]` | every node/edge/discovery must trace to source spans |
| `DISCOVERY_ENGINE` | Synthesize evidence into explainable AML/DD discoveries and company reputation score timeline | seed subgraph + official enrichments + market signals + `DiscoveryConfig` | `Discoveries[]`, `current_score`, `historical_time_series[]`, report bundle | no source-free inference; unresolved links remain `latent` |

**Optimization target**

- Maximize challenge score by centering `article-risk inference`, `industry-aware sentiment`, `entity unification`, `historical scoring`, `demo-grade explainability`.
- Treat crawling as replaceable infrastructure; treat `scoring logic + evidence traceability` as the product core.
- Use `risk sentiment`, not generic polarity. Positive/neutral/negative news classification is insufficient for AML/DD.

**Canonical runtime objects**

- `Case`: investigation container.
- `SeedProfile`: canonical investigated entity + aliases + board + UBO + subsidiaries + industry.
- `Clue`: normalized output from the external Scraper Module.
- `Graph`: directed property graph with provenance on all assertions.
- `Discovery`: evidence-backed probable/confirmed infraction or risk event.
- `TimelinePoint`: bucketed reputation score and delta.

**Entity-resolution decision rule**

\[
ER(q,e)=w_{nip}N+w_{exact}X+w_{alias}A+w_{acro}C+w_{board}B+w_{ubo}U+w_{industry}I+w_{jurisdiction}J-w_{homonym}H
\]

Default weights for seed resolution:

\[
(w_{nip},w_{exact},w_{alias},w_{acro},w_{board},w_{ubo},w_{industry},w_{jurisdiction},w_{homonym})=(0.45,0.20,0.10,0.05,0.06,0.04,0.04,0.06,0.20)
\]

Acceptance rule:

\[
accept(e^\*) \iff ER(q,e^\*) \ge T_{accept} \land \left(ER(q,e^\*)-ER(q,e_2)\right)\ge T_{margin}
\]

where `e2` is runner-up candidate. If false, case enters `AMBIGUOUS` state.

**Investigation control rule**

- Start with `seed aliases + board members + UBOs + subsidiaries + search vectors`.
- Every returned clue receives `Potential_Score(c) ∈ [0,1]`.
- Re-queue only if `Potential_Score(c) ≥ InvestigationConfig.threshold` and `depth < max_search_depth`.
- Terminate branch on low potential, duplicate dominance, or depth exhaustion.

**Evidence truth policy**

- `Direct evidence` > `role-linked evidence` > `latent association`.
- `Official/legal/sanctions evidence` dominates conflicting media evidence.
- Mentions without actor-role alignment do not directly score the company.
- Allegation, investigation, charge, sanction, conviction, denial, acquittal, settlement are distinct claim states with different weights and decay.

**Hackathon-fit build ordering**

1. `Entity resolution + SeedProfile`
2. `Context-aware article risk scoring`
3. `Explainable reputation score + history`
4. `Graph provenance + discovery synthesis`
5. `Bonus enrichments`: sanctions, stock correlation, continuous crawling

# 2. SYSTEM TOPOLOGY & AGENT WORKFLOW

```mermaid
sequenceDiagram
    autonumber
    actor User
    participant Intake as Case Intake
    participant Seed as SEED_GENESIS
    participant Registry as Entity Registry
    participant Orchestrator as TARGETED_INVESTIGATION_LOOP
    participant Scraper as External Scraper API
    participant Webhook as Task Webhook Receiver
    participant Normalizer as Clue Normalizer
    participant Graph as GRAPH_ASSEMBLER
    participant Connector as Connector Agent
    participant Gov as Gov/Registry/Sanctions APIs
    participant Market as Market Signals API
    participant Discover as DISCOVERY_ENGINE (MCP)
    participant Score as Reputation Scorer
    participant Report as Final Report Composer

    User->>Intake: submit {nip|fuzzy_name, SeedConfig, InvestigationConfig, DiscoveryConfig}
    Intake->>Seed: initialize_case()
    Seed->>Registry: resolve_entity(input, alias_rules, jurisdiction_scope)
    Registry-->>Seed: candidate_entities[] + ER scores + ambiguity flags

    alt unique high-confidence candidate
        Seed-->>Intake: SeedProfile
    else homonym/alias collision
        Seed-->>Intake: ranked candidates + rationale
        Intake-->>User: disambiguation request
        User->>Intake: candidate confirmation
        Intake->>Seed: finalize_seed(candidate_id)
    end

    Intake->>Orchestrator: start(seed_profile, investigation_config)

    par OSINT gather
        Orchestrator->>Scraper: POST /api/v1/gather/osint
        Scraper-->>Orchestrator: 202 Accepted {task_id, poll_url}
    and Legal gather
        Orchestrator->>Scraper: POST /api/v1/gather/legal
        Scraper-->>Orchestrator: 202 Accepted {task_id, poll_url}
    and Registry gather
        Orchestrator->>Scraper: POST /api/v1/gather/registry
        Scraper-->>Orchestrator: 202 Accepted {task_id, poll_url}
    and Sanctions gather
        Orchestrator->>Scraper: POST /api/v1/gather/sanctions
        Scraper-->>Orchestrator: 202 Accepted {task_id, poll_url}
    end

    Note over Orchestrator,Scraper: Async Boundary A: crawl/fetch/HTML-PDF parse isolated outside AI pipeline

    alt callback configured
        Scraper-->>Webhook: POST task.event {queued|running|partial|completed|failed}
        Webhook->>Orchestrator: task state update
    else polling fallback
        loop until task terminal
            Orchestrator->>Scraper: GET /api/v1/task/{task_id}
            Scraper-->>Orchestrator: status + progress + counters
        end
    end

    loop completed task result pages
        Orchestrator->>Scraper: GET /api/v1/task/{task_id}/results?cursor=...
        Scraper-->>Orchestrator: CluePage {clues[], next_cursor}
        Orchestrator->>Normalizer: normalize_language() + dedupe() + score_potential()
        Normalizer-->>Orchestrator: normalized clues + Potential_Score features

        loop branch expansion while potential >= threshold and depth < max_depth
            Orchestrator->>Scraper: POST /api/v1/gather/osint {parent_clue_ids, expanded_queries, depth+1}
            Scraper-->>Orchestrator: 202 Accepted {child_task_id, parent_task_id}
        end
    end

    Orchestrator->>Graph: upsert_clues_as_nodes_edges()
    Graph->>Connector: infer_latent_links(seed_subgraph)
    Connector-->>Graph: latent edges + confidence + validation_required=true

    par official enrichment
        Discover->>Gov: fetch court/registry/sanctions corroboration
        Gov-->>Discover: official evidence objects
    and market enrichment
        Discover->>Market: fetch event-window price anomalies
        Market-->>Discover: abnormal returns + volatility context
    end

    Note over Graph,Discover: Async Boundary B: evidence synthesis and score computation

    Discover->>Graph: request provenance-complete seed subgraph
    Graph-->>Discover: nodes + edges + source_parents + latent links
    Discover->>Score: compute discovery severities + company timeline
    Score-->>Discover: current_score + historical_time_series + score drivers
    Discover->>Report: compose Discoveries[] + evidence map + contradictions + trend data
    Report-->>Intake: investigation dossier
    Intake-->>User: final report + score history + explainability trace
```
