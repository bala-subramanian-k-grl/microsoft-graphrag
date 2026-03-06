# GraphRAG — Query Pipeline

The query engine loads the persisted knowledge index and uses one of four **search strategies** to answer user questions with an LLM.

---

## Query Method Selection

```mermaid
flowchart TD
    Q["User Query"] --> SEL{"--method"}
    SEL -->|local| LOCAL["Local Search\n(entity-centric)"]
    SEL -->|global| GLOBAL["Global Search\n(community-centric,\nmap-reduce)"]
    SEL -->|drift| DRIFT["DRIFT Search\n(exploratory /\nhybrid)"]
    SEL -->|basic| BASIC["Basic Search\n(vector-only)"]

    LOCAL & GLOBAL & DRIFT & BASIC --> LOAD["Load Knowledge Store\n(parquet tables + vector store)"]
    LOAD --> LLM["LLM completion\n(build context → generate answer)"]
    LLM --> ANS["Structured Answer\n+ source citations"]
```

---

## Local Search — Detailed Flow

Best for: **specific entities, facts, and relationships** within the graph.

```mermaid
flowchart TD
    Q2["User Query"] --> EMB_Q["Embed query\n(embedding model)"]

    EMB_Q --> VEC_LOOKUP["Vector similarity search\nentity description embeddings\n(top-k entities)"]

    VEC_LOOKUP --> CTX_BUILD

    subgraph CTX_BUILD["LocalSearchMixedContext — build_context()"]
        direction TB
        ENT_CTX["Entity records\n(top_k_entities)"]
        REL_CTX["Relationships\n(top_k_relationships)"]
        TU_CTX["Text units\n(text_unit_prop of token budget)"]
        CR_CTX["Community reports\n(community_prop of token budget)"]
        COV_CTX["Covariates\n(if available)"]
        CONV["Conversation history\n(max_turns window)"]
    end

    CTX_BUILD --> PROMPT["System prompt + context\n+ query → LLM"]
    PROMPT --> ANS2["Answer + context_data"]

    subgraph CFG_LS["Config: local_search"]
        LS1["completion_model_id"]
        LS2["embedding_model_id"]
        LS3["text_unit_prop: 0.5"]
        LS4["community_prop: 0.1"]
        LS5["top_k_entities: 10"]
        LS6["top_k_relationships: 10"]
        LS7["max_context_tokens: 12000"]
        LS8["conversation_history_max_turns: 5"]
    end
```

---

## Global Search — Map-Reduce Flow

Best for: **broad thematic questions** that span the entire corpus.

```mermaid
flowchart TD
    Q3["User Query"] --> CB["GlobalCommunityContext\n— build_context()"]

    subgraph CB["Context Builder"]
        RANK["Rank communities by\noccurrence weight"]
        SHUF["Shuffle + normalize weights"]
        PACK["Pack reports into\ntoken-budget batches"]
    end

    PACK --> MAP_PHASE

    subgraph MAP_PHASE["Map Phase  (parallel, per batch)"]
        M1["Batch 1 → LLM → partial answer + score"]
        M2["Batch 2 → LLM → partial answer + score"]
        MN["Batch N → LLM → partial answer + score"]
    end

    MAP_PHASE --> REDUCE_PHASE["Reduce Phase\nTop-scored partial answers → LLM → final answer"]

    REDUCE_PHASE --> ANS3["Final Answer"]

    subgraph DYN["Optional: Dynamic Community Selection"]
        DS1["LLM rates each community\nrelevance to query"]
        DS2["Only relevant communities\nenter map phase"]
    end

    subgraph CFG_GS["Config: global_search"]
        GS1["completion_model_id"]
        GS2["data_max_tokens: 12000"]
        GS3["map_max_length: 1000"]
        GS4["reduce_max_length: 2000"]
        GS5["max_context_tokens: 16000"]
        GS6["dynamic_search_threshold: 0.5"]
        GS7["dynamic_search_max_level: 3"]
        GS8["dynamic_search_keep_parent: bool"]
    end
```

---

## DRIFT Search — Exploratory Hybrid Flow

Best for: **open-ended exploration** mixing global themes with local entity detail.

