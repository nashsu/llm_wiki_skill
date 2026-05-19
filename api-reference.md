# LLM Wiki API v1 — Endpoint reference

Base URL: `http://127.0.0.1:19828`
Prefix:   `/api/v1`
Default content type: `application/json; charset=utf-8`

All non-`/health` endpoints require auth via one of:

```
Authorization: Bearer <token>
X-LLM-Wiki-Token: <token>
?token=<urlencoded-token>
```

---

## GET /api/v1/health

**Auth:** not required.

```json
{
  "ok": true,
  "status": "running",
  "version": "0.4.x",
  "enabled": true,
  "authRequired": true,
  "authConfigured": true,
  "allowUnauthenticated": false,
  "tokenSource": "store"
}
```

Field reference:

- `status` — `starting` / `running` / `port_conflict` / `error`
- `enabled` — `false` = user toggled the API off; all non-`/health` endpoints return 503
- `authRequired` — `false` iff `allowUnauthenticated: true`
- `authConfigured` — `true` if env `LLM_WIKI_API_TOKEN` or `apiConfig.token` is set
- `allowUnauthenticated` — anonymous local-process mode (rare; user opt-in only)
- `tokenSource` — `env` / `store` / `none`. If `env`, the desktop UI token field is **ignored**.

---

## GET /api/v1/projects

**Auth:** required.

```json
{
  "ok": true,
  "projects": [
    {
      "id": "a0e90b29-fcf3-4364-9502-8bd1272de820",
      "name": "Research Notes",
      "path": "/Users/me/wiki-projects/research",
      "current": true
    },
    {
      "id": "...",
      "name": "Reading",
      "path": "/Users/me/wiki-projects/reading",
      "current": false
    }
  ]
}
```

The `current` flag marks the project that is currently open in the desktop UI. Use it when the user doesn't name a project explicitly.

`{id}` placeholder in every project-scoped endpoint accepts:

- the project UUID (`p.id`)
- the project filesystem path (`p.path`, URL-encoded)
- the literal string `current`

### Resolving a user-spoken project name

There is **no `?name=` filter** on this endpoint and `{id}` does **not** accept a name directly — names are resolved entirely client-side after listing all projects.

Algorithm:

1. `GET /api/v1/projects` → array of `{id, name, path, current}`
2. Case-insensitive substring match on `name`:
   ```js
   const matches = projects.filter(p =>
     p.name.toLowerCase().includes(spokenName.toLowerCase())
   )
   ```
3. Handle the cardinality:
   - **0 matches** → tell the user, list available names, ask. Don't silently fall back to `current`.
   - **1 match** → use `matches[0].id` for all subsequent calls in this conversation.
   - **2+ matches** → ask the user to disambiguate (show `name` + `path` for each).
4. Cache the resolved `id` for the rest of the conversation. Only re-list when the user switches contexts.

If the user gives a filesystem path verbatim, you can URL-encode it and pass it as `{id}` directly — no list lookup needed:

```bash
PROJECT_PATH=$(printf %s "/Users/me/wiki/reading" | jq -sRr @uri)
curl -s -H "Authorization: Bearer $TOKEN" \
  "$BASE/api/v1/projects/$PROJECT_PATH/files?root=wiki"
```

If the path isn't registered in the desktop app, you'll get **404** — fall back to listing and asking.

---

## GET /api/v1/projects/{id}/files

**Auth:** required.

Query params:

| Param | Default | Notes |
|---|---|---|
| `root` | `wiki` | One of `wiki` / `sources` (alias `raw`, `raw/sources`) / `all`. `all` lists every public sub-tree (`purpose.md`, `schema.md`, `wiki/`, `raw/sources/`). |
| `recursive` | `true` | `false` → only one level. |
| `maxFiles` | `2000` | Clamped to `[1, 10000]`. Exceed → 413. |

Response:

