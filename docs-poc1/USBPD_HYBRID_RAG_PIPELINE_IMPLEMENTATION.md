# USB-PD Hybrid RAG Pipeline — Implementation Breakdown

> This document maps each stage of the Microsoft GraphRAG pipeline to the specific USB-PD AI Reasoning Platform requirements. Input sources are `spec.md` (USB-PD 3.2 Specification), `cts.md` (Compliance Test Specification), and `vif.md` (Vendor Info File Definition). The retrieval model is **Graph-first + Vector-augmented + Metadata-filtered**.

---

## Pipeline Architecture Overview

```mermaid
flowchart TD
    subgraph SOURCES["📥 Input Sources"]
        S1["spec.md\nUSB-PD 3.2 Specification\n~1000+ pages, protocol rules,\nstate diagrams, timers, message formats"]
        S2["cts.md\nCompliance Test Specification\nTest procedures, checks, timing tables,\nAMS-level test sequences"]
        S3["vif.md\nVendor Info File Definition\nDevice capability fields,\nPDO declarations, VIF constraints"]
    end

    subgraph STAGE1["Stage 1: Document Ingestion & Normalization"]
        P1["Parse + Tag by Source\nAssign doc_type metadata\n(spec | cts | vif)"]
    end

    subgraph STAGE2["Stage 2: Domain-Aware Chunking"]
        P2["Section-boundary chunking\nPreserve AMS boundaries,\ntest IDs, clause numbers,\ntimer/rule cross-refs"]
    end

    subgraph STAGE3["Stage 3: Graph Extraction"]
        P3_LLM["LLM Mode: extract\nentities + relationships\nper USB-PD ontology"]
        P3_NLP["NLP Mode (fast): extract\nnoun phrases from protocol\nvocabulary"]
    end

    subgraph STAGE4["Stage 4: Graph Finalization"]
        P4["Merge entity variants,\nlink spec ↔ cts ↔ vif,\ncompute AMS community clusters"]
    end

    subgraph STAGE5["Stage 5: Community Detection"]
        P5["Leiden clustering on\nprotocol entity graph\n→ AMS communities,\nrule groups, message families"]
    end

    subgraph STAGE6["Stage 6: Community Reports"]
        P6["LLM summaries per community:\nAMS flows, rule sets,\ntest procedure clusters"]
    end

    subgraph STAGE7["Stage 7: Metadata Enrichment & Filtering Schema"]
        P7["Tag all entities with:\ndoc_type, spec_revision,\nams_type, rule_id, test_id,\npdo_type, sop_domain, severity"]
    end

    subgraph STAGE8["Stage 8: Text Embeddings"]
        P8["Embed: spec clauses,\nrule descriptions, test steps,\ncommunity summaries,\nVIF field definitions"]
    end

    subgraph KNOWLEDGE["🗄️ Knowledge Store"]
        KG["Knowledge Graph\n(entities + relationships)"]
        VS["Vector Store\n(semantic embeddings)"]
        META["Metadata Filters\n(structured predicates)"]
    end

    subgraph QUERY["🔍 Hybrid Query Engine"]
        QG["Graph-first:\nAMS traversal, rule chains,\ncausal paths"]
        QV["Vector-augmented:\nsemantic spec clause retrieval"]
        QM["Metadata-filtered:\nspec_revision, ams_type,\ntest_id, severity"]
    end

    SOURCES --> STAGE1 --> STAGE2 --> STAGE3 --> STAGE4 --> STAGE5 --> STAGE6 --> STAGE7 --> STAGE8
    STAGE4 --> KG
    STAGE8 --> VS
    STAGE7 --> META
    KG & VS & META --> QUERY
```

---

## Stage 1 — Document Ingestion & Normalization

