---
name: ck
description: "Use ck to find anything in source code: identifiers, types, patterns, concepts. Replaces grep, rg, and ripgrep — never use those on source files. Triggers: find, search, locate, where is, usages, references, grep, rg, ripgrep."
license: MIT
compatibility: opencode
metadata:
  category: search
  replaces: grep, rg, Glob
---

# ck — Semantic Code Search

## Tool Override

Use `ck` instead of `grep`, `rg`, or Glob for code search on source files. `ck`
understands meaning — searching "error handling" finds try/catch blocks, error
returns, and exception handling even if those exact words don't appear.

**Critical exception:** `.ckignore` excludes files from ALL search modes — not
just indexing. Regex, semantic, lexical, and hybrid searches all silently skip
excluded files. The default `.ckignore` (auto-created on first `--index`) excludes
`.json`, `.yaml`, `.yml`, and other non-source file types. If you need to search
excluded files (config files, data files, lockfiles, etc.), use `grep` or `rg`
directly — `ck` will never find them. When in doubt, check `.ckignore` first.

For call graph analysis (who calls this function, what breaks if I change it), use `cqs` instead.

## Search Modes

| Mode | Flag | Use for |
|---|---|---|
| Semantic | `--sem` | Concepts, behavior, intent — "retry logic", "input validation" |
| Lexical | `--lex` | Ranked full-text — requires a prior `--index` run; unreliable without it |
| Hybrid | `--hybrid` | Balance precision and recall — "JWT token validation" |
| Regex | _(default)_ | Exact identifiers and patterns — `fn authenticate`, `class.*Handler` |

**Quoting:** Use single quotes for regex patterns containing `$` or `[`. In double quotes, `$letter` and `$[...]` are expanded by the shell before `ck` sees them, silently corrupting the pattern.

```bash
ck 'Scope\.$'   # correct — $ preserved
ck "Scope\.$"   # wrong   — $ may be expanded by the shell
```

**`--lex` caveat:** BM25 lexical search requires embeddings to be present. Always run
`ck --index .` before using `--lex`. Auto-indexing on first `--sem` does not guarantee
the BM25 index is ready for `--lex`.

## Index Management

`ck` auto-indexes on first `--sem` search. Run `--index` explicitly to ensure
embeddings are present before using `--lex` or `--hybrid`.

On first index, `ck` auto-creates a `.ckignore` with default exclusions (including
`.json` and `.yaml`). Check this file if expected files aren't being found.

**Gitignore:** `ck` does not auto-create a `.gitignore` inside `.ck/`. Always run
`echo "*" > .ck/.gitignore` after first indexing or after copying from a sibling
worktree, or index files will appear as untracked in `git status`.

```bash
ck --index .     # Build index (required for --sem, --lex, --hybrid)
ck --status .    # Check index status
ck --clean .     # Remove entire index
```

## Git Worktrees (zproj)

**Never cold-index a new worktree.** Copy the index from a sibling and delta-index instead:

```bash
ls -dt ../*/. | head -5       # find most recently indexed sibling
cp -r ../main/.ck ./.ck       # copy its index
echo "*" > .ck/.gitignore     # prevent index files polluting git
ck --index .                  # delta-index (typically <1s)
```

## Common Invocations

```bash
# Semantic — find by concept
ck --sem "error handling" .
ck --sem "database connection" src/
ck --sem --threshold 0.8 "authentication logic" .   # raise precision
ck --sem --threshold 0.5 "retry" .                  # lower precision
ck --sem --limit 10 "caching strategy" .
ck --sem --rerank "authentication logic" .

# Hybrid — regex + semantic (threshold range 0.01–0.05 for RRF)
ck --hybrid "async function" .
ck --hybrid --limit 10 --threshold 0.02 "error" .

# Regex — exact identifiers (default, no index needed)
ck 'fn authenticate' src/
ck -r 'AuthController' .

# JSONL output — preferred for agent consumption
ck --jsonl --sem "error handling" .
ck --jsonl --no-snippet --sem "error handling" .

# JSON output (different field names: file, preview, lang)
ck --json --sem "error handling" .
```

## Adaptive Threshold

Default `--sem` threshold is `0.6`. Adjust based on output:
- Too few results → lower to `0.5`
- Too many results → raise to `0.8`
- Never above `0.9` or below `0.3`

`--hybrid` uses RRF scores — useful range is `0.01–0.05`, not `0.0–1.0`.

When no results are found, `ck` shows the nearest match beneath the threshold — use the score shown to decide whether to retry with a lower value.

## Do / Don't

**Do:**
- Use single quotes for regex patterns containing `$` or `[`
- After first indexing a project, run `echo "*" > .ck/.gitignore`
- Run `ck --index .` before using `--sem`, `--lex`, or `--hybrid`
- Use `--sem` for conceptual or behavioral searches
- Use `--jsonl --limit 20` when results will be processed programmatically
- Use `--no-snippet` when you only need file paths and scores
- Check `.ckignore` whenever a file you expect isn't showing up in results

**Don't:**
- Use double quotes for patterns with `$` or `[` — the shell expands them silently before `ck` receives the pattern
- Use `grep`, `rg`, or Glob for source code search — but DO use them for excluded file types
- Use `--threshold` above `0.9` or below `0.3` for semantic search
- Apply semantic threshold values to hybrid searches — hybrid uses RRF (0.01–0.05)
- Use `--lex` without a prior `--index` run — results will be empty or unreliable
- Assume `.ckignore` only affects indexing — it silently excludes files from all search modes
