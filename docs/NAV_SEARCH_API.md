# Using `/nav/search` with n8n Workflows

This guide explains how to call the `POST /nav/search` endpoint from the CompanyTree API and integrate the results into an n8n workflow. It captures request/response structure, supported filters, pagination behaviour, and pragmatic tips for orchestrating the calls inside n8n.

## Endpoint Overview
- **Method & path**: `POST /nav/search`
- **Purpose**: Return NAV (node attribute value) rows that match one or more attribute-based conditions, along with node metadata and optional ancestor/type aggregations.
- **Content type**: Send JSON with `Content-Type: application/json`. Responses are JSON; large payloads are gzip-compressed if the request includes `Accept-Encoding: gzip` (many HTTP clients, including curl and n8n, negotiate this automatically).

## Request Schema
All requests share a common envelope:

```json
{
  "logic": "OR",
  "groups": [ /* required: at least one group */ ],
  "filters": { /* optional */ },
  "items_asofdate": "2026-01-01T00:00:00Z",
  "items": ["figi", "shareclassfigi"],
  "items_scope": "node",
  "sort": [ { "field": "valid_from", "dir": "asc" } ],
  "page": 1,
  "page_size": 500,
  "limit": 5000,
  "view": { "items": true, "ancestors": true, "type_buckets": true, "canonical_keys": false }
}
```

### Logical Combination
- `logic`: `"OR"` (default) or `"AND"`. Determines how the `groups` are combined when deciding which **nodes** match.

### Groups (required)
Each entry in `groups[]` describes one attribute test. Within a group, all comparisons are ANDed. Multiple groups are combined by `logic`.

```json
{
  "attribute_code": "figi",          // required; case-insensitive
  "asOf": "2026-05-06",              // optional but recommended; ISO date or datetime
  "value": {
    "type": "text",                   // optional hint; inferred from attribute metadata when omitted
    "eq": "BBG000B9XRY4",             // supported comparators listed below
    "ne": null,
    "gt": null,
    "regex": "^US[0-9]{10}$"
  }
}
```

Supported `value` comparators (all optional):
- Equality / inequality: `eq`, `ne`
- Range: `gt`, `gte`/`ge`, `lt`, `lte`
- Membership: `in` (array)
- Pattern matching on text: `regex`, `regex_i` (PostgreSQL regex), `like`, `ilike`
- Boolean and date comparisons inherit the correct NAV column automatically once `type` or attribute metadata is known.

> **Attribute lookup**: If `attribute_code` cannot be resolved in `dim.attribute_def`, that group evaluates to `false` and prevents matches for that portion of the request.

- `asOf`: When present, the NAV row must have a `valid_during` range that contains this timestamp. If you omit `asOf`, the attribute is matched regardless of validity window.

### Required: `items_asofdate`
- ISO date or datetime. All rows in the `items` array of the response must be valid at this timestamp.
- The endpoint rejects requests without this field.

### Optional: `items`
- Array of attribute codes to render in the response. When omitted, all matched attributes are returned.
- Supplying `items` changes pagination: `page`/`page_size` apply to this filtered list.

### Optional: `items_scope`
Controls where `items` are pulled from relative to the matched node set (ignored when `items` is omitted). Accepted values:
- `"node"` (default): fetch attributes on the matched nodes.
- `"ancestors"`: fetch attributes for each node’s ancestor chain.
- `"descendants"`: fetch attributes from matched nodes’ descendants.
- `"node+ancestors"`: union of the node and its ancestors.

Internally this leverages `fct.node_instance_closure`, so ensure the closure table is populated.

### Optional: `filters`
Restrict the overall scope before evaluating groups:
- `company_ids: [int, ...]`
- `node_type_codes: ["company", "security", ...]`
- `source: "securityresolution"` (matches `v.source`)
- Node instance scoping (aliases are interchangeable and may be scalars or arrays): `node_inst_id`, `node_instance_id`, `node_inst_ids`, `node_instance_ids`

### Optional: `sort`
Order of returned rows (defaults to `node_inst_id`, then `attribute_id`, then `valid_from`). Allowed fields:
`node_inst_id`, `company_id`, `node_type_id`, `node_type_code`, `attribute_id`, `attribute_code`, `valid_from`, `valid_to`, `source`, `val_text`, `val_numeric`, `val_boolean`, `val_date`.