```mermaid
flowchart LR
    subgraph INPUTS["Input Documents"]
        SP["spec.md\n(USB-PD 3.2 spec)"]
        CT["cts.md\n(CTS Q1 2026)"]
        VI["vif.md\n(VIF 3.36)"]
    end

    subgraph READER["graphrag-input: MarkItDown / Custom Readers"]
        R1["spec.md reader:\nextract sections by heading hierarchy\n(Chapter > Section > Subsection)\nassign: doc_type=spec, spec_revision=3.2"]
        R2["cts.md reader:\nextract by test ID pattern\n(TEST.PD.*, COMMON.CHECK.*,\nCOMMON.PROC.*)\nassign: doc_type=cts, test_id=<ID>"]
        R3["vif.md reader:\nextract by field name blocks\nassign: doc_type=vif, vif_version=3.36"]
    end

    subgraph OUT1["Document Table — Required Metadata Columns"]
        D1["id (sha256 hash)\ntitle (section heading or test ID)\ntext (section body)\ndoc_type: spec | cts | vif\nspec_revision: 3.2 | 3.1 | 2.0\nams_type (when identifiable)\ntest_id (CTS only)\nvif_field (VIF only)\nsource_file"]
    end

    SP --> R1 --> OUT1
    CT --> R2 --> OUT1
    VI --> R3 --> OUT1

    subgraph CFG1["Configuration: input"]
        C1["type: markitdown"]
        C2["file_pattern: *.md"]
        C3["text_column: text"]
        C4["title_column: title"]
        C5["metadata_columns:\n  [doc_type, spec_revision,\n   test_id, vif_field, ams_type]"]
    end
```

**Key decisions for this stage:**

- `spec.md` — split at clause level (e.g. `6.3.7 PS_RDY Message`, `8.3.2 AMS Diagrams`). Each clause becomes one document record. Tag with `spec_revision=3.2` and extract clause ID as title.
- `cts.md` — split at test procedure level (`TEST.PD.PROT.SRC3.1`, `COMMON.CHECK.PD.3`). Each named test/check procedure becomes one record. Tag with `test_id` and applicable UUT type.
- `vif.md` — split at field definition level (e.g. `Num_Src_PDOs`, `Has_Invariant_PDOs`, `EPR_Supported_As_Src`). Each field group becomes one record.
- All records get `doc_type` metadata — this is the primary filter dimension for the query engine.

---

## Stage 2 — Domain-Aware Chunking

```mermaid
flowchart LR
    subgraph CHK["graphrag-chunking Strategy"]
        direction TB
        TK["TokenChunker (primary)\nsize: 800 tokens\noverlap: 150 tokens\nencoding_model: cl100k_base"]
        SENT["SentenceChunker (fallback for\nshort field definitions)"]
    end

    subgraph RULES["Chunking Rules for USB-PD Content"]
        R1["MUST NOT split across\nAMS boundary markers\n(e.g. AMS: Power Negotiation table rows)"]
        R2["MUST NOT split across\ntest step sequences\n(numbered test procedure steps stay together)"]
        R3["MUST preserve timer references\n(tFirstSourceCap, tSenderResponse)\nwithin the same chunk as their constraints"]
        R4["MUST preserve message sequence tables\n(Table 8.32: Steps for Successful Power Negotiation)\nas a single chunk"]
        R5["CAN split spec chapter prose\nat paragraph boundaries with 150-token overlap"]
    end

    subgraph META_PREP["Metadata prepended to each chunk"]
        M1["prepend_metadata: true\n─────────────────────────────\n[doc_type: spec] [clause: 8.3.2]\n[spec_revision: 3.2]\n[ams_type: SourceCapNegotiation]"]
    end

    subgraph CFG2["Configuration: chunking"]
        C1["type: tokens"]
        C2["size: 800"]
        C3["overlap: 150"]
        C4["encoding_model: cl100k_base"]
        C5["prepend_metadata: true"]
    end

    TK & SENT --> META_PREP
    RULES -.guidance.-> TK
```

**Key decisions for this stage:**

- Smaller chunks (800 tokens) than default (1200) because USB-PD spec text is dense — each sentence can carry a separate normative requirement with a distinct SHALL/SHALL NOT.
- 150-token overlap ensures timing constraint context (e.g. `tSenderResponse` value) is never orphaned from the step that references it.
- `prepend_metadata: true` embeds `doc_type`, `clause`, `ams_type` at the start of each chunk so the embedding captures the semantic context of the domain.

---

## Stage 3 — Graph Extraction (Dual-Mode)