```json
{
  "ok": true,
  "projectId": "...",
  "root": "wiki",
  "files": [
    {
      "name": "concepts",
      "path": "wiki/concepts",
      "isDir": true,
      "size": null,
      "children": [
        {
          "name": "rope.md",
          "path": "wiki/concepts/rope.md",
          "isDir": false,
          "size": 4321,
          "children": null
        }
      ]
    }
  ],
  "truncated": false
}
```

Hidden files (dotfiles) and symlinks are silently skipped.

---

## GET /api/v1/projects/{id}/files/content

**Auth:** required.

Query params:

| Param | Required | Notes |
|---|---|---|
| `path` | yes | Project-relative. Allow-list: `purpose.md`, `schema.md`, `wiki/**`, `raw/sources/**`. No dotfile segments. No `..`. |

Response:

```json
{
  "ok": true,
  "projectId": "...",
  "path": "wiki/concepts/rope.md",
  "content": "---\ntitle: RoPE\n---\n\n# Rotary Position Embedding\n..."
}
```

Failure modes:

- `403` — path outside the allow-list (e.g., `../app-state.json`, `.llm-wiki/foo.json`)
- `404` — file does not exist
- `413` — file > 2 MB
- `415` — file is binary / non-UTF-8 (e.g., PNG, PDF). Use the desktop UI to view; the API is text-only.

---

## POST /api/v1/projects/{id}/search

**Auth:** required.

Body:

```json
{
  "query": "rope rotary position embedding",
  "topK": 10,
  "includeContent": false,
  "queryEmbedding": null
}
```

| Field | Default | Notes |
|---|---|---|
| `query` | (required) | Non-empty. Empty / whitespace-only → 400. |
| `topK` | `10` | Clamped to `[1, 50]`. |
| `includeContent` | `false` | When `true`, each hit carries `content` (full markdown). Skip the per-page content fetch round-trip. |
| `queryEmbedding` | `null` | Optional `number[]`. If you precomputed a query embedding (your own model, batched offline), pass it here and the server skips its own embed call. Must be a non-empty array of finite numbers, otherwise → 400. |

Response:

```json
{
  "ok": true,
  "projectId": "...",
  "mode": "hybrid",
  "tokenHits": 78,
  "vectorHits": 14,
  "note": "Search uses the shared backend retrieval service. When embeddingConfig is enabled, the API automatically includes LanceDB vector results; clients may also pass queryEmbedding explicitly.",
  "results": [
    {
      "path": "wiki/concepts/rope.md",
      "title": "Rotary Position Embedding",
      "snippet": "...inject positional information by rotating Q and K...",
      "titleMatch": true,
      "score": 0.0315136476426799,
      "vectorScore": 0.94,
      "images": [
        { "url": "wiki/media/rope-diagram.png", "alt": "RoPE rotation diagram" }
      ],
      "content": null
    }
  ]
}
```

### Retrieval mode

The server picks the mode automatically based on whether the active project has embeddings configured (Settings → Embeddings) **and** whether the vector index for the project has data:

| `mode` | Trigger | Score scale |
|---|---|---|
| `"keyword"` | No `embeddingConfig`, OR embedding fetch failed, OR vector index empty. | Additive keyword score: filename-exact ~200, phrase-in-title ~50+, token-bag scoring in single digits. |
| `"vector"` | Vector index returned hits but keyword scoring matched nothing. Rare in practice. | RRF rank score, typically `1 / (60 + rank)` ≈ `0.015–0.017`. |
| `"hybrid"` | Both keyword and vector pipelines produced hits — the common case when embeddings are enabled. | RRF combined: up to `1/61 + 1/61` ≈ `0.0328` for a top hit. |

`tokenHits` is the number of pages the keyword pass scored; `vectorHits` is the number of distinct pages LanceDB returned. Either can be 0.

### Per-result fields