### Pagination and Result Caps
- `page` (1-based) and `page_size` (max 20 000) control paging.
- `limit`: optional cap on the total number of rows across all pages. Once the cap is reached, `next_page` is omitted.
- You may supply these as query parameters instead of body keys (handy for n8n): `POST /nav/search?page=2&page_size=1000&limit=5000`.
  - Snake_case takes precedence; camelCase aliases (`pageSize`) are also accepted via query parameters.

### View Flags (`view`, `response`, or `include` aliases)
Booleans that control what the response includes. Defaults: `items=true`, `type_buckets=true`, `ancestors=true`, `canonical_keys=false`.
- Set `items=false` to only retrieve pagination metadata and buckets.
- Set `type_buckets=false` if you only need individual rows.
- Set `ancestors=false` to skip the ancestor JSON array per item.
- Set `canonical_keys=true` to mirror the standard type bucket keys (`listingexchange`, `listing`, `security`, `company`) at the top level.

## Response Structure
Core fields:
- `page`, `page_size`: echo the request (after overrides).
- `next_page`: next page number when more items are available; absent otherwise.
- `next_url`: convenience link containing the next-page query parameters.
- `items[]`: present when `view.items` is `true`.
- `type_buckets`: present when `view.type_buckets` is `true`; maps `node_type_code` to distinct node IDs.
- Optional top-level `listingexchange`, `listing`, `security`, `company`: only included when `view.canonical_keys=true` and mirror the corresponding bucket arrays.

Each entry in `items[]` contains:
- Identifiers: `node_inst_id`, `company_id`, `node_type_id`, `node_type_code`, `attribute_id`, `attribute_code`
- Value columns: one or more of `val_text`, `val_numeric`, `val_boolean`, `val_date`
- Validity window: `valid_from`, `valid_to` (upper bound can be `null` for open range)
- `source`: origin string (optional)
- `ancestors[]`: when included, an array ordered by `depth` with `node_inst_id`, `node_type_id`, `node_type_code`

Errors are surfaced as `400` with detail text (e.g., missing `items_asofdate`, invalid pagination values). Unhandled errors bubble up as `500`.

## Example Requests

### 1. Minimal lookup by FIGI
```bash
curl --compressed \
  -X POST "http://localhost:8000/nav/search" \
  -H "Content-Type: application/json" \
  -d '{
    "groups": [
      {"attribute_code": "figi", "asOf": "2026-05-06", "value": {"eq": "BBG000B9XRY4"}}
    ],
    "items_asofdate": "2026-05-06"
  }'
```

### 2. Combine attribute filters with additional items and buckets
```json
{
  "logic": "AND",
  "groups": [
    {
      "attribute_code": "lei",
      "asOf": "2024-01-01",
      "value": { "type": "text", "regex_i": "/^([A-Z0-9]{20})$/" }
    },
    {
      "attribute_code": "countryname",
      "asOf": "2024-01-01",
      "value": { "type": "text", "ilike": "%states%" }
    }
  ],
  "filters": { "node_type_codes": ["company", "security"] },
  "items": ["ticker", "shareclassfigi"],
  "items_scope": "node+ancestors",
  "items_asofdate": "2024-01-01T00:00:00Z",
  "sort": [ { "field": "valid_from", "dir": "desc" } ],
  "view": { "canonical_keys": true }
}
```

### 3. Using the total limit with pagination overrides
First page:
```bash
curl --compressed \
  -X POST "http://localhost:8000/nav/search?page=1&page_size=500&limit=2000" \
  -H "Content-Type: application/json" \
  -d '{
    "groups": [
      {"attribute_code": "shares", "asOf": "2023-12-31", "value": {"gt": 0}}
    ],
    "items_asofdate": "2023-12-31"
  }'
```
Use the `next_url` from the response for subsequent pages until it becomes `null`.

## Integrating with n8n

