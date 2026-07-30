# ast-grep (`sg`) — Tree-sitter-based Code Structure Tool

**Usage**: Structural code search and extraction powered by Tree-sitter AST parsing.
Accuracy: AST-based and grammar-dependent; Tree-sitter error recovery and the queried shape can limit results, so this tool does not prove semantic correctness or absence.

Verified locally on Windows with ast-grep 0.44.0 and RTK 0.43.0 on 2026-07-13. Re-check passthrough and query behavior after upgrades. This file is compatible with `AGENTS.md` 2.73.x; compatibility was re-verified on 2026-07-30.

## Quick Reference

### Kind-based extraction

```bash
# Compact declarations when relevant
rtk sg run --kind import_statement --lang tsx <path>
rtk sg run --kind interface_declaration --lang tsx <path>
rtk sg run --kind type_alias_declaration --lang tsx <path>

# These kinds include complete matched bodies and may be large
rtk sg run --kind export_statement --lang tsx <path>
rtk sg run --kind function_declaration --lang tsx <path>
```

`export_statement` is not a signature-only outline. In local tests it emitted 180/220 lines and 63/83 lines for exported functions with large bodies. Use it only for small files or short exports; narrow any output larger than needed.

### Pattern-based search (structural grep)

```bash
# Find files containing React hook calls before printing all matches
rtk sg run -p 'useState($$$ARGS)' --lang tsx <path> --files-with-matches

# Find specific function calls
rtk sg run -p 'parseFliAction($$$ARGS)' --lang tsx <path>

# Find `const` arrow-function assignments matching this exact shape
rtk sg run -p 'const $NAME = ($$$ARGS) => $BODY' --lang tsx <path>
```

PowerShell MUST use single quotes around patterns containing `$`; double quotes expand metavariables and can silently produce a different/no-match query. Use literal-safe quoting on other shells as well.

### Language support

Common `--lang` values: `tsx`, `typescript`, `javascript`, `jsx`, `css`, `json`, `html`, `python`

## When to use

For every covered project-code task, the mandatory Codegraph attempt precedes `sg`. Run `sg` before the first deliberate non-Codegraph full read of a code file not already fully read since its latest change and in each `CODEGRAPH.md` fallback that requires structural source inspection. Codegraph-returned source marked current/Read-equivalent by its active contract is already read; never invoke `sg` solely to re-verify it. Contract-reported stale, pending, frozen, unindexed, or unsupported states still follow the scoped handling in `CODEGRAPH.md`.

| Situation | Use sg? |
|---|---|
| Need structure not supplied by Codegraph after an authorized fallback | **YES** — use the narrowest relevant kind/pattern; do not default every file to exports |
| Looking for a symbol/pattern in a current indexed project | Use Codegraph; do not invoke `sg` merely to re-verify its current result |
| Broad structural discovery after a built-in-tools fallback | **YES** — start with `--files-with-matches`, then inspect selected files |
| Already know an exact small line range | Usually NO full-file probe; targeted reads remain allowed only when otherwise authorized by the active Codegraph contract and `AGENTS.md`, and a deliberate range sequence intended to reconstruct the entire file is a full-file read requiring `sg` before its first range |
| File already read and unchanged in current session | NO — already in context; run a new prerequisite only after a later change makes that read stale |
| Cross-file relationships (callers, trace, impact) | Use a narrower Codegraph query; use `sg` only after an active-contract or documented failure fallback authorizes built-in tools |

## Token saving strategy

```
Step 1:  Use --files-with-matches or a compact declaration kind for discovery
Step 2:  Narrow to the relevant file, symbol, kind, or call pattern
Step 3:  If output contains large bodies or >~80 lines, narrow again
Step 4:  Read only the specific lines/sections still needed
```

## Exit semantics and governing fail-closed behavior

- Diagnostic-first: any diagnostic containing an `ERROR node` or identifying an invalid selector, kind, pattern, path, or parse state is a query error regardless of exit code. Do not count it as a match or as satisfying the `sg` prerequisite. Correct once when clear; if it still fails, report it and do not perform a full code-file read, continuing only with narrower sources permitted by `AGENTS.md`.
- Exit 0 counts as matches found only when actual match output is present and no query-error diagnostic above appears.
- Exit 1 with no diagnostic is a valid no-match. It satisfies the `sg` prerequisite only when the query is syntactically valid, narrow, and task-relevant, `--lang` matches the target file's language, and the query was invoked on the exact target file rather than a directory or glob that may skip it; a wrong-`--lang`, irrelevant-shape, or unproven target scan does not count. If structure is expected, try one materially relevant alternative kind/pattern on that exact file for additional shape coverage, but the first eligible exact-target no-match already satisfies the prerequisite and proves absence only for its queried shape.
- Explicit unsupported-language diagnostic: use the narrow parser fallback in `CODEGRAPH.md` only when one of that reference's explicit triggers holds. The fallback still requires that Codegraph and `sg` were both attempted.
- Binary/command unavailable: report once per session and do not perform a full code-file read. Continue only with scoped Codegraph source, already-present context, non-code work, or reads that are not full-file; if completion requires a full code-file read, report the task as blocked.
- Any other non-zero exit, interrupted/I/O/permission state, or unclassified result is a query failure and does not satisfy the `sg` prerequisite. Correct or retry once only when a diagnostic establishes a clear non-destructive command/query issue; otherwise use an applicable documented `CODEGRAPH.md` fallback. If no such fallback authorizes the needed structural inspection or full-file read, report the blocker.

Do not use `--rewrite` merely to print signatures; without update mode it emits a potentially huge diff.

## RTK integration

All `sg` commands are shell commands → must be prefixed with `rtk`:
```bash
rtk sg run --kind interface_declaration --lang tsx src/
rtk sg run -p 'useState($$$ARGS)' --lang tsx src/components/ --files-with-matches
```

With RTK 0.43.0, `rtk sg ...` is a verified implicit passthrough. Re-verify after RTK upgrades; if it stops working, use the governing RTK fallback instead of bare `sg`.

## Installation verification

```bash
rtk sg --version    # Should show: ast-grep X.Y.Z
```
