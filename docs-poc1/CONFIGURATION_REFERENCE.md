# GraphRAG — Configuration Reference

All configuration lives in a single `settings.yaml` (or `.json`) at the `--root` directory. It maps to the `GraphRagConfig` Pydantic model.

---

## Top-Level Config Structure

```mermaid
graph TB
    CFG["GraphRagConfig\nsettings.yaml"] --> MODELS
    CFG --> STORAGE_GRP
    CFG --> CHUNKING_GRP
    CFG --> INDEXING_GRP
    CFG --> QUERY_GRP
    CFG --> INFRA

    subgraph MODELS["LLM Models"]
        CM["completion_models: dict"]
        EM["embedding_models: dict"]
    end

    subgraph STORAGE_GRP["Storage"]
        IS["input_storage"]
        OS["output_storage"]
        US["update_output_storage"]
        TP["table_provider"]
        VS_CFG["vector_store"]
        CACHE_CFG["cache"]
    end

    subgraph CHUNKING_GRP["Document Ingestion"]
        INPUT_CFG["input"]
        CHK_CFG["chunking"]
    end

    subgraph INDEXING_GRP["Indexing Stages"]
        EG["extract_graph"]
        EGN["extract_graph_nlp"]
        PG["prune_graph"]
        SD["summarize_descriptions"]
        CG_CFG["cluster_graph"]
        EC["extract_claims"]
        CR_CFG["community_reports"]
        ET["embed_text"]
        SNAP["snapshots"]
    end

    subgraph QUERY_GRP["Query Engines"]
        LS_CFG["local_search"]
        GS_CFG["global_search"]
        DR_CFG["drift_search"]
        BS_CFG["basic_search"]
    end

    subgraph INFRA["Infrastructure"]
        ASYNC["async_mode"]
        CONC["concurrent_requests"]
        RPT["reporting"]
        WF["workflows (custom list)"]
    end
```

---

## LLM Model Configuration

```mermaid
graph LR
    subgraph MODEL_KEYS["completion_models / embedding_models\n(keyed by model_id string)"]
        direction TB
        MK1["api: openai | azure | ollama | litellm"]
        MK2["model: gpt-4o | text-embedding-3-small | …"]
        MK3["api_key: env-var or literal"]
        MK4["api_base: custom endpoint URL"]
        MK5["deployment_name (Azure only)"]
        MK6["api_version (Azure only)"]
        MK7["max_tokens: 4096"]
        MK8["temperature: 0.0"]
        MK9["top_p: 1.0"]
        MK10["n: 1  (completions count)"]
        MK11["request_timeout: 180s"]
        MK12["call_args: dict  (extra kwargs)"]
    end

    subgraph RATE_LIMIT["rate_limit"]
        RL1["requests_per_minute: 0 (unlimited)"]
        RL2["tokens_per_minute: 0"]
    end

    subgraph RETRY_CFG["retry"]
        RC1["max_retries: 10"]
        RC2["strategy: exponential | immediate"]
        RC3["backoff_factor: 1.5"]
        RC4["max_retry_wait: 60s"]
    end

    MODEL_KEYS --> RATE_LIMIT & RETRY_CFG
```

---

## Input Configuration

```mermaid
graph LR
    subgraph INPUT_CFG2["input"]
        IN1["type:\ntext | csv | json | jsonl | markitdown"]
        IN2["file_pattern: *.txt"]
        IN3["file_encoding: utf-8"]
        IN4["text_column (csv/json)"]
        IN5["title_column (csv/json)"]
        IN6["metadata_columns: []"]
    end

    subgraph INPUT_STOR["input_storage"]
        IS1["type: file | blob | memory"]
        IS2["base_dir: ./input"]
        IS3["connection_string (blob)"]
        IS4["container_name (blob)"]
        IS5["storage_account_blob_url (Azure)"]
    end
```

---

## Chunking Configuration

