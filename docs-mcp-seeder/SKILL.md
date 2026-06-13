---
name: docs-mcp-seeder
description: >-
  Populate a running docs-mcp-server (arabold/docs-mcp-server) with the latest
  official documentation for a set of libraries/topics. Given a server URL and
  fuzzy "documentation categories" (e.g. "LangGraph, Next.js 15, FastAPI"), this
  skill uses web search to resolve each category to its canonical docs URL and
  current version, then enqueues scrape/index jobs on the server, polls them to
  completion, and verifies the index with a search. Use this WHENEVER the user
  wants to seed, populate, fill, refresh, re-index, or "scrape the latest docs
  into" a docs-mcp-server — including phrasings like "my docs-mcp-server is
  missing docs for X", "index the React 19 docs", "add these libraries to the
  docs server", "freshen the docs index", or "scrape docs for <library> at
  <url:port>". Also use when someone references the @arabold/docs-mcp-server CLI,
  the scrape_docs / search_docs MCP tools, or a docs index on port 6280.
---

# docs-mcp-seeder

Seed a running **docs-mcp-server** with up-to-date official documentation.

The server (arabold/docs-mcp-server) already knows how to crawl, chunk, and
embed a documentation site — it just needs to be *told what to fetch*. Your job
is the part it can't do: take the user's fuzzy list of libraries/topics, figure
out the **canonical docs URL and the right version** for each via web search,
hand those to the server's scraper, and confirm the docs actually landed in the
index. You are the librarian deciding what goes on the shelves; the server is
the machine that binds the books.

Drive ingestion through the **CLI** (`npx @arabold/docs-mcp-server@latest …
--server-url <URL>`). This delegates the actual scraping to the running server,
so you only need a reachable URL — no MCP connection or local install of the
package's data store. The full command/flag reference lives in
`references/docs-mcp-cli.md`; read it before your first scrape so you use the
correct flag names (they are kebab-case on the CLI, e.g. `--max-pages`).

## Inputs you need

1. **Server URL** — where docs-mcp-server is listening, e.g. `http://localhost:6280`.
   - The CLI's `--server-url` is the **tRPC worker endpoint and must end with
     `/api`** — it is *not* the MCP path and *not* the bare base. Normalize
     whatever the user gives you to `<scheme>://<host>:<port>/api`:
     strip any trailing `/mcp`, `/sse`, or `/` and append `/api`. So
     `http://localhost:6280`, `http://localhost:6280/mcp`, and
     `http://localhost:6280/sse` all become **`http://localhost:6280/api`**.
   - Verify this once at preflight: if `list --server-url …/api` returns data,
     the endpoint is right. A "Failed to connect… ends with '/api'" error means
     you passed the wrong path.
