# USB-PD Source Capability Sequence — RAG Test Plan

> This document defines the step-by-step test plan for validating the hybrid RAG pipeline against the **Source Capabilities Negotiation AMS** (Atomic Message Sequence) using the USB-PD 3.2 Specification (`spec.md`) as the knowledge source. It maps each CTS test case from `cts.md` (Section 5.3) to expected graph retrievals, vector lookups, and answer validation criteria.

---

## What Is Source Capability Sequence?

```mermaid
sequenceDiagram
    participant SRC as Source (UUT)
    participant SNK as Sink (Tester)

    Note over SRC,SNK: AMS: SourceCapNegotiation starts at PE_SRC_Send_Capabilities

    SRC->>SNK: Source_Capabilities [PDO list] ← within tFirstSourceCap
    SNK->>SRC: GoodCRC ← within tTransmit max

    Note over SRC,SNK: SenderResponseTimer starts after GoodCRC sent

    SNK->>SRC: Request [selected PDO] ← within tSenderResponse
    SRC->>SNK: GoodCRC ← within tTransmit max

    SRC->>SNK: Accept ← OR Reject ← OR Wait
    SNK->>SRC: GoodCRC

    Note over SRC,SNK: If Accept: Power Transition begins
    SRC-->>SNK: [VBUS transitions to negotiated voltage]
    SRC->>SNK: PS_RDY ← within tSrcReady max
    SNK->>SRC: GoodCRC

    Note over SRC,SNK: AMS complete: PE_SRC_Ready
```

**Key timers under test:**

| Timer | Min | Max | Source |
|-------|-----|-----|--------|
| `tFirstSourceCap` | — | 250 ms | spec §6.6.x |
| `tSenderResponse` | 24 ms | 30 ms | spec §6.6.2 |
| `tTypeCSendSourceCap` | 100 ms | 200 ms | spec (SourceCapabilityTimer) |
| `tReceive` | — | 1.1 ms | spec |
| `tRetry` | — | 195 µs | spec |
| `tTransmit` | — | 195 µs | spec |
| `tSrcReady` | — | 285 ms (SPR) | spec §7.x |

---

## Test Coverage Map — CTS Source-Capable Tests

```mermaid
graph TB
    subgraph PD2_PD3["Section 5.3.1 — PD2 and PD3 Modes (SRC)"]
        T01["TEST.PD.PROT.SRC.1\nGet_Source_Cap Response"]
        T02["TEST.PD.PROT.SRC.2\nGet_Source_Cap No Request"]
        T03["TEST.PD.PROT.SRC.3\nSenderResponseTimer Deadline"]
        T04["TEST.PD.PROT.SRC.4\nReject Request"]
        T05["TEST.PD.PROT.SRC.5\nReject Request Invalid Object Position"]
        T06["TEST.PD.PROT.SRC.6\nAtomic Message Sequence - Request"]
        T07["TEST.PD.PROT.SRC.7\nDR_Swap"]
        T08["TEST.PD.PROT.SRC.8\nVCONN_Swap Response"]
        T09["TEST.PD.PROT.SRC.9\nPR_Swap Response"]
    end

    subgraph PD3_ONLY["Section 5.3.2 — PD3 Mode Only (SRC)"]
        T10["TEST.PD.PROT.SRC3.1\nSourceCapabilityTimer Timeout"]
        T11["TEST.PD.PROT.SRC3.2\nSenderResponseTimer Timeout"]
        T12["TEST.PD.PROT.SRC3.3\nGet_Source_Cap_Extended Response"]
        T13["TEST.PD.PROT.SRC3.4\nAlert Response Source Input Change"]
        T14["TEST.PD.PROT.SRC3.5\nAlert Response Battery Status Change"]
        T15["TEST.PD.PROT.SRC3.6\nSoft Reset Sent when SinkTxOK"]
        T16["TEST.PD.PROT.SRC3.7\nGet_PPS_Status Response"]
        T17["TEST.PD.PROT.SRC3.13\nSource Power Rule Check"]
        T18["TEST.PD.PROT.SRC3.14\nSource Info Response"]
        T19["TEST.PD.PROT.SRC3.15\nAlert Response Extended Alert"]
    end

    subgraph COMMON["Common Checks Used Across SRC Tests"]
        CC1["COMMON.CHECK.PD.2\nMessage Header Check"]
        CC2["COMMON.CHECK.PD.3\nGoodCRC Check"]
        CC3["COMMON.CHECK.PD.4\nAMS Response Timing"]
        CC4["COMMON.CHECK.PD.5\nSoft Reset Check"]
        CC5["COMMON.CHECK.PD.7\nSource Capabilities Check"]
        CC6["COMMON.CHECK.PD.11\nSource Capabilities Extended Check"]
        CC7["COMMON.PROC.BU.1\nBring-up Procedure (SRC)"]
        CC8["COMMON.PROC.PD.18\nSource Capabilities Mismatch Handling"]
    end

    T01 & T02 & T03 & T04 & T05 & T06 --> SRCCAP_CORE["Core SourceCapNeg AMS"]
    T10 & T11 --> TIMER_TESTS["Timer Compliance Tests"]
    T12 & T13 & T14 & T15 --> EXTENDED_MSG["Extended Message Tests"]
    SRCCAP_CORE & TIMER_TESTS & EXTENDED_MSG --> CC1 & CC2 & CC3
```

