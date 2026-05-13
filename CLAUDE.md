# CLAUDE.md — gage-knowledge-mcp

## What This Repo Is

RAG pipeline + MCP server that indexes GW Allen / Gage Western's Dropbox files and exposes semantic search as MCP tools for Claude.

This is the **unstructured document** companion to the `operational-gage-mcp` (which covers structured Snowflake data — proves, hours, certs, miles).

## Key Context

- **Company**: GW Allen LLC / Gage Western — petroleum measurement services (proves, tank calibrations)
- **Data source**: Dropbox account holding operational docs — field tickets, SOPs, customer agreements, safety records, certs, prove sheets
- **Consumer**: Claude Code and claude.ai via MCP, used by Joshua Murray and GW Allen staff
- **Related MCP**: `operational-gage-mcp` in claude.ai already handles structured DB queries — don't duplicate that

## Tech Stack

- **Language**: Python 3.11+ managed via `uv`
- **MCP SDK**: Anthropic official `mcp` Python package
- **Vector store**: LanceDB (file-based, no server required)
- **Embeddings**: OpenAI `text-embedding-3-small` (key in env as `OPENAI_API_KEY`)
- **Dropbox access**: `dropbox` Python SDK (key in env as `DROPBOX_ACCESS_TOKEN`)

## Project Layout

```
gage-knowledge-mcp/
├── CLAUDE.md
├── README.md
├── pyproject.toml
├── .env.example
├── src/
│   ├── server.py          # MCP server entrypoint
│   ├── ingest.py          # Dropbox → parse → chunk → embed → LanceDB
│   ├── parsers/
│   │   ├── pdf.py         # pdfplumber + pytesseract OCR fallback
│   │   ├── word.py        # python-docx
│   │   └── excel.py       # openpyxl / pandas
│   └── search.py          # LanceDB vector search helpers
├── data/
│   └── lancedb/           # Vector index (gitignored)
└── scripts/
    └── reindex.py         # One-shot full re-ingest CLI
```

## MCP Tools (Planned)

| Tool | Args | Returns |
|------|------|---------|
| `search_documents` | `query`, optional `folder_filter`, `doc_type_filter`, `limit` | Top-k chunks with file path, page/row, score |
| `list_indexed_files` | optional `folder` | Files in the index with last-indexed timestamp |
| `get_document_summary` | `file_path` | Full extracted text of one document |
| `reindex` | optional `folder` | Trigger incremental re-ingest, return stats |

## Development Notes

- Use `uv` for all package management (`uv add`, `uv run`)
- Never commit the `data/lancedb/` directory or `.env` files
- Dropbox token needs to be a **long-lived** token or refreshed via OAuth — short-lived tokens will break scheduled re-index runs
- OCR for scanned PDFs is slow — consider a separate background job for OCR-heavy folders (field tickets are likely scanned)
- Keep chunk size ~500 tokens with ~50-token overlap — field documents tend to be short and dense
- Always include `file_path` and `page_number`/`row_range` in chunk metadata so Claude can cite sources

## Environment Variables

```
OPENAI_API_KEY=sk-...
DROPBOX_ACCESS_TOKEN=...
LANCEDB_PATH=./data/lancedb
```

## Running Locally

```bash
uv sync
uv run python scripts/reindex.py          # full re-ingest
uv run python -m src.server               # start MCP server
```

## Open Questions (resolve in early spikes)

1. Is Dropbox mounted locally on the machine that will host this, or do we use the API?
2. Are field ticket PDFs digital-native or scanned? (Determines whether OCR is needed)
3. What Dropbox folder paths should be indexed? (Get from GW Allen admin)
4. Where does this run — EC2 alongside donoco-journal, or a GW Allen local machine?
5. Do GW Allen staff want to query this themselves, or is it Joshua Murray only?