### 1. Basic HTTP Request node
1. Add an **HTTP Request** node.
2. Set **Method** to `POST` and **URL** to your API host, e.g. `https://api.companytree.local/nav/search`.
3. Under **Authentication** choose the scheme your deployment requires (if none, leave at *None*).
4. In **Headers**, add `Content-Type: application/json` if n8n does not auto-set it.
5. Switch **Body Content Type** to `JSON/RAW` and populate the JSON body. You can either paste the JSON directly or add individual `Body Parameters` (n8n will assemble the JSON).
6. Define `items_asofdate` and at least one `groups[]` entry. Example body parameter structure:
   - `logic` → `OR`
   - `groups[0][attribute_code]` → `figi`
   - `groups[0][asOf]` → `2026-05-06`
   - `groups[0][value][eq]` → `BBG000B9XRY4`
   - `items_asofdate` → `2026-05-06`
7. (Optional) Add **Query Parameters**: `page`, `page_size`, `limit`. Using n8n expressions lets you drive pagination dynamically, e.g. `{{$json.page || 1}}`.
8. Set **Response Format** to *JSON*. The node output exposes `items`, `type_buckets`, `next_page`, etc., for downstream nodes.

### 2. Handling pagination loops
Use `next_page` or `next_url` to paginate:
1. Start with a **Set** node initialising `page` to `1` and (optionally) `accumulator` arrays.
2. Connect to your **HTTP Request** node. In the URL or query parameters reference `{{$json.page}}` so each iteration requests the current page.
3. Follow with an **IF** node that checks `{{$json["next_page"] !== null}}` on the HTTP node’s output.
4. Use an **Increment** (Set) node to bump the page number and loop back via an n8n *Loop* (e.g. `Merge` node in wait or a `Split In Batches` pattern). Stop when `next_page` is `null`.
5. If you prefer to follow `next_url`, expose it via `{{$json["next_url"]}}` and set the HTTP node’s URL to that expression when present, falling back to the base URL on the first run.

### 3. Parsing results in n8n
- To flatten the items, use an **Item Lists → Split Out Items** node or a **Function** node to iterate through `items` and emit one n8n item per NAV row.
- Use the `type_buckets` object for quick access to distinct node IDs per type. Example (Function node):
  ```javascript
  const buckets = $json.type_buckets || {};
  return Object.entries(buckets).map(([type, ids]) => ({ json: { type, ids } }));
  ```
- Ancestor data is nested; to extract the top-most company ancestor you can use `{{$json["ancestors"][0]}}` (after verifying the array exists).

### 4. When to call `/nav/distinct-nodes`
If you only need unique nodes that match the criteria (no attribute rows), reuse the same payload with `POST /nav/distinct-nodes`. The response schema becomes:
```json
{
  "items": [ { "node_inst_id": 123, "company_id": 456, "node_type_id": 3, "node_type_code": "company" }, ... ],
  "page": 1,
  "page_size": 200,
  "next_page": null,
  "next_url": null
}
```
This endpoint ignores `view` flags and omits ancestor calculations for performance.

## Troubleshooting
- **400 `nav search failed: items_asofdate is required`** – supply `items_asofdate` in the body.
- **No matches returned** – confirm the `attribute_code` exists in `dim.attribute_def` and that `asOf` aligns with a `valid_during` window. Use a broader `items_asofdate` or remove narrow filters to debug.
- **Unexpected pagination** – remember that setting `items` filters the returned list; pagination applies after that filter, so the number of rows per page can change when you adjust `items` or `items_scope`.
- **Large responses** – reduce payload size via `view` flags (`ancestors=false`, `type_buckets=false`) or request smaller `page_size`.

## Quick Reference
- **Mandatory fields**: `groups`, `items_asofdate`
- **Comparators**: `eq`, `ne`, `gt`, `gte`, `lt`, `lte`, `in`, `regex`, `regex_i`, `like`, `ilike`
- **Pagination**: `page`, `page_size`, optional `limit`; query parameters override body values.
- **Ancestors**: controlled by `view.ancestors`; needed for `type_buckets` to include ancestor IDs.

This documentation is versioned with the repository. Update it alongside any future changes to `app/services/nav.py` or `app/routers/nav.py` to keep n8n workflows accurate.