---

## RAG Test Plan — Per Test Case

### TEST GROUP 1: Core Source Capability Message Sequence

---

#### TEST.PD.PROT.SRC.1 — Get_Source_Cap Response

```mermaid
flowchart TD
    TQ1["Query:\n'What is the expected Source response to\nGet_Source_Cap in the SourceCapNeg AMS?'"]

    TQ1 --> FILTER1["Metadata Filters:\n  ams_type = SourceCapNegotiation\n  doc_type IN [spec, cts]\n  test_id = TEST.PD.PROT.SRC.1\n  uut_type = SRC"]

    FILTER1 --> GRAPH1["Expected Graph Retrieval:\n  TestCase[TEST.PD.PROT.SRC.1]\n  → VALIDATES → AMS[SourceCapNegotiation]\n  → CONTAINS → Message[Source_Capabilities]\n  → GOVERNED_BY → Rule[tFirstSourceCap]\n  → REFERENCES → SpecClause[6.3.8 / 8.3.2]"]

    FILTER1 --> VEC1["Expected Vector Retrieval:\n  Spec: 'Source shall send Source_Capabilities\n    within tFirstSourceCap of Get_Source_Cap'\n  CTS: SRC.1 test step #1, #4 procedure text"]

    GRAPH1 & VEC1 --> ANS1["Expected Answer:\n  Source_Capabilities message sent within tFirstSourceCap max\n  after receiving Get_Source_Cap\n  Cite: [TEST.PD.PROT.SRC.1#1, spec §6.3.8]"]

    subgraph PASS_CRITERIA1["Pass Criteria for RAG Answer"]
        P1["✓ Cites TEST.PD.PROT.SRC.1#1 check ID"]
        P2["✓ States tFirstSourceCap timing constraint"]
        P3["✓ References Source_Capabilities message"]
        P4["✓ Does NOT hallucinate additional steps not in spec/CTS"]
    end
```

---

#### TEST.PD.PROT.SRC.2 — Get_Source_Cap No Request (Hard Reset Path)

```mermaid
flowchart TD
    TQ2["Query:\n'What happens if Source sends Source_Capabilities\nbut Sink never sends a Request?'"]

    FILTER2["Metadata Filters:\n  ams_type = SourceCapNegotiation\n  test_id = TEST.PD.PROT.SRC.2\n  timer_name = tSenderResponse"]

    TQ2 --> FILTER2 --> GRAPH2["Expected Graph Retrieval:\n  TestCase[TEST.PD.PROT.SRC.2]\n  → VALIDATES → Timer[tSenderResponse]\n  → CONSTRAINS → AMS[SourceCapNegotiation]\n  → TRIGGERS_ON_EXPIRY → Message[Hard_Reset]"]

    GRAPH2 --> ANS2["Expected Answer:\n  tSenderResponse timer expires (24–30ms after GoodCRC)\n  → UUT issues Hard Reset\n  Timing: tSenderResponse min + tHardResetComplete max\n  Cite: [TEST.PD.PROT.SRC.2#2, spec §6.6.2]"]

    subgraph PASS_CRITERIA2["Pass Criteria"]
        P2A["✓ Identifies tSenderResponse timer (min: 24ms, max: 30ms)"]
        P2B["✓ Identifies Hard Reset as the timeout response"]
        P2C["✓ Cites TEST.PD.PROT.SRC.2#2 check anchor"]
        P2D["✓ Distinguishes from SoftReset (wrong failure)"]
    end
```

