n8n Workflows Repository

Purpose: version-controlled storage for n8n workflow JSON exports, with clear structure and guidance for safe collaboration across environments.

Directory layout

- workflows: workflow JSON exports grouped by type (no credentials).
- credentials: never commit real credentials; store examples/templates only.
- environments: env-specific notes/config conventions (dev/staging/prod).
- custom-nodes: optional custom node packages or notes.
- docs: additional documentation (architecture, runbooks, etc.).

Conventions

- Export format: use n8n "Export" for a single workflow; do NOT include credentials in exports.
- Naming: one workflow per file using kebab-case, e.g. `daily-sync-accounts.json`. Prefer Git history for versioning; if you need parallel versions, suffix with `.v2.json`.
- Categorization: place workflows under the closest matching folder (webhook, scheduled, event, manual, callable). Move retired flows to `deprecated/`.
- Reusability: put sub-workflows intended for Execute Workflow in `workflows/callable/`.
- Secrets: credentials must never be stored here. Use named credentials within n8n and document names under `credentials/README.md`.

Typical commands

- Import a workflow file: `n8n import:workflow --input=workflows/scheduled/daily-sync-accounts.json`
- Export a workflow by ID: `n8n export:workflow --id=<id> --output=workflows/scheduled/daily-sync-accounts.json`

Notes

- Keep workflow names, tags, and descriptions meaningful for searchability.
- Prefer environment-agnostic logic; branch on env only when necessary.
- Review diffs carefully—JSON can be noisy; consider pretty-printed exports.
