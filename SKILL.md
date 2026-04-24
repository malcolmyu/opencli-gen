# /opencli-gen

Auto-generate an opencli plugin for a frontend project. Selects endpoints by **user task**, **probes real responses**, and **verifies every command runs** before declaring success.

## Usage

```
/opencli-gen                 # auto-detect & analyze current directory
/opencli-gen <path>          # analyze specific path
/opencli-gen --site <name>   # override cli site name
/opencli-gen --dry-run       # print plan, don't write files
```

## Pipeline (6 phases)

1. **Purpose** — Read `README.md` + `package.json` + router + `src/views/` to understand what USERS of this platform do. Write one-sentence platform summary and a bullet list of concrete user tasks. If you can't, STOP and ask the user.
2. **Discovery** — Detect API structure using the 3-layer strategy. See **[discovery.md](./discovery.md)**.
3. **Selection** — Map user tasks → endpoints, apply three filters. See **[selection-rules.md](./selection-rules.md)**. Print the justification ledger before writing any file.
4. **Generate** — Emit `.ts` files + `package.json` + `README.md` + `_probe.ts`. See **[generation.md](./generation.md)**.
5. **Verify** — Install, compile, smoke-test every command. Auto-fix with `_probe` up to 2 passes. See **[verification.md](./verification.md)**.
6. **Report** — Honest status table: ✓ PASS / ⚠ EMPTY / ✗ FAIL / ⊘ SKIP. Never claim success for unverified commands.

## Hard rules

- **User tasks drive selection.** Frequency is a tiebreaker only. Never generate commands like `whoami` / `check-admin` / `kconf` because they were called a lot.
- **Probe before parsing.** For each endpoint, run `opencli <site> _probe --path <path>` to see the real response shape. Don't guess `.data.list` vs `.list`.
- **Verify before declaring success.** Every generated command must be smoke-tested. Any ✗ FAIL must be fixed or clearly marked broken in the README. No silent failures.
- **No writes without approval.** POST/PUT/DELETE that mutate state (`create`, `update`, `delete`, `add`, `save`, `toggle`, `submit`, …) are skipped unless the user explicitly opts in.
- **Skip if unclear.** Don't hallucinate endpoints, params, or response shapes.

## Prerequisites

```bash
which opencli || echo "MISSING"
```
If missing: `npm install -g @jackwener/opencli`.

## Files in this skill

- `SKILL.md` (this file) — pipeline overview & hard rules
- `discovery.md` — how to find endpoints (3 layers)
- `selection-rules.md` — user-task filters + justification ledger
- `generation.md` — `.ts` templates + `_probe` command
- `verification.md` — install/compile/smoke-test pipeline + status reporting
- `gotchas.md` — opencli-specific traps (arg naming, baseURL, page.evaluate)

## Case study — why this skill is strict about verification

First-pass generation on `ks-fe-radar-plus` (a perf-monitoring dashboard) produced:

```
kconf, check-admin, whoami, report-kpn, compat-data-*, sso-login  ← all useless
```

They dominated by call frequency (every page loads auth/config on hydration) but answered zero user questions. After switching to user-task-driven selection, the right commands emerged:

```
projects, project-detail, stat, trend, top-errors, dimension-values,
alarms, alarm-detail, alarm-attribution, monitor-rules, custom-apis
```

Then verification revealed that several commands returned 0 rows because the response shape was guessed wrong (`.data.list[].projectList` vs real `.data.list[].children`). Probing fixed it.

**Two lessons:** frequency lies about value, and guessed response shapes silently fail. This skill enforces both.