---

#### TEST.PD.PROT.SRC.4 — Reject Request

```mermaid
flowchart TD
    TQ4["Query:\n'Under what conditions does a Source send Reject,\nand what must happen after?'"]

    FILTER4["Metadata Filters:\n  ams_type = SourceCapNegotiation\n  test_id = TEST.PD.PROT.SRC.4\n  doc_type IN [spec, cts]"]

    TQ4 --> FILTER4 --> GRAPH4["Expected Graph Retrieval:\n  TestCase[TEST.PD.PROT.SRC.4]\n  → VALIDATES → AMS[SourceCapNegotiation]\n  → CONTAINS → Message[Reject]\n  → FOLLOWED_BY → Message[Source_Capabilities]\n  → GOVERNED_BY → Rule[SpecClause: Source after Reject]"]

    GRAPH4 --> ANS4["Expected Answer:\n  Source sends Reject if Request is for unsupported PDO\n  After Reject: Source must re-send Source_Capabilities\n  Timing: within tFirstSourceCap\n  Cite: [TEST.PD.PROT.SRC.4#1, TEST.PD.PROT.SRC.4#2]"]

    subgraph PASS_CRITERIA4["Pass Criteria"]
        P4A["✓ Identifies Reject trigger condition (invalid PDO)"]
        P4B["✓ States Source_Capabilities must follow within tFirstSourceCap"]
        P4C["✓ Does NOT say PS_RDY follows Reject (wrong)"]
    end
```

---

#### TEST.PD.PROT.SRC.6 — AMS Sequence Violation (Interrupt by non-AMS message)

```mermaid
flowchart TD
    TQ6["Query:\n'What must Source do if it receives an\nunexpected message during SourceCapNeg AMS?'"]

    FILTER6["Metadata Filters:\n  ams_type = SourceCapNegotiation\n  test_id = TEST.PD.PROT.SRC.6\n  rule_category = sequence"]

    TQ6 --> FILTER6 --> GRAPH6["Expected Graph Retrieval:\n  TestCase[TEST.PD.PROT.SRC.6]\n  → VALIDATES → Rule[AMS Non-Interruptible]\n  → REFERENCES → SpecClause[6.8.1 Soft Reset]\n  AMS[SourceCapNegotiation] is NON-INTERRUPTIBLE\n  → TRIGGERS → Message[Soft_Reset]"]

    GRAPH6 --> ANS6["Expected Answer:\n  Source sends Soft_Reset within tProtErrSoftReset\n  AMS: SourceCapNegotiation is non-interruptible\n  After Soft_Reset: Source re-sends Source_Capabilities\n  with MessageID=1 within tTypeCSinkWaitCap max\n  Cite: [TEST.PD.PROT.SRC.6#2, TEST.PD.PROT.SRC.6#3]"]

    subgraph PASS_CRITERIA6["Pass Criteria"]
        P6A["✓ Identifies Soft Reset as the AMS violation response"]
        P6B["✓ Cites tProtErrSoftReset timing"]
        P6C["✓ States Source_Capabilities with MessageID=1 follows"]
        P6D["✓ References tTypeCSinkWaitCap"]
    end
```

---

### TEST GROUP 2: Timer Compliance Tests (PD3 Mode)

---

#### TEST.PD.PROT.SRC3.1 — SourceCapabilityTimer Timeout