```mermaid
flowchart TD
    TU["text_units table\n(all chunks from spec + cts + vif)"]

    subgraph MODE_SEL["Pipeline Mode Selection"]
        STD["Standard Mode\n(extract_graph ← LLM)\nFor spec.md clauses and\nCTS test procedures"]
        FAST["Fast Mode\n(extract_graph_nlp)\nFor VIF field definitions\n(structured, low ambiguity)"]
    end

    subgraph LLM_EXT["Standard: LLM Graph Extraction"]
        EP["Custom Entity-Extraction Prompt\n─────────────────────────────────────\nentity_types:\n  - AMS (Atomic Message Sequence)\n  - Message (protocol message type)\n  - State (policy engine state)\n  - Timer (protocol timer)\n  - Rule (normative requirement)\n  - SpecClause (spec section/table)\n  - TestCase (CTS test procedure)\n  - DataObject (PDO/BIST/VDO)\n  - Port (SRC/SNK/DRP)\n  - Role (Source/Sink/Cable)\n  - FailureMode (violation category)\n  - VIFField (VIF capability field)\n  - PowerContract (negotiated contract)"]
        MG["max_gleanings: 2\n(spec text dense — second pass\noften finds sub-entity refs)"]
    end

    subgraph NLP_EXT["Fast: NLP Extraction (VIF only)"]
        NP["extractor_type: syntactic_parser\n(SpaCy NER + dep-parse captures\ncapability field names as entities)"]
        PR["prune_graph:\n  min_node_freq: 2\n  min_edge_weight_pct: 5"]
    end

    subgraph REL_TYPES["Expected Relationship Types"]
        RL1["AMS -[CONTAINS]-> Message"]
        RL2["AMS -[STARTS_WITH]-> State"]
        RL3["AMS -[GOVERNED_BY]-> Rule"]
        RL4["Rule -[REFERENCES]-> SpecClause"]
        RL5["TestCase -[VALIDATES]-> AMS"]
        RL6["TestCase -[CHECKS]-> Rule"]
        RL7["Message -[HAS_DATA_OBJECT]-> DataObject"]
        RL8["Timer -[CONSTRAINS]-> AMS"]
        RL9["VIFField -[CONSTRAINS]-> DataObject"]
        RL10["FailureMode -[VIOLATES]-> Rule"]
        RL11["TestCase -[REFERENCES]-> SpecClause"]
    end

    TU --> MODE_SEL
    MODE_SEL --> LLM_EXT & NLP_EXT
    LLM_EXT --> REL_TYPES
    NLP_EXT --> REL_TYPES

    subgraph CFG3["Configuration: extract_graph"]
        E1["completion_model_id: default_chat"]
        E2["entity_types: [AMS, Message, State, Timer,\n Rule, SpecClause, TestCase, DataObject,\n Port, Role, FailureMode, VIFField, PowerContract]"]
        E3["max_gleanings: 2"]
        E4["num_threads: 50"]
        E5["prompt: prompts/usbpd_entity_extract.txt\n(custom domain prompt)"]
    end
```

**Custom prompt guidance (entity extraction):**

The entity extraction prompt must be tuned to recognize USB-PD-specific patterns:
- AMS names: `SourceCapNegotiation`, `PowerRoleSwap`, `VCONNSwap`, `CableDiscovery`, `EPREntry`
- Message names: `Source_Capabilities`, `Request`, `Accept`, `Reject`, `Wait`, `PS_RDY`, `GoodCRC`, `Soft_Reset`, `Hard_Reset`, `Get_Source_Cap`, `Get_Source_Cap_Extended`
- Timer names: `tFirstSourceCap`, `tSenderResponse`, `tTypeCSendSourceCap`, `tReceive`, `tRetry`, `tHardResetComplete`
- Rule patterns: `[TEST.PD.PROT.SRC3.1#1]` anchors, `[COMMON.CHECK.PD.3#1]` anchors, `SHALL`, `shall not`
- State names: `PE_SRC_Ready`, `PE_SNK_Ready`, `PE_SRC_Send_Capabilities`, `PE_SRC_Wait_New_Capabilities`

---

## Stage 4 — Graph Finalization & Cross-Document Linking