2. **Documentation categories** — a list of libraries/topics in whatever form
   the user typed them ("LangGraph", "Next.js 15 App Router", "the FastAPI
   docs", "anthropic python sdk"). These are *intentions*, not URLs; resolving
   them is the core work below.

If either is missing, ask once — but infer sensibly: a bare list of libraries
with no URL almost always means the local default `http://localhost:6280`
(confirm before scraping). If the user is in a repo, check `.env` /
`docker-compose.yml` for a `DOCS_MCP_URL` and offer it as the default.

## Workflow

### 1. Preflight — is the server up, and what's already indexed?

```bash
npx -y @arabold/docs-mcp-server@latest list --server-url <API_URL> --output json
```

(`<API_URL>` is the normalized `…/api` endpoint from above.)

- A clean JSON/table response means the server is reachable. A connection error
  (especially "…ends with '/api'") means stop — fix the URL or tell the user
  it's unreachable. Don't try to scrape against a bad endpoint.
- Note which `library` + `version` pairs already exist, their doc counts, **and
  the exact version label each uses**. This drives three decisions in step 3:
  - **scrape vs refresh** per category;
  - whether to skip anything already well-indexed (if the user only asked to
    "fill gaps");
  - **which version label to reuse** — see the trap below.
- A library showing a **very low or zero `documentCount`** (e.g. `langgraph`
  with 3 docs, `anthropic-api` with 0) is a *failed/partial* prior scrape, not a
  healthy entry — treat it as needing a fresh scrape even though it "exists".

> **Version-bucket trap.** docs-mcp-server keys docs on (library, version), and
> an empty version `""` is its own bucket. If a library already exists as
> `version=""` and you scrape it with `--version 0.2`, you create a *second*,
> parallel bucket and leave the broken one in place — searches default to
> "latest" and may still hit the empty one. **Match the existing version label
> found in `list`.** If the server's convention is unversioned (common — and the
> case for this project's 150+ libraries), scrape with no `--version` so you
> overwrite the right bucket. Only introduce a version label when the user
> explicitly wants version isolation *and* the library isn't already present
> under a different label.

### 2. Resolve each category → canonical URL + version (the web-search core)

For **each** category, run a web search and decide four things. Resolve all
categories in one batch of parallel searches where possible — they're
independent.

| Field | How to decide |
|---|---|
| `library` | A short, stable, lowercase slug you'll reuse for search later (`langgraph`, `nextjs`, `fastapi`). **If the category already exists in the preflight `list`, reuse that exact slug** so you update it in place instead of creating a near-duplicate. Keep it consistent across re-runs. |
| `url` | The **canonical docs root or API-reference root** on the *official* domain — not the marketing homepage, not a blog post, not a third-party tutorial, not an old versioned mirror. Prefer the page from which the doc tree fans out, so `--scope subpages` captures the whole set. |
| `version` | **Match the label already in the index for this library** (see the version-bucket trap above). If it's new and the server's convention is unversioned, omit `--version`. Only pin a clean label (`0.2`, `19`, `15` — major or major.minor, never a full patch) when the user wants version isolation. |
| `scope` | Almost always `subpages` (crawl under the URL path). Use `hostname`/`domain` only when the docs sprawl across the host and subpages would miss them — and bound it harder with `--max-pages`. |

Web-search judgment that matters:
- **Trust official sources.** `react.dev`, `nextjs.org/docs`, `fastapi.tiangolo.com`,
  `docs.pydantic.dev`, `langchain-ai.github.io/langgraph`. If the top hit is a
  package registry (npm/PyPI), follow its "Documentation"/"Homepage" link to the
  real docs site rather than scraping the registry page.
- **Pin the version the user implied.** "Next.js 15" → the v15 docs, not v14.
  If the user named no version, take current stable. If the site has a version
  switcher, capture the URL for the intended version.
- **Aim the URL at the doc tree, not the landing page.** `https://nextjs.org/docs`
  (fans out into the whole manual), not `https://nextjs.org`.

Produce a **resolution plan** — one row per category — and show it to the user
before scraping anything:

```
category            library    version  url                                          scope     action
LangGraph           langgraph  0.2      https://langchain-ai.github.io/langgraph/    subpages  scrape (new)
Next.js 15          nextjs     15       https://nextjs.org/docs                      subpages  refresh (exists)
FastAPI             fastapi    0.115    https://fastapi.tiangolo.com/                subpages  scrape (new)
```

Flag any category you **couldn't** confidently resolve (ambiguous name, no clear
official docs) instead of guessing a URL — ask the user for the URL for just
that one.

### 3. Confirm, then enqueue scrapes

Scraping crawls an external site and consumes server CPU + embedding API calls.
That's outward-facing and not free, so **confirm the plan once** before firing —
unless the user already said "just do it" / "scrape them all". Then for each row:

**New, or fixing a broken/partial entry → `scrape`** (defaults to `--clean`,
replacing any prior docs for that exact library+version — so it also *repairs*
an entry stuck at 0–few docs):

```bash
npx -y @arabold/docs-mcp-server@latest scrape <library> <url> \
  --scope subpages \
  --max-pages 200 --max-depth 4 \
  --server-url <API_URL>
# add `--version <label>` ONLY to match an existing non-empty label,
# or when the user explicitly wants version isolation. Omit for unversioned.
```

**Healthy entry you just want freshened → `refresh`** (ETag-based, skips
unchanged pages — much faster than a clean re-scrape; pass `--version` only if
the entry uses a non-empty label):

```bash
npx -y @arabold/docs-mcp-server@latest refresh <library> --server-url <API_URL>
```

Bounding the crawl matters — an unbounded scrape of a large docs domain is slow
and can balloon embedding costs. Start with `--max-pages 200 --max-depth 4` and
only raise it if a verification search in step 4 shows the docs are clearly
incomplete. If you bound it, **say so in the report** — silent truncation reads
as "fully indexed" when it isn't.