```mermaid
flowchart TD
    TQ10["Query:\n'How does SourceCapabilityTimer work?\nWhat is the retransmission pattern when\nGoodCRC is never received?'"]

    FILTER10["Metadata Filters:\n  test_id = TEST.PD.PROT.SRC3.1\n  timer_name = tTypeCSendSourceCap\n  ams_type = SourceCapNegotiation\n  uut_type = SRC\n  pd_mode = PD3"]

    TQ10 --> FILTER10 --> GRAPH10["Expected Graph Retrieval:\n  TestCase[TEST.PD.PROT.SRC3.1]\n  → VALIDATES → Timer[SourceCapabilityTimer]\n  → CONSTRAINS → Message[Source_Capabilities]\n  Timer values: tTypeCSendSourceCap min=100ms, max=200ms\n  Retry: up to 2 retries before tTypeCSendSourceCap\n  → REFERENCES → SpecClause[tFirstSourceCap, tReceive, tRetry]"]

    GRAPH10 --> ANS10["Expected Answer:\n  Step 1: SrcCap sent; no GoodCRC\n  Step 2: Retry 1 within tReceive max + tRetry (first 2 retries)\n  Step 3: 3rd→4th SrcCap within tTypeCSendSourceCap (100–200ms)\n  Step 4: Retry after 4th SrcCap within tReceive max + tRetry\n  Check intervals:\n    1st→2nd: tReceive max + tRetry\n    2nd→3rd: tReceive max + tRetry\n    3rd→4th: tTypeCSendSourceCap min(100.9ms) to max(201.1ms)\n    4th→5th: tReceive max + tRetry\n  Cite: [TEST.PD.PROT.SRC3.1#1, TEST.PD.PROT.SRC3.1#2]"]

    subgraph PASS_CRITERIA10["Pass Criteria"]
        P10A["✓ Correctly describes 5-message retransmission pattern"]
        P10B["✓ States tTypeCSendSourceCap applies to 3rd→4th interval ONLY"]
        P10C["✓ Gives correct numeric range: 100.9ms – 201.1ms"]
        P10D["✓ Does NOT confuse with SenderResponseTimer"]
    end
```

---

#### TEST.PD.PROT.SRC3.2 — SenderResponseTimer Timeout

```mermaid
flowchart TD
    TQ11["Query:\n'What is the exact timing check for\nSenderResponseTimer expiry in Source?'"]

    FILTER11["Metadata Filters:\n  test_id = TEST.PD.PROT.SRC3.2\n  timer_name = tSenderResponse\n  rule_category = timing\n  uut_type = SRC"]

    TQ11 --> FILTER11 --> GRAPH11["Expected Graph Retrieval:\n  TestCase[TEST.PD.PROT.SRC3.2]\n  → VALIDATES → Timer[tSenderResponse]\n  Timer: min=24ms, max=30ms\n  Measurement: last bit of GoodCRC EOP\n  → to last bit of Hard Reset EOP\n  → REFERENCES → SpecClause[6.6.2 SenderResponseTimer]"]

    GRAPH11 --> ANS11["Expected Answer:\n  tSenderResponse: 24ms (min) to 30ms (max)\n  PLUS tHardResetComplete max\n  Measurement start: last bit of GoodCRC EOP (sent by Tester)\n  Measurement end: last bit of Hard Reset EOP (received from UUT)\n  Hard Reset must complete within this window\n  Cite: [TEST.PD.PROT.SRC3.2#1, TEST.PD.PROT.SRC3.2#2,\n    spec §6.6.2, CTS Table 19 Timing Calculations]"]

    subgraph PASS_CRITERIA11["Pass Criteria"]
        P11A["✓ States tSenderResponse = 24–30ms"]
        P11B["✓ Includes tHardResetComplete max in total window"]
        P11C["✓ Measurement anchored to GoodCRC EOP (not SrcCap EOP)"]
        P11D["✓ Cites CTS Table 19 timing table"]
    end
```

---

### TEST GROUP 3: Extended Message Tests (PD3 Mode)

---

#### TEST.PD.PROT.SRC3.3 — Get_Source_Cap_Extended Response