```mermaid
flowchart TD
    ENT["raw_entities from\nspec + cts + vif chunks"]
    REL["raw_relationships"]

    subgraph MERGE["Entity Merge Strategy"]
        M1["Canonical name normalization:\n'Source Capabilities' = 'Source_Capabilities'\n= 'SourceCap' → SAME entity"]
        M2["Cross-document linking:\nRule entity in cts.md LINKS TO\nSpecClause entity in spec.md\nwhen IDs match (e.g. tSenderResponse)"]
        M3["VIF field entities LINK TO\nDataObject entities in spec.md\n(e.g. Num_Src_PDOs ↔ Source PDO fields)"]
    end

    subgraph SUMM["Description Summarization"]
        DS["summarize_descriptions ← LLM\n──────────────────────────────────────────\nFor entities appearing in BOTH spec and CTS:\ncombine spec definition + CTS check behavior\ninto a single summary description\n\nmax_summary_length: 600\nprompt: prompts/usbpd_desc_summarize.txt"]
    end

    subgraph META_NODE["Metadata on Graph Nodes (spec_revision)"]
        SV["All entities tagged with:\nspec_revision: [3.2 | 3.1 | 2.0]\nintroduced_in: version\ndeprecated_in: version (if applicable)\nsop_domain: [SOP | SOP' | SOP'']\nuut_type: [SRC | SNK | DRP | CBL]"]
    end

    subgraph OUT_FIN["Output Tables"]
        ET["entities table\n──────────────────────\nid, name, type, description,\nspec_revision, ams_type,\ntest_id, doc_source,\nsop_domain, uut_type, rank"]
        RT["relationships table\n──────────────────────────\nid, source, target,\nrelationship_type, description,\nweight, spec_revision"]
    end

    ENT --> MERGE --> SUMM --> META_NODE --> ET
    REL --> META_NODE --> RT
```

**Critical cross-document links to establish:**

- `TEST.PD.PROT.SRC3.1` (CTS test) → `LINKS_TO` → `SourceCapabilityTimer` (spec entity) → `GOVERNED_BY` → clause `6.6.x` (spec)
- `TEST.PD.PROT.SRC3.2` (CTS test) → `LINKS_TO` → `SenderResponseTimer` → `CONSTRAINS` → `SourceCapNegotiation` AMS
- `COMMON.CHECK.PD.3` → `VALIDATES` → `GoodCRC` message sequence in spec
- `Num_Src_PDOs` (VIF field) → `CONSTRAINS` → `Source_Capabilities` message → `GOVERNS` → `SourceCapNegotiation` AMS
- `Has_Invariant_PDOs` (VIF field) → `GOVERNS` → `COMMON.PROC.PD.18` procedure

---

## Stage 5 — Community Detection (AMS Clustering)

```mermaid
flowchart LR
    RELS["relationships\ntable"] --> LEIDEN

    subgraph LEIDEN["Leiden Clustering — Expected Communities"]
        C1["Community L0: Source Capability Negotiation AMS\n(Source_Capabilities, Request, Accept/Reject/Wait,\nPS_RDY, GoodCRC, tFirstSourceCap, tSenderResponse,\nTEST.PD.PROT.SRC3.1–3.15, related VIF fields)"]
        C2["Community L0: Protocol Error & Reset AMS\n(Soft_Reset, Hard_Reset, tProtErrSoftReset,\nCOMMON.CHECK.PD.5, COMMON.CHECK.PD.3)"]
        C3["Community L0: Power Role Swap AMS\n(PR_Swap, DR_Swap, Accept/Reject/Wait,\nPS_RDY, vSafe0V, COMMON.PROC.BU.*)"]
        C4["Community L0: VCONN Swap AMS\n(VCONN_Swap, tVCONNSourceOn, tVCONNSourceOff,\nTEST.PD.PROT.SRC.8)"]
        C5["Community L0: EPR Entry AMS\n(EPR_Mode, EPR_Request, EPR_Source_Capabilities,\nEPR_Supported_As_Src VIF fields)"]
        C6["Community L0: VIF Capability Model\n(VIF fields: Has_Invariant_PDOs, Num_Src_PDOs,\nSrc_PDO_*, EPR_Mode_Capable, PD_Port_Type)"]
        C7["Community L1 (parent): Full Source Protocol\nSubcommunities: SrcCap + Reset + PRSwap"]
        C8["Community L1 (parent): EPR Protocol\nSubcommunities: EPR Entry + EPR SrcCap"]
    end

    subgraph CFG5["Configuration: cluster_graph"]
        CL1["max_cluster_size: 12\n(USB-PD AMS groups are\nnaturally 8-15 entities)"]
        CL2["use_lcc: true"]
        CL3["seed: 42 (reproducibility)"]
    end

    LEIDEN --> COMM["communities table\n───────────────────────────\nid, level, parent,\ntitle, entity_ids\n(AMS-aligned clusters)"]
```

