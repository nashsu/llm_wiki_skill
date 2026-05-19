---
name: llm-wiki
description: "Query the user's LLM Wiki knowledge base (the LLM Wiki desktop app at 127.0.0.1:19828 — NOT Obsidian, Notion, Apple Notes, Logseq, or any other PKM tool). Trigger ONLY when the user explicitly names LLM Wiki, says 'my wiki', 'my 知识库 / 知识库 / knowledge base', or asks things like 'what does my wiki say about X', 'read wiki page Y', 'show my wiki graph / 知识图谱', 'search in my LLM Wiki project', 'rescan my wiki sources / 重新索引', or names a wiki project by ID. DO NOT trigger on generic 'search my notes', 'find in my notebook', 'check my Obsidian', etc. — those belong to other tools the user may have installed. Covers wiki page search, file listing, content read, knowledge graph navigation, and source rescan against the running LLM Wiki desktop app. Read-only except for source rescan."
---

# LLM Wiki Local API Skill

Talk to the user's locally-running LLM Wiki app over its built-in HTTP API. This is a **standard JSON API** — call it directly with whatever HTTP tool is already in your environment (`curl`, `fetch`, `requests`, `http` middleware, etc.). No client library to install, no SDK to learn.

Treat the wiki as a **private, structured knowledge base** the user has been curating: pages live as `wiki/**.md`, raw documents under `raw/sources/`, wikilinks form a graph.

## When to invoke

Invoke **only** when the user is clearly referring to **LLM Wiki** specifically — by app name, by `wiki` framing, or by `知识库` framing. Concretely:

- asks a question framed as "what does my **wiki** / my **knowledge base** / 我的**知识库** / **LLM Wiki** say about X"
- asks to "search **my wiki** / **LLM Wiki** project / 我的**知识库** for X"
- references a **wiki page** by stem / title and wants to read or cross-link
- asks for the **wiki graph / 知识图谱 / wiki overview / wiki structure**
- has just added or edited files under the LLM Wiki **source folder** and wants ingest re-run / **重新索引**
- says "use **my wiki** for context" / "ground your answer in **my wiki**" / "check **my LLM Wiki**"
- names a wiki project (by ID, by absolute path, or by `current`)

**Do NOT invoke when the user says:**

- "search **my notes**" without further qualification — likely Obsidian / Apple Notes / Notion / Logseq / Bear / etc.
- "find in **my notebook**" — likely Jupyter / OneNote / Notability
- "check **my Obsidian / Notion / Roam / Logseq vault**" — explicitly a different tool
- "look up **my Anki / Readwise / Pocket**" — different tool
- "search **my files / my Documents folder**" — generic filesystem, not the wiki
- general world knowledge, current events, or anything the user clearly wants from the open web

When in doubt about which knowledge tool the user means, ask: *"Do you mean your LLM Wiki specifically, or another tool?"* — don't silently call the LLM Wiki API on what might be an Obsidian vault.

## Quick start

The whole API is plain HTTP + JSON. The fastest path:

```bash
BASE=http://127.0.0.1:19828
TOKEN="${LLM_WIKI_API_TOKEN:-<paste-from-Settings>}"

# 1. probe state — no auth needed
curl -s $BASE/api/v1/health

# 2. list projects
curl -s -H "Authorization: Bearer $TOKEN" $BASE/api/v1/projects

# 3. search
curl -s -H "Authorization: Bearer $TOKEN" \
  -H 'Content-Type: application/json' \
  -d '{"query":"rope embedding","topK":5}' \
  $BASE/api/v1/projects/current/search

# 4. read a page
curl -s -H "Authorization: Bearer $TOKEN" \
  "$BASE/api/v1/projects/current/files/content?path=wiki/concepts/rope.md"
```

If you're writing TypeScript / JavaScript:

```ts
const res = await fetch("http://127.0.0.1:19828/api/v1/projects/current/search", {
  method: "POST",
  headers: { "Authorization": `Bearer ${process.env.LLM_WIKI_API_TOKEN}`, "Content-Type": "application/json" },
  body: JSON.stringify({ query: "rope embedding", topK: 5 }),
})
const { results } = await res.json()
```

Python is the same shape — `urllib.request`, `requests`, `httpx`, whatever you already have. **Don't install anything new.**

## Auth model

The API is **localhost-only**. The token is one of:

