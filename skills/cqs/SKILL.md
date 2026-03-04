---
name: cqs
description: "Use cqs for call graph analysis: callers, callees, impact of changing a function, dead functions, refactoring safety, diff risk. Requires a callable function name — does not find text, types, or identifiers. Triggers: who calls this, what breaks if I change, impact analysis, callers, callees, dead code, refactor function, rename function, review diff."
license: MIT
compatibility: opencode
metadata:
  category: code-intelligence
  complements: ck
---

# cqs — Code Intelligence CLI

## Setup (required before first use in any project)

```bash
cqs init        # Download ML model (~547MB, one-time per machine)
cqs index       # Index the codebase (builds embeddings + call graph + HNSW)
```

Run `cqs watch` afterward to keep the index fresh automatically.

**Re-index when:** new files were added outside of `watch`, or call graph shows 0 entries.

## Git Worktrees (zproj)

**Never cold-index a new worktree.** Copy the index from a sibling and delta-index instead:

```bash
ls -dt ../*/. | head -5        # find most recently indexed sibling
cp -r ../main/.cqs ./.cqs      # copy its index
cqs index                      # delta-index
```

For finding code by text, pattern, or concept, use `ck` instead — `cqs` does not search source files.

## Critical Syntax Rules

**`-q` is a global flag — it must come before the subcommand:**
```
cqs -q <command> <args> --json     ✓
cqs <command> <args> --json -q     ✗ (error: unexpected argument)
```

**`notes add/remove/update` do not accept `-q` at all.**

**`trace`, `impact`, `review`, `ci` use `--format json`, not `--json`:**
```
cqs impact "encode" --format json     ✓
cqs -q impact "encode" --json         ✗
```

## Recommended Workflows

### New task / feature implementation
```bash
cqs -q scout "<task description>" --json
cqs -q task "<task description>" --json
```

### Understanding unfamiliar code
```bash
cqs -q onboard "<concept>" --json
cqs -q gather "<concept>" --json
cqs explain "<function>" --json
```

### Before refactoring (rename / move / change signature)
```bash
cqs -q callers "<function>" --json
cqs impact "<function>" --format json
cqs -q deps "<type>" --json
cqs trace "<fn-a>" "<fn-b>" --format json
```

### Code review / pre-commit
```bash
cqs review --format json
cqs ci --format json
```

### Codebase health
```bash
cqs -q dead --json
cqs -q health --json
cqs -q stale --json
```

## Do / Don't

**Do:**
- Use single quotes for queries containing `$` — in double quotes, `$letter` is silently expanded by the shell, changing the query without any error
- Run `cqs init` + `cqs index` at the start of every new project
- Start with `scout` or `task` before implementing
- Use `impact` before any refactor — even "small" changes can have transitive callers
- Use `--format json` for `trace`, `impact`, `review`, and `ci`
- Use `-q` before the subcommand, never after
- Use `cqs watch` in the background during long coding sessions

**Don't:**
- Skip `cqs index` — without it, call graph and type graph will be empty
- Use `--json` with `trace` or `impact` — it will error; use `--format json`
- Append `-q` after the subcommand — it will error
- Run `cqs index` repeatedly — it's expensive; `watch` handles incremental updates

---

## Search & Discovery

### Semantic search — `cqs -q "<query>" [flags] --json`

**There is no `search` subcommand.** The query is a bare positional argument.

```bash
cqs -q "error handling" --json          # correct
cqs -q search "error handling" --json   # WRONG — search is not a subcommand
```

| Flag | Description |
|------|-------------|
| `--lang <L>` | Filter by language (rust, python, typescript, javascript, go, c, java, sql, markdown, scala) |
| `-n/--limit <N>` | Max results (default 5, max 20) |
| `-t/--threshold <N>` | Similarity threshold (default 0.3) |
| `--name-only` | Definition lookup, skips embedding |
| `--semantic-only` | Pure vector similarity, no hybrid RRF |
| `--rerank` | Cross-encoder re-ranking (slower, more accurate) |
| `-p/--path <glob>` | Path pattern filter (e.g., `src/cli/**`) |
| `--chunk-type <T>` | Filter: function, method, class, struct, enum, trait, interface, constant, section |
| `--pattern <P>` | Pattern: builder, error_swallow, async, mutex, unsafe, recursion |
| `--tokens <N>` | Token budget |
| `--expand` | Include parent context (boolean flag, no value) |
| `-C/--context <N>` | Lines of context around chunk |
| `--no-content` | Omit code content from display output |
| `--no-stale-check` | Skip per-file staleness checks |

