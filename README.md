# Version-Aware Technical Documentation RAG

A proposed retrieval assistant for support engineers answering installation, connector, and configuration questions from product manuals and troubleshooting guides.

**Repository status:** system design and evaluation specification. This repository currently contains documentation; ingestion code, an index, a running assistant, and measured results are not included.

## The problem

Technical support questions often need several pieces of evidence. An error-code explanation may identify the cause, while connector documentation supplies the remedy and a compatibility page limits which versions it applies to. Retrieving one relevant sentence can leave the answer incomplete; supplying entire manuals can exhaust the context budget.

The proposed system combines precise passage retrieval with recovery of the surrounding section and explicit version handling. TDengine's public documentation provides the intended example domain. Private customer tickets, internal manuals, and production traffic are outside the proposed dataset.

## Proposed retrieval pipeline

**Preserve document structure.** LlamaIndex would organize HTML or Markdown while retaining heading paths, code blocks, tables, source URLs, and version information. Parent sections would start around 800–1,200 tokens. Within each parent, 256-token child windows with 64-token overlap would provide finer retrieval targets, with boundaries adjusted to preserve meaningful text and code.

**Combine exact and semantic matches.** Elasticsearch BM25 would handle error codes, configuration keys, and version strings. BGE-M3 dense embeddings would support semantically similar descriptions. Each route would retrieve 20 candidates; reciprocal rank fusion would combine their rankings before `bge-reranker-v2-m3` selects five child passages. This design uses BGE-M3's dense output, with BM25 supplied separately by Elasticsearch.

**Recover sufficient context.** Selected children would point back to their parent sections. Duplicate parents would be removed, and relevant sections would be assembled within a fixed token budget. Evidence coverage would be measured after this assembly and any truncation.

**Respect applicability.** Known product versions, languages, and connection modes would filter candidate evidence. Missing critical details would prompt clarification. Page update dates would not automatically be treated as product versions, and documents with unknown applicability would retain that uncertainty.

**Answer with traceable support.** Qwen3-8B is the proposed generator. Answers would explain the cause, steps, applicable conditions, and section references. Citation checks would examine whether the cited material supports the associated instruction. When evidence is insufficient or incompatible, the assistant would explain what remains unresolved rather than supply unsupported operations.

## Evaluation design

The proposed held-out set contains 300 questions: 240 answerable from the selected documentation and 60 without sufficient evidence. Development and evaluation would be separated by question family so that paraphrases of the same problem stay together.

A dense-only baseline would use fixed chunks, BGE-M3 retrieval, and five direct retrieval results. The full design would add hybrid retrieval, reranking, parent recovery, version handling, and evidence-based abstention. Both would use the same corpus snapshot, generation settings, and final context budget.

| Metric | Required judgment |
|---|---|
| Complete-evidence coverage | Final context contains all evidence needed for the answer |
| Correct and complete answers | Applicable version, necessary steps, supported claims, and no substantive errors |
| Appropriate abstention | Insufficient-evidence questions receive a clear, justified refusal |
| False abstention | Answerable questions incorrectly receive a refusal |

Human review would verify reference evidence and answers. Planned ablations would remove BM25 fusion, reranking, parent recovery, or child overlap separately. No performance improvement, latency reduction, or business impact is claimed here.

## References

- [BGE-M3 model card](https://huggingface.co/BAAI/bge-m3)
- [bge-reranker-v2-m3 model card](https://huggingface.co/BAAI/bge-reranker-v2-m3)