| Field | Always present? | Notes |
|---|---|---|
| `path` | yes | Project-relative path to the markdown page. |
| `title` | yes | Front-matter `title:` if present; else first `# Heading`; else filename with dashes → spaces. |
| `snippet` | yes | ~160-char window. In keyword mode: centered on the query/anchor token in the page body. In vector-only matches: the actual matching chunk text, optionally prefixed with the chunk's heading path (e.g. `"Section > Detail: chunk text..."`). |
| `titleMatch` | yes | `true` when a token or phrase hit the title (boosts ranking). |
| `score` | yes | Final ranking score. See "Retrieval mode" for scale. |
| `vectorScore` | optional | Raw vector similarity (≈ cosine 0–1) when the page matched via the vector index. Useful for "how strong was the semantic match" decisions. Absent on pure keyword hits. |
| `images` | yes | Embedded `![alt](url)` references discovered in the markdown, deduped by URL. Useful for the agent to surface diagrams. |
| `content` | optional | Full markdown, only when `includeContent: true`. |

`results` is sorted descending by `score`. Don't compare scores **across** modes (keyword scores are 100×+ larger than RRF scores by construction). Rely on relative ordering within one response.

---

## GET /api/v1/projects/{id}/graph

**Auth:** required.

Query params:

| Param | Default | Notes |
|---|---|---|
| `q` | — | Substring filter on `id` or `label`, case-insensitive. |
| `nodeType` | — | Filter on frontmatter `type:` (e.g., `entity`, `concept`, `query`, `other`). |
| `limit` | `200` | Clamped to `[1, 1000]`. |

Response:

```json
{
  "ok": true,
  "projectId": "...",
  "nodes": [
    {
      "id": "rope",
      "label": "Rotary Position Embedding",
      "nodeType": "concept",
      "path": "wiki/concepts/rope.md",
      "linkCount": 4
    }
  ],
  "edges": [
    {
      "source": "rope",
      "target": "attention",
      "weight": 1.0
    }
  ]
}
```

Edges are derived from `[[wikilink]]` references inside `wiki/*.md`. Deduplicated by unordered pair `(source, target)` — `[[a]]` in b and `[[b]]` in a produce **one** edge. Self-edges are dropped. `weight` is `1.0` in v1.

`linkCount` is the node's degree in the deduped graph.

---

## POST /api/v1/projects/{id}/sources/rescan

**Auth:** required. **Mutates state.**

No body required.

```json
{
  "ok": true,
  "projectId": "...",
  "result": {
    "queue": {
      "version": 1,
      "tasks": []
    },
    "changedTasks": [
      {
        "id": "...",
        "path": "raw/sources/new-paper.pdf",
        "kind": "created"
      }
    ]
  }
}
```

`changedTasks` contains the files this rescan **actually detected as changed** (created / modified / deleted). The downstream ingest queue picks these up asynchronously — the API call returns as soon as the diff is queued, not when ingest finishes.

The user's `sourceWatchConfig` (file type filters, exclude dirs, max size) is honored. If the user disabled auto-ingest, files are still detected but not queued for the LLM pipeline.

---

## POST /api/v1/projects/{id}/chat

**Returns 501.** Chat / RAG pipeline lives in the WebView in v1. Don't invoke. Tell the user to use the desktop chat UI.

---

## Limits & defenses

| Limit | Value | Effect |
|---|---|---|
| Body size | 1 MiB | Exceed → 400. |
| File content read | 2 MiB | Exceed → 413. |
| File-tree node count | 10000 hard cap | Exceed → 413. |
| Search `topK` | 50 max | Silently clamped. |
| Graph `limit` | 1000 max | Silently clamped. |
| Rate limit | 120 req/sec (global) | 429 with `Retry-After` semantics implied (back off ≥1s). |
| In-flight requests | 64 concurrent | 503 "API server is busy" — back off ≥2s. |

CORS: `Access-Control-Allow-Origin: *`. Preflight cached 10 min via `Access-Control-Max-Age: 600`. Allowed headers include `Authorization`, `X-LLM-Wiki-Token`, `Content-Type`.

Non-`GET` / non-`POST` methods → 405.
