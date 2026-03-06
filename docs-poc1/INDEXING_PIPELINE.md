# GraphRAG — Indexing Pipeline

The indexing pipeline is a sequenced set of **workflow steps** orchestrated by `PipelineFactory` and executed by `run_pipeline()`. Each step reads from and writes to a shared `PipelineRunContext` (storage + cache + callbacks).

---

## Standard Pipeline — Full Workflow Sequence

```mermaid
flowchart TD
    A([Raw Documents\ntxt / csv / json / jsonl / docx / pdf]) --> W1

    W1["① load_input_documents\n─────────────────────\nInput:  file storage\nOutput: documents table"]

    W1 --> W2["② create_base_text_units\n─────────────────────────\nInput:  documents table\nOutput: text_units table\n(chunked)"]

    W2 --> W3["③ create_final_documents\n─────────────────────────\nInput:  documents table + text_units\nOutput: final_documents table"]

    W3 --> W4["④ extract_graph  ← LLM\n────────────────────────────\nInput:  text_units\nOutput: raw_entities, raw_relationships"]

    W4 --> W5["⑤ finalize_graph\n────────────────────────────\nInput:  raw entities + relationships\nOutput: entities table\n        relationships table\n        (merged & de-duplicated)"]

    W5 --> W6["⑥ extract_covariates  ← LLM (optional)\n─────────────────────────────────────────\nInput:  text_units + entities\nOutput: covariates table (claims / facts)"]

    W5 --> W7["⑦ create_communities\n──────────────────────────────\nInput:  relationships table\nOutput: communities table\n(Leiden hierarchical clustering)"]

    W6 & W7 --> W8["⑧ create_final_text_units\n──────────────────────────────\nInput:  text_units + entities\n        + relationships + covariates\nOutput: final_text_units table\n(with graph cross-refs)"]

    W7 --> W9["⑨ create_community_reports  ← LLM\n──────────────────────────────────────\nInput:  communities + entities\n        + relationships + text context\nOutput: community_reports table"]

    W8 & W9 --> W10["⑩ generate_text_embeddings  ← Embedding Model\n──────────────────────────────────────────────────\nInput:  entities / relationships / communities\n        / text_units / community_reports\nOutput: vector store indices"]

    W10 --> Z([Persisted Index\nParquet tables + Vector Store])
```

---

## Fast Pipeline — NLP-Based Graph Extraction

```mermaid
flowchart TD
    A([Raw Documents]) --> F1
    F1["① load_input_documents"] --> F2
    F2["② create_base_text_units"] --> F3
    F3["③ create_final_documents"] --> F4

    F4["④ extract_graph_nlp  ← SpaCy / Regex\n──────────────────────────────────────\nInput:  text_units\nOutput: raw noun-phrase graph\n        (no LLM calls)"] --> F5

    F5["⑤ prune_graph\n───────────────────────\nInput:  raw NLP graph\nOutput: pruned graph\n(removes low-weight edges)"] --> F6

    F6["⑥ finalize_graph"] --> F7
    F7["⑦ create_communities"] --> F8
    F8["⑧ create_final_text_units"] --> F9

    F9["⑨ create_community_reports_text  ← LLM\n(text-only; no entity descriptions to summarize)"] --> F10

    F10["⑩ generate_text_embeddings"] --> Z([Persisted Index])
```

---

## Stage 1 — Document Loading (`load_input_documents`)

```mermaid
flowchart LR
    subgraph IN["Input Storage (file / blob / memory)"]
        TXT[".txt files"]
        CSV[".csv files"]
        JSON[".json / .jsonl"]
        RICH["Rich formats\n(docx, pdf, xlsx…\nvia MarkItDown)"]
    end

    subgraph READER["graphrag-input  InputReaderFactory"]
        TR["TextInputReader"]
        CR["CSVInputReader"]
        JR["JsonInputReader"]
        MR["MarkItDownInputReader"]
    end

    subgraph OUT["Output"]
        DOCS["documents DataFrame\n─────────────────\nid, title, text,\nmetadata columns"]
    end

    TXT --> TR --> OUT
    CSV --> CR --> OUT
    JSON --> JR --> OUT
    RICH --> MR --> OUT
```

