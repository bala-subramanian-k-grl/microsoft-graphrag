# GraphRAG — Packages & Modules Reference

The repository is a Python monorepo of **9 packages** under `packages/`. The `graphrag` core package depends on the 8 support libraries below.

---

## Package Dependency Map

```mermaid
graph TB
    subgraph CORE["graphrag (core package)"]
        G_IDX["graphrag.index"]
        G_QRY["graphrag.query"]
        G_CFG["graphrag.config"]
        G_PT["graphrag.prompt_tune"]
        G_API["graphrag.api"]
        G_CLI["graphrag.cli"]
        G_CB["graphrag.callbacks"]
        G_DM["graphrag.data_model"]
        G_GRAPHS["graphrag.graphs"]
    end

    subgraph SUPPORT["Support Libraries"]
        INP["graphrag-input"]
        CHK["graphrag-chunking"]
        LLM["graphrag-llm"]
        CAC["graphrag-cache"]
        STO["graphrag-storage"]
        VEC["graphrag-vectors"]
        COM["graphrag-common"]
    end

    G_IDX --> INP & CHK & LLM & CAC & STO & VEC
    G_QRY --> LLM & STO & VEC
    G_CFG --> INP & CHK & LLM & CAC & STO & VEC
    G_PT --> LLM & STO
    G_API --> G_IDX & G_QRY & G_CFG
    G_CLI --> G_API
    LLM --> COM
```

---

## `graphrag` Core Package

```mermaid
graph TB
    subgraph CLI["graphrag.cli"]
        CLI_IDX["index()\ngraphrag index --root …"]
        CLI_QRY["query()\ngraphrag query --method …"]
        CLI_PT["prompt_tune()\ngraphrag prompt-tune …"]
        CLI_INIT["initialize()\ngraphrag init --root …"]
    end

    subgraph API["graphrag.api  (public surface)"]
        API_IDX["build_index()"]
        API_QRY["local_search()\nglobal_search()\ndrift_search()\nbasic_search()"]
        API_PT["generate_indexing_prompts()"]
    end

    subgraph IDX["graphrag.index"]
        IDX_WF["workflows/\nfactory.py  (PipelineFactory)\nWorkflow step functions"]
        IDX_OPS["operations/\nextract_graph\nsummarize_descriptions\nsummarize_communities\ncluster_graph\nembed_text\nextract_covariates\nbuild_noun_graph\nprune_graph\nfinalize_*"]
        IDX_RUN["run/\nrun_pipeline.py"]
        IDX_UPD["update/\n(incremental merge helpers)"]
    end

    subgraph QRY["graphrag.query"]
        QRY_LS["structured_search/\nlocal_search/"]
        QRY_GS["structured_search/\nglobal_search/"]
        QRY_DS["structured_search/\ndrift_search/"]
        QRY_BS["structured_search/\nbasic_search/"]
        QRY_CTX["context_builder/\nentity_extraction.py"]
        QRY_FAC["factory.py\nget_*_search_engine()"]
    end

    subgraph CFG["graphrag.config"]
        CFG_MOD["models/\nGraphRagConfig\n+ per-stage configs"]
        CFG_LOAD["load_config.py"]
        CFG_ENUM["enums.py\n(IndexingMethod, SearchMethod…)"]
        CFG_DEF["defaults.py"]
    end

    subgraph OTHER["Other Core Modules"]
        DM["data_model/\nEntity, Relationship,\nCommunity, CommunityReport,\nTextUnit, Covariate, Document"]
        GRAPHS["graphs/\nhierarchical_leiden.py\nstable_lcc.py"]
        PT["prompt_tune/\ngenerators + loaders"]
        CB["callbacks/\nWorkflowCallbacks\nQueryCallbacks"]
    end

    CLI --> API
    API --> IDX & QRY & CFG
```

---

## `graphrag-input` — Document Reader

