> A system that classifies roughly **15,000 public audit results** with AI and provides search and statistics by category, task, and region.

Developed as a research project for the Board of Audit and Inspection of Korea at Hallym University's Intelligent Decision Systems Lab (LIT LAB). I handled the backend and data pipeline in a three-person team.

## My contribution and implementation

- **Classification criteria and prompts:** Wrote prompts to classify each audit finding into category → task → subtask and made it possible to compare predictions alongside practitioner labels.
- **Search backend:** Built FastAPI endpoints for filtered search, statistical aggregation, and links to original documents; deployed the service on AWS.
- **Coordination and handover:** Coordinated requirements and review results with the audit agency and handed over the source code and execution procedure.

The roughly 15,000 records refer to the **public audit results built into the system**. Classification agreement was evaluated on a separate sample of 136 records.

![Home screen of the public audit results viewer](/projects/AUDIT/audit-home.png)

For this screenshot, an available JSON file was loaded into a separate Docker PostgreSQL database and the original FastAPI and React applications were run locally. The JSON file and the roughly 15,000-record deployment cover different counting scopes.

## Problem and approach

Audit practitioners had difficulty finding past cases because the results lacked consistent category and task labels. Public results were scattered across agencies and years, and titles alone did not reveal the work involved.

We collected these results, applied **consistent classification labels**, and built search and statistics views that non-specialists could use.

---

## Data pipeline

```text
Collect public audit results
   |
   +-- Clean and deduplicate --- File hashes identify the same document
   |                             even when it arrives through multiple paths
   |
   +-- AI classification ------- Batch calls to GPT-4.1-mini
   |                             category -> task -> subtask
   |
   +-- Store both labels ------- AI predictions and original practitioner
   |                             labels stay in separate columns for review
   |
   +-- Apply feedback ---------- Revise the classification scheme itself
   |                             from practitioner review
   |
   v
Load into PostgreSQL; store source PDFs in S3
```

### Why retain the original labels?

AI labels may disagree with practitioner decisions. Overwriting one with the other would make errors impossible to trace and remove the evidence needed to improve the scheme. Storing both values side by side allowed feedback to inform changes to the classification criteria.

### Why use a batch process?

Calling the model separately for roughly 15,000 records would increase time and cost. A batch script processes groups of documents and can resume from an interrupted point.

> Classification is a **separate batch operation performed once before loading** and is not included in the linked viewer repository. That repository contains the API and UI for already classified data.

---

## Search and statistics

The application makes classified records directly usable by practitioners.

| Feature | Behavior |
|---|---|
| **Filtered search** | Filter by category, task, audit type, and agency. |
| **Category and task charts** | Show counts as bar and pie charts and a drill-down donut chart. |
| **Regional map** | Select a province or metropolitan area and view counts by category. |
| **Comparison** | Compare results from multiple filter sets. |
| **Source documents** | Open the audit result PDF in the viewer. |
| **Excel export** | Export search results. |

The backend aggregates counts by category and task; the frontend displays regional maps and charts.

![Public audit search filters](/projects/AUDIT/audit-viewer.png)

The search screen is followed by regional and category statistics computed by the API from the same local JSON data.

![Regional public audit map and category statistics](/projects/AUDIT/audit-map.png)

![Public audit category bar chart](/projects/AUDIT/audit-chart.png)

---

## Technology

| Layer | Technology |
|---|---|
| **Backend** | FastAPI · SQLAlchemy · Pydantic Settings |
| **Database** | PostgreSQL · Alembic migrations |
| **Classification** | OpenAI GPT-4.1-mini batch calls |
| **Storage** | AWS S3 for audit-result PDFs |
| **Frontend** | React · Vite |
| **Visualization** | Bar and pie charts, drill-down donut chart, GeoJSON map |
| **Infrastructure** | AWS deployment and domain configuration |

---

## Validation and outcomes

Built a service to classify and search roughly **15,000 public audit results**, operated it on AWS, and handed over the source code and execution procedure.

The final evaluation used a **preprocessed sample of 136 records**. The figures below are agreement with reference labels after GPT classification and post-processing. Percentages are the confirmed agreement counts divided by 136 and rounded to one decimal place.

| Classification level | Matches with reference labels | Agreement |
|---|---:|---:|
| Category | **121/136** | **89.0%** |
| Task | **108/136** | **79.4%** |
| Subtask | **65/136** | **47.8%** |

Alongside the lower subtask agreement, reviewers observed over-generalization to broader categories, incorrect mappings, and numbering-system errors. Ambiguous classification boundaries may also have contributed.

## Limitations and reflection

The 136-record result should not be presented as accuracy across all roughly 15,000 records. Keeping AI predictions separate from the reviewer's original labels made it possible to revise criteria and compare results during operation. The internal evaluation sheet and raw data are not public.
