> A research system that extracts and translates legal documents that cannot be sent outside the institution, then lets researchers compare the source and results on their own database and GPU server.

Developed for a joint research project with an overseas university laboratory at the Intelligent Decision Systems Lab (LIT LAB), Hallym University.

## My contribution and implementation

- **Entire frontend:** Designed and built the researcher-facing workflow from upload through extraction and translation progress, block-level review, edits, and retries.
- **Extractor change and block review:** Marker-pdf could not reliably structure headers and footers inserted between paragraphs; tagging and LLM post-processing took too long. I replaced it with MinerU and used block coordinates to compare the source PDF with extracted text, then extracted text with its translation. Reviewers can merge a wrongly split block with the next paragraph or split a combined block at the cursor.
- **Translation model and progress:** Selected and integrated TranslateGemma, displayed live processing progress as a percentage, and added retries for failed extraction and translation jobs.
- **Shared GPU scheduling:** Designed a queue so new jobs wait while the GPU is reserved instead of failing. Connected the frontend to the GPU manager API to display reservation and model status periodically.

The system uses Gemma4 to create context and TranslateGemma to translate. The following sections describe how extraction, translation, review, and GPU scheduling fit together.

## Why we built it

The legal decisions used in the joint study are **private documents that cannot be transferred externally**. Commercial services such as DeepL and Google Translate send documents to outside servers and therefore were not options.

Translation quality also mattered: a changed case number or amount could undermine the reliability of the entire document. We therefore ran **extraction, translation, and review on a local GPU server** and built a screen where researchers can compare source and translated text block by block.

---

## Three-stage pipeline

```text
PDF upload
   |
   +-- 1. MinerU ------------ Layout-aware block extraction
   |                          (preserves body, tables, and footnotes)
   |
   +-- 2. Gemma4 ------------ Section context + document-type classification
   |                          1–2 sentences; excludes names
   |                          Nine types, including decisions, precedents,
   |                          pleadings, and evidence
   |
   +-- 3. TranslateGemma ---- Block translation with section context
                              sv -> en / ko
   |
   v
Block alignment -> synchronized side-by-side review -> edits -> download
```

### Why the second stage matters

Translating isolated blocks **loses the surrounding context**, so the same term may be rendered differently across paragraphs. Before translation, Gemma4 summarizes the section in one or two sentences and supplies that context to the translation prompt. The prompt asks only for facts that help disambiguate the upcoming translation and explicitly excludes personal names.

The section is also classified, with a confidence score, as a decision, evidence, pleading, or another document type so the translation can use an appropriate register.

---

## Automatic checks for translation corruption

Changing a legal identifier or number can change a document's meaning. The system checks each translation before accepting it.

| Check | Behavior |
|---|---|
| **Identifier and number comparison** | Compare case numbers, precedent references, and monetary amounts with the source; warn when any value differs. |
| **Length anomaly** | Warn when translated length falls outside 0.25–4 times source length; reject and rerun translation above 8 times source length. |

An identifier mismatch does **not** automatically stop the translation. It points the reviewer to the affected text; a person decides whether the meaning was actually damaged.

---

## Review interface

- **Block alignment:** Source and translated blocks appear in matching pairs.
- **Synchronized scrolling:** Scrolling either side moves the other to the corresponding position.
- **Revision history:** Reviewer edits are retained as revisions.
- **Retry:** A failed extraction can be restarted without uploading the PDF again.

---

## Access control and audit trail

The system handles private documents, so it records **who opened what**.

- Invite-code registration and session authentication, plus Google sign-in integration.
- Project- and document-level ownership.
- Document access and processing history recorded as `AuditEvent` and available to administrators.
- **Allowlisted audit API fields:** Source text, storage paths, and credentials are excluded from responses even if they appear in older records.
- Internal service URLs are restricted to loopback and tunnel hosts by `_validate_private_service_url`; an external URL prevents server startup.

---

## Shared GPU operation

Several researchers share the GPU server. Keeping every model loaded blocks other work, while restarting for every request creates long waits.

The GPU manager **starts a needed model on demand and stops it after sustained idle time**. A researcher changes the reservation state on the GPU server with `litailab-reserve`. The web backend reads that state through the GPU manager API; it does not run the command itself. The frontend polls `/api/system/gpu-status` about every five seconds to show reservation and model status.

- Starting a queued job loads only its required model.
- An idle model is stopped automatically.
- When another researcher reserves the GPU, the current job finishes, its model is stopped, and new jobs **wait in the queue instead of failing**.
- An administrator can inspect GPU status and user activity logs.

---

## Technology

| Layer | Technology |
|---|---|
| **Backend** | FastAPI · SQLAlchemy (async) · Pydantic · Alembic |
| **Database** | PostgreSQL (asyncpg) |
| **Models** | MinerU (PDF extraction) · Gemma4 12B (context) · TranslateGemma 12B (translation), all on the local GPU |
| **Frontend** | React · TypeScript · Vite |
| **Infrastructure** | Docker Compose · NAS deployment · SSH tunnel to GPU server |

---

## Validation and outcomes

Built the MinerU extraction → Gemma4 context → TranslateGemma translation → source comparison and review workflow and deployed it in a NAS Docker environment. Failed job states are retained so work can be retried.

## Limitations and reflection

There is no quantitative evaluation yet for translation accuracy, identifier retention, or processing speed. The document-processing models run on our own GPU, but the entire system should not be described as fully offline because services such as sign-in may use external connections. Extraction quality and human review mattered as much as the translation model.