```mermaid
flowchart TD
    TQ12["Query:\n'How should a Source respond to\nGet_Source_Cap_Extended?'"]

    FILTER12["Metadata Filters:\n  test_id = TEST.PD.PROT.SRC3.3\n  ams_type = SourceCapNegotiation\n  doc_type IN [spec, cts]\n  uut_type IN [SRC, DRP]"]

    TQ12 --> FILTER12 --> GRAPH12["Expected Graph Retrieval:\n  TestCase[TEST.PD.PROT.SRC3.3]\n  → VALIDATES → Message[Source_Capabilities_Extended]\n  → REFERENCES → SpecClause[6.5.1 SrcCap_Ext]\n  Branching:\n    IF supports Ext → send Source_Capabilities_Extended\n    IF not supported → send Not_Supported\n  VIF check: Has_Invariant_PDOs → GOVERNS → COMMON.PROC.PD.18"]

    GRAPH12 --> ANS12["Expected Answer:\n  Source responds with Source_Capabilities_Extended OR Not_Supported\n  VIF condition: check Has_Invariant_PDOs; if N and SrcCaps\n    don't match VIF → run COMMON.PROC.PD.18 first\n  Cite: [TEST.PD.PROT.SRC3.3#1, spec §6.5.1,\n    VIF: Has_Invariant_PDOs]"]

    subgraph PASS_CRITERIA12["Pass Criteria"]
        P12A["✓ Identifies both valid responses (Ext or Not_Supported)"]
        P12B["✓ Mentions VIF Has_Invariant_PDOs check"]
        P12C["✓ References COMMON.PROC.PD.18 for PDO mismatch handling"]
    end
```

---

#### TEST.PD.PROT.SRC3.6 — Soft Reset Sent when SinkTxOK

```mermaid
flowchart TD
    TQ15["Query:\n'When must a Source send Soft Reset in response\nto an unacknowledged Source_Capabilities?'"]

    FILTER15["Metadata Filters:\n  test_id = TEST.PD.PROT.SRC3.6\n  rule_category = protocol\n  timer_name IN [tReceive, tSoftReset]"]

    TQ15 --> FILTER15 --> GRAPH15["Expected Graph Retrieval:\n  TestCase[TEST.PD.PROT.SRC3.6]\n  Trigger: Source sends SrcCap; Tester doesn't reply GoodCRC\n  → TRIGGERS → Message[Soft_Reset]\n  Timing: tReceive max + tSoftReset max\n  Measurement: last bit of retransmitted SrcCap EOP\n  → to last bit of Soft Reset EOP"]

    GRAPH15 --> ANS15["Expected Answer:\n  Soft Reset sent within tReceive max + tSoftReset max\n  measured from the last retransmitted Source_Capabilities EOP\n  Condition: SinkTxOK (Rp value indicates OK to send)\n  Cite: [TEST.PD.PROT.SRC3.6#1]"]

    subgraph PASS_CRITERIA15["Pass Criteria"]
        P15A["✓ Cites tReceive + tSoftReset combined window"]
        P15B["✓ Measurement anchored to last retransmission EOP"]
        P15C["✓ Mentions SinkTxOK Rp condition"]
    end
```

---

### TEST GROUP 4: Common Checks — GoodCRC and Message Header

---

#### COMMON.CHECK.PD.3 — GoodCRC Sequence Validation

```mermaid
flowchart TD
    TQ_GCR["Query:\n'What are all the checks on GoodCRC\nduring Source Capabilities sequence?'"]

    FILTER_GCR["Metadata Filters:\n  test_id = COMMON.CHECK.PD.3\n  doc_type = cts\n  ams_type = SourceCapNegotiation"]

    TQ_GCR --> FILTER_GCR --> GRAPH_GCR["Expected Graph Retrieval:\n  TestCase[COMMON.CHECK.PD.3]\n  → VALIDATES → Message[GoodCRC]\n  Checks:\n    1. GoodCRC received within tTransmit max\n    2. MessageID matches the message being ACKed\n    3. Port Power Role bit correct for SRC\n    4. Specification Revision matches sent message\n    5. Port Data Role bit correct\n  Measurement: last bit of Tester message EOP\n  → first bit of UUT GoodCRC preamble"]

    GRAPH_GCR --> ANS_GCR["Expected Answer:\n  5 checks on GoodCRC:\n  1. Timing: within tTransmit max after EOP\n  2. MessageID = MessageID of acked message\n  3. Port Power Role = Source (for SOP)\n  4. Spec Revision = same as sent message\n  5. Port Data Role = UUT's current data role\n  Cite: [COMMON.CHECK.PD.3#1, COMMON.CHECK.PD.3#2]"]

    subgraph PASS_CRITERIA_GCR["Pass Criteria"]
        PGCRA["✓ All 5 check dimensions identified"]
        PGCRB["✓ Timing anchor is EOP of tester-sent message"]
        PGCRC["✓ Does NOT include Spec Revision check for PD2 GoodCRC in PD3 mode (conditional rule)"]
    end
```

