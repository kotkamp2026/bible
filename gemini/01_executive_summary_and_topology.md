# EXECUTIVE ARCHITECTURE SUMMARY

## 1.1 CORE PURPOSE & SYSTEM DOCTRINE
This AML/Due Diligence investigation system operates as a **Decoupled Intelligence & Acquisition Pipeline**. It maps entity lineage, synthesizes multi-lingual OSINT (focusing on Polish context and strict regulatory frameworks), identifies anomalous correlations, and computes algorithmic severity scores. 

## 1.2 ARCHITECTURE PARADIGM
- **Asynchronicity Base:** The core AI Pipeline (Graph Assembler + Discovery Engine) is strictly decoupled from the web-scraping/OSINT data acquisition modules via HTTP webhooks and queuing systems.
- **Topological Data Backbone:** All data (seeds, text clues, discoveries) is mapped onto a **Directed Property Graph** enabling multihop semantic reasoning.
- **Bipartite Metric Yield:**
  - `Severity_Score` ($S_t \in [0, 1]$) – Instantaneous localized risk per edge.
  - `Reputation Vector` ($\mathbf{R} = [R_{t-n}, \dots, R_t]$) – Temporally decayed Bayesian time-series index.

## 1.3 SYSTEM TOPOLOGY & AGENT WORKFLOW
```mermaid
sequenceDiagram
    participant UI as API Gateway
    participant CP as Core Pipeline (Orchestrator)
    participant UN as Unification Engine (NLP)
    participant SC as Scraper Worker Queue (External Module)
    participant GA as Graph Assembler (Graph Database)
    participant MX as Embedding Semantic Matrix
    participant DE as Discovery Engine (MCP / LLM)
    
    UI->>CP: Initialize(NIP / Entity Name)
    CP->>UN: Dispatch seed resolution
    UN-->>CP: UnifiedEntity[Aliases, UBOs, NIPs]
    CP->>GA: Initialize Graph [Node: Seed]
    
    loop Depth-Bounded Investigation Loop (Phase 2)
        CP->>SC: POST /api/v1/gather/osint (Payload: Keywords, Entity)
        SC-->>CP: 202 Accepted {task_id}
        SC-->>CP: POST /webhook/callback [Raw Clues]
        CP->>CP: Evaluate Potential_Score(Clues) thresholds
        opt Potential_Score > \theta_{thresh} & depth < MAX_DEPTH
            CP->>SC: Re-queue [Depth++]
        end
        CP->>GA: Ingest Payload -> [Node: Clue]
        GA->>MX: Trigger vectorization of Clue Node abstract
        MX-->>GA: Semantic Vectors [High-d]
        GA->>GA: Draw Latent Edges (Seed <-> Clue) using Vector Proximity
    end
    
    CP->>DE: Trigger Discovery Phase (Graph Traversal Payload)
    DE->>GA: Query Path Topologies & Lineage Provenance
    DE->>DE: Synthesize Sub-graphs & Compute Discoveries
    DE->>DE: Compute Temporal Reputation Trend Vector
    DE-->>CP: Return { Discoveries[], R_vector }
    CP-->>UI: Complete JSON Final Report
```
