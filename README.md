# Document Extraction API

REST API that turns PDFs into structured JSON against a **client-defined schema**, running on
self-hosted LLM inference. Documents never leave the host machine and there is no per-page API
cost.

> **Status: design phase.** This repository currently contains the technical specification only —
> no code yet. [`SPEC.md`](SPEC.md) is written in Vietnamese; this README summarises it in English.

---

## What it does

```
PDF upload → text extraction (PDFBox) → LLM extraction against a JSON schema
          → per-field confidence scoring → business rule validation
          → suspect fields into a human review queue
```

## Two things that make it different

**1. The client defines the schema.** Nothing is hardcoded for invoices. The same binary extracts
invoices, contracts, and delivery notes — only the supplied schema changes. Schemas are a
deliberately constrained subset of JSON Schema, validated on creation.

**2. A field-level accuracy benchmark.** Precision, recall, and F1 per field over a 50-document
labelled set, runnable as a single command, with a model-comparison table. Measured numbers rather
than "it works well" — which is the part most portfolio projects skip, because labelling 50
documents is tedious.

## Architecture

A single Spring Boot process. No microservices, no message broker.

```
HTTP ──▶ Controller ──▶ Service ──▶ PostgreSQL
                 │
                 ├──▶ disk     (original PDF)
                 ├──▶ PDFBox   (PDF → text)
                 └──▶ Ollama   (text + schema → JSON)   HTTP localhost:11434
```

**Stack:** Java 21 · Spring Boot · Spring AI · PostgreSQL · PDFBox · Ollama

## API

Prefix `/api/v1`, JSON in and out. Errors follow RFC 9457 Problem Details via Spring's built-in
`ProblemDetail`.

| Method | Path | Description |
|---|---|---|
| `POST` | `/documents` | Multipart upload; optional `?schemaId=&model=` chains straight into extraction → `202` |
| `GET` | `/documents` | Paginated list, filter by `status` and filename |
| `GET` | `/documents/{id}/text` | Raw extracted text; `409` if not yet `TEXT_READY` |
| `POST` | `/schemas` | Create an extraction schema |
| `POST` | `/documents/{id}/extractions` | Start a run → `202` + run id |
| `GET` | `/extractions/{runId}` | Result JSON, per-field confidence, validation issues |
| `GET` | `/review` | Review queue — low-confidence fields across all documents |
| `PATCH` | `/extractions/{runId}/fields` | Correct several fields in one call |

No `DELETE` in v1. Nobody has needed it.

## Confidence scoring

The model is **not** asked to rate its own output — self-reported confidence from an LLM is not a
measurement. Scores are computed from deterministic signals instead, and the spec requires proving
the score is not decorative: a calibration table showing that low-confidence fields really are
wrong more often than high-confidence ones.

## Design constraints

These are written into the spec to keep the scope from drifting:

- **Every phase must run.** No table or endpoint exists "for a later phase" — with one considered
  exception (`tenant_id`, present from the first migration).
- **No DSL, no plugin system, no abstraction with one implementation.** Specifically: no expression
  language for validation rules, and no `ExtractorStrategy` interface while PDFBox is the only
  extractor.
- **Crude but measurable beats sophisticated but unverifiable.** Applied hardest to confidence
  scoring.

## Out of scope for v1

OCR · vision models · web UI · cloud deployment · non-PDF formats · streaming responses · webhooks ·
chunking long documents (v1 truncates and flags) · fine-tuning · optimal line-item matching (v1
compares in order).

## Roadmap

| Phase | Deliverable |
|---|---|
| 0 | Stack verification |
| 1 | Upload, list, input validation |
| 2 | PDF → text, scanned-PDF detection |
| 3 | Schemas and the extraction core |
| 4 | **Accuracy benchmark** ⭐ |
| 5 | Business rules and review queue |
| 6 | Multi-tenancy and API key auth |
| 7 | Model comparison results |

## License

MIT