```mermaid
graph LR
    subgraph CHK2["chunking"]
        CK1["type: tokens | sentence"]
        CK2["size: 1200\n(tokens or sentences)"]
        CK3["overlap: 100\n(tokens or sentences)"]
        CK4["encoding_model: cl100k_base"]
        CK5["prepend_metadata: false\n(prepend doc metadata to each chunk)"]
    end

    CK1 -->|tokens| T_CHK["TokenChunker\n(tiktoken)"]
    CK1 -->|sentence| S_CHK["SentenceChunker\n(NLTK)"]
```

---

## Graph Extraction Configuration

```mermaid
graph LR
    subgraph EG2["extract_graph  (LLM mode)"]
        direction TB
        EG1["completion_model_id: default_chat"]
        EG2["entity_types:\n[organization, person, geo, event]"]
        EG3["max_gleanings: 1\n(extra LLM passes to find more entities)"]
        EG4["num_threads: 50"]
        EG5["async_mode: asyncio | threaded"]
        EG6["prompt: path/to/custom_prompt.txt"]
    end

    subgraph NLP2["extract_graph_nlp  (NLP mode)"]
        NLP1["extractor_type:\nregex_english | syntactic_parser | cfg"]
        NLP2["num_threads: 50"]
    end

    subgraph PRUNE2["prune_graph  (Fast mode post-NLP)"]
        PR1["min_node_freq: 3"]
        PR2["min_edge_weight_pct: 10"]
        PR3["max_node_freq_std: 3"]
    end
```

---

## Description Summarization

```mermaid
graph LR
    subgraph SD2["summarize_descriptions"]
        SD1["completion_model_id: default_chat"]
        SD2["max_summary_length: 500 chars"]
        SD3["num_threads: 50"]
        SD4["async_mode: asyncio | threaded"]
        SD5["prompt: path/to/custom_prompt.txt"]
    end
```

---

## Community Detection Configuration

```mermaid
graph LR
    subgraph CG2["cluster_graph"]
        CG1["max_cluster_size: 10\n(Leiden max community size)"]
        CG2["use_lcc: true\n(use Largest Connected Component only)"]
        CG3["seed: 0xDEADBEEF\n(reproducibility)"]
    end
```

---

## Community Reports Configuration

```mermaid
graph LR
    subgraph CR2["community_reports"]
        CR1["completion_model_id: default_chat"]
        CR2["max_input_length: 8000 tokens"]
        CR3["max_length: 2000 tokens"]
        CR4["num_threads: 50"]
        CR5["async_mode: asyncio | threaded"]
        CR6["prompt: path/to/custom_prompt.txt"]
    end
```

---

## Covariate (Claims) Configuration

```mermaid
graph LR
    subgraph EC2["extract_claims"]
        EC1["enabled: false  ← disabled by default"]
        EC2["completion_model_id: default_chat"]
        EC3["description: 'Any claims or facts…'"]
        EC4["max_gleanings: 1"]
        EC5["num_threads: 50"]
        EC6["prompt: path/to/custom_prompt.txt"]
    end
```

---

## Embeddings Configuration

```mermaid
graph TB
    subgraph ET2["embed_text"]
        ET1["names: list of fields to embed\n─────────────────────────────────\nentity.description (required for local/drift)\nrelationship.description\ncommunity.title\ncommunity.summary\ncommunity_report.full_content\ntext_unit.text (required for basic/drift)\ndocument.text (optional)"]
        ET2b["batch_size: 16"]
        ET3["batch_max_tokens: 8192"]
        ET4["embedding_model_id: default_embedding"]
        ET5["num_threads: 50"]
    end

    subgraph VS3["vector_store"]
        VS1b["type: lancedb | azure_ai_search | cosmosdb"]
        VS2b["db_uri: ./lancedb  (LanceDB local path)"]
        VS3b["url (Azure AI Search endpoint)"]
        VS4b["api_key (Azure AI Search)"]
        VS5b["audience (Azure Managed Identity)"]
        VS6b["connection_string (CosmosDB)"]
        VS7b["vector_size: 1536"]
        VS8b["index_schema: dict of IndexSchema"]
    end
```

---

## Query Engine Configuration