```mermaid
graph LR
    subgraph READERS["Readers (InputReaderFactory)"]
        TR2["TextInputReader\n→ plain .txt files"]
        CR2["CSVInputReader\n→ .csv with configurable columns"]
        JR2["JsonInputReader\n→ .json (array of objects)"]
        JLR["JsonlInputReader\n→ .jsonl (one object per line)"]
        MKD["MarkItDownInputReader\n→ pdf, docx, xlsx, pptx, html…\n(via MarkItDown library)"]
    end

    subgraph CFG_INP["InputConfig"]
        I1["type: text | csv | json | jsonl\n| markitdown"]
        I2["file_pattern: *.txt"]
        I3["file_encoding: utf-8"]
        I4["text_column (csv/json)"]
        I5["title_column (csv/json)"]
        I6["metadata_columns []"]
    end

    READERS --> OUT2["TextDocument\n─────────────────\nid (sha256 hash)\ntitle\ntext\nmetadata dict"]
```

---

## `graphrag-chunking` — Text Splitter

```mermaid
graph LR
    subgraph CHUNKERS["Chunkers (ChunkerFactory)"]
        TOK["TokenChunker\n(tiktoken-based)"]
        SENT["SentenceChunker\n(NLTK sentence tokenizer)"]
    end

    subgraph CFG_CHK["ChunkingConfig"]
        CK1["type: tokens | sentence"]
        CK2["size: 1200"]
        CK3["overlap: 100"]
        CK4["encoding_model: cl100k_base"]
        CK5["prepend_metadata: bool"]
    end

    IN3["Document text"] --> CHUNKERS
    CHUNKERS --> OUT3["TextChunk[]\n─────────────────\ntext\nn_tokens\nsource_doc_id\nchunk_index"]
```

---

## `graphrag-llm` — LLM Abstraction Layer

```mermaid
graph TB
    subgraph FACTORIES["Factories"]
        CF["create_completion(ModelConfig)"]
        EF["create_embedding(ModelConfig)"]
    end

    subgraph COMPLETION["Completion (chat)"]
        LLC["LiteLLMCompletion"]
        MLC["MockLLMCompletion (testing)"]
    end

    subgraph EMBEDDING["Embedding"]
        LLE["LiteLLMEmbedding"]
        MLE["MockLLMEmbedding (testing)"]
    end

    subgraph MIDDLEWARE["Middleware Pipeline"]
        MW1["with_logging"]
        MW2["with_metrics"]
        MW3["with_rate_limiting\n(TPM / RPM buckets)"]
        MW4["with_cache\n(hash-keyed)"]
        MW5["with_retries\n(exponential / immediate)"]
        MW6["with_request_count"]
    end

    subgraph TOKENIZER["Tokenizer"]
        TK1["TiktokenTokenizer"]
        TK2["LiteLLMTokenizer"]
    end

    subgraph METRICS["Metrics"]
        ME1["MetricsStore\n(memory / noop)"]
        ME2["MetricsWriter\n(log / file)"]
        ME3["MetricsProcessor"]
    end

    CF --> LLC & MLC
    EF --> LLE & MLE
    LLC & LLE --> MIDDLEWARE
    MIDDLEWARE --> LITELLM["LiteLLM\n(OpenAI, Azure OpenAI,\nOllama, Cohere, …)"]
```

**Key config fields (ModelConfig):**

```mermaid
graph LR
    subgraph MC["ModelConfig"]
        MC1["api: openai | azure | ollama | …"]
        MC2["model: gpt-4o"]
        MC3["api_key / api_base"]
        MC4["deployment_name (Azure)"]
        MC5["max_tokens"]
        MC6["temperature / top_p"]
        MC7["rate_limit.requests_per_minute"]
        MC8["rate_limit.tokens_per_minute"]
        MC9["retry.max_retries"]
        MC10["retry.backoff_factor"]
        MC11["cache.type: json | memory | noop"]
    end
```

---

## `graphrag-storage` — Persistence Layer

