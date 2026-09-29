> A multi-agent desktop application for internal information security self-audits, with employee consent and an opportunity to respond to findings.

A Hallym University School of Software capstone project at the Intelligent Decision Systems Lab (LIT LAB). I led the three-person team and handled the architecture, multi-agent analysis, ETL pipeline, and presentation.

## My contribution and implementation

- **Evidence ETL:** Built a 16-stage pipeline that processes a provided CTF C-drive dataset and loads evidence into PostgreSQL, Qdrant, and Neo4j according to each store's purpose.
- **Agent system:** Designed and implemented the overall orchestration and result synthesis, and built the baseline, behavioral analysis, and counter-evidence agents.
- **Validation data:** Created three simulated scenarios and expected risk levels in virtual machines, then checked that the system's decisions matched all three.
- **Team leadership and presentation:** Coordinated development direction, prepared the presentation materials, and delivered the final presentation.

The measured change from about **8 minutes to 2–3 minutes** applies to the **agent analysis stage**. Agreement on three simulated scenarios is not a measure of detection accuracy in a real organization.

![AUTH system flow from evidence collection to agent analysis and employee response](/projects/AUTH/system-flow-en.svg)

The diagram summarizes how evidence collection, ETL, storage, agent analysis, and result review connect.

## Project overview

**AUTH** is an **internal information security self-audit system** designed for employees to consent to periodic AI-assisted reviews of their work activity and respond to the findings.

Rather than one-sided surveillance, the goal is a **transparent self-audit process based on employee consent** that helps an organization strengthen compliance.

### Who uses it?

| Role | Workflow |
|---|---|
| **Employee** | Join a quarterly review → electronically sign consent forms → inspect AI findings → submit a written response if needed. |
| **Administrator** | Check submitted reports in the inbox → review explanation, evidence, network graph, and timeline → download a PDF report → mark the review complete. |

---

## Core features

### Employee workflow

```mermaid
flowchart LR
    accTitle: AUTH employee workflow
    A["1. Sign in"] --> B["2. Consent form<br/>e-signature 1"]
    B --> C["3. Consent form<br/>e-signature 2"]
    C --> D["4. ETL<br/>pipeline"]
    D --> E["5. AI analysis<br/>and report"]
    E --> F["6. Write response"]
    F --> G["7. Submit"]

    classDef step fill:#e8f4ff,stroke:#3b82f6;
    class A,B,C,D,E,F,G step;
```

- **Sign-in:** Employee ID.
- **E-signature 1:** Consent to inspect the work PC, files, and company email.
- **E-signature 2:** Consent to access messenger and personal email.
- **ETL → AI analysis:** Live progress through SSE.
- **Report review:** Suspicious files, email, and behavior patterns visualized.
- **Response:** Free-text explanation of the AI finding.

### Administrator workflow

```mermaid
flowchart LR
    accTitle: AUTH administrator workflow
    A["1. Administrator<br/>sign-in"] --> B["2. Dashboard<br/>submitted reports"]
    B --> C["3. Detailed analysis<br/>report + response"]
    C --> D["4. Evidence review<br/>network + timeline"]
    D --> E["5. Download<br/>PDF / PNG"]
    E --> F["6. Mark reviewed"]

    classDef step fill:#fff7e6,stroke:#f59e0b;
    class A,B,C,D,E,F step;
```

- **Dashboard:** Filterable inbox by status and need for an employee response.
- **Detailed review:** Employee report and response, suspicious files and email bodies in modals, evidence network graph, and behavioral timeline.
- **Downloads:** PDF report and PNG network graph.
- **Completion:** Change the review status in the employee inbox.

---

## System architecture

