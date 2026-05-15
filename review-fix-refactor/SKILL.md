---
name: review-fix-refactor
description: Three-stage parallel-subagent pipeline. Multi-perspective code + architecture review → parallel code-generation to apply the most-recommended fixes → parallel refactoring. Use when the user wants a deep, multi-expert cleanup pass over a scope. Scope can be a file path, a directory path, or a natural-language prompt that the skill resolves to a file list. Works on any project regardless of language, framework, or source-control state. Examples "/review-fix-refactor src/", "/review-fix-refactor app.py", "/review-fix-refactor packages/api/handlers/", "/review-fix-refactor the authentication module", "/review-fix-refactor everything that touches the SSE event broker".
---

# Review · Fix · Refactor

Three-stage parallel-subagent pipeline. Execute the phases in order. Within a phase, fan out subagents in parallel. Between phases, wait for every agent to return before moving on.

This skill is project-agnostic and source-control-agnostic. Do not call `git diff`, `git status`, or any other git command. Read files directly from the filesystem. Operate against any directory, in any language, regardless of whether the project is version-controlled.

## Phase 0 — Resolve scope

Read the argument the user passed with the slash command. Accept three shapes:

1. **A single file path** → the scope is exactly that file.
2. **A directory path** → recursively enumerate source files. Apply common exclude patterns to avoid noise: `node_modules/`, `.venv/`, `venv/`, `__pycache__/`, `dist/`, `build/`, `target/`, `.git/`, `out/`, `coverage/`, lock files (`package-lock.json`, `yarn.lock`, `pnpm-lock.yaml`, `uv.lock`, `poetry.lock`, `Cargo.lock`, `Gemfile.lock`), and any path component starting with `.`.
3. **A natural-language prompt** (anything that isn't a valid filesystem path) → resolve to a concrete file list by launching one `Explore` subagent. Brief it with the user's prompt verbatim plus the working directory, and request a ranked list of files most relevant to the prompt, capped at 50. Apply the same exclude patterns to its output. Use the returned list as the scope.

### Disambiguation rule

Try the argument as a filesystem path first (relative to the working directory or absolute). Use `Read` for a single path, `Glob` for a directory walk.

- Resolves to an existing file or directory → **path mode**.
- Doesn't resolve AND argument is **path-like** (starts with `./` or `/`, or contains `/`) → stop with `scope not accessible: <path>`. Do NOT fall through to prompt mode for a broken path; it almost always means the user typed the path wrong.
- Doesn't resolve AND argument is not path-like → **prompt mode**. Launch the Explore subagent.

If no argument is passed → echo `scope required: pass a file path, a directory path, or a natural-language prompt` and stop. No implicit fallback to the current directory.

### Materialize the scope

After resolution, write three temp files so every subagent shares one view of the scope without re-walking the filesystem:

- `$TMPDIR/rfr_files.txt` — one resolved file path per line.
- `$TMPDIR/rfr_scope.txt` — concatenated file contents. Wrap each file's body with:
  ```
  --- BEGIN <path> ---
  <file contents>
  --- END <path> ---
  ```
- `$TMPDIR/rfr_scope_meta.txt` — two lines: the original argument verbatim, then `mode=path` or `mode=prompt`.

### Echo the resolved scope

Print a concise inline summary to the user before launching Phase 1: total file count + the first 20 paths + `... and N more` when truncated. For prompt-mode invocations this is load-bearing — the user needs to see which files the Explore agent picked so they can interrupt and re-invoke with a refined prompt if the resolution is wrong.

### Phase 0 failure handling

| Failure | Action |
|---|---|
| No scope argument | Echo `scope required`, stop. |
| Path-like argument doesn't resolve | Echo `scope not accessible: <path>`, stop. |
| Directory walk yields zero files after excludes | Echo `no source files in scope under <path>`, stop. |
| Prompt-mode Explore agent returns zero files | Echo `no files matched the prompt: <prompt>`, stop. Suggest narrowing or broadening the prompt. |
| Prompt-mode Explore agent crashes | Echo the error, stop. Suggest re-invoking with a path-mode argument as a workaround. |

## Phase 1 — Five parallel reviewers

Use the Agent tool to launch all five reviewer agents concurrently in a single message. Pass each agent the full scope so it has the complete context. Wait for every agent to return before moving on to Phase 2.

The five slots:

| Slot | `subagent_type` | Focus + brief |
|---|---|---|
| 1 | `code-reviewer` | Overall code quality, bug risk, anti-patterns, missed best practices. |
| 2 | `architect-reviewer` | SOLID, layer boundaries, module coupling, abstraction leaks, dependency direction. |
| 3 | `general-purpose` | Reuse: flag new code that duplicates existing utilities; suggest the existing function to use instead. |
| 4 | `general-purpose` | Efficiency: unnecessary work, missed concurrency, hot-path bloat, recurring no-op updates, unbounded memory. |
| 5 | `general-purpose` | Security: command injection, XSS, SQLi, unsafe deserialization, auth/authz bypass, secret exposure. |

### Reviewer prompt template

For each slot, brief the agent with this structure:

- Open with the slot's focus brief (one paragraph).
- Tell the agent to read `$TMPDIR/rfr_scope.txt` for the materialized scope, and `$TMPDIR/rfr_files.txt` if it wants to inspect specific files in isolation.
- State the working directory and remind the agent it may Read additional adjacent context if needed (e.g. neighboring files) but should not modify anything in this phase.
- End with the output contract verbatim:

> Report findings as a JSON array (one object per finding) with fields: `file` (string), `line` (int or null), `severity` (`"blocker"` / `"warning"` / `"info"`), `category` (string), `finding` (one-sentence summary), `suggested_fix` (concrete action). Cap output at 25 findings. Skip false positives. Under 600 words of narrative beside the JSON.

### Phase 1 failure handling

| Failure | Action |
|---|---|
| One reviewer crashes or returns malformed JSON | Continue with the four that returned. Mark the missing perspective in Phase 5's report. |
| All five reviewers crash | Stop. Report the error; the pipeline can't proceed without findings. |

## Phase 2 — Aggregate + prioritize

Do this in the main loop (not a subagent).

1. **Parse the five JSON blobs**. If a reviewer's output isn't valid JSON, extract the JSON array between the first `[` and the last `]` and retry. If still invalid, drop that reviewer's findings and continue.
2. **Hash each finding** by `(file, normalized_summary)`. Normalize: lowercase, strip articles (`the`, `a`, `an`) + punctuation, take the first 60 chars.
3. **Count reviewer agreement** per hash. Tag each finding with `agreement = N` (how many reviewers raised it).
4. **Priority rule (this is what "most recommended" means)**:
   - `agreement >= 2` OR `severity == "blocker"` → include in the fix list.
   - Otherwise → defer. Single-reviewer non-blocker suggestions surface in Phase 5 as advisory; they do NOT enter Phase 3.
5. **Group findings by file**. Each `file` becomes one work item containing all its prioritized findings.
6. **Partition files into agent slices**. Target ≤5 partitions, each owning ≤8 files. Distribute by file count (not by finding count) so each agent gets a roughly equal workload. If the fix list spans 1 or 2 files, use a single agent.

Print a one-screen summary before Phase 3: `N files, M total fixes, X agents, top 5 categories`.

If the fix list is empty (no `agreement >= 2` AND no blockers): **skip Phase 3 and Phase 4**. Go straight to Phase 5 with a `code is clean from a 5-perspective review; no consensus issues to fix` verdict and the deferred suggestions appended as advisory.

## Phase 3 — Parallel code-generation (file-partitioned)

Use the Agent tool to launch one agent per file partition concurrently in a single message. Each agent is `subagent_type: general-purpose`. Wait for every agent to return before moving on to Phase 4.

### Per-agent prompt template

- Open with: `You are applying review-validated fixes to a partition of files. Your partition is exclusive — no other agent will touch these files concurrently.`
- List the partition's file paths.
- For each file, list the prioritized findings as a numbered list: `<file>:<line>` (or `<file>` when line is null), severity, finding, suggested_fix.
- Hand-off the materialized scope path (`$TMPDIR/rfr_scope.txt`) for context, plus `$TMPDIR/rfr_files.txt` for the full project view.
- Add the **non-negotiable directive** verbatim:

> You may only edit the files in your partition. Do not Read or Edit any file outside this list. If a fix you've been assigned requires a change in another partition's file, leave a `TODO(review-fix-refactor): cross-partition fix needed: <description>` comment at the call site in your file instead (use the language-appropriate comment syntax). Do not chase the change into the other file.

- Add the **lint/test directive** verbatim:

> After applying all fixes, scan the project root for lint, typecheck, and test configuration (e.g. `pyproject.toml`, `package.json` scripts, `tsconfig.json`, `Cargo.toml`, `Gemfile`, `Makefile`, `composer.json`, `pom.xml`, `build.gradle`). Run any commands that apply to the files you modified — `ruff check`, `npx tsc --noEmit`, `cargo check`, etc. — scoped to your partition where possible. Report which commands you ran, the pass/fail state, and any errors. If you cannot identify a relevant command for the project, skip this step and say so explicitly.

### Aggregate Phase 3 output

After every agent returns:

- Collect the list of files each agent actually modified.
- Write `$TMPDIR/rfr_phase3_touched.txt` — one path per line — for Phase 4 to consume.
- Note any lint/test failures and any cross-partition TODO comments left behind. Both go into Phase 5's report.

### Phase 3 failure handling

| Failure | Action |
|---|---|
| One code-gen agent fails | Continue with the others. Surface the failed partition + which files weren't touched in Phase 5. |
| One agent reports lint/test failures | Surface in Phase 5. Do NOT roll back automatically — the user reviews and decides. |
| All code-gen agents fail | Skip Phase 4. Phase 5 reports the across-the-board failure. |

## Phase 4 — Parallel refactoring (after Phase 3 completes)

Phase 4 starts only when every Phase 3 agent has returned (success or failure). No Phase-4 work overlaps with Phase 3.

Use the Agent tool to launch the refactoring agents concurrently in a single message. Each agent is `subagent_type: refactoring-specialist`. Recompute the file partitions against `$TMPDIR/rfr_phase3_touched.txt` — only files Phase 3 actually modified are in scope. Target ≤5 agents, each owning ≤8 files.

### Per-agent prompt template

- Open with: `You are refactoring an exclusive partition of files that were just modified by automated fixes. Your scope is closed — no other refactoring agent will touch your files. Behaviour preservation is non-negotiable: do not change observable semantics.`
- List the partition's files.
- Hand off the architecture-reviewer's findings from Phase 1 as steering context. Filter to the findings whose `file` is in this partition.
- State the refactoring mandate:

> Apply the full refactoring playbook to the files in your partition. Targets: long-method extraction (split methods that exceed ~50 lines or have >3 levels of nesting); deep-nesting flattening (guard clauses, lookup tables, early returns); duplicate-block consolidation **within your partition only** (do not chase duplicates that span other partitions); design-pattern application where the architecture-reviewer flagged opportunities (strategy, template-method, builder, etc.). Preserve every public API and observable behaviour. If you'd need to change a public signature, leave a TODO comment instead and keep the existing signature.

- Reuse the **exclusivity directive** from Phase 3 (only edit files in your partition).
- Reuse the **lint/test directive** from Phase 3, scoped to your partition, so each agent verifies its own slice didn't regress.

### Phase 4 failure handling

| Failure | Action |
|---|---|
| One refactoring agent fails | Continue with the others. Surface the failed partition in Phase 5. |
| Lint/test failures after refactoring | Surface in Phase 5. Do NOT roll back automatically. |

### Cross-partition refactoring tradeoff (called out so future-me doesn't think it's a bug)

File-based partitioning means cross-partition duplicate consolidation won't be caught (a duplicated block split between agent A's files and agent B's files stays duplicated). This is the same compromise Phase 3 makes for the same reason: exclusive write privilege per agent eliminates merge conflicts by construction. If the user routinely needs cross-partition refactoring, future revisions of this skill can add an isolation-based design; the current default favors safety.

## Phase 5 — Final summary

Write a single report to the user covering:

1. **Scope**: original argument + resolved mode (`path` / `prompt`) + total file count. Pull from `$TMPDIR/rfr_scope_meta.txt`.
2. **Phase 1 reviewer activity**: per-slot findings count, total findings, any reviewer that crashed.
3. **Phase 2 prioritization**: how many findings were prioritized (`agreement >= 2` OR blocker), how many deferred, top categories.
4. **Phase 3 code-gen activity**: per-agent file count + finding count, lint/test results when commands were detected, any cross-partition TODOs left behind.
5. **Phase 4 refactoring activity**: per-agent file count, what was refactored (extracted methods, flattened nesting, consolidated duplicates, applied patterns), lint/test results.
6. **Deferred suggestions appendix**: the single-reviewer findings that didn't meet the agreement threshold, grouped by file. The user can act on these manually if interested.
7. **Suggested next steps**: inspect the modified files, run the project's full test suite, save / commit using whatever workflow the user has (git, hg, no source control at all).

Do NOT auto-commit or auto-save outside the file edits the subagents already performed. The user reviews and persists changes manually.

## Operating principles

- **Parallel within a stage; sequential between stages.** Phase 1, Phase 3, and Phase 4 each fan out their agents in a single message. Phase 2 and Phase 5 happen in the main loop. Each phase must complete before the next begins.
- **One source of truth.** Every subagent reads `$TMPDIR/rfr_scope.txt`. Don't restate the scope inline in agent prompts — point them at the file.
- **No source-control commands.** Don't call `git diff`, `git status`, `git add`, `git commit`, or any equivalent in hg, jj, fossil, etc. The skill must work against any project regardless of source control.
- **No project-specific assumptions.** Don't assume the project uses any particular language, framework, build tool, lint tool, or test runner. Detect what's there from configuration files; skip steps gracefully when nothing is detected.
- **Don't roll back automatically.** If lint/tests fail after fixes or refactoring, surface the failure in Phase 5 and let the user decide. Auto-rollback is destructive.
- **Exclusivity by file partition.** Both Phase 3 and Phase 4 partition by file with exclusive ownership. Don't try concern-based partitioning — two concerns can target the same line and collide.
