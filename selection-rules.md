# Selection Rules — pick endpoints by user task, not by frequency

## The trap: frequency ≠ user value

**Call frequency in source code is a LAGGING indicator and often MISLEADING.** Infrastructure endpoints (auth / config / permission / session) are called by every page on every load — they rank highest by `grep | sort -rn` but deliver **zero value** to a CLI user.

Real regression: on `ks-fe-radar-plus` (a frontend monitoring dashboard) the top-frequency endpoints were `/sso/login`, `/session/check-is-administrator`, `/kconf/get`, `/session/check-is-taster`. A monitoring-platform user actually wants PV, error counts, top-N slow URLs, active alarms, attribution results. **None of the high-frequency endpoints answer any user question.**

## Rule

User tasks drive selection. Frequency is only a tiebreaker.

For every candidate endpoint, ask:
> "What real user task does this endpoint answer?"
>
> - Permission gate / config key / session auth / hardcoded enum → **SKIP**, even if called 100 times.
> - Core business data this platform is for → **PICK**, even if called once.

## Heuristics

**High user value (prefer):**
- Returns domain business data (metrics, logs, events, records, reports, transactions)
- Answers a question a human would ask ("what's my X?", "show me top Y", "when did Z happen")
- Lives in `query.ts` / `report.ts` / `alarm.ts` / `metric.ts` / `list.ts` / `detail.ts`
- Function names like `useStat*`, `useTrend*`, `useList*`, `query*`, `get*Detail`, `search*`

**Low user value (skip):**
- Names containing: `check*`, `verify*`, `is*`, `has*`, `auth*`, `session*`, `sso*`, `config*`, `kconf*`, `privilege*`, `permission*`
- Paths containing: `/sso/`, `/auth/`, `/session/`, `/kconf/`, `/config/get`, `/privilege/`, `/check-is-*`, `/health`, `/ping`
- Returns boolean / enum / scalar (admin flag, feature flag, config value)
- Returns UI-only data (CSRF tokens, hydration, A/B variants)

## Three filters (apply in order)

**Filter A — User-task match (MANDATORY):** does it answer a user task from Step 2.5.2? No → SKIP.

**Filter B — Anti-infrastructure (MANDATORY):** skip if ANY apply:
- Path contains: `/sso/`, `/auth/`, `/session/`, `/kconf/`, `/config/get`, `/privilege/`, `/check-is-*`, `/health`, `/ping`, `/csrf`, `/token/refresh`, `/login`, `/logout`
- Function matches: `check*`, `verify*`, `is*Admin*`, `has*Permission*`, `use*Auth*`, `get*Config`, `get*Kconf*`, `get*Token*`
- Returns `Promise<boolean>`, `Promise<string>`, `Promise<number>` for non-data purposes
- Return type ends with `Config`, `Permission`, `Auth`, `Token`, `Session`

**Filter C — Read vs write:**
- GET + query-style POST (queryList/queryTrend/queryDetail/search/list/detail/get) → OK
- POST/PUT/DELETE mutating state (add/update/delete/create/save/remove/toggle/enable/disable/push/submit) → SKIP by default, require explicit user approval

## Justification ledger (MANDATORY before emitting code)

Print this before writing any `.ts` file:

```
Selected endpoints (user-task driven):

  COMMAND          USER TASK                               ENDPOINT
  ───────────────────────────────────────────────────────────────────
  projects         "What projects can I access?"           GET  /new/project/list
  stat             "Total PV/FMP/errors for project X"     POST /report/queryTrend
  top-errors       "Top N JS errors in the last hour"      POST /report/queryDetail
  ...

Rejected endpoints (infrastructure / no user task):

  /sso/login                      — session bootstrap
  /session/check-is-administrator — UI permission gate
  /kconf/get                      — generic config fetch
  ...
```

If you cannot write a plausible one-sentence user task for an endpoint, DO NOT generate the command.

## Ranking the survivors

After A+B+C, sort by:
1. Strength of match to a user task (weight: 3)
2. Centrality to the platform's core purpose (weight: 2)
3. Call frequency (weight: 1, tiebreaker)

Pick 10–15 commands that cover the **breadth of user tasks**. Better to have 1 command for each of 12 tasks than 12 commands for the same task.
