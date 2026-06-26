# HMR Knowledge Base Ingestion Pipeline

A [Claude](https://claude.com/claude-code) **Skill** that turns a list of URLs into a clean,
deduplicated, metadata-tagged documentation corpus — ready for human review and manual upload to
[Flowise](https://flowiseai.com/).

It harvests mobile-hardware documentation (PDF manuals and HTML support pages), converts it to
clean text/Markdown, removes duplicates, and generates structured `.meta.json` files aligned with
HMR's **Five Product Pillars**. A human reviews the result before anything is uploaded to the
`HMR-Chatbot-V4` knowledge base.

---

## Why this exists

Building a retrieval knowledge base from scattered manufacturer manuals and support pages is
repetitive, error-prone work: downloading files, stripping page chrome, avoiding duplicates,
naming things consistently, and tagging each document so it can be retrieved later. This skill
automates the mechanical parts deterministically and reserves the **judgement** parts (summaries,
topic tagging) for the language model — then hands a clean corpus to a human for final approval.

### Design principle: deterministic engine + model judgement

The work is split in two so every run is reproducible:

| Layer | Owner | Responsibilities |
|-------|-------|------------------|
| **Mechanical** | `scripts/ingest.py` | fetch, content-type sniff, PDF text extraction, HTML→Markdown, SHA-256 dedup, filename sanitization, `doc_id` generation, real timestamps, crash-safe state |
| **Judgement** | Claude | clean title, executive summary, pillar mapping, semantic tags, device model |
| **Approval** | Human | review accuracy, drop bad scrapes, move approved files, upload to Flowise |

The model never re-implements HTTP, hashing, or state handling by hand, so two runs of the same
`targets.txt` produce the same files and IDs.

---

## Features

- **Resumable** — progress is tracked in `agent_state.json`, written after *every* URL, so an
  interrupted run continues cleanly.
- **Content-hash deduplication** — SHA-256 is computed on the fetched **bytes before anything is
  written**, so duplicate content is never saved to disk (no orphaned files).
- **Explicit PDF text extraction** — PDFs are extracted to text that feeds metadata generation;
  failures are flagged, never silently skipped.
- **Collision-free naming** — every artifact (content file, extracted text, metadata) shares a
  unique `doc_id` stem, so two documents that share a basename can't overwrite each other.
- **Retry cap** — failed URLs accumulate an attempt counter and are abandoned after 3 tries, so
  dead links don't retry forever.
- **Brand organization with fallback** — files are foldered by detected brand; anything unmatched
  lands in `Misc/` instead of stalling.
- **Polite fetching** — descriptive User-Agent, request timeout, and a courtesy delay between
  remote requests.
- **Schema validation** — `scripts/validate_meta.py` gates the corpus before the human handoff.
- **Human-in-the-loop** — the pipeline never touches Flowise; a person approves every file.

---

## The Five Product Pillars

Each document is tagged with one or more pillars (see
[`hmr-kb-ingestion-pipeline/references/pillars.md`](hmr-kb-ingestion-pipeline/references/pillars.md)
for the full mapping guide):

| Pillar key | Covers |
|------------|--------|
| `1_new_phone_buying_guide` | specs comparison, purchase advice, choosing a new device |
| `2_used_phone_fraud_detection` | counterfeit detection, used-phone inspection, IMEI/serial checks |
| `3_hardware_troubleshooting` | fault diagnosis, repair steps, error codes, hardware issues |
| `4_hardware_education` | how components work, teardowns, technical education |
| `5_accessories_guidance` | chargers, cables, cases, compatibility, accessories |

---

## Installation

### As a Claude Skill

Copy the `hmr-kb-ingestion-pipeline/` folder into your Claude skills directory, or package it as a
`.skill` archive and install it. A pre-built archive is included at the repository root:
`hmr-kb-ingestion-pipeline.skill`.

Once installed, Claude consults the skill automatically when you ask it to ingest manuals, process
a `targets.txt`, or build the HMR knowledge base.

### Dependencies

The scripts run on the **Python 3.8+ standard library** alone. Two optional packages dramatically
improve extraction quality and are used automatically when present:

```bash
pip install pypdf trafilatura
```

- `pypdf` — extracts text from PDF manuals.
- `trafilatura` — extracts clean article content from HTML (strips nav, ads, cookie banners).

Without them the pipeline still runs: PDFs are flagged `EXTRACTION_FAILED` for the reviewer, and
HTML falls back to a crude tag-strip.

---

## Usage

### 1. First-run setup

On first use the skill asks for a staging directory and the path to your `targets.txt`, then writes
`agent_config.json`:

```json
{
  "staging_dir": "C:/HMR_Staging",
  "targets_file": "C:/HMR_Staging/targets.txt",
  "ready_for_flowise_dir": "C:/HMR_Staging/Ready_For_Flowise"
}
```

Populate `targets.txt` with one URL per line (see [`examples/targets.txt`](examples/targets.txt)).
Lines beginning with `#` are ignored.

### 2. Run the pipeline

Claude drives the loop below, one URL at a time. You can also run the engine directly:

```bash
# Show the next URL to process (derived live from targets.txt minus done/abandoned)
python hmr-kb-ingestion-pipeline/scripts/ingest.py next    --config agent_config.json

# Fetch + extract + dedup + save one URL (writes a .meta.json stub)
python hmr-kb-ingestion-pipeline/scripts/ingest.py fetch   --config agent_config.json --url "https://..."

# Record the outcome in state (crash-safe, written immediately)
python hmr-kb-ingestion-pipeline/scripts/ingest.py commit  --config agent_config.json --url "https://..." --status processed
python hmr-kb-ingestion-pipeline/scripts/ingest.py commit  --config agent_config.json --url "https://..." --status failed --reason "HTTP 404"

# Print a session summary
python hmr-kb-ingestion-pipeline/scripts/ingest.py summary  --config agent_config.json
```

Between `fetch` and `commit`, Claude reads the generated `*.extracted.txt` and fills the metadata
stub (title, summary, pillars, tags, device model).

### 3. Validate before handoff

```bash
python hmr-kb-ingestion-pipeline/scripts/validate_meta.py --corpus "C:/HMR_Staging/Corpus"
```

The validator checks JSON validity, that every model-filled field is complete, that pillar keys are
valid, that the tag count is 6–12, and that the described content file exists. It exits non-zero if
anything fails, so it can gate a CI step.

---

## Output layout

```
<staging_dir>/
├── Corpus/
│   └── <Brand>/
│       ├── samsung_s24_manual_pdf_001.pdf            # original content
│       ├── samsung_s24_manual_pdf_001.extracted.txt  # text for the model to read
│       └── samsung_s24_manual_pdf_001.meta.json      # structured metadata
├── Ready_For_Flowise/    # files a human has approved
└── agent_state.json      # run progress (script-managed)
```

### Metadata schema (`.meta.json`)

```json
{
  "doc_id": "samsung_galaxy_s24_ultra_manual_001",
  "brand": "Samsung",
  "device_model": "Galaxy S24 Ultra",
  "source_type": "pdf_manual",
  "source_url": "https://...",
  "local_file_name": "samsung_galaxy_s24_ultra_manual_001.pdf",
  "ingested_timestamp": "2026-06-26T12:00:00Z",
  "content_sha256": "<hash>",
  "hmr_target_pillars": ["3_hardware_troubleshooting"],
  "ai_clean_title": "Samsung Galaxy S24 Ultra User Guide",
  "ai_executive_summary": "Three-paragraph conceptual summary in English.",
  "semantic_tags": [
    "OLED burn-in", "screen ghosting", "battery health",
    "phone won't charge", "overheating", "fake charger"
  ]
}
```

Mechanical fields (`doc_id`, `content_sha256`, `ingested_timestamp`, `source_*`, `local_file_name`,
`source_type`, `brand`) are produced by the engine. The remaining fields are completed by the model.

---

## Human-in-the-loop handoff

The pipeline never uploads anything. After ingestion a person:

1. Reviews files under `Corpus/<Brand>/` and checks `.meta.json` accuracy.
2. Deletes poorly scraped files (and their companion `.meta.json` / `.extracted.txt`).
3. Moves approved files to `Ready_For_Flowise/`.
4. Manually uploads them in the Flowise dashboard → `HMR-Chatbot-V4` → Document Loader node.

---

## Configuration

A few constants at the top of `scripts/ingest.py` can be tuned:

| Constant | Default | Purpose |
|----------|---------|---------|
| `MAX_ATTEMPTS` | `3` | retries before a failing URL is abandoned |
| `POLITE_DELAY_SECONDS` | `1.0` | courtesy pause between remote requests |
| `REQUEST_TIMEOUT` | `30` | per-request timeout (seconds) |
| `BRAND_MAP` | — | host substring → brand name; extend as your source list grows |
| `FALLBACK_BRAND` | `Misc` | folder for unmatched hosts |

---

## Limitations

- **Exact-duplicate detection only.** SHA-256 catches byte-identical content. Pages whose markup
  changes between crawls (timestamps, dynamic blocks) won't be recognized as duplicates.
- **Extraction quality depends on optional libraries.** Install `pypdf` and `trafilatura` for best
  results; without them some documents are flagged for manual handling.
- **No robots.txt enforcement.** The pipeline processes the curated URLs you provide. Be sure you
  have the right to download and store the documents you list.

---

## Repository structure

```
HMR-Knowledge-Base-Ingestion-Pipeline/
├── README.md
├── CLAUDE.md                       # guidance for Claude working in this repo
├── LICENSE
├── .gitignore
├── examples/
│   └── targets.txt                 # sample input
└── hmr-kb-ingestion-pipeline/      # the installable skill
    ├── SKILL.md
    ├── scripts/
    │   ├── ingest.py               # fetch → extract → dedup → save → state engine
    │   └── validate_meta.py        # metadata schema validator
    └── references/
        └── pillars.md              # Five Pillars mapping + tagging guide
```

---

## License

[MIT](LICENSE) © 2026 wikigoo
