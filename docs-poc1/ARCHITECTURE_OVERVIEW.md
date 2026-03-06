# GraphRAG — Architecture Overview

> Microsoft GraphRAG is a **data pipeline and transformation suite** that converts unstructured text into a richly structured knowledge graph, then uses that graph to answer questions with a language model.

---

## High-Level System Map

```mermaid
graph TB
    subgraph INPUT["📥 Input Layer"]
        RAW["Raw Documents\n(txt / csv / json / jsonl\n/ pdf / docx / …)"]
    end

    subgraph INDEXING["⚙️ Indexing Pipeline"]
        direction TB
        LOAD["Load & Parse\ngraphrag-input"]
        CHUNK["Chunk Text\ngraphrag-chunking"]
        GRAPH_EXT["Graph Extraction\n(LLM or NLP)"]
        GRAPH_FIN["Finalize Graph\n(merge + prune)"]
        COMMUNITY["Community Detection\n(Leiden algorithm)"]
        REPORTS["Community Reports\n(LLM summaries)"]
        EMBED["Text Embeddings\ngraphrag-llm"]
        STORE["Persist Artifacts\ngraphrag-storage"]
    end

    subgraph KNOWLEDGE["🗄️ Knowledge Store"]
        PARQUET["Parquet / CSV Tables\n(entities, relationships,\ncommunities, text units, …)"]
        VECTORS["Vector Store\n(LanceDB / Azure AI Search\n/ CosmosDB)"]
    end

    subgraph QUERY["🔍 Query Engine"]
        direction TB
        LOCAL["Local Search\n(entity-centric)"]
        GLOBAL["Global Search\n(community-centric)"]
        DRIFT["DRIFT Search\n(exploratory)"]
        BASIC["Basic Search\n(vector-only)"]
    end

    subgraph OUTPUT["📤 Output Layer"]
        ANSWER["Structured Answer\n+ Source Context"]
    end

    RAW --> LOAD
    LOAD --> CHUNK
    CHUNK --> GRAPH_EXT
    GRAPH_EXT --> GRAPH_FIN
    GRAPH_FIN --> COMMUNITY
    COMMUNITY --> REPORTS
    REPORTS --> EMBED
    EMBED --> STORE
    STORE --> PARQUET
    STORE --> VECTORS

    PARQUET --> LOCAL & GLOBAL & DRIFT & BASIC
    VECTORS --> LOCAL & DRIFT & BASIC

    LOCAL & GLOBAL & DRIFT & BASIC --> ANSWER
```

---

## Monorepo Package Map

```mermaid
graph LR
    subgraph CORE["graphrag (core)"]
        CLI["CLI\ngraphrag.cli"]
        API["Public API\ngraphrag.api"]
        IDX["Indexing Engine\ngraphrag.index"]
        QRY["Query Engine\ngraphrag.query"]
        CFG["Config System\ngraphrag.config"]
        PT["Prompt Tuner\ngraphrag.prompt_tune"]
    end

    subgraph LIBS["Support Libraries"]
        INPUT["graphrag-input\n(document readers)"]
        CHUNK["graphrag-chunking\n(text splitters)"]
        LLM["graphrag-llm\n(LLM abstraction)"]
        CACHE["graphrag-cache\n(request cache)"]
        STORAGE["graphrag-storage\n(file / blob / memory)"]
        VECTORS["graphrag-vectors\n(vector stores)"]
        COMMON["graphrag-common\n(shared utilities)"]
    end

    CLI --> API
    API --> IDX & QRY
    IDX --> INPUT & CHUNK & LLM & CACHE & STORAGE & VECTORS
    QRY --> LLM & STORAGE & VECTORS
    CFG --> IDX & QRY
    PT --> LLM & STORAGE
```

---

## Two-Phase Lifecycle

```mermaid
sequenceDiagram
    actor User
    participant CLI as graphrag CLI
    participant IDX as Indexing Pipeline
    participant KS as Knowledge Store
    participant QRY as Query Engine
    participant LLM as LLM / Embeddings

    Note over User,LLM: Phase 1 — Build Index (run once or incrementally)
    User->>CLI: graphrag index --root ./
    CLI->>IDX: run_pipeline(config, method)
    IDX->>LLM: extract entities / relationships
    IDX->>LLM: summarize descriptions
    IDX->>LLM: generate community reports
    IDX->>LLM: embed text
    IDX->>KS: write parquet tables + vector store

    Note over User,LLM: Phase 2 — Query (run many times)
    User->>CLI: graphrag query --method local "…"
    CLI->>QRY: search(query, context)
    QRY->>KS: load tables + vector lookup
    QRY->>LLM: build context + generate answer
    QRY-->>User: answer + source citations
```

---

## Indexing Methods

| Method | Graph Extraction | Community Summaries | Notes |
|--------|-----------------|---------------------|-------|
| `standard` | LLM (GPT-4 class) | LLM | Highest quality, most expensive |
| `fast` | NLP (SpaCy / regex) | LLM | Cheaper; skips LLM graph extraction |
| `standard-update` | LLM (incremental) | LLM | Merges new docs into existing index |
| `fast-update` | NLP (incremental) | LLM | Incremental fast mode |

## Query Methods

| Method | Context Source | Best For |
|--------|---------------|----------|
| `local` | Entity graph + text units | Specific entities / facts |
| `global` | Community reports (map-reduce) | Broad themes / summaries |
| `drift` | Hybrid exploratory | Open-ended exploration |
| `basic` | Vector-only text units | Simple semantic search |
