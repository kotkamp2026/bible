# ARCHITECTURE EXPANSION & IMPLEMENTATION INDEX

This split captures cross-cutting implementation doctrine and artifact mapping retained from the full Codex architecture bible.

## Optimization Target

- Maximize challenge score by centering `article-risk inference`, `industry-aware sentiment`, `entity unification`, `historical scoring`, `demo-grade explainability`.
- Treat crawling as replaceable infrastructure; treat `scoring logic + evidence traceability` as the product core.
- Use `risk sentiment`, not generic polarity. Positive/neutral/negative news classification is insufficient for AML/DD.

## Canonical Runtime Objects

- `Case`: investigation container.
- `SeedProfile`: canonical investigated entity + aliases + board + UBO + subsidiaries + industry.
- `Clue`: normalized output from the external Scraper Module.
- `Graph`: directed property graph with provenance on all assertions.
- `Discovery`: evidence-backed probable/confirmed infraction or risk event.
- `TimelinePoint`: bucketed reputation score and delta.

## Investigation Control Rule

- Start with `seed aliases + board members + UBOs + subsidiaries + search vectors`.
- Every returned clue receives `Potential_Score(c) ∈ [0,1]`.
- Re-queue only if `Potential_Score(c) ≥ InvestigationConfig.threshold` and `depth < max_search_depth`.
- Terminate branch on low potential, duplicate dominance, or depth exhaustion.

## Evidence Truth Policy

- `Direct evidence` > `role-linked evidence` > `latent association`.
- `Official/legal/sanctions evidence` dominates conflicting media evidence.
- Mentions without actor-role alignment do not directly score the company.
- Allegation, investigation, charge, sanction, conviction, denial, acquittal, settlement are distinct claim states with different weights and decay.

## Hackathon-Fit Build Ordering

1. `Entity resolution + SeedProfile`
2. `Context-aware article risk scoring`
3. `Explainable reputation score + history`
4. `Graph provenance + discovery synthesis`
5. `Bonus enrichments`: sanctions, stock correlation, continuous crawling

## Artifact Map

- `01_executive_summary_and_topology.md`: doctrine, topology, async workflow.
- `02_seed_genesis_unification.md`: seed contract, `SeedConfig`, entity resolution.
- `03_targeted_investigation_scraper.md`: external scraper API, `InvestigationConfig`, clue triage.
- `04_graph_assembler_provenance.md`: graph contracts, provenance, latent edge logic.
- `05_discovery_engine_scoring.md`: `DiscoveryConfig`, discovery scoring, time-series, MCP prompt.
- `architecture_bible.md`: combined full Codex document.
