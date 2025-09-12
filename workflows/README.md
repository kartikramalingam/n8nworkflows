Workflows

This directory contains n8n workflow JSON exports. One workflow per file.

Subfolders

- webhook: flows triggered by HTTP/webhook endpoints.
- scheduled: cron/time-based flows.
- event: flows triggered by external events (e.g., queues, webhooks from third parties that are not direct inbound APIs), or app triggers.
- manual: flows run manually within the editor (ad-hoc/utility).
- callable: reusable sub-workflows invoked via Execute Workflow.
- deprecated: retired flows kept for historical reference; do not modify.
- _templates: skeletons and examples for new workflows.

Rules

- Do not include credentials in exports.
- Use kebab-case filenames; keep names aligned with the workflow display name.
- Prefer descriptive tags within the workflow for discoverability.
