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
        GM[Google Gemini 2.5 Flash]
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

## Tech Stack

| Layer | Tool | Purpose |
|---|---|---|
| Workflow automation | n8n (cloud) | All five workflow chains |
| AI / LLM | Google Gemini 2.5 Flash | CV scoring, GitHub relevance analysis, assessment scoring |
| Database | Airtable | Jobs, candidates, talent pool |
| Forms | Tally | Application, assessments, opt-in |
| Profile scanning | GitHub REST API | Public repository and README analysis |
| Notifications | Gmail + Slack | Candidate emails + HR alerts |
| Cost | All free tiers | Zero monthly operating cost |



## Implementation Note

This repository documents the system architecture and design 
decisions behind this build. Workflow configurations, AI prompt 
engineering, scoring logic, and integration specifics are not 
included in this public repository.

If you are looking to build something similar or want to discuss 
the architecture in more detail, feel free to reach out.


## Workflow Overview
| Workflow | Workflow Description |
|---|---|
| 1. Application Submission | The first step is about keeping the database clean. When an application comes in, the system immediately checks if it is a duplicate or if the role is still open. If either fails, it stops right there and emails the candidate. Otherwise, it logs the applicant, generates a reference ID, and kicks off the evaluation. This saves HR from digging through hundreds of dead-end applications. |
| 2. AI Scoring | This is where the actual filtering happens. The system pulls the job description and scoring weights, then runs the candidate's CV against them. It returns a base score and flags any missing skills. The most important part here is the hard disqualifiers. If someone applies for a senior finance role without an ICAN certification, they are instantly rejected. It keeps unqualified candidates out of the pipeline completely. |
| 3. GitHub Scanner | Workflow 3 runs only when an applicant includes a GitHub URL in their application. It extracts the username, calls the GitHub REST API to retrieve all public repositories sorted by recency, and fetches the README for each, *(up to thirty repositories per scan)*. The compiled repository data is sent to Gemini alongside the role's requirements. The key output beyond a relevance score is what I called a ***hidden gem flag***: a signal that the system found a repository directly relevant to the role that the applicant never mentioned in their CV or application. This addresses a real pattern where capable candidates undersell their own work. |
| 4. Routing | Once we have a final score, this workflow decides what to do next. Anyone scoring over **75** gets shortlisted, and the talent team is notified on Slack. Mid-range scores (45 to 74) trigger an email with a department-specific technical test. Under 45 is an automatic rejection. A webhook is also included so rejected candidates can opt into a talent pool. It automates the triage phase. |
| 5. Assessment Follow-Up | Following up with people is usually the worst part of HR, <ins>so this handles it asynchronously</ins>. One track listens for completed assessments, grades them, and recalculates the applicant's final score. The second track runs every morning at 8 AM. It checks who hasn't submitted their test, nudges them on day five, and archives them by day eight. Nobody falls through the cracks, and HR doesn't have to remember to chase anyone down. |