---

#### COMMON.CHECK.PD.7 — Source Capabilities Validity

```mermaid
flowchart TD
    TQ_CC7["Query:\n'What checks does COMMON.CHECK.PD.7 perform\non Source Capabilities message content?'"]

    FILTER_CC7["Metadata Filters:\n  test_id = COMMON.CHECK.PD.7\n  doc_type = cts\n  uut_type = SRC"]

    TQ_CC7 --> FILTER_CC7 --> GRAPH_CC7["Expected Graph Retrieval:\n  TestCase[COMMON.CHECK.PD.7]\n  → VALIDATES → Message[Source_Capabilities]\n  Checks:\n    PDOs match VIF Num_Src_PDOs\n    First PDO is Fixed 5V\n    Src_PDO_PDP_Rating consistent with Guaranteed PDP\n    Source Capabilities match VIF when Has_Invariant_PDOs=Y\n    EPR PDOs only when EPR_Supported_As_Src=YES\n  VIFFields: [Has_Invariant_PDOs, Num_Src_PDOs,\n    Src_PDO_*, EPR_Supported_As_Src]"]

    GRAPH_CC7 --> ANS_CC7["Expected Answer:\n  Checks PDO count vs VIF Num_Src_PDOs\n  Verifies 5V Fixed PDO is first in list\n  If Has_Invariant_PDOs=Y: SrcCaps must match VIF exactly\n  If Has_Invariant_PDOs=N: run COMMON.PROC.PD.18\n  EPR PDOs only valid if EPR_Supported_As_Src=YES\n  Cite: [COMMON.CHECK.PD.7, VIF §3.2.9, spec §6.5.2]"]
```

---

### TEST GROUP 5: Bring-Up Procedure Validation

---

#### COMMON.PROC.BU.1 — Standard Source Bring-Up

```mermaid
sequenceDiagram
    participant T as Tester
    participant U as UUT (Source)
    participant V as Verification Check

    Note over T,U: COMMON.PROC.BU.1 start

    T->>U: Apply Rd (simulate Sink attach)
    T->>U: Wait for VBUS (vSafe5V)

    U->>T: Source_Capabilities ← must arrive within tFirstSourceCap max (250ms)
    V->>V: [COMMON.PROC.BU.1#1]: FAIL if tFirstSourceCap violated
    T->>U: GoodCRC

    Note over T,U: SenderResponseTimer starts

    T->>U: Request [Contract PDO from VIF]
    U->>T: GoodCRC
    U->>T: Accept
    T->>U: GoodCRC

    U->>T: PS_RDY ← within tSrcReady max
    T->>U: GoodCRC

    Note over T,U: Explicit contract established. BU.1 complete.

    V->>V: Run COMMON.CHECK.PD.2 (Message Header)
    V->>V: Run COMMON.CHECK.PD.3 (GoodCRC)
    V->>V: Run COMMON.CHECK.PD.7 (Source Capabilities content)
```

**RAG Query for this procedure:**

