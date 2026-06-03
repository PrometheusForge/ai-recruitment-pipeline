```mermaid
graph TB
    subgraph Forms["Application Layer — Tally Forms"]
        AF[Job Application Form]
        AS[Assessment Forms × 6]
        TF[Talent Pool Form]
    end

    subgraph Engine["Automation Engine — n8n"]
        WF1[WF1 · Intake & Validation]
        WF2[WF2 · AI CV Scoring]
        WF3[WF3 · GitHub Scanner]
        WF4[WF4 · Routing Decision]
        WF5[WF5 · Assessment Follow-Up]
    end

    subgraph AI["AI Layer"]
        GM[Google Gemini 1.5 Flash]
    end

    subgraph Data["Data Layer — Airtable"]
        DB1[(Jobs Table)]
        DB2[(Candidates Active)]
        DB3[(Talent Pool)]
    end

    subgraph GitHub["Profile Scanner"]
        GH[GitHub REST API]
    end

    subgraph Output["Notification Layer"]
        SL[Slack — HR Alerts]
        ML[Gmail — Candidate Emails]
    end

    AF -->|Webhook| WF1
    WF1 -->|Valid application| WF2
    WF1 -->|Closed role / duplicate| ML
    WF2 -->|CV text + job context| GM
    GM -->|Score + summary| WF2
    WF2 -->|GitHub URL present| WF3
    WF2 -->|No GitHub URL| WF4
    WF3 -->|Repos + READMEs| GM
    GM -->|GitHub score + hidden gem flag| WF3
    WF3 -->|Combined score| WF4
    WF4 -->|Score 75-100| SL
    WF4 -->|Score 75-100| ML
    WF4 -->|Score 45-74| AS
    WF4 -->|Score 0-44| ML
    AS -->|Webhook| WF5
    TF -->|Webhook| WF4
    WF5 -->|Recalculated score| SL
    WF5 -->|Recalculated score| ML
    WF1 & WF2 & WF3 & WF4 & WF5 <-->|Read / Write| DB1
    WF1 & WF2 & WF3 & WF4 & WF5 <-->|Read / Write| DB2
    WF4 <-->|Read / Write| DB3
    GH -->|Repos + READMEs| WF3

    style Forms fill:#1a1a2e,stroke:#4a4a8a,color:#ffffff
    style Engine fill:#0f2027,stroke:#2d6a4f,color:#ffffff
    style AI fill:#1a0a2e,stroke:#7b2d8b,color:#ffffff
    style Data fill:#0a1628,stroke:#1e6091,color:#ffffff
    style GitHub fill:#0d1117,stroke:#656d76,color:#ffffff
    style Output fill:#1a0a0a,stroke:#8b2d2d,color:#ffffff
```