```mermaid
graph TB
    subgraph LS2["local_search"]
        LS1b["completion_model_id: default_chat"]
        LS2b["embedding_model_id: default_embedding"]
        LS3b["text_unit_prop: 0.5  (token budget for text units)"]
        LS4b["community_prop: 0.1  (token budget for community reports)"]
        LS5b["top_k_entities: 10"]
        LS6b["top_k_relationships: 10"]
        LS7b["max_context_tokens: 12000"]
        LS8b["conversation_history_max_turns: 5"]
    end

    subgraph GS2["global_search"]
        GS1b["completion_model_id: default_chat"]
        GS2b["data_max_tokens: 12000"]
        GS3b["map_max_length: 1000  (partial answer token limit)"]
        GS4b["reduce_max_length: 2000  (final answer token limit)"]
        GS5b["max_context_tokens: 16000"]
        GS6b["dynamic_search_threshold: 0.5"]
        GS7b["dynamic_search_max_level: 3"]
        GS8b["dynamic_search_keep_parent: false"]
        GS9b["dynamic_search_num_repeats: 1"]
        GS10["dynamic_search_use_summary: false"]
    end

    subgraph DR2["drift_search"]
        DR1b["completion_model_id: default_chat"]
        DR2b["embedding_model_id: default_embedding"]
        DR3b["drift_k_followups: 20\n(sub-queries to generate)"]
        DR4b["primer_folds: 10\n(global context splits)"]
        DR5b["n_depth: 1  (search depth)"]
        DR6b["max_context_tokens: 12000"]
        DR7b["local_search_* params\n(inherits local_search defaults)"]
    end

    subgraph BS2["basic_search"]
        BS1b["completion_model_id: default_chat"]
        BS2b["embedding_model_id: default_embedding"]
        BS3b["k: 10  (top-k text units to retrieve)"]
        BS4b["max_context_tokens: 12000"]
    end
```

---

## Storage & Cache Configuration

```mermaid
graph LR
    subgraph OUT_STO["output_storage"]
        OS1["type: file | blob | memory"]
        OS2["base_dir: ./output"]
        OS3["connection_string / container (blob)"]
    end

    subgraph TP2["table_provider"]
        TP1["type: parquet | csv"]
    end

    subgraph CACHE2["cache"]
        CA1["type: json | memory | noop"]
        CA2["base_dir: ./cache"]
        CA3["connection_string / container (blob)"]
    end

    subgraph RPT2["reporting"]
        RP1["type: file | blob"]
        RP2["base_dir: ./logs"]
    end
```

---

## Pipeline Methods & Workflow Override

```mermaid
graph LR
    subgraph METHOD["IndexingMethod"]
        M1["standard\n(LLM graph extraction)"]
        M2["fast\n(NLP graph extraction)"]
        M3["standard-update\n(incremental standard)"]
        M4["fast-update\n(incremental fast)"]
    end

    subgraph CUSTOM["Custom Workflows Override"]
        WF2["workflows: list\n─────────────────────────────\nOverrides any built-in method.\nList exact workflow function names\nin execution order."]
    end

    subgraph ASYNC["Concurrency"]
        AS1["async_mode: asyncio | threaded"]
        AS2["concurrent_requests: 25"]
    end
```

---

## Snapshots (Debug Artifacts)

```mermaid
graph LR
    subgraph SNAP2["snapshots"]
        SN1["graphml: false\n(export graph as .graphml)"]
        SN2["embeddings: false\n(save raw embedding arrays)"]
        SN3["transient: false\n(keep intermediate parquet files)"]
    end
```

---

## Configuration Loading Flow

```mermaid
flowchart LR
    ROOT["--root directory"] --> YAML["settings.yaml\nor settings.json"]
    YAML --> LOAD["load_config()\ngraphrag.config.load_config"]
    ENV["Environment variables\nGRAPHRAG_API_KEY etc."] --> LOAD
    LOAD --> VALIDATE["Pydantic validation\n+ model_validator()"]
    VALIDATE --> GRC["GraphRagConfig object\n(passed to all pipeline stages)"]
```
