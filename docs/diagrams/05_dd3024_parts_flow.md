# 05 — DD Form 3024 PHA Workflow (Parts A → C2)

> Embed in: L100 Domain Context

```mermaid
flowchart LR
    subgraph PART_A["Part A — Intake & Screening"]
        A1["Behavioral Health Screening"]
        A2["Deployment History"]
        A3["Occupational Exposures"]
        A4["Women's Health Screening"]
        A5["MHA Screening Responses"]
    end

    subgraph PART_B["Part B — Medical Record Review"]
        B1["Problem List"]
        B2["Allergies"]
        B3["Active Medications"]
        B4["Procedure History"]
        B5["Immunization Record"]
        B6["Laboratory Results"]
        B7["Completeness Check"]
    end

    subgraph PART_C1["Part C1 — AI/ML Risk Scoring"]
        C1_1["Preliminary Risk Flag"]
        C1_2["MHA AI/ML Risk Score"]
        C1_3["MHA AI Summarization"]
        C1_4["Discrepancy Flags"]
        C1_5["Queue Priority"]
        C1_6["Throughput Metrics"]
    end

    subgraph PART_C2["Part C2 — Provider Review"]
        C2_1["Composite Risk"]
        C2_2["AI-Drafted Recommendation"]
        C2_3["Provider Approval"]
        C2_4["IMR Determination"]
        C2_5["Commander Notification"]
    end

    PART_A -->|"Input"| PART_B
    PART_B -->|"AI/ML"| PART_C1
    PART_C1 -->|"Provider"| PART_C2

    style PART_A fill:#07a,stroke:#fff,color:#fff
    style PART_B fill:#0a6,stroke:#fff,color:#fff
    style PART_C1 fill:#a60,stroke:#fff,color:#fff
    style PART_C2 fill:#a06,stroke:#fff,color:#fff
```

![DD Form 3024 PHA Workflow](./05_dd3024_parts_flow.svg)
