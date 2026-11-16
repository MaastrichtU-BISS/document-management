# @legalblocks/document-management

A reusable **document upload + management** module for LegalBlocks apps, extracted
from Lawnotation and made storage-agnostic.

- Upload `txt`, `md`, `html`, `pdf`, `docx` (PDF page-picker + DOCX text
  extraction run in-browser via CDN-loaded `pdfjs-dist` / `mammoth`). Legacy binary
  `.doc` is not supported — mammoth's `extractRawText` only reliably parses OOXML
  `.docx`.
- List, search, view, delete, and bulk-delete documents.
- Stores **one artifact per document** (`document.<id>`) plus a derived
  `document-set` aggregate for the downstream annotate screen.

## Storage-agnostic

The module persists through `@legalblocks/module-runtime`. Its `documents_endpoint`
param binds the `document` prefix:

- default `{ "adapter": "local" }` → browser storage (zero backend).
- `{ "adapter": "http", "endpoint": "…/v1" }` → a database, via any backend
  implementing the artifact REST contract (see `services/artifact-store`).

No module code changes between the two — see the repo's
`docs/module-authoring-guide.md`.

## Config params

| param | type | default |
|---|---|---|
| `allowed_types` | string | `txt,md,html,pdf,docx` |
| `max_file_size_mb` | string | `10` (0/blank = no limit) |
| `enable_pdf` | enum on/off | `on` |
| `enable_docx` | enum on/off | `on` |
| `documents_endpoint` | artifact_endpoint (prefix `document`) | `{adapter:"local"}` |

## Screen

Provides `list` → route `/list`.