```mermaid
flowchart TB
    accTitle: AUTH system architecture
    subgraph CLIENT["Desktop client"]
        ELECTRON["Electron app<br/>(React + TypeScript)"]
    end

    subgraph API["API layer"]
        FASTAPI["FastAPI<br/>(REST + SSE progress stream)"]
    end

    subgraph PROCESS["Processing layer"]
        ETL["ETL pipeline<br/>(collect, transform, embed evidence)"]
        AGENT["Multi-agent analysis<br/>(LangGraph Supervisor)"]
    end

    subgraph STORE["Storage"]
        PG[(PostgreSQL)]
        QD[(Qdrant vectors)]
        NEO[(Neo4j graph)]
    end

    CLIENT <--> FASTAPI
    FASTAPI --> ETL
    FASTAPI --> AGENT
    ETL --> PG & QD & NEO
    AGENT --> PG & QD & NEO

    classDef client fill:#e8f4ff,stroke:#3b82f6;
    classDef api fill:#fff7e6,stroke:#f59e0b;
    classDef proc fill:#fdf2f8,stroke:#ec4899;
    classDef store fill:#f5f3ff,stroke:#8b5cf6;

    class ELECTRON client;
    class FASTAPI api;
    class ETL,AGENT proc;
    class PG,QD,NEO store;
```

---

## ETL pipeline in brief

A 16-stage pipeline scans one evidence disk and loads data into **three kinds of stores: relational, vector, and graph databases** (`STAGES_BASE` in `etl/pipeline.py`).

```mermaid
flowchart LR
    accTitle: AUTH ETL pipeline
    A["1. Collect and classify<br/>scan + classify"]
    B["2. Extract and transform<br/>mail, documents, speech, images<br/>(9 stages)"]
    C["3. Index and structure<br/>embeddings, entities, graph"]
    D["4. Analyze and audit<br/>audit_findings + audit"]
    E[("PostgreSQL<br/>Qdrant<br/>Neo4j")]
    A --> B --> C --> D --> E
```

| Conceptual step | Actual stages | Purpose |
|---|---|---|
| 1. Collect and classify | `scan`, `classify` | Scan the disk and group files and mail by type. |
| 2. Multi-format extraction and transformation | Nine stages | PST/OST, HWP↔HWPX, DOC↔DOCX, STT (CLOVA), and vision. |
| 3. Index and structure | `upstage_embeddings`, `entity_extract`, `graphdb_load` | Embeddings → entities → graph load. |
| 4. Analyze and audit | `audit_findings`, `audit`, `complete` | Quality audit and completion. |

### Rerun only the failed stage

All 16 stages use **one execution wrapper**.

```python
def run_stage(job_id, stage_name, fn, *args) -> dict:
    set_stage(job_id, stage_name, "running")
    try:
        result = fn(*args) or {}
    except Exception as exc:
        set_stage(job_id, stage_name, "failed", error=str(exc)[:500])
        raise                       # Retain the failure point in the database.
    set_stage(job_id, stage_name, "done", processed=..., success=..., failed=...)
```

Stage state and elapsed time are recorded in `ingest_stage_runs`, allowing a job to **resume from the point of interruption** instead of reprocessing a disk of tens of gigabytes. Optional flags (`process_audio`, `process_images`, `process_embeddings`, `process_entities`, `process_graphdb`) control individual stages.

---

## Multi-agent analysis

A LangGraph Supervisor dynamically coordinates **five specialist sub-agents**.

### The Main Supervisor's four-step cycle

```mermaid
flowchart TD
    accTitle: AUTH multi-agent analysis
    A(["Investigation request<br/>subject + period"]) --> MAIN

    subgraph MAIN["Main Supervisor (LangGraph)"]
        direction TD
        P1["1. Plan investigation<br/>LLM generates instructions"] --> P2
        P2["2. Delegate to agents<br/>task + instructions as JSON"] --> E

        E["STEP 1. Baseline agent"] --> P2b
        P2b["Parallel delegation<br/>STEP 2, 3, 4"] --> F & G & H

        F["STEP 2. Exfiltration agent"] --> I
        G["STEP 3. Sensitive-file agent"] --> I
        H["STEP 4. Behavior agent"] --> I

        I["3. Synthesize and cross-check<br/>suspected findings"] --> P2c
        P2c["Delegate counter-evidence review"] --> J
        J["STEP 5. Counter-evidence agent"] --> N
    end

    N["4. Generate final LLM report"]

    style MAIN fill:#0d1f35,color:#fff
    style P1 fill:#2a5298,color:#fff
    style P2 fill:#2a5298,color:#fff
    style P2b fill:#2a5298,color:#fff
    style P2c fill:#2a5298,color:#fff
    style I fill:#2a5298,color:#fff
    style E fill:#1a4a6b,color:#fff
    style F fill:#1a4a6b,color:#fff
    style G fill:#1a4a6b,color:#fff
    style H fill:#1a4a6b,color:#fff
    style J fill:#1a4a6b,color:#fff
    style N fill:#7a1e1e,color:#fff
```