---

## Stage 6 — Community Reports (AMS Knowledge Summaries)

```mermaid
flowchart TD
    COMM["communities"] & ENTS["entities"] & RELS2["relationships"] --> CTX

    subgraph CTX["Context Builder for USB-PD Communities"]
        MIX["Mixed context:\n- All entities in community (messages, timers, rules, states)\n- All relationships between them\n- Up to 3 text unit excerpts from spec + CTS"]
    end

    CTX --> CR

    subgraph CR["CommunityReportsExtractor ← LLM\n(custom USB-PD prompt)"]
        P1["Report structure for USB-PD communities:\n1. AMS title and USB-PD version scope\n2. Message sequence summary (A→B→C)\n3. Timer constraints (values + effects)\n4. Normative rules (SHALL / SHALL NOT)\n5. CTS test IDs that validate this community\n6. Common failure patterns\n7. VIF fields that constrain behavior\n8. Spec clause cross-references"]
    end

    CR --> CREP["community_reports table\n─────────────────────────────────────────\nid, community_id, level, title,\nams_type, spec_revision,\nsummary, full_content,\nrule_ids[], test_ids[], timer_ids[],\nrank, findings[]"]

    subgraph CFG6["Configuration: community_reports"]
        R1["completion_model_id: default_chat"]
        R2["max_input_length: 10000 tokens\n(USB-PD AMS reports need full\nmessage table + timer context)"]
        R3["max_length: 2500 tokens"]
        R4["prompt: prompts/usbpd_community_report.txt"]
    end
```

---

## Stage 7 — Metadata Schema for Filtering

This stage defines the filterable metadata dimensions applied to all graph entities, vector store documents, and community reports. This is the "metadata filtration" layer of the hybrid pipeline.

```mermaid
graph TB
    subgraph META_DIMS["Metadata Dimensions (Filter Keys)"]
        MD1["doc_type\nValues: spec | cts | vif\nUsage: route query to the right knowledge source"]
        MD2["spec_revision\nValues: 3.2 | 3.1 | 3.0 | 2.0\nUsage: version-scoped rule evaluation\n(SR-VER-001, SR-VER-002)"]
        MD3["ams_type\nValues: SourceCapNegotiation | PowerRoleSwap\n| VCONNSwap | CableDiscovery | EPREntry\n| HardReset | SoftReset | DataReset | ...\nUsage: AMS-scoped graph traversal and retrieval"]
        MD4["rule_category\nValues: timing | sequence | field_validation\n| protocol | retry | power\nUsage: SR-RULE-002 — compliance rule categories"]
        MD5["severity\nValues: SHALL | SHOULD | MAY | conditional\nUsage: filter mandatory vs. optional requirements"]
        MD6["test_id\nValues: TEST.PD.PROT.SRC3.*, COMMON.CHECK.*\nUsage: link CTS checks to spec entities"]
        MD7["uut_type\nValues: SRC | SNK | DRP | CBL | VPD\nUsage: filter tests applicable to device role\n(SR-ING-006)"]
        MD8["sop_domain\nValues: SOP | SOP_PRIME | SOP_DOUBLE_PRIME\nUsage: filter cable vs. port messages\n(SR-GRAPH-005)"]
        MD9["pdo_type\nValues: Fixed | Variable | Battery | PPS_APDO\n| SPR_AVS | EPR_AVS\nUsage: filter power capability queries"]
        MD10["timer_name\nValues: tFirstSourceCap | tSenderResponse\n| tTypeCSendSourceCap | tReceive | ...\nUsage: timer constraint retrieval"]
        MD11["pd_mode\nValues: PD2 | PD3\nUsage: CTS test applicability filtering"]
    end

    subgraph VS_SCHEMA["Vector Store Index Schema"]
        VS1["entities index\nmetadata: doc_type, spec_revision, ams_type,\nuut_type, sop_domain, rule_category"]
        VS2["text_units index\nmetadata: doc_type, spec_revision, ams_type,\ntest_id, clause_id, pd_mode"]
        VS3["community_reports index\nmetadata: ams_type, spec_revision, level,\nrule_ids[], test_ids[]"]
        VS4["spec_clauses index\nmetadata: doc_type=spec, clause_id,\nspec_revision, ams_type"]
    end
```

---

## Stage 8 — Text Embeddings