```mermaid
flowchart TD
    Q4["User Query"] --> DRIFT_CB

    subgraph DRIFT_CB["DRIFTSearchContextBuilder"]
        direction TB
        EMB_D["Embed query"]
        GLOBAL_Q["Global query\n(top community reports)"]
        PRIMER["Generate primer answer\n← LLM (global context)"]
        FOLLOW["Generate follow-up\nsub-queries ← LLM"]
    end

    PRIMER --> LOCAL_ITERS

    subgraph LOCAL_ITERS["Local Search Iterations (per sub-query)"]
        LI1["sub-query 1 → LocalSearch → partial answer + score"]
        LI2["sub-query 2 → LocalSearch → partial answer + score"]
        LIN["sub-query N → LocalSearch → partial answer + score"]
    end

    LOCAL_ITERS --> RED["Reduce: merge answers ← LLM"]
    RED --> ANS4["Final Exploratory Answer"]

    subgraph CFG_DR["Config: drift_search"]
        DR1["completion_model_id"]
        DR2["embedding_model_id"]
        DR3["drift_k_followups: 20"]
        DR4["primer_folds: 10"]
        DR5["n_depth: 1"]
        DR6["local_search_text_unit_prop"]
        DR7["local_search_community_prop"]
        DR8["local_search_top_k_entities"]
        DR9["max_context_tokens: 12000"]
    end
```

---

## Basic Search — Vector-Only Flow

Best for: **simple semantic similarity** search without graph traversal.

```mermaid
flowchart LR
    Q5["User Query"] --> EMB_B["Embed query\n(embedding model)"]
    EMB_B --> VS_SEARCH["Vector Store\nsimilarity search\n(text_unit embeddings, top-k)"]
    VS_SEARCH --> PACK_B["Pack text units\ninto token budget"]
    PACK_B --> LLM_B["System prompt + text context\n+ query → LLM"]
    LLM_B --> ANS5["Answer"]

    subgraph CFG_BS["Config: basic_search"]
        BS1["completion_model_id"]
        BS2["embedding_model_id"]
        BS3["k: 10 (top-k text units)"]
        BS4["max_context_tokens: 12000"]
    end
```

---

## Context Window Budget Management

All search methods share the same token-budget approach:

```mermaid
flowchart LR
    BUDGET["max_context_tokens\n(model limit)"] --> ALLOC{"Budget\nAllocation"}

    ALLOC -->|local| L1["text_unit_prop × budget\n→ text units"]
    ALLOC -->|local| L2["community_prop × budget\n→ community reports"]
    ALLOC -->|local| L3["remainder\n→ entities + relationships"]

    ALLOC -->|global| G1["data_max_tokens\n→ packed community reports"]

    ALLOC -->|basic| B1["max_context_tokens\n→ top-k text units"]

    L1 & L2 & L3 & G1 & B1 --> TOKENIZER["Tokenizer\n(tiktoken / LiteLLM)"]
    TOKENIZER --> TRIM["Truncate to fit\nwithin budget"]
```

---

## Query Data Loading

```mermaid
flowchart LR
    subgraph TABLES["Parquet / CSV Tables (graphrag-storage)"]
        T1["entities"]
        T2["relationships"]
        T3["communities"]
        T4["community_reports"]
        T5["text_units"]
        T6["covariates (optional)"]
        T7["documents"]
    end

    subgraph VS2["Vector Store (graphrag-vectors)"]
        V1["entity.description embeddings"]
        V2["text_unit.text embeddings"]
        V3["community.summary embeddings"]
    end

    T1 & T2 & T3 & T4 & T5 & T6 --> LOCAL2["Local / DRIFT / Global\nContext Builders"]
    V1 --> LOCAL2
    T5 --> BASIC2["Basic Search\nContext Builder"]
    V2 --> BASIC2
```

---

## Question Generation (Prompt Tuning helper)

```mermaid
flowchart LR
    DOCS2["Sample documents"] --> QG["QuestionGenerator\n← LLM"]
    QG --> QUESTIONS["Candidate questions\n(tests / evaluation prompts)"]

    subgraph QUESTION_CFG["Config"]
        QC1["completion_model_id"]
        QC2["community_report context"]
        QC3["question count: N"]
    end
```
