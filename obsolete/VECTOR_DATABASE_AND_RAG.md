# Vector Database and RAG Capabilities

Status: draft
Last updated: 2026-03-27

## Vision

TruthDB is a hybrid data system. Beyond its WAL-centric event-sourcing core, full-text search engine, and relational capabilities, TruthDB should natively support **vector embeddings** and **similarity search** as first-class data types and query operations.

This positions TruthDB as a single platform that can serve structured queries, full-text search, and semantic similarity retrieval without requiring external vector databases (Pinecone, Weaviate, Milvus, Qdrant, etc.) or bolt-on extensions (pgvector). For AI/ML workloads — particularly Retrieval-Augmented Generation (RAG) — this eliminates an entire infrastructure dependency and keeps all data under one durability, consistency, and replay model.

## Why this belongs in TruthDB

1. **Hybrid by design**: The capability catalogue already envisions hybrid retrieval across structured, textual, and historical sources (F-032). Vector similarity is the missing third retrieval modality alongside exact-match and full-text search.

2. **WAL-native embeddings**: Embedding vectors are derived data, just like search indexes. They fit naturally into TruthDB's model: the WAL is the source of truth, vectors are rebuildable projections. If an embedding model changes, vectors can be recomputed from the authoritative log.

3. **Single consistency boundary**: RAG pipelines today stitch together a primary database, a search engine, and a vector store — each with its own consistency guarantees. TruthDB can serve all three from a single transactional boundary, eliminating stale-vector bugs and sync failures.

4. **Event-sourced embedding lineage**: Every vector stored in TruthDB traces back to a WAL event. This gives operators full provenance: which document version produced which embedding, with which model, at which point in the event stream.

## Capability domain: Vector storage, indexing, and retrieval

This would extend the capability catalogue as a new domain (proposed: domain 10, feature IDs F-063 to F-070).

### F-063 Dense vector field type

TruthDB should support storing dense floating-point vectors as a native field type in document mappings and schemas. Vectors should be stored alongside the documents they describe, not in a separate system.

**Acceptance criteria**

- A mapping can declare a field of type `dense_vector` with a specified dimensionality (e.g., 768, 1024, 1536).
- The system validates vector dimensionality on write and rejects mismatched vectors with a clear error.
- Vectors are stored in the WAL as part of the document event, ensuring they are covered by the same durability and replay guarantees as all other fields.
- Vectors can coexist with text, keyword, float, and other field types in the same document.
- Storage format supports common floating-point precisions (f32, f16, binary quantization as future options).

### F-064 Approximate nearest neighbor (ANN) index

TruthDB should build and maintain an ANN index over dense vector fields to enable sub-linear similarity search. The index is a derived structure, rebuildable from the WAL.

**Acceptance criteria**

- An ANN index can be created on a `dense_vector` field with a specified distance metric (cosine, dot product, euclidean).
- The index supports incremental updates as new documents are inserted, without requiring a full rebuild.
- Index builds are observable: progress, memory usage, and estimated completion are exposed.
- The index can be rebuilt from scratch (e.g., after model change) using existing WAL replay or snapshot + replay.
- Index corruption or divergence from source data is detectable and repairable.
- The system documents recall/precision trade-offs for the chosen ANN algorithm.

### F-065 k-nearest-neighbor (kNN) search

TruthDB should support querying for the k most similar vectors to a given query vector, returning the associated documents ranked by similarity score.

**Acceptance criteria**

- A search request can include a `knn` clause specifying the target field, query vector, and k.
- Results include similarity scores alongside document payloads.
- The query can combine kNN with structured filters (pre-filtering or post-filtering).
- Exact brute-force kNN is available as a fallback or for small datasets where ANN index overhead is unjustified.
- Query latency and recall metrics are exposed per query for operational tuning.

### F-066 Hybrid search: vector + full-text + structured

TruthDB should support queries that combine vector similarity, full-text relevance, and structured predicates in a single request. This is the core differentiator for a hybrid database in RAG and AI-assisted search scenarios.

**Acceptance criteria**

- A single query can combine a `knn` vector clause, a `match` full-text clause, and structured `term`/`range` filters.
- Score fusion strategies (reciprocal rank fusion, weighted sum, or custom) are configurable.
- The response model is clear about which score components came from which retrieval path.
- The query planner can explain the execution strategy for hybrid queries.
- Performance of hybrid queries is benchmarkable and cost drivers (vector scan, text scan, filter) are attributable.

### F-067 Embedding lifecycle and model versioning

Because embedding models evolve, TruthDB should track which model produced which vectors and support re-embedding workflows without data loss.

**Acceptance criteria**

- Vector fields can carry metadata about the embedding model (name, version, dimensionality).
- A re-embedding operation can update vectors for existing documents without deleting the source documents.
- During re-embedding, the old and new vector indexes can coexist (similar to reindex-and-cutover, F-030).
- The system can report what percentage of vectors in an index were produced by each model version.
- Re-embedding progress is observable and resumable after interruption.

### F-068 Quantization and storage efficiency