**Config keys:** `input.type`, `input.file_pattern`, `input.file_encoding`, `input.text_column`, `input.title_column`, `input_storage.type / base_dir / connection_string / container_name`

---

## Stage 2 — Text Chunking (`create_base_text_units`)

```mermaid
flowchart LR
    DOCS["documents\n(text column)"] --> CHUNKER

    subgraph CHUNKER["graphrag-chunking  ChunkerFactory"]
        TC["TokenChunker\n(tiktoken tokens)"]
        SC["SentenceChunker\n(NLTK sentences)"]
    end

    TC & SC --> TU["text_units DataFrame\n──────────────────────\nid, document_id, text,\nn_tokens, chunk_index"]

    subgraph CFG["Config: chunking"]
        C1["type: tokens | sentence"]
        C2["size: 1200 (tokens)"]
        C3["overlap: 100 (tokens)"]
        C4["encoding_model: cl100k_base"]
        C5["prepend_metadata: bool"]
    end
```

---

## Stage 3 — Graph Extraction (`extract_graph`)

```mermaid
flowchart TD
    TU["text_units"] --> GE

    subgraph GE["GraphExtractor  ← LLM"]
        P["Custom entity-extraction prompt\n(entity_types list injected)"]
        GL["max_gleanings loop\n(repeat until LLM finds nothing new)"]
        P --> GL
    end

    GL --> ENT["raw_entities DataFrame\n─────────────────────\nname, type, description,\nsource_id"]
    GL --> REL["raw_relationships DataFrame\n──────────────────────────\nsource, target, description,\nweight, source_id"]

    subgraph CFG["Config: extract_graph"]
        E1["completion_model_id"]
        E2["entity_types list"]
        E3["max_gleanings: 1"]
        E4["num_threads: 50"]
        E5["async_mode: asyncio | threaded"]
        E6["prompt (custom Jinja2 template)"]
    end
```

---

## Stage 3-alt — NLP Graph Extraction (`extract_graph_nlp`)

```mermaid
flowchart TD
    TU["text_units"] --> NLP

    subgraph NLP["NLP Extractors (no LLM)"]
        RE["RegexEnglish\n(fast, English-only)"]
        SYN["SyntacticParser\n(SpaCy dep-parse + NER)"]
        CFG_NLP["CFG\n(chunk grammar + NER)"]
    end

    NLP --> NOUNGRAPH["noun-phrase graph\n(entities = noun phrases,\nrelationships = co-occurrence)"]

    subgraph PRUNE["prune_graph"]
        PR1["min_node_freq threshold"]
        PR2["min_edge_weight_pct threshold"]
        PR3["max_node_freq threshold"]
    end

    NOUNGRAPH --> PRUNE --> ENT2["pruned entities\n+ relationships"]
```

---

## Stage 4 — Graph Finalization (`finalize_graph`)

```mermaid
flowchart LR
    ENT["raw_entities"] & REL["raw_relationships"] --> FIN

    subgraph FIN["finalize_graph"]
        MERGE["merge duplicate\nentity names"]
        SUMM["summarize_descriptions  ← LLM\n(if multiple descriptions exist)"]
        COMPD["compute_edge_combined_degree"]
        MERGE --> SUMM --> COMPD
    end

    COMPD --> ENTS_OUT["entities table\n──────────────\nid, name, type,\ndescription, rank"]
    COMPD --> RELS_OUT["relationships table\n────────────────────\nid, source, target,\ndescription, weight,\ncombined_degree"]

    subgraph CFG2["Config: summarize_descriptions"]
        SD1["completion_model_id"]
        SD2["max_summary_length: 500"]
        SD3["num_threads: 50"]
        SD4["prompt (custom template)"]
    end
```

---

## Stage 5 — Community Detection (`create_communities`)

```mermaid
flowchart LR
    RELS["relationships\ntable"] --> CG

    subgraph CG["cluster_graph"]
        LCC["Largest Connected Component\n(use_lcc flag)"]
        LEIDEN["Hierarchical Leiden\nalgorithm"]
        LCC --> LEIDEN
    end

    LEIDEN --> COMM["communities table\n──────────────────────\nid, level, parent,\ntitle, entity_ids"]

    subgraph CFG3["Config: cluster_graph"]
        CL1["max_cluster_size: 10"]
        CL2["use_lcc: true"]
        CL3["seed: int (reproducibility)"]
    end
```