1. `LLM_WIKI_API_TOKEN` environment variable (if set, overrides UI)
2. The user's `apiConfig.token` saved via Settings → API Server
3. `allowUnauthenticated: true` mode (no token needed; rare, user opt-in only)

Always check `/api/v1/health` first — it returns `{ enabled, authConfigured, allowUnauthenticated, tokenSource }`. **If `authConfigured: false && allowUnauthenticated: false`, ask the user to open `Settings → API Server → Generate new token`**. Do not proceed without auth being set up.

Three equivalent ways to send the token:

```
Authorization: Bearer <token>          # preferred
X-LLM-Wiki-Token: <token>              # alternative header
?token=<urlencoded-token>              # query param — last resort, leaks into logs
```

**Never log or echo the token. Never put it in any URL the user can see in your output** (Referer / shell history / logs all leak it).

## Standard workflow

When the user asks "look it up in my wiki":

1. **Resolve project** (see [Project resolution](#project-resolution) below).
2. **Search**: `POST /api/v1/projects/{id}/search` with `{ query, topK: 5..10 }` → ranked hits (`path`, `title`, `snippet`, `score`, `titleMatch`, optional `vectorScore`, `images`). Inspect `response.mode` to know whether hybrid retrieval kicked in.
3. **Read top hits**: for each promising hit, `GET /api/v1/projects/{id}/files/content?path=...` for the full markdown. Or pass `includeContent: true` to the search to avoid the round-trip.
4. **Cite + answer**: synthesize an answer grounded in the read pages. **Quote the `path` of each page you used** so the user can verify and jump in-app.

### Reading the score

The `score` field's scale depends on `mode`:

- **`mode: "keyword"`** — additive keyword score. Filename-exact hits are ~200; phrase-in-title ~50+; bag-of-tokens lands in single digits. Treat anything below ~5% of the top result as low-confidence.
- **`mode: "hybrid"` or `"vector"`** — RRF (Reciprocal Rank Fusion) score, typically in the **0.015–0.035** range. The absolute number is small; relative ordering is what matters. Use the per-result `vectorScore` (raw cosine 0–1) for "how strongly did the embedding match" if you need it.

Don't apply a fixed score threshold across modes. Sort by `score` descending and rely on relative gaps.

### Project resolution

`{id}` in every project-scoped endpoint accepts **four forms**:

| Form | When to use | Example |
|---|---|---|
| `current` (literal) | Default for "my wiki / 我的知识库 / this project / this wiki". The user is referring to whatever is open in the desktop UI. | `/api/v1/projects/current/search` |
| UUID | The user pasted a project ID, OR you previously resolved a name to an ID and want to re-use it. | `/api/v1/projects/a0e90b29-fcf3-4364-9502-8bd1272de820/files` |
| Absolute filesystem path (URL-encoded) | The user named the path (e.g. `~/notes/research`). Useful when the user has multiple projects with similar names. | `/api/v1/projects/%2FUsers%2Fme%2Fwiki%2Fresearch/files` |
| Project name | **Not supported directly.** You must `GET /api/v1/projects` first, find a match by `name`, then use that project's `id`. |

**Decision tree** for what the user said:

```
"my wiki" / "my 知识库" / "this wiki" / "this project" / unspecified
    → use `current`

"my Research project" / "in Reading"
    → GET /api/v1/projects
    → name-match (case-insensitive substring on `name`)
    → use the resulting `id`
    → if 0 matches: tell the user, list available names, fall back to `current` only if they confirm
    → if 2+ matches: ask the user to disambiguate, quoting both names + paths

"the project at /Users/me/foo"
    → URL-encode the path, use directly
    → if the API returns 404, the project isn't registered — list and let user pick

"project a0e90b29-…"
    → use the UUID literally
```

Cache the resolved `id` for the rest of the conversation — there's no need to re-`GET /projects` for every call. But if the user switches contexts mid-conversation ("now look in my Reading project"), re-resolve.

When the user is silent about which project, **default to `current`** and mention it once: *"Looking in your active project (Research Notes)…"*. This avoids cross-project surprises.

For graph / cross-reference questions:

- `GET /api/v1/projects/{id}/graph?limit=200` → `{ nodes: [{id, label, nodeType, path, linkCount}], edges: [{source, target, weight}] }`
- Filter via `?q=term` (substring of id/label, case-insensitive) and `?nodeType=entity|concept|...`

For "I added new docs" requests:

- `POST /api/v1/projects/{id}/sources/rescan` → returns `{ queue: { tasks }, changedTasks: [...] }`. Tell the user how many files changed. Actual ingest runs asynchronously via the desktop queue.

## Endpoint contract (v1)

| Method | Path | Notes |
|---|---|---|
| GET | `/api/v1/health` | No auth. Returns `{ ok, status, version, enabled, authRequired, authConfigured, allowUnauthenticated, tokenSource }`. |
| GET | `/api/v1/projects` | List projects. Each: `{ id, name, path, current }`. |
| GET | `/api/v1/projects/{id}/files?root=wiki\|sources\|all&recursive=true&maxFiles=2000` | Tree of `{ name, path, isDir, size, children }`. Capped at 10000 nodes (413). |
| GET | `/api/v1/projects/{id}/files/content?path=wiki/foo.md` | Text files only (md/mdx/txt/json/yaml/yml/csv/html/htm/xml/rtf/log). 2 MB max. 415 on binary, 413 on oversize, 403 on out-of-scope path. |
| POST | `/api/v1/projects/{id}/search` | Body: `{ "query": "...", "topK": 10, "includeContent": false }`. **Hybrid (keyword + vector)** when the user has embeddings configured in Settings; falls back to keyword-only otherwise. Response carries `mode: "keyword" \| "vector" \| "hybrid"`, plus `tokenHits` / `vectorHits` and per-result `vectorScore`. Empty query → 400. |
| GET | `/api/v1/projects/{id}/graph?q=&nodeType=&limit=200` | Wikilinks graph from `wiki/*.md`. Limit clamped to 1000. |
| POST | `/api/v1/projects/{id}/sources/rescan` | Triggers a backend rescan using the user's Source Watch config. Returns post-rescan queue + actually-changed tasks. |
| POST | `/api/v1/projects/{id}/chat` | **501** — not implemented in v1. Don't call. |

`{id}` accepts a UUID, an absolute filesystem path (URL-encoded), or the literal string `current`.

## Error handling

Always treat the status code as the contract:

| Status | Meaning | What to do |
|---|---|---|
| 200 | OK | Use `body.ok === true` belt-and-suspenders; payload is in the same object. |
| 400 | Bad request | Show `body.error`. Typical: empty `query`, invalid `?root=`, oversized body. |
| 401 | Unauthorized | Token missing/wrong. Tell user to set/regenerate in Settings → API Server. |
| 403 | Forbidden | Path traversal or out-of-scope (e.g. `../app-state.json`). Don't retry the same path. |
| 404 | Not found | Unknown project id or unknown route. On unknown project, list projects first to recover. |
| 405 | Method not allowed | Wrong HTTP verb. |
| 413 | Payload too large | File > 2 MB, file tree > maxFiles, or request body > 1 MB. Suggest narrower scope. |
| 415 | Unsupported media | Binary or non-UTF-8 file content. API is text-only. |
| 429 | Too many requests | Rate limit (120 req/sec global). Back off ≥1 second. |
| 500 | Internal error | Log + report; don't loop. |
| 501 | Not implemented | `/chat` stub. Don't retry. |
| 503 | Service unavailable | Two flavors: API toggled off (`error` contains "disabled"); in-flight cap (64) reached ("busy"). Back off ≥2s. |

If the HTTP call itself fails (connection refused / ENOTFOUND): the desktop app is **not running**. Tell the user: "Launch LLM Wiki, then re-try."

## Etiquette

- **Cite paths.** When you answer using wiki content, name the page: `(from wiki/concepts/rope.md)`. The user uses these to verify and to jump in-app.
- **Stay read-only by default.** Only `sources/rescan` mutates state; everything else is reads. Don't invent write endpoints — they don't exist in v1.
- **Don't dump full pages unless asked.** Snippet + path is usually enough. Pull full content only when reasoning genuinely needs it.
- **Respect the project boundary.** The current project is the user's active context. Do not silently switch projects.
- **Honor the rate limit.** 120 req/sec is plenty for sequential work, but parallel page reads can burst close to it. Batch where the API allows (`includeContent: true` on search avoids N+1 reads).
- **Never leak the token.** Headers are safe; query params and your own output text are not.

## See also

- `api-reference.md` — full endpoint shapes with request / response examples
- `examples.md` — common conversational patterns mapped to direct `curl` / `fetch` sequences
- `README.md` — human setup notes (token generation, port conflicts, troubleshooting)