```mermaid
flowchart LR
    subgraph EMBED_TARGETS["Fields to Embed (embed_text.names)"]
        ET1["entity.description\n(ALL entity types — critical for\nentity vector lookup in local search)"]
        ET2["text_unit.text\n(all spec clauses, CTS test steps,\nVIF field definitions)"]
        ET3["community_report.full_content\n(AMS summaries for global search)"]
        ET4["community_report.summary\n(short AMS overviews)"]
        ET5["relationship.description\n(protocol links — optional but\nhelpful for DRIFT search)"]
    end

    EMBED_TARGETS --> EMB["graphrag-llm EmbeddingModel\n(text-embedding-3-large recommended\nfor USB-PD technical vocabulary)"]

    EMB --> VS2["Vector Store\n────────────────────────────────\nLanceDB (local dev)\nAzure AI Search (production)\n\nPer-index metadata fields:\n  doc_type, spec_revision, ams_type,\n  test_id, uut_type, sop_domain"]

    subgraph CFG8["Configuration: embed_text"]
        E1["names:\n  - entity.description\n  - text_unit.text\n  - community_report.full_content\n  - community_report.summary\n  - relationship.description"]
        E2["batch_size: 16"]
        E3["batch_max_tokens: 8192"]
        E4["embedding_model_id: default_embedding"]
        E5["num_threads: 50"]
    end
```

---

## Stage 9 — Hybrid Query Engine (Graph + Vector + Metadata Filter)

This is the core runtime of the platform. Per `SR-RAG-001`: graph-first retrieval, then vector augmentation.

```mermaid
flowchart TD
    Q["User / System Query\n(NL question or structured intent)"]

    Q --> CLASSIFY["Intent Classification\n─────────────────────────────────\nCompliance Query → Local Search\nRCA / Causal Query → Local + Graph Traversal\nBroad Protocol Query → Global Search\nVIF Capability Check → Metadata Filter + Local\nTimer / Rule Lookup → Metadata Filter + Local\nExploration → DRIFT Search"]

    CLASSIFY --> META_FILTER["Pre-Filter: Metadata Extraction from Query\n─────────────────────────────────────────\nExtract: ams_type, spec_revision, uut_type,\n  test_id, timer_name, doc_type\nApply as vector store filter predicates\nbefore any similarity search"]

    META_FILTER --> GRAPH_FIRST

    subgraph GRAPH_FIRST["① Graph-First Retrieval (SR-RAG-001)"]
        GF1["Identify seed entity from query\n(e.g. 'SourceCapNegotiation AMS')"]
        GF2["Graph traversal:\nAMS → CONTAINS → Messages\nAMS → GOVERNED_BY → Rules\nRules → REFERENCES → SpecClauses\nTestCases → VALIDATES → AMS"]
        GF3["Collect: entity facts, rule chains,\nspec clause IDs, timer constraints,\nassociated CTS test IDs"]
        GF1 --> GF2 --> GF3
    end

    GRAPH_FIRST --> VEC_AUG

    subgraph VEC_AUG["② Vector-Augmented Retrieval (SR-RAG-002)"]
        VA1["Embed query + graph context summary"]
        VA2["Search entity.description embeddings\nwith metadata filter:\n  ams_type = <extracted>\n  spec_revision = <extracted>\n  doc_type IN [spec, cts]"]
        VA3["Search text_unit embeddings\n(spec clauses + CTS test steps)\nwith same metadata filters"]
        VA4["Search community_report embeddings\n(AMS summaries)"]
        VA1 --> VA2 & VA3 & VA4
    end

    VEC_AUG --> ASSEMBLE

    subgraph ASSEMBLE["③ Context Assembly (SR-RAG-003)"]
        CA1["Combine:\n- Graph structural facts (entity + rel records)\n- Relevant spec clause text units\n- Applicable CTS test procedure excerpts\n- AMS community report summary\n- VIF field constraint facts"]
        CA2["Annotate with node IDs, rule IDs,\nspec clause refs for grounding\n(SR-RAG-004, SR-NL-004)"]
        CA3["Enforce token budget:\ntotal context ≤ max_context_tokens\npriority: graph facts > spec clauses > CTS steps > VIF"]
        CA1 --> CA2 --> CA3
    end

    ASSEMBLE --> LLM_GEN["④ Grounded LLM Response Generation\n─────────────────────────────────────────────\nSystem prompt: 'You are a USB-PD compliance\nexpert. Answer using ONLY the provided graph\nfacts and spec excerpts. Cite rule IDs and\nspec clauses in every claim. Do not hallucinate.'\n\nOutput includes:\n- Answer text\n- Cited: [Rule-IDs], [AMS-IDs], [Clause-IDs]\n- Reasoning chain (SR-RCA-005)\n- Confidence markers"]
```

