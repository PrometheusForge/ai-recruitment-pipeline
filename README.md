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



## Implementation note

This repository documents the system architecture and design 
decisions behind this build. Workflow configurations, AI prompt 
engineering, scoring logic, and integration specifics are not 
included in this public repository.

If you are looking to build something similar or want to discuss 
the architecture in more detail, feel free to reach out.


## workflow overview
Workflow 1 acts as the initial gatekeeper, running immediately when a new application is submitted. It performs crucial validation checks to identify *duplicate submissions* and verify if the requested role is still marked as actively open in the database. If the application fails either check, the candidate receives an automated notification and the process halts; if successful, the system generates a unique reference number, logs the record, sends a confirmation email, and triggers the AI evaluation phase. This ensures the system remains clean, prevents duplicate spam, and guarantees HR is not clogged with redundant applications for closed roles.

Workflow 2 handles the heavy lifting of initial CV evaluation by fetching the specific job description, required skills, and department scoring weights directly from the central database. It evaluates these metrics against the candidate's CV using artificial intelligence, returning a comprehensive assessment that includes a base score, identified skills, and potential red flags. Importantly, this workflow implements hard disqualifier checks—such as instantly rejecting a finance candidate lacking mandatory industry certifications—to definitively filter out legally or technically unqualified candidates before any human review occurs.

Workflow 3 runs only when an applicant includes a GitHub URL in their application. It extracts the username, calls the GitHub REST API to retrieve all public repositories sorted by recency, and fetches the README for each — up to thirty repositories per scan. The compiled repository data is sent to Gemini alongside the role's requirements. The key output beyond a relevance score is what I called a hidden gem flag: a signal that the system found a repository directly relevant to the role that the applicant never mentioned in their CV or application. This addresses a real pattern where capable candidates undersell their own work.

Workflow 4 is the intelligent decision engine that routes candidates based on their final computed score. It categorizes applicants into three distinct paths: top scorers are automatically shortlisted with an assessment summary sent to the talent acquisition team; mid-range scorers are emailed a dynamically selected, department-specific technical assessment; and low scorers receive a warm rejection email. The workflow also seamlessly transitions rejected candidates who opt-in into a dedicated talent pool, completely automating the traditional manual triage process and ensuring immediate, appropriate next steps for every single applicant.

Workflow 5 manages the asynchronous assessment phase and eliminates the need for manual follow-ups by operating on two distinct tracks. The first track listens for assessment submissions, matches them back to the candidate's original record, and uses artificial intelligence to grade the technical answers before recalculating a final, weighted candidate score. The second track operates as a daily automated audit of pending assessments, sending friendly nudge emails to candidates approaching their deadline and archiving those who fail to respond. This design ensures that the evaluation pipeline keeps moving automatically and completely removes the cognitive load of chasing candidates from the HR team.