```mermaid
flowchart LR
    Q_BU1["Query:\n'What is the full bring-up sequence\nfor a Source UUT in COMMON.PROC.BU.1?'"]

    FILTER_BU1["Filters:\n  test_id = COMMON.PROC.BU.1\n  doc_type = cts\n  uut_type = SRC"]

    Q_BU1 --> FILTER_BU1 --> GRAPH_BU1["Graph Retrieval:\n  Procedure[COMMON.PROC.BU.1]\n  → CONTAINS → Step[Apply Rd]\n  → CONTAINS → Step[Wait tFirstSourceCap]\n  → CONTAINS → Step[Send Request]\n  → CONTAINS → Step[Expect PS_RDY]\n  → REFERENCES → COMMON.CHECK.PD.7\n  → REFERENCES → COMMON.CHECK.PD.3"]

    GRAPH_BU1 --> ANS_BU1["Expected Answer:\n  Step-by-step BU.1 sequence with all check anchors\n  Cites all COMMON.PROC.BU.1#N check IDs\n  No hallucinated steps\n  Timing values from spec tables referenced"]
```

---

## VIF-Conditioned Test Routing

```mermaid
flowchart TD
    Q_VIF["Query:\n'Which SourceCap tests apply given:\nPD_Port_Type=Provider_Only,\nHas_Invariant_PDOs=N,\nEPR_Supported_As_Src=YES?'"]

    META_VIF["Metadata Filters:\n  doc_type IN [vif, cts]\n  uut_type = SRC\n  pd_mode = PD3"]

    Q_VIF --> META_VIF --> GRAPH_VIF["Graph Traversal:\n  VIFField[PD_Port_Type=Provider_Only]\n  → ROUTES_TO → COMMON.PROC.BU.1 (not BU.7)\n\n  VIFField[Has_Invariant_PDOs=N]\n  → TRIGGERS → COMMON.PROC.PD.18\n    for every test requiring SrcCap validation\n\n  VIFField[EPR_Supported_As_Src=YES]\n  → INCLUDES → EPR SrcCap test procedures\n  → REQUIRES → EPR_Source_Capabilities check\n  → VIF: Num_Src_PDOs < total PDO count"]

    GRAPH_VIF --> ANS_VIF["Expected Answer:\n  Bring-up: COMMON.PROC.BU.1\n  Before each test: run COMMON.PROC.PD.18\n    (PDO mismatch re-negotiation)\n  Additional EPR tests applicable:\n    TEST.PD.EPR.SRC3.*, COMMON.PROC.PD3.5\n  Has_Invariant_PDOs=N means PDO check\n    cannot use simple equality comparison\n  Cite: [VIF §Has_Invariant_PDOs, CTS §5.3.2]"]
```

---

## End-to-End Test Execution Plan

```mermaid
flowchart TD
    subgraph PREP["Preparation: Index Validation"]
        PR1["Verify graph contains AMS[SourceCapNegotiation]\n(entity type = AMS, ams_type = SourceCapNegotiation)"]
        PR2["Verify all 10 CTS test cases are\nindexed as TestCase entities\n(TEST.PD.PROT.SRC3.1 through SRC3.6, SRC.1-SRC.6)"]
        PR3["Verify all timers indexed:\ntFirstSourceCap, tSenderResponse,\ntTypeCSendSourceCap, tReceive, tRetry"]
        PR4["Verify cross-links exist:\nTestCase → VALIDATES → AMS\nTestCase → CHECKS → Timer/Rule\nRule → REFERENCES → SpecClause"]
        PR5["Verify VIF fields indexed:\nHas_Invariant_PDOs, Num_Src_PDOs,\nEPR_Supported_As_Src, PD_Port_Type"]
    end

    subgraph EXEC["Execution: Query Tests"]
        E1["Run 10 compliance queries\n(one per CTS test case above)"]
        E2["Run 5 timer-specific queries\n(values, measurement anchors, combined windows)"]
        E3["Run 3 VIF-conditioned routing queries"]
        E4["Run 2 sequence violation queries\n(Reject path, Soft Reset path)"]
        E5["Run 1 RCA traversal query\n(SenderResponseTimer expiry RCA)"]
    end

    subgraph VAL["Validation: Answer Quality Checks"]
        V1["Citation completeness:\nEvery answer cites at least one\n[TEST.PD.PROT.* check anchor]"]
        V2["No hallucination:\nNo timer values beyond spec\ntFirstSourceCap, tSenderResponse bounds"]
        V3["Graph grounding:\nAnswer references entity IDs\n(ams_id, rule_id, clause_id)"]
        V4["Metadata filter effectiveness:\nSRC tests do NOT bleed into SNK\nPD3 tests do NOT appear in PD2 queries"]
        V5["Cross-document consistency:\nVIF field constraints align with\nCTS procedure applicability"]
        V6["Sequence ordering:\nAll multi-step sequences in\ncorrect AMS-defined order"]
    end

    subgraph FAIL_MODES["Expected Failure Modes to Test"]
        FM1["Timer value confusion:\nQuery mixing tSenderResponse vs\ntTypeCSendSourceCap → must return distinct values"]
        FM2["SRC vs SNK confusion:\nQuery for Source test must NOT\nreturn Sink-specific procedures"]
        FM3["PD2 vs PD3 mode:\nTEST.PD.PROT.SRC3.* tests are PD3 only\n→ must filter PD2 queries appropriately"]
        FM4["EPR boundary:\nQueries on SPR Source tests must NOT\ncontain EPR AMS steps unless UUT\nhas EPR_Supported_As_Src=YES"]
    end

    PREP --> EXEC --> VAL
    VAL --> FAIL_MODES
```