---

## Stage 9a — Query Mode: Compliance Check

```mermaid
flowchart LR
    Q2["'Does TEST.PD.PROT.SRC3.2 pass\nfor this trace?'"] --> MF1["Filter:\ndoc_type IN [cts, spec]\ntest_id = TEST.PD.PROT.SRC3.2\nams_type = SourceCapNegotiation"]

    MF1 --> GQ1["Graph:\nTEST.PD.PROT.SRC3.2\n→ VALIDATES → SourceCapNegotiation AMS\n→ GOVERNED_BY → tSenderResponse Rule\n→ REFERENCES → spec clause 6.6.2"]

    GQ1 --> VQ1["Vector:\nSearch spec clause 6.6.2 text (exact timer values)\nSearch CTS test steps for SRC3.2"]

    VQ1 --> ANS1["Answer:\n'SenderResponseTimer (tSenderResponse: 24–30ms)\nmust elapse after GoodCRC EOP\nbefore Hard Reset is issued.\nRef: [TEST.PD.PROT.SRC3.2#2, spec 6.6.2]'"]
```

---

## Stage 9b — Query Mode: RCA Traversal

```mermaid
flowchart LR
    Q3["'Why did power negotiation fail?\n(trace shows SenderResponseTimer expired)'"] --> MF2["Filter:\nams_type = SourceCapNegotiation\ndoc_type IN [spec, cts]\nuut_type = SRC"]

    MF2 --> GQ2["Graph traversal:\nFailureMode: SenderResponseTimerExpired\n→ VIOLATES → Rule: tSenderResponse\n→ CAUSED_BY → Missing Request Message\n→ IN_AMS → SourceCapNegotiation\n→ GOVERNED_BY → TEST.PD.PROT.SRC3.2"]

    GQ2 --> VQ2["Vector:\nSpec clause on SenderResponseTimer behavior\nCTS SRC3.2 expected behavior description"]

    VQ2 --> ANS2["RCA Answer:\nObserved: No Request Message received\nExpected: Request within tSenderResponse (24-30ms)\nViolated Rule: tSenderResponse [spec 6.6.2]\nCTS Ref: TEST.PD.PROT.SRC3.2\nMitigation: Check Sink firmware Request path\nRef chain: [AMS-SrcCapNeg-001] → [Rule-tSR-001]"]
```

---

## Stage 9c — Query Mode: VIF Capability Filter

```mermaid
flowchart LR
    Q4["'Which tests apply to a Source-only\ndevice with no EPR support?'"] --> MF3["Filter:\nuut_type = SRC\ndoc_type = cts\nEPR_Supported_As_Src = NO\npd_mode IN [PD2, PD3]"]

    MF3 --> GQ3["Graph:\nVIFField: EPR_Supported_As_Src=NO\n→ CONSTRAINS → TestCase applicability\n  → excludes EPR AMS communities"]

    GQ3 --> VQ3["Vector:\nSearch CTS text_units filtered by\nuut_type=SRC AND ams_type NOT IN [EPR*]"]

    VQ3 --> ANS3["Filtered test list:\nApplicable: TEST.PD.PROT.SRC.1–9,\nTEST.PD.PROT.SRC3.1–14,\nTEST.PD.PS.SRC.1-2,\nCOMMON.CHECK.PD.3–7,\nCOMMON.PROC.BU.1\nExcluded: TEST.PD.EPR.*, EPR AMS tests"]
```

---

## Incremental Update Pipeline (Phase 2+)

```mermaid
flowchart TD
    NEW["New test trace log\n(field failure event)"] --> INGEST["Normalize trace event\n→ TraceEvent entities\n→ link to AMS, Rule, Port"]

    INGEST --> DELTA_IDX["Incremental index run\n(standard-update method)\nOnly new/changed entities re-indexed"]

    DELTA_IDX --> PATTERN["FailurePattern Mining\n(SR-MINE-001 to SR-MINE-005)\n─────────────────────────────────────\nAggregate violations by:\n  ams_type, firmware_version,\n  cable_type, environment\n→ generate FailurePattern entities"]

    PATTERN --> APPROVAL["Approval Workflow\n(SR-GOV-004)\nNew patterns require human review\nbefore promoting to trusted knowledge"]

    APPROVAL --> KB["Updated Knowledge Graph\n+ Vector Store\n(institutional memory: SR-BR-005)"]
```