High-dimensional vectors are storage-intensive. TruthDB should support quantization and compression techniques to reduce memory and disk footprint while preserving acceptable recall.

**Acceptance criteria**

- Scalar quantization (f32 to f16 or int8) is supported at index build time.
- Binary quantization is available for high-dimensional models where recall trade-offs are acceptable.
- Product quantization (PQ) or similar techniques are available as a future option.
- The system reports actual storage consumption per vector field and index.
- Recall benchmarks comparing quantized vs. full-precision indexes are reproducible using built-in tooling.

### F-069 RAG retrieval pipeline support

TruthDB should provide primitives that make building RAG pipelines straightforward without requiring external orchestration for the retrieval step.

**Acceptance criteria**

- A retrieval query can return document chunks with surrounding context, not just top-level document IDs.
- Chunk boundaries (e.g., by paragraph, sentence, or fixed token window) can be defined at index time or query time.
- Results include enough metadata (source document, chunk position, similarity score) for the LLM prompt assembly step.
- The retrieval API is stateless and composable with any LLM framework (LangChain, LlamaIndex, custom).
- Retrieval latency targets are documented: p99 under 50ms for typical RAG workloads (1M chunks, 1536-dim).

### F-070 Vector-aware replication and snapshots

Vector indexes should participate in TruthDB's existing replication, snapshot, and backup infrastructure without special-case handling.

**Acceptance criteria**

- Vector data in the WAL is replicated like any other event payload.
- Snapshots include vector index state or enough metadata to rebuild the index after restore.
- A restored node can serve vector queries once replay or index rebuild completes.
- Replication lag metrics include vector index rebuild progress on followers.
- Backup size estimates account for vector storage and index overhead.

## Architectural notes

### Index algorithm candidates

- **HNSW (Hierarchical Navigable Small World)**: The dominant choice for in-memory ANN. Used by Qdrant, Weaviate, pgvector, Elasticsearch. Good recall, reasonable build time, natural fit for incremental updates.
- **IVF (Inverted File Index)**: Better for very large datasets with disk-based access. Could combine well with TruthDB's existing inverted index infrastructure.
- **DiskANN / Vamana**: Microsoft's disk-optimized ANN graph. Strong fit for TruthDB's O_DIRECT / io_uring storage path since it is designed for SSD random reads.

Initial recommendation: start with HNSW for in-memory workloads, investigate DiskANN for disk-resident large-scale deployments. Both are well-documented and have Rust implementations available.

### Storage integration

Vectors stored in the WAL are part of the document event. The ANN index is a derived structure, similar to how text postings lists are rebuilt from the WAL today. This means:

- `insert document` with a vector field writes the vector bytes to the WAL
- The engine materializes the vector into the ANN index during event application
- On restart, the ANN index is rebuilt from the snapshot + WAL replay (same as text postings)
- Checkpoints can optionally serialize the HNSW graph to the data region for faster recovery

### Distance metrics

| Metric            | Formula                     | Use case                                         |
| ----------------- | --------------------------- | ------------------------------------------------ |
| Cosine similarity | 1 - (a . b) / (\|a\| \|b\|) | Normalized embeddings (OpenAI, Cohere)           |
| Dot product       | a . b                       | Pre-normalized embeddings, maximum inner product |
| Euclidean (L2)    | sqrt(sum((a_i - b_i)^2))    | Spatial data, image features                     |

### Embedding generation

TruthDB does **not** generate embeddings itself. Embedding generation is the client's responsibility (or handled by an ingestion pipeline). TruthDB stores, indexes, and retrieves vectors — it does not run inference. This keeps the database deterministic and avoids coupling to specific ML runtimes.

## Relationship to existing capabilities

| Existing capability                   | Vector extension                                           |
| ------------------------------------- | ---------------------------------------------------------- |
| F-019 Secondary indexes               | ANN index is a new secondary index type                    |
| F-022 Full-text search                | Vector search complements text search in hybrid queries    |
| F-026 Mapping and analyzer management | Vector field type added to mapping declarations            |
| F-027 Query DSL                       | `knn` clause added to the query model                      |
| F-029 Relevance ranking               | Score fusion across vector + text + structured             |
| F-030 Reindex and alias-based cutover | Re-embedding uses same cutover pattern                     |
| F-032 Hybrid retrieval                | Vector search is the key enabler for true hybrid retrieval |

## Success criteria

- TruthDB can store and retrieve 1M 1536-dimensional vectors with p99 kNN query latency under 50ms
- Hybrid queries (vector + text + filter) work in a single round-trip
- Vector data participates in WAL replay, snapshots, and crash recovery with no special handling
- No external vector database dependency required for RAG retrieval workloads
- The capability catalogue is extended with domain 10 (F-063 to F-070)

## Implementation priority

This is a roadmap item, not an immediate implementation target. Prerequisites:

1. Stable snapshot and WAL reclamation (in progress)
2. Multi-connection concurrency improvements
3. Query DSL extensions for compound queries

The vector field type and brute-force kNN can be prototyped early. The ANN index (HNSW) is the larger engineering effort and should follow once the storage and query foundations are solid.
