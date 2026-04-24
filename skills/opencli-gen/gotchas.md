# opencli gotchas — don't repeat these

## arg naming
`args[].name` must NOT start with `--`. opencli adds the prefix. Using `'--foo'` produces `----foo` at runtime.

```typescript
args: [{ name: 'projectId', type: 'str', help: '...' }]   // ✅
args: [{ name: '--projectId', type: 'str', help: '...' }] // ❌ → "----projectId"
```

## install / update commands
- `opencli plugin install <path>` — **absolute** path only. Relative (`./foo`) rejected.
- `opencli plugin update <site-name>` — uses the `site:` field in `cli(...)`, NOT the npm package name. e.g. `opencli plugin update radarplus` (not `opencli plugin update opencli-plugin-radarplus`).

## compile
```bash
esbuild *.ts --bundle=false --format=esm --platform=node --outdir=.
```
Do NOT use `npx esbuild` — esbuild is global on homebrew machines.

## page.evaluate context
`page.evaluate(\`...\`)` runs in browser. Node-side variables are unreachable. Always serialize:

```typescript
// ✅ correct — values baked into the template string
const id = encodeURIComponent(kwargs.id);
const bodyJson = JSON.stringify(body);
await page.evaluate(`
  const res = await fetch('/api/x?id=${id}', {
    method: 'POST',
    body: ${JSON.stringify(bodyJson)},      // double-stringify: string literal safe to embed
  });
`);

// ❌ wrong — kwargs not defined in browser
await page.evaluate(`await fetch('/api/x?id=' + kwargs.id)`);
```

## baseURL awareness
When the project uses multiple axios instances (common pattern: `api.dp`, `api.dpp`, `api.rp`), each has a different baseURL that is prepended. A call to `api.dpp.get('/foo')` hits `<baseURL-of-dpp>/foo`, not `/foo`. You MUST use the full path in `page.evaluate`:
- Source code: `api.dpp.get('/report/queryTrend')`
- Real URL:   `/dp/platform/da/radar/plus/report/queryTrend`

Read `src/utils/api.ts` (or equivalent) to find the baseURL mapping before generating paths.

## response unwrapping
Most internal APIs wrap their response as:
```json
{ "code": 0, "message": "", "data": { ... } }
```
The frontend's axios interceptor typically returns `body.data` directly. From `page.evaluate`, you get the RAW response (the wrapper `{code, data}`), so you must check `result.code !== 0` and read `result.data` yourself.

## wide objects → table unusable
If an endpoint returns a single object with 20+ fields, do NOT flatten it into `[{field, value}]` rows — the resulting table is unreadable. Instead:
- Return ONE row containing all fields.
- Declare `columns: [...]` listing only the ~5–10 most useful keys (for the default table view).
- Tell users to pass `--format json` in the command description for full detail.

```typescript
columns: ['id', 'name', 'owner', 'env', 'status'],  // table subset
func: async (...) => {
  const row: Record<string, any> = {};
  for (const [k, v] of Object.entries(data)) row[k] = typeof v === 'object' ? JSON.stringify(v) : v;
  return [row];   // one row, wide — table shows columns subset, json shows all
}
```

## honesty in output
- If parsing returns 0 rows, DO NOT hide it as "success" in the report. Print `⚠ EMPTY`.
- If an API returns `code != 0`, print the `message` field in the error.
- Never claim success for a command that wasn't actually run.
