# gage-knowledge-mcp

A RAG-powered MCP server that lets Claude query GW Allen / Gage Western operational knowledge stored in Dropbox and archived email.

## Problem

GW Allen's operational knowledge is scattered across Dropbox in dozens of file formats — PDFs, Word docs, Excel sheets, scanned forms, SOPs, customer agreements, prove sheets, safety records — and buried in gigabytes of archived Outlook email from current and former employees. This makes it impossible for AI tools to answer questions like:

- "What's the procedure for a tank calibration at a Plains site?"
- "What cert does James Alvarez have and when does it expire?"
- "What were the notes on last month's ONEOK prove sheets?"
- "Did we ever discuss meter drift issues with Apache in 2022?"
- "What did the previous ops manager say about the Midland site?"

## Data Sources

| Source | Format | Volume | Access |
|--------|--------|--------|--------|
| Dropbox operational docs | PDF, Word, Excel, scanned images | Ongoing | Dropbox API or local mount |
| Archived PST files (ex-employees + Joshua's inbox) | `.pst` → extracted `.md` | Gigabytes (one-time) | Local files |
| Live Microsoft 365 mailboxes | Email via Graph API | Ongoing | MS Graph API |

## Solution Architecture

```
┌─────────────────────────────────────────────┐
│              DATA SOURCES                   │
│                                             │
│  Dropbox Files    PST Archives    M365 Mail │
│  (PDF/Word/XLS)   (ex-employee    (live     │
│                    + Joshua)       inbox)   │
└────────┬──────────────┬──────────────┬──────┘
         │              │              │
         ▼              ▼              ▼
┌─────────────────────────────────────────────┐
│           EXTRACTION LAYER                  │
│                                             │
│  pdfplumber     readpst → .eml    MS Graph  │
│  pytesseract    pypff              API      │
│  python-docx    html2text                   │
│  pandas                                     │
└────────────────────┬────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────┐
│         NORMALIZED TEXT STORE               │
│                                             │
│   data/extracted/                           │
│   ├── dropbox/  (mirrors folder structure)  │
│   └── email/    (.md per message)           │
│                                             │
│   One .md or .txt file per source doc —     │
│   inspectable, grep-able, re-embeddable     │
└────────────────────┬────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────┐
│         CHUNKING + EMBEDDING                │
│   OpenAI text-embedding-3-small             │
│   ~500 token chunks, 50 token overlap       │
└────────────────────┬────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────┐
│           LANCEDB VECTOR INDEX              │
└────────────────────┬────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────┐
│            MCP SERVER (Python)              │
└────────────────────┬────────────────────────┘
                     │
                     ▼
         Claude (Claude Code / claude.ai)
```

## Email Ingestion Pipeline (PST + Graph)

### Why extract to text/markdown first

Rather than embedding directly from raw PST binary or API payloads, the pipeline writes clean `.md` files to `data/extracted/email/` as an intermediate step. This means:
- One-time extraction is decoupled from (re-)embedding
- Files are inspectable and grep-able — easy to audit what's in the index
- Can switch embedding models or chunk strategies without re-parsing PSTs
- Dedup by `Message-ID` header works naturally on flat files

### PST extraction (archived email)

```
archive.pst
    ↓  readpst -r -e -o ./data/extracted/email/pst/ archive.pst
    ↓  (produces folder tree of .eml files matching Outlook folder structure)
    ↓  Python email stdlib → parse headers + body
    ↓  html2text → strip HTML, preserve structure
    ↓  write one .md per message
```

Each output file looks like:
```markdown
---
message_id: <abc123@mail.gwallen.com>
from: prev.employee@gwallen.com
to: joshua.murray@donoco.com
date: 2022-08-15
subject: ONEOK meter drift — Midland site
folder: Inbox/Customers/ONEOK
pst_source: josh-archive-2023.pst
---

Body text here, HTML stripped and converted to plain markdown...

---
attachments:
  - meter-drift-report.pdf  (extracted to data/extracted/email/attachments/)
```

### Live M365 mail (Graph API)

- `GET /users/{upn}/messages` with delta tokens for incremental sync
- Same `.md` output format, tagged `source: m365`
- Attachments extracted and fed through the same document parsers (PDF, Word, etc.)
- GW Allen is already on M365 (Graph API used in donoco-journal for Outlook calendar)

### Volume estimate

Gigabytes of PST = likely 100k–500k+ messages. The one-time extraction job will take hours but runs once. After that, only new M365 mail is synced incrementally. Recommend running extraction on a machine with the PST files mounted locally (not over the network).

## Open Design Questions

### 1. Sync Strategy
- **Option A: Dropbox API polling** — periodic pull of changed files, good for automated refresh
- **Option B: Local Dropbox folder watch** — simpler if Dropbox is mounted on the server running this, use `watchdog` or similar
- Which machine hosts this — EC2, a GW Allen workstation, or Joshua's machine?

### 2. Document Parsing
- PDFs: `pdfplumber` (text-based) + `pytesseract` OCR (for scanned/image PDFs — field tickets are likely scanned)
- Word: `python-docx`
- Excel/CSV: `openpyxl` / `pandas` (structure-aware chunking — preserve row context)
- **Key question**: are most field documents scanned images or digital-native PDFs?

### 3. PST File Inventory
- How many PST files total? Whose inboxes?
- Are they already on a local machine or on a network share?
- Any sensitive/personal email that should be excluded from the index?

### 4. Vector Store
- **LanceDB** — file-based, no server, easy to run embedded in the MCP process. Best fit for a single-node setup.
- Leaning toward LanceDB for simplicity; revisit if multi-user search becomes a requirement.

### 5. Embedding Model
- **OpenAI `text-embedding-3-small`** — cheap (~$0.02/1M tokens), good quality
- **Local via Ollama** (`nomic-embed-text`) — free, no API dependency, better for sensitive docs
- Given that email contains confidential business correspondence, offline embedding is worth considering.

### 6. MCP Tools to Expose
```
search_documents(query, source_filter?, folder_filter?, date_from?, date_to?) → chunks + source
search_email(query, from_filter?, date_from?, date_to?) → email chunks + message metadata
list_indexed_files(folder?) → what's in the index
get_document(file_path) → full extracted text of one document
reindex(source?) → trigger incremental re-ingest
```

### 7. Metadata to Index Per Chunk

**Documents:**
- `file_path`, `file_name`, `doc_type`, `last_modified`, `page_number`/`row_range`

**Email:**
- `message_id`, `from`, `to`, `date`, `subject`, `folder`, `pst_source` or `m365`

### 8. Dropbox Folder Structure
Need to map out the actual layout to decide which folders to index and how to tag chunks.
Key folders expected:
- Prove Sheets / Field Tickets
- SOPs / Procedures
- Customer Agreements / Contracts
- Safety Records / Certifications
- Equipment Records

## Tech Stack (Proposed)

| Layer | Choice | Notes |
|-------|--------|-------|
| Language | Python 3.11+ | Natural fit for RAG ecosystem |
| MCP SDK | `mcp` (Anthropic official Python SDK) | |
| Dropbox sync | `dropbox` Python SDK | |
| PST extraction | `readpst` (libpst CLI) + `pypff` | `readpst` for bulk, `pypff` for programmatic |
| Email parsing | Python `email` stdlib + `html2text` | |
| M365 live mail | MS Graph API | delta sync, same creds as donoco-journal |
| PDF parsing | `pdfplumber` + `pytesseract` | |
| Word | `python-docx` | |
| Excel | `openpyxl` / `pandas` | |
| Embeddings | OpenAI `text-embedding-3-small` | swappable to local Ollama |
| Vector store | LanceDB | file-based, no server |
| Chunking | LangChain `RecursiveCharacterTextSplitter` | or custom |
| Package mgmt | `uv` | consistent with donoco-journal |

## Milestones (Draft)

1. **Repo skeleton** — this file, CLAUDE.md, pyproject.toml, folder structure ✓
2. **PST extraction spike** — run `readpst` on one archive, parse `.eml` → `.md`, inspect output quality
3. **Dropbox ingestion spike** — parse 10 sample docs, chunk, embed, store in LanceDB
4. **MCP server v1** — expose `search_documents` and `search_email` tools
5. **Full ingest run** — all PST archives + Dropbox
6. **M365 live sync** — Graph API delta sync for ongoing email
7. **Deploy** — wired into Claude Code / claude.ai MCP settings
8. **Incremental sync** — scheduled re-index for new Dropbox files and new email

## Related Projects

- `donoco-journal` — Donoco internal newspaper; MS Graph credentials already set up (reuse for M365 email sync)
- `operational-gage-mcp` (live in claude.ai) — covers structured data (proves, hours, miles, certs) from Snowflake; this project covers **unstructured** document + email knowledge
