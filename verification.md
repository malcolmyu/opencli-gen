# Verification — MANDATORY before declaring success

Generated commands fail silently more often than not. The most common failure modes:

1. Response shape guessed wrong (no rows returned, but no error thrown)
2. Request body missing required fields (server returns `code != 0`)
3. Path prefix wrong (e.g. forgot baseURL like `/dp/platform/da/radar/plus/`)
4. POST body not properly double-stringified inside `page.evaluate`
5. `args[].name` starts with `--` (opencli adds prefix itself)

The skill MUST run a verification pass and report results honestly.

## Step V1 — Probe real responses BEFORE writing commands

For every candidate endpoint, use the `_probe` command (see `generation.md`) to dump the real response shape:

```bash
opencli <site> _probe --path /api/foo --format json
opencli <site> _probe --path /api/bar --method POST --body '{"k":"v"}' --format json
```

Record the observed shape as a comment above each generated `.ts`:
```typescript
// Response (observed): { code, data: { list: [...] } }
```

Never guess `.data.list` vs `.list` vs `.data.records` — probe and verify.

## Step V2 — Install & compile check

```bash
opencli plugin install /absolute/path/to/opencli-plugin-<site> 2>&1
cd opencli-plugin-<site> && esbuild *.ts --bundle=false --format=esm --platform=node --outdir=. 2>&1
opencli plugin update <site> 2>&1
```

If any step errors, fix before continuing. Common compile errors:
- `args[].name` starts with `--` — strip the prefix.
- TypeScript syntax error — run `esbuild` on the single failing file to see the line.

## Step V3 — Smoke-test every command

For every generated command, run it at minimum in `--help` mode, and for parameter-free or defaultable commands, actually execute it:

```bash
# Help check (every command)
opencli <site> --help
for cmd in $(opencli <site> --help | awk '/^\s+[a-z]/ {print $1}' | grep -v help); do
  opencli <site> $cmd --help >/dev/null 2>&1 && echo "✓ $cmd help ok" || echo "✗ $cmd help FAIL"
done

# Actual run for zero-arg or defaultable commands
opencli <site> projects --format json 2>&1 | head -5
opencli <site> _probe --path /api/health --format json 2>&1 | head -10
```

For commands that need a required arg, use a known-good sample value (e.g. a projectId discovered via `projects`).

## Step V4 — Honest smoke-test report

After running, print a status table:

```
Smoke-test results:

  COMMAND              STATUS      NOTES
  ─────────────────────────────────────────────────────────────
  projects             ✓ PASS      returned 124 rows
  project-detail       ✓ PASS      sample code=528734399a ok
  stat                 ✗ FAIL      API code=1, "invalid dataType"
  trend                ⚠ EMPTY     0 rows; response had data but no rows parsed
  alarms               ✓ PASS      3 alarms returned
  _probe               ✓ PASS      diagnostic helper
```

Status codes:
- ✓ PASS — returned non-empty data
- ✗ FAIL — threw error or API returned non-zero code
- ⚠ EMPTY — succeeded but returned 0 rows (possibly wrong response parsing or just no data)
- ⊘ SKIP — requires parameter you don't have

**DO NOT report success until every ✗ FAIL is either fixed or clearly marked as broken in the README.** An ⚠ EMPTY with a valid sample arg should be re-probed to confirm the parsing is correct.

## Step V5 — Auto-fix loop (up to 2 passes)

For each ✗ FAIL or ⚠ EMPTY:

1. Run `opencli <site> _probe --path <failing-path> [--method POST --body <bodyJson>]`
2. Read actual response shape.
3. Update the command's response parsing code.
4. `esbuild` + `opencli plugin update <site>`.
5. Re-run the command.

If still failing after 2 passes, mark the command as broken in README and MOVE ON. Better to ship 8 working commands than 13 half-broken ones.

## Step V6 — Include findings in the final summary

The report to the user must distinguish working vs broken:

```
✅ Working (8): projects, project-detail, alarms, alarm-detail, dimension-values, _probe, employee-suggest, keep-product-list
❌ Broken/unverified (3): stat (wrong body shape), trend (unknown rows schema), top-errors (needs real sample filter)
⊘ Needs arg (2): alarm-attribution, monitor-rules (need live ids)
```

**Never** claim success if verification was skipped.
