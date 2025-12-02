## How to help in this repository

This repository contains exported n8n workflow JSONs that implement an invoice ingestion and validation pipeline. AI coding agents should treat the workflows as the primary source-of-truth and follow the conventions below when editing, extending, or suggesting changes.

### Big picture
- **Platform**: n8n workflows (exported JSON files at repository root). Key workflows:
  - `Insert (8).json` — pipeline that ingests PDFs, creates company schema, inserts `invoices` and `excel_purchase_tax` rows.
  - `Compare (9).json` — compares normalized Excel rows vs `invoices` stored in Postgres and writes summary to `invoice_check_summary`.
  - `parallel sub workflow (18).json` — LLM-driven pipeline that parses PDFs, normalizes fields, validates VAT amounts and diffs.
  - `RD Buyer VAT Lookup (2).json` and `RD Vender VAT Lookup (2).json` — SOAP calls to RD (Thai tax authority) VAT lookup and normalization.

### Data model & DB conventions (explicit in workflows)
- Primary tables (created by SQL in `Insert (8).json`): `invoices`, `invoice_check_summary`, `invoice_check_reasons`, `excel_purchase_tax`, `vat_companies`.
- Per-company schemas: workflows compute a `schema` value derived from the input folder name (hashed, e.g. `s_<hex>`) and use it as the Postgres schema for company-specific tables.
- Composite uniqueness: `invoices` uses unique constraint on `(invoice_number, vendor_vat_id, vendor_branch)`; `invoice_check_summary` uses the same composite for upserts.

### Important patterns & helpers (copyable examples)
- VAT normalization: see `Code: Normalize rows` / `Code in JavaScript` in `Compare (9).json` for `normVatId` (pad to 13 digits, convert non-digits) and `normBranch` (pad to 5 digits, `00000` = head office).
- Company/folder schema hashing: workflows compute `schema = 's_' + Buffer.from(folderName).toString('hex').slice(0,16)` — preserve this transformation when adding features that reference schemas.
- LLM prompts: `parallel sub workflow (18).json` includes strict instructions for models (Gemini) — agents must not change the prompt semantics without test coverage. The LLM output is expected to be a single JSON (no commentary) matching a specific invoice schema.
- Date rules: LLM/normalizers convert Thai BE years → AD by subtracting 543 and return `DD/MM/YYYY` (zero-padded). See `Code: Normalize Fields` for examples.
- Amount validation: VAT validation nodes compute cents, allow a 1-cent tolerance, and infer inclusive/exclusive modes. See `Code Node: Validate VAT amounts` for exact logic.

### Integration points & credentials
- Postgres: all DB nodes use a Postgres credential (credential id visible in the JSON). Agents should **not** hardcode credentials — follow project standards and use n8n credentials.
- RD VAT SOAP API: URL `https://rdws.rd.go.th/serviceRD3/vatserviceRD3.asmx` is used in `RD Buyer/Vender VAT Lookup` workflows. Keep SOAP envelope structure intact if editing these calls.
- LLM provider: Google Gemini via n8n Langchain nodes (`@n8n/n8n-nodes-langchain.googleGemini`). Prompts and model selection are embedded in workflow nodes; changing the model or prompt requires regression testing.

### Project-specific developer workflows
- To test workflows locally or in staging: import the JSON into an n8n instance (Cloud or self-hosted) and configure credentials: Postgres + Google/PaLM API keys.
- Webhook endpoints used by workflows:
  - `POST /insert` (see `Insert (8).json` webhook node) — ingestion entrypoint
  - `POST /compare` (see `Compare (9).json` webhook node) — Excel compare entrypoint
- To create company schema and tables the workflows already include `CREATE SCHEMA ...` and `CREATE TABLE IF NOT EXISTS ...` SQL statements. Agents should prefer running the workflows to provision DB structure rather than duplicating SQL elsewhere.

### Conventions and coding patterns
- Workflows are organized by logical pipeline (insert → parallel sub-workflow → lookups → compare). Keep naming of n8n nodes consistent when adding new nodes (use the existing `Code: <purpose>` and `Execute a SQL query` style).
- Keep normalization logic in code nodes (JS) rather than scattering transformations into many SQL statements. Reuse existing helper functions (date/VAT/branch normalization, amount parsing) where possible.
- LLM outputs are parsed by `Code: Parse LLM JSON` nodes that look for fenced ```json``` blocks or raw JSON — maintain that parsing approach when changing prompt formats.

### What to watch for when changing code or prompts
- Do not loosen the LLM instruction formats: the downstream parsing expects valid JSON blocks; changing prompt output format will break `Parse LLM JSON` nodes.
- When updating normalization logic (VAT id, branch, dates), run a round-trip test by importing sample PDFs/Excel into n8n and checking `invoice_check_summary` flags.
- Many Postgres nodes use `skipOnConflict: true` and `onError: continueRegularOutput`. If you change these, consider the effect on idempotency and error propagation.

### Example snippets to reference
- VAT normalization (from `Compare (9).json`): normVatId pads digits to 13 and treats all-zero as `'0'`.
- Branch normalization: pad to 5 digits, `00000` means head office; used across vendor/buyer flows.
- LLM prompt requirement (from `parallel sub workflow (18).json`): outputs must be valid JSON only, numbers numeric, date `DD/MM/YYYY` after BE→AD conversion.

### Minimal checklist for PRs that touch workflows
- Validate: import updated JSON into a dev n8n instance and run the affected workflow with a small sample payload.
- Credentials: confirm Postgres and LLM credentials are referenced (do not inline secrets).
- Backwards compatibility: keep output JSON schema stable (especially keys consumed by other workflows).
- Update this file if you change high-level architecture, DB schema, or LLM contract.

If any section is unclear or you'd like me to expand examples (e.g., add a small runnable test harness, or convert key JS helpers into a sharable module), tell me which part to elaborate. Which area should I document next? (DB queries, example payloads, or LLM prompt tests?)