---

## Configuration File Summary

```mermaid
graph TB
    subgraph SETTINGS["settings.yaml — Key Configuration Blocks"]
        S1["completion_models:\n  default_chat:\n    api: azure\n    model: gpt-4o\n    max_tokens: 4096\n    rate_limit:\n      requests_per_minute: 60\n    retry:\n      max_retries: 10"]

        S2["embedding_models:\n  default_embedding:\n    api: azure\n    model: text-embedding-3-large\n    rate_limit:\n      tokens_per_minute: 350000"]

        S3["input:\n  type: markitdown\n  file_pattern: '*.md'\n  metadata_columns:\n    - doc_type\n    - spec_revision\n    - ams_type\n    - test_id\n    - uut_type"]

        S4["chunking:\n  type: tokens\n  size: 800\n  overlap: 150\n  prepend_metadata: true"]

        S5["extract_graph:\n  entity_types: [AMS, Message, State, Timer,\n    Rule, SpecClause, TestCase, DataObject,\n    Port, Role, FailureMode, VIFField]\n  max_gleanings: 2\n  prompt: prompts/usbpd_entity_extract.txt"]

        S6["cluster_graph:\n  max_cluster_size: 12\n  use_lcc: true\n  seed: 42"]

        S7["community_reports:\n  max_input_length: 10000\n  max_length: 2500\n  prompt: prompts/usbpd_community_report.txt"]

        S8["embed_text:\n  names:\n    - entity.description\n    - text_unit.text\n    - community_report.full_content\n    - community_report.summary\n  batch_size: 16"]

        S9["vector_store:\n  type: lancedb  # or azure_ai_search\n  vector_size: 3072  # text-embedding-3-large\n  index_schema:\n    entity.description:\n      metadata: [doc_type, spec_revision,\n        ams_type, uut_type, sop_domain]\n    text_unit.text:\n      metadata: [doc_type, clause_id,\n        test_id, ams_type, pd_mode]"]

        S10["local_search:\n  text_unit_prop: 0.4\n  community_prop: 0.15\n  top_k_entities: 15\n  top_k_relationships: 15\n  max_context_tokens: 14000"]

        S11["global_search:\n  data_max_tokens: 14000\n  map_max_length: 1500\n  reduce_max_length: 3000\n  max_context_tokens: 18000"]
    end
```

---

## Phase Mapping to Pipeline Stages

```mermaid
gantt
    title Implementation Phases vs. Pipeline Stages
    dateFormat  X
    axisFormat  Phase %s

    section Phase 1 MVP
    Stage 1 Document Ingestion (spec.md only + cts SRC section) :done, p1s1, 1, 2
    Stage 2 Chunking (SourceCapNeg AMS chunks) :done, p1s2, 2, 3
    Stage 3 Graph Extraction (5 core entities) :done, p1s3, 3, 4
    Stage 4 Finalization (spec↔CTS links) :done, p1s4, 4, 5
    Stage 5 Communities (SourceCapNeg community) :done, p1s5, 5, 6
    Stage 6 Community Reports (1 AMS report) :done, p1s6, 6, 7
    Stage 7 Metadata Schema (core filter keys) :done, p1s7, 7, 8
    Stage 8 Embeddings (entity + text_unit) :done, p1s8, 8, 9
    Stage 9 Local Search + Compliance query mode :done, p1s9, 9, 10

    section Phase 2
    All 3 sources ingested (spec + cts + vif) :p2s1, 10, 12
    All entity types extracted :p2s2, 11, 13
    VIF↔spec cross-links :p2s3, 12, 14
    Multi-AMS communities :p2s4, 13, 15
    Global + DRIFT search modes :p2s5, 14, 16
    Failure Pattern mining :p2s6, 15, 17

    section Phase 3
    Incremental update pipeline :p3s1, 17, 20
    Causal pattern clustering :p3s2, 18, 21
    Cross-version regression detection :p3s3, 19, 22
```
