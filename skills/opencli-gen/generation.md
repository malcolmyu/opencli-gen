# Generation Template — writing command `.ts` files

## Package scaffolding

```
opencli-plugin-<site>/
  package.json
  README.md
  _probe.ts           ← diagnostic command, always include first
  <cmd1>.ts
  <cmd2>.ts
  ...
```

**`package.json`:**
```json
{
  "name": "opencli-plugin-<site>",
  "version": "0.1.0",
  "type": "module",
  "description": "CLI plugin for <site> API access",
  "peerDependencies": { "@jackwener/opencli": ">=1.0.0" }
}
```

## Command template (GET)

```typescript
// Importance: HIGH ★★★
// Purpose: <what user task this answers>
// Source: src/services/<file>.ts (<axios call site>)
// Endpoint: GET /<path>
// Response (observed via _probe): { code, data: { ... } }

import { cli, Strategy } from '@jackwener/opencli/registry';

cli({
  site: '<site>',
  name: '<cmd>',
  description: '<user-facing description>',
  domain: '<domain>',
  strategy: Strategy.COOKIE,
  args: [
    { name: 'id', type: 'str', help: '...' },
  ],
  columns: ['col1', 'col2'],
  func: async (page: any, kwargs: any) => {
    await page.goto('https://<domain>');
    const id = encodeURIComponent(kwargs.id);
    const result = await page.evaluate(`
      (async () => {
        const res = await fetch('/path?id=${id}', { credentials: 'include' });
        return res.json();
      })()
    `);
    if (result?.code !== 0) throw new Error(result?.message || 'API error');
    const data = result.data;
    // ...shape to rows...
    return rows;
  },
});
```

## Command template (POST with JSON body)

```typescript
func: async (page: any, kwargs: any) => {
  await page.goto('https://<domain>');
  const body = { /* build from kwargs */ };
  const bodyJson = JSON.stringify(body);
  const result = await page.evaluate(`
    (async () => {
      const res = await fetch('/path', {
        method: 'POST',
        credentials: 'include',
        headers: { 'Content-Type': 'application/json' },
        body: ${JSON.stringify(bodyJson)},    // ⚠ double-stringify for safe interpolation
      });
      return res.json();
    })()
  `);
  if (result?.code !== 0) throw new Error(result?.message || 'API error');
  return result.data.map((row: any) => ({ /* columns */ }));
},
```

## The `_probe` command (always include)

Every plugin MUST ship with a probe command so the skill (and the user) can inspect real response shapes before/after generation:

```typescript
import { cli, Strategy } from '@jackwener/opencli/registry';

cli({
    site: '<site>',
    name: '_probe',
    description: 'Diagnostic: dump raw response of any URL (GET or POST)',
    domain: '<domain>',
    strategy: Strategy.COOKIE,
    args: [
        { name: 'path', type: 'str', help: 'API path, e.g. /api/...' },
        { name: 'method', type: 'str', help: 'GET or POST', default: 'GET' },
        { name: 'body', type: 'str', help: 'JSON body for POST', default: '' },
    ],
    columns: ['field', 'value'],
    func: async (page: any, kwargs: any) => {
        await page.goto('https://<domain>');
        const path = kwargs.path;
        const method = String(kwargs.method || 'GET').toUpperCase();
        const body = kwargs.body || '';
        const result = await page.evaluate(`
            (async () => {
                const init = { credentials: 'include', method: ${JSON.stringify(method)} };
                if (${JSON.stringify(method)} === 'POST' && ${JSON.stringify(body)}) {
                    init.headers = { 'Content-Type': 'application/json' };
                    init.body = ${JSON.stringify(body)};
                }
                const res = await fetch(${JSON.stringify(path)}, init);
                const text = await res.text();
                let parsed; try { parsed = JSON.parse(text); } catch { parsed = null; }
                return { status: res.status, raw: text.slice(0, 2000), parsed };
            })()
        `);
        const p = result.parsed;
        return [
            { field: 'status', value: String(result.status) },
            { field: 'topKeys', value: p ? Object.keys(p).join(',') : '(not JSON)' },
            { field: 'code', value: p?.code !== undefined ? String(p.code) : '' },
            { field: 'message', value: String(p?.message ?? '') },
            { field: 'dataType', value: p?.data ? (Array.isArray(p.data) ? 'array len=' + p.data.length : typeof p.data + ' keys=' + Object.keys(p.data || {}).join(',')) : '' },
            { field: 'rawHead', value: String(result.raw).slice(0, 500) },
        ];
    },
});
```

This is the single biggest reliability lever. Use `opencli <site> _probe --path <path>` to verify every response shape BEFORE writing the command that parses it.

## kwargs serialization in `page.evaluate`

`page.evaluate` runs JS in browser context — Node-side `kwargs` is inaccessible. Always serialize before embedding:

```typescript
// ✅ Correct
const key = encodeURIComponent(kwargs.key);
const result = await page.evaluate(`
  (async () => {
    const res = await fetch('/api/get?key=${key}', { credentials: 'include' });
    return res.json();
  })()
`);

// ❌ Wrong (runtime undefined)
const result = await page.evaluate(`
  (async () => {
    const res = await fetch('/api/get?key=' + kwargs.key);  // kwargs is not defined here
    return res.json();
  })()
`);
```

For POST bodies: `const bodyJson = JSON.stringify(body);` then embed via `${JSON.stringify(bodyJson)}` (double-wrap so template literals and quotes stay safe).

## Compile & install

```bash
opencli plugin install /absolute/path/to/opencli-plugin-<site>
cd opencli-plugin-<site> && esbuild *.ts --bundle=false --format=esm --platform=node --outdir=.
opencli plugin update <site>        # ← site name, NOT package name
```
