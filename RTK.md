# RTK - Rust Token Killer (Codex CLI)
<!-- Compatible with AGENTS.md 2.73.x | Last updated: 2026-07-30 -->

**Usage**: Token-optimized CLI proxy for shell commands.

## Rule

Always prefix shell commands with `rtk`.

Examples:

```bash
rtk git status
rtk rg "TODO" .
rtk read README.md
rtk sg --version
```

## Routing Priority

For shell operations, use this order:

1. A verified RTK-native route that matches the operation.
2. `rtk proxy <command>` only when no verified route applies, the native route is incompatible, or unfiltered output is required for verification.
3. Never run an unprefixed shell command.

Do not choose `rtk proxy` merely because the exact native syntax is not recalled. Check this file or `rtk <subcommand> --help` first.

**Route failure vs domain exit**: A route-level failure means RTK or the proxied wrapper could not route or launch the target (binary-resolution, wrapper, or transport error); treat it as uncertain per `AGENTS.md` and use the documented fallback. A normal non-zero exit from a target RTK did run (no matches, test failures, or another domain exit code) is not a route failure and does not invalidate a verified native route. If instead an argument/usage error points to RTK route incompatibility (see Known-Incompatible routes below), treat it as route-level.

## Verified Native Routes on Windows

Verified locally with RTK 0.43.0 on 2026-07-13. Re-check after a future RTK upgrade.

| Operation | Required route | Notes |
|-----------|----------------|-------|
| Text/ripgrep search | `rtk rg <pattern> [path]` | Native compact ripgrep route; verified with RTK 0.43.0 and ripgrep 15.1.0 on 2026-07-11 |
| File read | `rtk read <file>` | Use `--max-lines`, `--tail-lines`, or `--level` when full content is unnecessary |
| File discovery | `rtk find ...` | Compact output; suitable replacement for broad PowerShell traversal |
| Git | `rtk git <subcommand>` | Includes `status`, `diff`, `log`, and `show`; use for repository state/history diffs |
| Direct file comparison | `rtk diff <file-a> <file-b>` | Verified with identical and different files; use `rtk git diff` for repository working-tree/index changes |
| JSON structure | `rtk json <file>` | Use `--keys-only` when values are unnecessary or sensitive |
| Dependency summary | `rtk deps [path]` | Summarizes supported dependency manifests |
| Command summary | `rtk summary <command>` | Heuristic compact summary |
| Errors/warnings | `rtk err <command>` | Shows only failures and warnings |
| Tests | `rtk test <command>` | Shows failures and compact trailing output |
| Logs | `rtk log <file>` | Heuristic; false positives can occur even on real logs, so never use it as the sole verification source |
| HTTP/curl | `rtk curl ...` | Default curl route; JSON may be compacted, but HTML can pass through uncompressed |
| npm run script | `rtk npm <script> [args]` | This wrapper maps only to `npm run`; never use it for dependency or other non-run npm CLI subcommands |
| npx | `rtk npx --no-install <binary> ...` | Verified no-download route in the current environment; stop if the binary is missing |
| Go tooling | `rtk go ...` | Use whenever invoking the installed Go toolchain |
| Lint | `rtk lint ...` | Preferred for supported lint output |
| ast-grep | `rtk sg ...` | Verified implicit passthrough on RTK 0.43.0; do not run bare `sg`, and re-verify after future RTK upgrades |

For a bounded raw UTF-8 middle range on Windows (`-Skip` is zero-based), use the unfiltered proxy only when full-fidelity range output is required. Construct the complete inner PowerShell script in agent/tool memory; represent the path as a single-quoted literal with every embedded `'` doubled, encode the complete script as UTF-16LE Base64 without passing the raw path through shell interpolation, then invoke:

```powershell
rtk proxy powershell -NoProfile -EncodedCommand <utf16le-base64>
```

The decoded script must set `[Console]::OutputEncoding = [System.Text.UTF8Encoding]::new($false)` and `$OutputEncoding = [Console]::OutputEncoding`, then use `Get-Content -LiteralPath <path> -Encoding UTF8 | Select-Object -Skip <start> -First <count>`. This preserves BOM-less UTF-8 on both input and output. A `$` in the path remains literal because the path is inside the encoded single-quoted literal. If no in-memory or structured encoder is available, do not fall back to raw `-Command`; use a safe native read route or report the blocker.

`rtk read --max-lines` is a compact output cap, not a selector for a guaranteed contiguous middle range.

## Known-Incompatible or Unsafe Native Routes on This Windows Setup

Do not use these until they are re-verified after an RTK upgrade:

| Route | Observed problem | Preferred fallback |
|-------|------------------|--------------------|
| `rtk ls` | Unix `ls` binary is not on PATH | workspace file tool, `rtk find`, or proxied PowerShell |
| `rtk tree` | RTK arguments are incompatible with Windows `tree.exe` | workspace file tool or `rtk find` |
| `rtk wc` | Unix `wc` binary is not on PATH | proxied PowerShell measurement |
| `rtk grep` | RTK 0.43.0 invokes external `grep`, which is not on PATH | `rtk rg <pattern> [path]` |
| `rtk npm <non-run-subcommand>` | RTK 0.43.0 maps all arguments to `npm run`; it does not invoke the requested npm CLI subcommand | After satisfying any applicable gate, inspect `rtk proxy npm <subcommand> --help` for the installed version and verify every applicable preflight/lifecycle condition in `AGENTS.md`; if the mutation boundary remains uncertain, stop. Use `rtk proxy npm <subcommand> ...`; never use native `rtk npm` for dependency changes |
| `rtk pnpm` | RTK 0.42.0 could run an implicit `pnpm install` even for `pnpm exec`; not re-verified on 0.43.0 | Never use the native route until re-verified. For an otherwise authorized invocation use `rtk proxy pnpm <args>`; apply dependency confirmation by effect under `AGENTS.md`. An inspected metadata/version or project-local verification chain may proceed only when it cannot mutate persistent package state or trigger another gate; mutating or uncertain chains remain gated |

## Meta Commands

```bash
rtk gain            # Token savings analytics
rtk gain --history  # Recent command savings history
rtk proxy <cmd>     # Last resort: run raw command without filtering
```

Treat `rtk gain --history` as command-history inspection: depending on the RTK version, it may expose recent commands or arguments. Do not use it for routine verification or when raw credentials may have appeared; prefer `rtk gain`, and apply the metadata-first/redaction rule in `AGENTS.md` before any history inspection.

## Verification

```bash
rtk --version
rtk gain
rtk proxy where.exe rtk
```