---

## Timer Lookup Test (Dedicated Vector Test)

```mermaid
flowchart LR
    TQ_T["Query:\n'Give me exact values and measurement\nanchors for all timers in\nSourceCapNegotiation AMS'"]

    FILTER_T["Metadata Filters:\n  ams_type = SourceCapNegotiation\n  doc_type = spec\n  entity_type = Timer"]

    TQ_T --> FILTER_T --> VEC_T["Vector Search:\n  Search entity.description embeddings\n  Filter: entity_type=Timer,\n  ams_type=SourceCapNegotiation\n\n  Search text_unit embeddings:\n  Filter: doc_type=spec, clause_id~=6.6.*\n  (timer definition sections)"]

    VEC_T --> ANS_T["Expected Answer Table:\n  tFirstSourceCap: — / 250ms\n    Anchor: VBUS vSafe5V present\n  tSenderResponse: 24ms / 30ms\n    Anchor: last bit of GoodCRC EOP (Source sent)\n  tTypeCSendSourceCap: 100ms / 200ms\n    Anchor: retry interval for SourceCapabilityTimer\n  tReceive: — / 1.1ms\n    Anchor: EOP to first preamble bit\n  tSrcReady (SPR): — / 285ms\n    Anchor: Accept GoodCRC EOP\n  Source: [spec §6.6, CTS Appendix F Timing Table]"]

    subgraph PASS_T["Pass Criteria"]
        PT1["✓ All 5 timers returned"]
        PT2["✓ Each has correct min AND max values"]
        PT3["✓ Each has correct measurement anchor point"]
        PT4["✓ tSenderResponse and tTypeCSendSourceCap NOT confused"]
    end
```

---

## Pass/Fail Summary Matrix

```mermaid
graph LR
    subgraph MATRIX["Test Query Pass Criteria Summary"]
        ROW1["TEST.PD.PROT.SRC.1 ✓ SrcCap timing cited\n✓ tFirstSourceCap value correct\n✓ GoodCRC handshake present"]
        ROW2["TEST.PD.PROT.SRC.2 ✓ Hard Reset as timeout response\n✓ tSenderResponse 24-30ms\n✓ Measurement anchor: GoodCRC EOP"]
        ROW3["TEST.PD.PROT.SRC3.1 ✓ 5-message retransmit pattern\n✓ tTypeCSendSourceCap on 3rd→4th ONLY\n✓ Numeric values correct"]
        ROW4["TEST.PD.PROT.SRC3.2 ✓ tSenderResponse + tHardResetComplete\n✓ Anchor = GoodCRC EOP not SrcCap EOP"]
        ROW5["COMMON.CHECK.PD.3 ✓ All 5 GoodCRC checks\n✓ Timing anchor = tester EOP\n✓ MessageID match check present"]
        ROW6["COMMON.CHECK.PD.7 ✓ VIF PDO count check\n✓ Has_Invariant_PDOs routing\n✓ EPR PDO condition"]
        ROW7["VIF Routing ✓ BU.1 vs BU.7 selection\n✓ COMMON.PROC.PD.18 trigger\n✓ EPR test inclusion condition"]
    end
```
