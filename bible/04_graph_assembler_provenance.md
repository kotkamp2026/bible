# PHASE 3: GRAPH ASSEMBLER & PROVENANCE

## 3.1 DIRECTED PROPERTY GRAPH (DPG) TOPOLOGY
The entire pipeline ingests results into a Directed Property Graph. This isolated context layer guarantees zero hallucinations down the line since the LLM can only query data actively existing as nodes.

### Node & Edge Schemas
```json
{
  "Node_Clue": {
    "type": "NODE",
    "schema": {
      "id": "UUID",
      "source_url": "URI",
      "timestamp_published": "ISO8601",
      "timestamp_ingested": "ISO8601",
      "raw_extract_hash": "SHA256",
      "raw_extract": "TEXT",
      "source_veracity": "FLOAT",
      "potential_score": "FLOAT",
      "lang": "ENUM(PL, EN, RU)"
    }
  },
  "Edge_Mentions": {
    "type": "EDGE",
    "schema": {
      "source": "UUID(Node_Clue)",
      "target": "UUID(Node_Seed)",
      "semantic_proximity": "FLOAT",
      "context_snippet": "STRING",
      "sentiment_polarity": "FLOAT[-1.0, 1.0]"
    }
  }
}
```

## 3.2 SEMANTIC LATENT EDGE CREATION
To solve the issue of missed mentions via pure Regex, the Graph Assembler translates `Node_Clue.raw_extract` into high-dimensional vectors.
- Pipeline passes extract into a structural embedding model. 
- It generates a high-dimensional vector embedding $v_c$.
- It queries the graph vector database for embeddings corresponding to existing `Node_Seed` structures $v_s$.
- **Latent Edge Formulation:** If cosine similarity metrics indicate proximity:
  $$CosineSimilarity(v_c, v_s) > 0.88$$
  The Graph Assembler will instantiate an `Edge_Mentions` mapping, drawing connections when an entity is referenced implicitly.

## 3.3 STRICT PROVENANCE CONTRACT
`Data Sources` $\rightarrow$ `Clues` $\rightarrow$ `Discoveries`.
Every hypothesis constructed by the final pipeline MUST enforce an unbreakable lineage mapping. No inference or risk score can exist without a verifiable topological path terminating at an existing URI point (e.g., a registry document or news URL).