### similar `<function>` — Find similar code
```
cqs -q similar "<name>" --json
```

### gather `<query>` — Smart context assembly
```
cqs -q gather "<query>" [flags] --json
```

| Flag | Description |
|------|-------------|
| `--expand <N>` | BFS depth (default 1, max 5) |
| `--direction <D>` | both, callers, callees (default both) |
| `-n/--limit <N>` | Max results (default 10) |
| `--tokens <N>` | Token budget |

### where `<description>` — Placement suggestion
```
cqs -q where "<description>" --json
```

### scout `<task>` — Pre-investigation dashboard
```
cqs -q scout "<task>" [-n N] [--tokens N] --json
```

### task `<description>` — Implementation brief
```
cqs -q task "<description>" [-n N] [--tokens N] --json
```

### onboard `<concept>` — Guided codebase tour
```
cqs -q onboard "<concept>" [-d/--depth N] [--tokens N] --json
```

### read `<path>` — File with contextual notes
```
cqs read <path> [--focus <function>] --json
```

`--focus <function>` returns only the target function + type dependencies.

### context `<file>` — Module overview
```
cqs context <path> [--compact] [--summary] --json
```

### explain `<function>` — Function card
```
cqs explain "<name>" --json
```

---

## Call Graph

### callers `<function>` — Who calls this?
```
cqs -q callers "<name>" --json
```

### callees `<function>` — What does this call?
```
cqs -q callees "<name>" --json
```

### trace `<source>` `<target>` — Shortest call path
Uses `--format json`, not `--json`.
```
cqs trace "<source>" "<target>" --format json
```

### deps `<name>` — Type dependencies
```
cqs -q deps "<name>" --json           # forward: who uses this type?
cqs -q deps --reverse "<name>" --json # reverse: what types does this function use?
```

### related `<function>` — Co-occurrence
```
cqs -q related "<name>" --json
```

### impact `<function>` — What breaks if you change X?
Uses `--format json`, not `--json`.
```
cqs impact "<name>" [--depth N] [--suggest-tests] [--include-types] --format json
```

### impact-diff — Diff-aware impact analysis
```
cqs impact-diff [--base <ref>] [--stdin] [--json] [--tokens N]
```

### test-map `<function>` — Map function to tests
```
cqs -q test-map "<name>" --json
```

### blame `<function>` — Semantic git blame
```
cqs blame "<name>" [--callers]
```

---

## Quality & Review

### dead — Find dead code
```
cqs -q dead [--include-pub] [--min-confidence low|medium|high] --json
```

### stale — Index freshness
```
cqs -q stale --json
```

### health — Codebase quality snapshot
```
cqs -q health --json
```

### ci — CI pipeline analysis (exit 3 on gate fail)
```
cqs ci [--base <ref>] [--stdin] [--gate high|medium|off] [--format json] [--tokens N]
```

### review — Comprehensive diff review
```
cqs review [--base <ref>] [--stdin] [--format json] [--tokens N]
```

### suggest — Auto-suggest notes
```
cqs -q suggest [--apply] --json
```

### gc — Clean stale index entries
```
cqs -q gc --json
```

### stats — Index statistics
```
cqs -q stats --json
```

---

## Notes

Notes do not accept `-q`.

### notes add `<text>`
```
cqs notes add "<text>" [--sentiment N] [--mentions a,b,c]
```
Sentiment: -1 (serious pain), -0.5 (notable pain), 0 (neutral), 0.5 (notable gain), 1 (major win).

### notes list
```
cqs notes list [--json] [--warnings] [--patterns]
```

### notes update `<exact text>`
```
cqs notes update "<exact text>" [--new-text "..."] [--new-sentiment N] [--new-mentions a,b,c]
```

### notes remove `<exact text>`
```
cqs notes remove "<exact text>"
```

### audit-mode
```
cqs audit-mode [on|off] [--expires 30m] --json
```

---

## Infrastructure

### ref — Reference index management
```
cqs ref add <name> <path> [--weight 0.8]
cqs ref list
cqs ref update <name>
cqs ref remove <name>
```

### watch — File watcher
```
cqs watch [--debounce <ms>] [--no-ignore]
```

### convert `<path>` — Document conversion
```
cqs convert <path> [-o/--output <dir>] [--overwrite] [--dry-run] [--clean-tags <tags>]
```
