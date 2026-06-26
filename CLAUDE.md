# CLAUDE.md

Guidance for Claude (and other AI assistants) working in this repository.

## What this repository is

This repo contains a single Claude **Skill**: the **HMR Knowledge Base Ingestion Pipeline**. The
skill turns a list of URLs (`targets.txt`) into a clean, deduplicated, metadata-tagged corpus of
mobile-hardware documentation, staged for human review before manual upload to Flowise
(`HMR-Chatbot-V4`).

The installable skill lives in [`hmr-kb-ingestion-pipeline/`](hmr-kb-ingestion-pipeline/). Everything
else at the repo root (`README.md`, this file, `LICENSE`, `examples/`) is project scaffolding.

## Core design principle: engine vs. judgement

The work is deliberately split so runs are deterministic and reproducible:

- **`scripts/ingest.py` is the deterministic engine.** It owns all mechanical work: HTTP fetching,
  content-type sniffing, PDF text extraction, HTML→Markdown conversion, SHA-256 deduplication,
  filename sanitization, `doc_id` generation, timestamping, and crash-safe state I/O.
- **You (the model) own only the judgement layer.** After the engine saves a file you read its
  `*.extracted.txt` and complete the null fields in the `*.meta.json` stub: `device_model`,
  `ai_clean_title`, `ai_executive_summary`, `hmr_target_pillars`, `semantic_tags`.

**Do not re-implement the engine's work by hand** (don't fetch with ad-hoc tools, hash manually,
or hand-edit `agent_state.json`). That defeats the determinism the engine guarantees. If a
mechanical capability is missing, add it to the engine rather than working around it.

## How the pipeline runs

One URL at a time, which keeps state crash-safe:

1. `python scripts/ingest.py next --config agent_config.json` → the next URL (or `null`).
2. `python scripts/ingest.py fetch --config agent_config.json --url "<url>"` → fetches, dedups,
   saves the content file + `*.extracted.txt` + a `*.meta.json` stub. Possible statuses:
   `saved`, `skipped_duplicate`, `fetch_failed`.
3. Read the `extracted_text_file` and fill the stub's null fields (see
   [`references/pillars.md`](hmr-kb-ingestion-pipeline/references/pillars.md)).
4. `python scripts/ingest.py commit --config ... --url "<url>" --status processed|failed [--reason ...]`.
5. When the queue is empty: `validate_meta.py --corpus <Corpus>` then `ingest.py summary`.

## Conventions and invariants

- **English only.** All metadata content — titles, summaries, and `semantic_tags` — is written in
  English. (An earlier draft used bilingual Persian/English tags; that was intentionally removed.)
- **`semantic_tags`: 6–12 entries**, mixing technical terms and colloquial/symptom phrasing.
  The validator enforces this range.
- **Pillar keys are exact strings.** Use only the five keys in `references/pillars.md`; the
  validator rejects anything else. Most documents map to a single dominant pillar — don't over-tag.
- **`doc_id` is the filename stem.** Content file, extracted text, and metadata for one document all
  share the same `doc_id` (e.g. `samsung_manual_001.pdf`, `.extracted.txt`, `.meta.json`). The
  engine guarantees `doc_id` uniqueness — never invent your own.
- **Never hand-edit `agent_state.json` or the mechanical metadata fields** (`doc_id`,
  `content_sha256`, `ingested_timestamp`, `source_url`, `source_type`, `local_file_name`, `brand`).
- **Real timestamps come from the engine.** Don't write a date yourself — `ingest.py` stamps
  `ingested_timestamp` in UTC.
- **`EXTRACTION_FAILED`** in `*.extracted.txt` means the engine couldn't extract text (e.g. a
  scanned PDF, or `pypdf` not installed). Leave the summary flagged and note it for the human
  reviewer — do not invent content from the filename or URL.

## Editing the engine

- `scripts/ingest.py` and `scripts/validate_meta.py` are **standard-library Python** at the core.
  Optional imports (`pypdf`, `trafilatura`) must stay optional: guard them with `try/except
  ImportError` and degrade gracefully, because the skill must run without them.
- Tunable constants live at the top of `ingest.py` (`MAX_ATTEMPTS`, `POLITE_DELAY_SECONDS`,
  `REQUEST_TIMEOUT`, `BRAND_MAP`, `FALLBACK_BRAND`). Extend `BRAND_MAP` when new source hosts appear.
- After any change to either script, verify they still compile and pass an offline run:
  `python -m py_compile scripts/ingest.py scripts/validate_meta.py`.

## Testing

There is no network dependency for testing — use `file://` URLs in a temporary `targets.txt` and a
throwaway staging directory to exercise `next` → `fetch` → fill stub → `commit` → `validate_meta`.
Check the key invariants: duplicates are skipped without writing a file, two documents that share a
basename get distinct `doc_id` filenames, failed URLs are abandoned after `MAX_ATTEMPTS`, and the
validator exits non-zero on an incomplete stub.

## What this skill must never do

- It must **never upload to Flowise** or any external service. The final step is always a human
  moving approved files into `Ready_For_Flowise/` and uploading them manually.
- It must **not download content the user has no right to store.** It processes the curated URLs
  provided in `targets.txt`; it is not a broad web crawler.