The CLI `scrape` blocks until the job finishes and prints the page count. To
seed many categories faster, run several in the background (the server processes
~3 jobs concurrently by default) and poll with
`list --server-url <API_URL> --output json` until counts stabilize. For a
handful of categories, sequential is simpler and fine.

### 4. Verify each library actually landed

A scrape that "succeeds" with 0 pages, or indexes pages but produces no
searchable chunks (e.g. the server has no embedding model configured), is a
silent failure. Confirm with a real query per library:

```bash
npx -y @arabold/docs-mcp-server@latest search <library> "<representative query>" \
  --limit 3 --server-url <API_URL>
# search defaults to the "latest" version; add `--version <label>` only if you
# scraped under a specific non-empty label.
```

Pick a query a developer would actually ask of that library ("how to define a
graph node", "app router layouts", "dependency injection"). Expect ≥1 relevant
hit. Zero hits after a successful scrape is the signal to investigate (see
Troubleshooting).

### 5. Report

Summarize as a table the user can act on:

```
ALWAYS report, per category:
  category | library@version | action taken | pages indexed | verify (hits) | status
```

- Lead with the headline: "Indexed N/M categories into <server>."
- List any **unresolved** categories and what you need from the user.
- List any category that scraped but **failed verification**, with the likely
  cause.
- If you capped any crawl (`--max-pages`/`--max-depth`), note which ones so the
  user knows coverage may be partial.
- Don't claim success for a library whose verification search returned nothing —
  report it honestly as "indexed, but search returned 0 results — check server
  embedding config."

## Troubleshooting

- **`--server-url` "Failed to connect… ends with '/api'"** — you passed the
  wrong path. The CLI needs the tRPC endpoint `…/api`, not `/mcp`, `/sse`, or
  the bare base. Re-normalize and retry once.
- **Connection refused / timeout even on `…/api`** — server down or wrong
  host/port. Confirm the container is running; don't retry blindly.
- **Scrape succeeds but `search` returns 0 hits** — the running server most
  likely has no embedding model configured (`DOCS_MCP_EMBEDDING_MODEL` +
  provider key are set on the *server process*, not by this skill). Docs still
  index, but semantic search is degraded. Surface this; you can't fix server
  config from the client.
- **Scrape indexes far fewer pages than expected** — the `url` was a landing
  page, not the docs root, or `--scope subpages` excluded sibling sections.
  Re-aim the URL at the doc tree, or widen scope to `hostname` with a higher
  `--max-pages`.
- **Wrong/old content shows up in search** — a stale version is indexed under
  the same library. `remove <library> --version <old>` then re-scrape, or scrape
  the correct version (note `scrape` defaults to `--clean`, replacing the same
  library+version in place).
- **A library has a semver version AND an unversioned bucket, and default
  searches return `[]`** — `find-version` resolves "latest" to the highest
  *semver* label **regardless of its doc count**, so an empty `1.2.3` bucket
  shadows a populated unversioned (`""`) one and breaks default search. This is
  the version-bucket trap biting after the fact. Check with
  `find-version <library>` (look at `bestMatch`). Fix by getting rid of the
  empty semver bucket — but note the next gotcha.
- **`remove --version X` reports success but the version still shows in `list`
  with `documentCount: 0`** — `remove` deletes the *documents* but can leave an
  orphan version row, and that empty row keeps shadowing "latest" (above).
  `remove <library>` (whole library, no `--version`) has the same limitation.
  If you must fully purge an orphan row, it lives in the server's store
  (`versions` table, linked to `libraries`); a zero-doc row can be deleted
  directly, but that's server-side surgery — only do it with the operator's
  go-ahead, when no scrape is active, and after confirming the row has 0 pages /
  0 docs. The cleaner everyday fix is to **avoid creating the semver bucket in
  the first place** by matching the existing unversioned convention (step 2).
- **`npx` can't find the package** — pass `-y` and pin
  `@arabold/docs-mcp-server@latest`; the CLI only needs Node, it talks to the
  remote server for the actual work.

## Reference

`references/docs-mcp-cli.md` — exact CLI commands, every `scrape` flag, the MCP
tool equivalents (if you ever need to drive it via tools instead of CLI),
server topology/ports, and version/idempotency semantics. Read it before your
first scrape in a session.
