> A web service that finds youth policies and scholarships for a user's circumstances, then checks the original announcement for existence and deadline status.

I led a four-person course project at Hallym University and handled the policy verification pipeline, backend system, presentation materials, and final presentation.

## My contribution and implementation

- **Policy processing and verification pipeline:** Designed and built the backend flow that normalizes collected policies and tracks their verification state.
- **Original-announcement check:** Used Playwright and browser-use to visit agency announcements, compare their names and contents, and record distinct outcomes for connection failure, missing announcement, and cases requiring review.
- **Team leadership and presentation:** Coordinated development priorities, prepared the presentation materials, and delivered the final presentation.

![POLYSTEP home screen](/projects/POLYSTEP/fig3-home.png)

## Problem and approach

Young people may miss support programs because **information is scattered and changes often**, not because programs do not exist.

- Policies are spread across public data portals, local governments, and universities.
- Announcements close or change frequently, but **expired listings can remain in search results**.
- A result may be found even when the user can no longer apply.

POLYSTEP goes beyond finding a listing: it **checks whether the original announcement still exists and whether it has closed**, then shows how to apply. If verification fails, the service **shows the reason**.

---

## Policy-source graph

![Graph of youth policy and scholarship websites](/projects/POLYSTEP/fig4-graph.png)

Below the home screen, a graph makes scattered sources visible in one place. **Twenty-five sites** connect to POLYSTEP, including Ontong Youth, the Korea Student Aid Foundation, Government24, Employment24, Bokjiro, and local youth portals.

**Clicking a node opens the source site in a new tab** (`onNodeClick` → `window.open`). Even when POLYSTEP has not indexed a program yet, a visitor can go directly to the source.

The graph uses `force-graph`; resistance in the node-drag interaction keeps nodes from snapping abruptly to the pointer.

---

## Core workflow

![POLYSTEP system architecture](/projects/POLYSTEP/fig1-architecture.png)

```text
1. Enter criteria       2. Find policies       3. Verify source        4. Application guide
   Age, region,            Match conditions        Browser agent visits     Steps, required
   keywords, category      against the database    official announcement   documents, source link
                                                  and checks existence
                                                  and deadline
                                                        |
                                           +------------+------------+
                                        SUCCESS                    FAILED
                                     Show verified             Show failure reason
                                     announcement              (missing / review
                                                                needed / connection)
```

When the team disagreed over adding more features versus finishing the core workflow, we used **“Can the user actually apply?”** as a shared decision rule and reprioritized around these four steps. Features that did not support that goal were left out of scope.

---

## Official-source verification

![Fast Track and Deep Track features and screens](/projects/POLYSTEP/fig2-core-features.png)

Instead of displaying policy details from an LLM's memory, the service **checks the actual announcement**.

| Stage | Processing |
|---|---|
| `PENDING` | Create a verification record. |
| Source visit | A browser agent opens the announcement URL with Playwright and browser-use. |
| Comparison | Check that the policy name and details actually appear on the page. |
| `SUCCESS` | The announcement is confirmed; generate application guidance. |
| `FAILED` | Save and show the reason: `POLICY_NOT_FOUND`, review needed, or connection error. |

**Live screenshots are streamed during verification**, so users can inspect the basis for the guidance.

> Showing failures is intentional. Presenting an unverified policy as confirmed could send an applicant to a closed or nonexistent announcement.

---

## Data collection

| Collector | Source | Data |
|---|---|---|
| `gonggong_api.py` | Public Data Portal | Youth policies |
| `gangone_api.py` | Gangwon Province | Regional youth policies |
| `hallym_broswer.py` | University announcements, via browser automation | Scholarships |

After collection, the data passes through normalization, duplicate detection, expired-policy filtering, and UI preparation.

```text
crawler/collectors/* -> normalize_policies -> check_duplicate_policies
                     -> active-policy filter -> UI preparation
                     -> database normalization -> database load
```

**Final datasets at the time:** **407 youth policies** (`data/policies_cleaned_final.csv`) and **38 scholarships** (`data/scholarship.csv`). Intermediate collection files and regional splits are not in the repository; the collectors can recreate them.

The `artifact_service` extracts text from attachments downloaded during announcement checks. It walks ZIP contents, uses `pypdf` for PDFs and `pytesseract` OCR for images. If a library is unavailable, it skips that step and records the reason in metadata.

For HWP and HWPX, **only extension recognition and download** were implemented. We considered conversion through headless LibreOffice and PDF, but it fell below the semester's priorities. The service marks these files with `HWP/HWPX convert not implemented` so unprocessed attachments remain traceable.

---

## Technology

| Layer | Technology used |
|---|---|
| **Backend** | FastAPI · SQLAlchemy · Pydantic |
| **Database** | PostgreSQL (`psycopg2`) |
| **LLM** | Google Gemini (`google-generativeai`) |
| **Browser automation** | browser-use · Playwright for source verification and scholarship collection |
| **Collection and parsing** | requests for public APIs · pypdf for PDF text · pytesseract for image OCR |
| **Frontend** | React · Vite (JavaScript) · force-graph for the source graph |
| **Authentication** | JWT login (`security.py`) |

> The system does not use a vector database. Policy records are structured enough that **relational filtering plus verification against the original announcement** was a better fit.

---

## Validation and outcomes

The built dataset contained **407 youth policies** and **38 scholarships** at the time, and the team completed a final demonstration. The interface shows verification successes, failures, and failure reasons.

## Limitations and reflection

Confirming an announcement does not determine an individual's application eligibility. HWP/HWPX conversion was not implemented beyond download and extension recognition, and the warning remains visible. We did not measure real-user outcomes or source-verification accuracy.
