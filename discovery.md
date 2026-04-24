# API Discovery — 3-layer strategy

Auto-detect which strategy applies, then extract endpoints.

## Strategy 1 — OpenAPI generated client

Look for generated client files:
```bash
find . -path "*/open-api-docs/*/clients/*client.ts" -o -path "*/openapi/*/client*.ts" 2>/dev/null | head -1
```

Each export has JSDoc with method + path:
```typescript
/**
 * 分享回流
 * * __method__: POST
 * * __path__: /rest/wd/cny2026/warmup/richtree/share/backFlow
 */
export const cny2025ShareControllerBackFlow = { ... }
```

Usage count:
```bash
grep "^export const" client.ts | while read line; do
  FN=$(echo "$line" | grep -oP "export const \K\w+")
  COUNT=$(grep -r "$FN" src/ --include="*.ts" --include="*.tsx" --include="*.vue" 2>/dev/null | wc -l)
  echo "$COUNT $FN"
done | sort -rn
```

## Strategy 2 — URL constants file

```bash
find . -type f \( -name "*url-api*" -o -name "*endpoints*" -o -name "*api.const*" \) \
  \( -name "*.ts" -o -name "*.js" \) 2>/dev/null | head -1
```

Extract `KEY: '/path'` mappings, count `URL_API.$KEY` usage.

## Strategy 3 — Direct HTTP calls

```bash
find src -name "*.ts" -o -name "*.tsx" -o -name "*.vue" -o -name "*.js" 2>/dev/null | \
  xargs grep -h "fetch\|axios\.\(get\|post\|put\|delete\)\|\\$http" 2>/dev/null | \
  grep -oP "['\"]/(api|rest|graphql)/[^'\"]*" | \
  sort | uniq -c | sort -rn | head -30
```

For projects using a centralized axios wrapper (common pattern: `api.ts` with multiple baseURLs), **read the wrapper** first to understand:
- How many axios instances exist and what each baseURL is
- How response is unwrapped in interceptors (e.g. `return body.data` — caller receives `.data` directly)
- How error codes are checked (e.g. `body.code === 0` means success)

Without this, your generated code will parse the response wrong.

## Frequency is context, not a decision

Use frequency rank only to understand which endpoints are central entry points. The final pick is driven by user tasks (see `selection-rules.md`).
