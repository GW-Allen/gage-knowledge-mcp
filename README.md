# gage-knowledge-mcp

A RAG-powered MCP server that lets Claude query GW Allen / Gage Western operational knowledge stored in Dropbox.

## Problem

GW Allen's operational knowledge is scattered across Dropbox in dozens of file formats — PDFs, Word docs, Excel sheets, scanned forms, SOPs, customer agreements, prove sheets, safety records. This makes it impossible for AI tools to answer questions like:

- "What's the procedure for a tank calibration at a Plains site?"
- "What cert does James Alvarez have and when does it expire?"
- "What were the notes on last month's ONEOK prove sheets?"

## Solution Architecture

```
Dropbox Files
    ↓  (Dropbox API or local sync)
Document Ingestion Layer
    ↓  (parse PDFs, Word, Excel, etc.)
Chunking + Embedding
    ↓  (OpenAI text-embedding-3-small or local)
Vector Store (LanceDB)
    ↓
MCP Server (Python)
    ↓
Claude (via Claude Code or claude.ai)
```

## Open Design Questions

### 1. Sync Strategy
- **Option A: Dropbox API polling** — periodic pull of changed files, good for automated refresh
- **Option B: Local Dropbox folder watch** — simpler if Dropbox is mounted on the server running this, use `watchdog` or similar
- Which GW Allen machine (or EC2 instance) will host this?

### 2. Document Parsing
- PDFs: `pdfplumber` (text-based) + `pytesseract` OCR (for scanned/image PDFs — field tickets are likely scanned)
- Word: `python-docx`
- Excel/CSV: `openpyxl` / `pandas` (structure-aware chunking — preserve row context)
- Images: skip or OCR?
- **Key question**: are most field documents scanned images or digital-native PDFs?

### 3. Vector Store
- **LanceDB** — file-based, no server, easy to run embedded in the MCP process. Best fit for a single-node setup.
- **Chroma** — similar simplicity, slightly more ecosystem support
- **pgvector** — if we ever need to colocate with a Postgres DB (overkill here)
- Leaning toward **LanceDB** for simplicity

### 4. Embedding Model
- **OpenAI `text-embedding-3-small`** — cheap (~$0.02/1M tokens), good quality, API call
- **Local via Ollama** (`nomic-embed-text`) — free, runs on-device, no API dependency
- Decision depends on how sensitive the documents are and whether we want offline capability

### 5. MCP Tools to Expose
```
search_documents(query, folder_filter?, doc_type_filter?) → chunks + source file path
list_indexed_files(folder?) → what's in the index
get_document_summary(file_path) → full extracted text of a specific file
reindex(folder?) → trigger a re-ingest of changed files
```

### 6. Metadata to Index Per Chunk
- `file_path` — Dropbox path (e.g., `/GW Allen/Prove Sheets/2026/April/ONEOK-123.pdf`)
- `file_name`
- `doc_type` — inferred from folder or filename pattern
- `last_modified` — for freshness filtering
- `page_number` / `row_range` — for traceability

### 7. Dropbox Folder Structure
Need to map out the actual folder layout to decide which folders to index and how to tag chunks.
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
| PDF parsing | `pdfplumber` + `pytesseract` | |
| Word | `python-docx` | |
| Excel | `openpyxl` / `pandas` | |
| Embeddings | OpenAI `text-embedding-3-small` | swappable |
| Vector store | LanceDB | file-based, no server |
| Chunking | LangChain `RecursiveCharacterTextSplitter` | or custom |
| Package mgmt | `uv` | consistent with donoco-journal |

## Milestones (Draft)

1. **Repo skeleton** — this file, CLAUDE.md, pyproject.toml, folder structure
2. **Ingestion spike** — parse 10 sample docs from Dropbox, chunk, embed, store in LanceDB
3. **MCP server** — expose `search_documents` and `list_indexed_files` tools
4. **Full ingest run** — index all relevant Dropbox folders
5. **Deploy** — EC2 or local machine, wired into Claude Code / claude.ai MCP settings
6. **Incremental sync** — re-index only changed/new files on a schedule

## Related Projects

- `donoco-journal` — Donoco internal newspaper; has the existing `operational-gage-mcp` for structured GAGE database queries
- `operational-gage-mcp` (live in claude.ai) — covers structured data (proves, hours, miles, certs) from Snowflake; this project covers **unstructured** document knowledge