```mermaid
graph LR
    subgraph STORAGE_TYPES["Storage Backends (StorageFactory)"]
        FS["FileStorage\n(local disk)"]
        ABS["AzureBlobStorage\n(Azure Blob)"]
        MEM["MemoryStorage\n(in-process, testing)"]
    end

    subgraph TABLE_PROVIDERS["Table Providers (TableProviderFactory)"]
        PARQ["ParquetTableProvider\n(.parquet files)"]
        CSVP["CSVTableProvider\n(.csv files)"]
    end

    subgraph CFG_STO["StorageConfig"]
        ST1["type: file | blob | memory"]
        ST2["base_dir: ./output"]
        ST3["connection_string (blob)"]
        ST4["container_name (blob)"]
    end

    STORAGE_TYPES --> TABLE_PROVIDERS
    TABLE_PROVIDERS --> TABLES2["DataFrame tables\n(entities, relationships,\ncommunities, text_units, …)"]
```

---

## `graphrag-vectors` — Vector Store

```mermaid
graph LR
    subgraph VS_TYPES["Vector Store Backends (VectorStoreFactory)"]
        LDB["LanceDBVectorStore\n(local, default)"]
        AIS["AzureAISearchVectorStore\n(cloud)"]
        CDB["CosmosDBVectorStore\n(NoSQL)"]
    end

    subgraph OPS["Operations"]
        LOAD_V["load_documents()\n(upsert embeddings)"]
        SEARCH_V["similarity_search_by_vector()\nsimilarity_search_by_text()"]
        FILTER_V["Filtered search\n(metadata predicates)"]
    end

    subgraph CFG_VS["VectorStoreConfig"]
        VS1["type: lancedb | azure_ai_search | cosmosdb"]
        VS2["db_uri: ./lancedb (LanceDB)"]
        VS3["url + api_key (Azure AI Search)"]
        VS4["connection_string (CosmosDB)"]
        VS5["vector_size: 1536"]
        VS6["index_schema: {name: IndexSchema}"]
    end

    VS_TYPES --> OPS
```

---

## `graphrag-cache` — LLM Request Cache

```mermaid
graph LR
    subgraph CACHE_TYPES["Cache Backends (CacheFactory)"]
        JC["JsonCache\n(JSON files on disk)"]
        MC2["MemoryCache\n(in-process dict)"]
        NC["NoopCache\n(disabled)"]
    end

    subgraph CFG_CACHE["CacheConfig"]
        CC1["type: json | memory | noop"]
        CC2["base_dir: ./cache"]
        CC3["connection_string (blob)"]
        CC4["container_name (blob)"]
    end

    KEY["cache_key =\nhash(model + params + prompt)"] --> CACHE_TYPES
    CACHE_TYPES -->|hit| RESP["Cached LLM response\n(skips API call)"]
    CACHE_TYPES -->|miss| LLMCALL["→ LLM API call\n→ store response"]
```

---

## `graphrag-common` — Shared Utilities

```mermaid
graph LR
    COM2["graphrag-common"] --> C1["Shared type definitions"]
    COM2 --> C2["Utility functions"]
    COM2 --> C3["Used by graphrag-llm\nand other packages"]
```

---

## Data Model Objects (`graphrag.data_model`)

```mermaid
classDiagram
    class Entity {
        id: str
        name: str
        type: str
        description: str
        rank: int
        text_unit_ids: list
        community_ids: list
    }

    class Relationship {
        id: str
        source: str
        target: str
        description: str
        weight: float
        combined_degree: int
        text_unit_ids: list
    }

    class Community {
        id: str
        level: int
        parent: int
        title: str
        entity_ids: list
        relationship_ids: list
    }

    class CommunityReport {
        id: str
        community: str
        level: int
        title: str
        summary: str
        full_content: str
        rank: float
        findings: list
    }

    class TextUnit {
        id: str
        document_id: str
        text: str
        n_tokens: int
        entity_ids: list
        relationship_ids: list
        covariate_ids: list
    }

    class Covariate {
        id: str
        subject_id: str
        subject_type: str
        object_id: str
        claim_type: str
        claim_status: str
        text: str
    }

    class Document {
        id: str
        title: str
        text: str
        text_unit_ids: list
    }

    Entity "1..*" --o "0..*" Community : "grouped into"
    Entity "1" -- "0..*" Relationship : "source / target"
    Community "1" -- "1" CommunityReport : "summarized by"
    TextUnit "1..*" -- "0..*" Entity : "source of"
    TextUnit "1..*" -- "0..*" Covariate : "source of"
    Document "1" -- "1..*" TextUnit : "split into"
```