---

## Stage 6 — Community Reports (`create_community_reports`)

```mermaid
flowchart TD
    COMM["communities"] & ENTS["entities"] & RELS["relationships"] & TU2["text_units"] --> CTX

    subgraph CTX["Context Builder"]
        MIX["build_mixed_context\n(graph context + text unit context)"]
        EXP["explode_communities\n(add sub-community nodes)"]
    end

    CTX --> CR_EXT

    subgraph CR_EXT["CommunityReportsExtractor  ← LLM"]
        PROMPT["community_report prompt\n(rating + summary + findings)"]
    end

    CR_EXT --> CREP["community_reports table\n──────────────────────────\nid, community, level,\ntitle, summary, full_content,\nrank, rating, findings[]"]

    subgraph CFG4["Config: community_reports"]
        R1["completion_model_id"]
        R2["max_input_length: 8000 tokens"]
        R3["max_length: 2000 tokens"]
        R4["num_threads: 50"]
        R5["prompt (custom template)"]
    end
```

---

## Stage 7 — Covariate Extraction (`extract_covariates`, optional)

```mermaid
flowchart LR
    TU3["text_units"] & ENTS2["entities"] --> COVE

    subgraph COVE["CovariateExtractor  ← LLM"]
        CP["claim extraction prompt\n(claim_description, claim_type injected)"]
    end

    COVE --> COV["covariates table\n───────────────────────\nid, subject_id, subject_type,\nobject_id, claim_type,\nclaim_status, text, source_id"]

    subgraph CFG5["Config: extract_claims"]
        CC1["enabled: false (default)"]
        CC2["completion_model_id"]
        CC3["description: str"]
        CC4["max_gleanings: 1"]
        CC5["prompt (custom template)"]
    end
```

---

## Stage 8 — Text Embeddings (`generate_text_embeddings`)

```mermaid
flowchart LR
    subgraph SOURCES["Embeddable fields"]
        E1["entity.description"]
        E2["relationship.description"]
        E3["community.title + summary"]
        E4["community_report.full_content"]
        E5["text_unit.text"]
        E6["document.text (optional)"]
    end

    SOURCES --> EMB["graphrag-llm\nEmbeddingModel\n(LiteLLM facade)"]
    EMB --> VS["Vector Store\n────────────────────\nLanceDB (local default)\nAzure AI Search\nCosmosDB (NoSQL)"]

    subgraph CFG6["Config: embed_text / vector_store"]
        V1["names: list of fields to embed"]
        V2["batch_size: 16"]
        V3["batch_max_tokens: 8192"]
        V4["vector_store.type: lancedb | azure_ai_search | cosmosdb"]
        V5["vector_store.db_uri (LanceDB path)"]
        V6["vector_store.vector_size: 1536"]
    end
```

---

## Incremental Update Pipeline

```mermaid
flowchart TD
    NEW["New Documents"] --> LUD["load_update_documents"]
    LUD --> DELTA_IDX["Run Standard/Fast\nworkflows on delta only"]
    DELTA_IDX --> DELTA_STORE["delta storage\n(timestamped)"]

    PREV["Previous Index\n(backup copy)"] --> MERGE

    DELTA_STORE --> MERGE["Merge workflows\n─────────────────────\nupdate_final_documents\nupdate_entities_relationships\nupdate_text_units\nupdate_covariates\nupdate_communities\nupdate_community_reports\nupdate_text_embeddings\nupdate_clean_state"]

    MERGE --> OUTPUT["Updated full index"]
```

---

## LLM Middleware Stack (all LLM calls)

```mermaid
flowchart LR
    CALL["LLM call"] --> MW

    subgraph MW["graphrag-llm Middleware Pipeline"]
        LOG["with_logging\n(debug traces)"]
        MET["with_metrics\n(token counts)"]
        RL["with_rate_limiting\n(TPM / RPM)"]
        CACHE2["with_cache\n(content hash key)"]
        RETRY["with_retries\n(exponential backoff)"]
        CNT["with_request_count"]
        LOG --> MET --> RL --> CACHE2 --> RETRY --> CNT
    end

    CNT --> LLM2["LiteLLM\n(OpenAI / Azure / Ollama\n/ any compatible model)"]
    LLM2 --> CNT
```
