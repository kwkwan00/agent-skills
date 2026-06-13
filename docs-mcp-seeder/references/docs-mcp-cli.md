# docs-mcp-server reference (arabold/docs-mcp-server)

Package: `@arabold/docs-mcp-server` · Image: `ghcr.io/arabold/docs-mcp-server:latest`
Verified against the project's `main` branch (README, `src/mcp/tools.ts`,
`src/tools/ScrapeTool.ts`, `src/tools/SearchTool.ts`, deployment docs).

This skill drives the server through the **CLI with `--server-url`**, which
delegates work to a running/remote server. The MCP-tool equivalents are listed
at the end in case you ever need them.

> **`--server-url` MUST end with `/api`.** It's the tRPC worker RPC endpoint,
> not the MCP path and not the bare base. For a standalone server on `:6280`,
> use `http://localhost:6280/api`. Passing `http://localhost:6280`,
> `…/mcp`, or `…/sse` fails with: *"Failed to connect… ends with '/api'"*.
> Normalize any user-supplied URL by stripping a trailing `/mcp`/`/sse`/`/` and
> appending `/api`. (Verified live against this project's server.)

## CLI commands

Invoke as `npx -y @arabold/docs-mcp-server@latest <command> …`. All read/write
commands accept `--server-url <API_URL>` (the `…/api` endpoint) to target a
running server instead of processing locally.

### `scrape <library> <url> [options]`
Crawls `url`, chunks + (if configured) embeds the pages, and indexes them under
`library`[`@version`].

| Flag | Meaning | Default |
|---|---|---|
| `--version, -v <label>` | Version bucket. Omit = unversioned. | (unversioned) |
| `--max-pages, -p <n>` | Cap total pages crawled. | server default |
| `--max-depth, -d <n>` | Cap link-follow depth. | server default |
| `--max-concurrency, -c <n>` | Parallel fetches for this job. | server default |
| `--scope <subpages\|hostname\|domain>` | Crawl boundary relative to `url`. | `subpages` |
| `--follow-redirects` / `--no-follow-redirects` | Follow HTTP redirects. | follow |
| `--ignore-errors` | Skip pages that error instead of aborting. | true |
| `--scrape-mode <auto\|fetch\|playwright>` | `auto` picks fetch vs headless browser. | `auto` |
| `--include-pattern <glob>` | Only crawl matching paths (repeatable). | — |
| `--exclude-pattern <glob>` | Skip matching paths (repeatable; wins over include). | — |
| `--header "<Key: value>"` | Extra request header, e.g. auth (repeatable). | — |
| `--embedding-model <model>` | Override embedding model for this job. | server env |
| `--server-url <url>` | Delegate to a running server/worker. | local |
| `--clean` / `--no-clean` | Wipe existing docs for this library+version first / append. | **`--clean`** |
| `--preserve-hashes` | Keep `#/route` fragments (hash-routed SPAs). | off |

Global: `--output json|yaml|toon`, `--quiet`, `--verbose`.

Notes:
- CLI flags are **kebab-case** (`--max-pages`); the MCP tool params are
  camelCase (`maxPages`) — don't mix them.
- `scrape` **blocks until the job completes** and reports pages scraped.
- `--clean` is **on by default**: re-scraping the same `library`+`version`
  **replaces** those docs (no duplicates). Use `--no-clean` to append.
- Local files: use `file:///abs/path` as the `url`.

Example:
```bash
npx -y @arabold/docs-mcp-server@latest scrape react https://react.dev/reference/react \
  --version 19 --scope subpages --max-pages 200 --max-depth 4 \
  --server-url http://localhost:6280/api
```

### `refresh <library> [--version <v>] --server-url <url>`
Incremental re-scrape using HTTP **ETags** — skips unchanged pages. Much faster
than a clean re-scrape; preferred for periodic freshening of an already-indexed
library.

### `search <library> <query> [options]`
| Flag | Meaning | Default |
|---|---|---|
| `--version, -v <label>` | Version to search. | `latest` |
| `--limit, -l <n>` | Max results (1–100). | 5 |
| `--exact-match, -e` | Disable fuzzy/semantic, exact only. | off |

Use this to **verify** a scrape: ≥1 relevant hit = docs are searchable.

### `list [--output yaml|json] --server-url <url>`
Lists indexed libraries, their versions, and doc counts. Use for preflight
(reachability + what already exists) and for polling job progress (counts climb
while a background scrape runs).

### Others
- `find-version <library> [-v <pattern>]` — resolve which indexed version matches.
- `remove <library> [--version <v>]` — delete a library or one version (destructive).
- `fetch-url <url>` — fetch a single page as Markdown, **not** indexed (one-off peek).

## Server topology / ports

- Default port **6280** for everything in standalone mode.
- HTTP mode serves: MCP at `/mcp` (streamable-HTTP) and `/sse` (SSE), the job
  web UI, an embedded worker, and a tRPC API at `/api`.
- The CLI's `--server-url` is the tRPC API at **`/api`** (`http://host:6280/api`),
  *not* `/mcp` or `/sse`. MCP clients use `/mcp` or `/sse`; the CLI uses `/api`.
- This project's compose service: `ghcr.io/arabold/docs-mcp-server:latest`,
  `--protocol http --host 0.0.0.0 --port 6280`, reachable at
  `http://localhost:6280` from the host and `http://docs-mcp-server:6280`
  inside the compose network. `.env` exposes `DOCS_MCP_URL` (the `/mcp`
  endpoint, for the backend's MCP client) — for the CLI, derive the API URL by
  stripping `/mcp` and appending `/api` → `http://localhost:6280/api`.

## Embeddings (server-side prerequisite)

- Set on the **server process at startup** via `DOCS_MCP_EMBEDDING_MODEL`
  (`provider:model`, e.g. `openai:text-embedding-3-small`) + provider key
  (`OPENAI_API_KEY`, `GOOGLE_API_KEY`, Azure vars, or Ollama base URL). This
  project's compose passes `OPENAI_API_KEY` and `DOCS_MCP_EMBEDDING_MODEL`.
- Scraping/indexing works **without** embeddings, but **semantic search is
  degraded** — that's the usual cause of "scrape succeeded, search returns 0".
- This skill **cannot** set embedding config per scrape; it can only report when
  it looks unconfigured.

## Versions & idempotency

- `version` is a free-form label (`19`, `0.2`, `15.4`). Keep it short + stable
  so re-runs stack under one library. Omit (or `null` at the tool layer) =
  unversioned bucket.
- `search`/`remove`/`refresh` default to the **latest** indexed version when
  `--version` is omitted.
- `scrape --clean` (default) replaces same library+version in place — safe to
  re-run.

## MCP tool equivalents (alternative to CLI)

If a session has the server registered as an MCP server, these tools mirror the
CLI. Tool params are **camelCase**.

- `scrape_docs(library, url, version?, maxPages?, maxDepth?, scope?,
  followRedirects?, ignoreErrors?, scrapeMode?, includePatterns?,
  excludePatterns?, headers?)` — returns `{pagesScraped}` (waits) or `{jobId}`
  (when `waitForCompletion:false`).
- `search_docs(library, query, version?, limit=5, exactMatch=false)`
- `list_libraries()`, `find_version(library, version?)`, `remove_docs(library, version?)`
- `refresh_docs(library, version?)` — ETag incremental re-scrape.
- `fetch_url(url)` — single page → Markdown, not indexed.
- Async jobs: `list_jobs()`, `get_job_info(jobId)`, `cancel_job(jobId)`.
  States: `QUEUED → RUNNING → COMPLETED` (or `FAILED`/`CANCELLED`). Poll
  `get_job_info` until terminal when you enqueue with `waitForCompletion:false`.

Prefer the CLI for this skill — it needs only a URL and Node, no MCP wiring.