### Five specialist agents

| # | Agent | Role | Main stores |
|---|---|---|---|
| 1 | **Baseline** | Establish normal external-mail, file-execution, and activity patterns. | PostgreSQL |
| 2 | **Exfiltration detection** | Analyze messages sent through external mail, personal mail, messengers, and anonymous channels. | PostgreSQL + Qdrant |
| 3 | **Sensitive files** | Classify contracts, personnel files, and confidential documents through semantic vector search. | Qdrant + Neo4j |
| 4 | **Behavioral analysis** | Find out-of-scope access, unusual hours, and concealment patterns. | PostgreSQL |
| 5 | **Counter-evidence** | Recheck all areas for legitimate work and reduce false positives. | PostgreSQL + Qdrant + Neo4j |

### Final risk rating

After synthesis, a **weighted quantitative score** assigns one of four levels: **HIGH / MEDIUM / LOW / CLEAN**.

---

## Validation and outcomes

| Measure | Result |
|---|---|
| **Simulated scenarios** | The expected risk levels matched system decisions in **3/3** virtual-machine scenarios. |
| **Agent analysis stage** | Measured reduction from about **8 minutes to 2–3 minutes** by parallelizing exfiltration, sensitive-file, and behavior agents. |
| **False-positive controls** | The counter-evidence agent checks shared folders and colleagues' documents before unrelated evidence contributes to a decision. |

### Parallel execution

The Baseline Agent first establishes ordinary activity. Three independent agents then run through `ThreadPoolExecutor(max_workers=3)` (`agent/graph.py`). The Counter-evidence Agent runs after their results are synthesized.

```text
Baseline --+--> Exfiltration --+
           +--> Sensitive file -+--> Synthesis --> Counter-evidence --> Final report
           +--> Behavior -------+
           (three parallel workers)
```

### Design decisions

- **Sensitive-file scope:** Adopt as evidence only files under the subject's paths and above a sensitivity threshold, rather than every search hit. This avoids rating someone based on shared folders or a colleague's documents.
- **Baseline window:** Exclude **the final 30 days before departure** from the baseline. Mixing a suspicious period into normal behavior would compromise the reference.
- **Risk-rating authority:** Agents gather evidence; **code calculates the risk level using weights**. The Counter-evidence Agent checks facts and can lower the score.

---

## Technology

| Layer | Technology |
|---|---|
| **Frontend** | React 18 · TypeScript · Vite · Electron · @xyflow/react · @tanstack/react-query |
| **Backend** | FastAPI · LangGraph · LangChain · LangSmith · Pydantic |
| **Stores** | PostgreSQL · Qdrant · Neo4j |
| **AI services** | GPT-family LLM · Upstage embeddings · CLOVA STT |
| **Document parsing** | pdfplumber · python-docx · python-evtx · extract-msg · hwp.js |
| **Infrastructure** | Docker · electron-builder for Windows .exe distribution |

---

## Data handling and privacy

The system is designed around **employee consent for self-audits**.

- **Two e-signatures:** Separate consent for the system review and for messenger/personal-email access.
- **Defined collection scope:** Consent forms state which data are inspected, including file access history, sent email, and messenger logs.
- **Local evidence processing:** Evidence is processed on local Docker infrastructure and is not sent out. **Some text may be transmitted to an external LLM API for inference**, with separate notice when used.

---

## Limitations and reflection

The three virtual-machine scenarios and the measured agent-analysis time describe a bounded validation. Long-term effectiveness in a real organization needs separate evaluation. Some AI inference may use external APIs, so the product is designed to disclose that scope. Evidence review and the chance for an employee response are part of the workflow, not just the risk score.

The project received a Bronze Prize at Hallym University's Spring 2026 SW Capstone Design Competition (SW-Centered University Project, June 5, 2026).
