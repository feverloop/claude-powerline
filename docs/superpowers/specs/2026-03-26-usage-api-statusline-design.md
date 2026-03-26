# Design: Claude API Usage in Statusline

**Date:** 2026-03-26
**Status:** Approved

## Overview

Display real Claude API usage data (5-hour and 7-day utilization + time-until-reset) in the
Claude Code powerline statusline. A hook-triggered background poller writes `~/.claude/usage.json`;
the powerline reads it via a polished `jsonFile` segment.

---

## Component 1: Poller Script

**Location:** `~/bin/refresh-usage.sh`

A standalone bash script that fetches the Claude API usage endpoint and writes the result
atomically to `~/.claude/usage.json`.

**Behaviour:**
- Checks mtime of `~/.claude/usage.json`; exits with code 0 if file is younger than 60 seconds
- Reads OAuth access token from `~/.claude/.credentials.json` via `jq`
- If credentials file is missing or token field is empty, exits with code 0 (silent)
- GETs `https://api.claude.ai/api/usage` with Bearer auth
- Writes to `~/.claude/usage.json.tmp`, then renames to `~/.claude/usage.json` (atomic swap
  prevents the statusline reading a partial write)
- On any failure, exits with code 0 and leaves the old file intact

**Script:**
```bash
#!/usr/bin/env bash
USAGE_FILE="$HOME/.claude/usage.json"
CREDS="$HOME/.claude/.credentials.json"

if [ -f "$USAGE_FILE" ] && \
   [ $(( $(date +%s) - $(stat -c %Y "$USAGE_FILE") )) -lt 60 ]; then
  exit 0
fi

TOKEN=$(jq -r '.claudeAiOauth.accessToken' "$CREDS" 2>/dev/null)
[ -z "$TOKEN" ] && exit 0

curl -sf \
  -H "Authorization: Bearer $TOKEN" \
  -H "Accept: application/json" \
  "https://api.claude.ai/api/usage" \
  -o "$USAGE_FILE.tmp" && mv "$USAGE_FILE.tmp" "$USAGE_FILE"
exit 0
```

---

## Component 2: Claude Code Hook

**Type:** `Stop` (fires after each Claude response)
**Location:** `~/.claude/settings.json` hooks config

```bash
(~/bin/refresh-usage.sh &)
```

Spawns the poller as a detached background process. Returns immediately — Claude Code is
never blocked. The 60s mtime cache means rapid back-and-forth sessions hit the API at most
once per minute.

---

## Component 3: `jsonFile` Segment — Additions

The existing partial implementation is extended with three improvements.

### 3a. Array support for multiple `jsonFile` segments per line

`LineConfig` currently allows only one `jsonFile` entry. Change the type to an array:

```typescript
// src/config/loader.ts
jsonFile?: JsonFileSegmentConfig[];
```

In `src/powerline.ts`, the `renderSegment()` dispatch handles `jsonFile` as an array,
rendering each entry and returning all non-null results in order. The caller (line renderer)
must handle an array return for this segment type.

Existing configs that provide a single object (not an array) are normalized to a single-element
array in the config loader (`src/config/loader.ts`), so the renderer always receives an array.

### 3b. `fieldType: "timeUntil"` support

New optional field on `JsonFileSegmentConfig`:

```typescript
fieldType?: "timeUntil";  // treat field value as ISO 8601 string, render as countdown
```

Rendering rules:
- Field value must be a string; any other type (number, boolean, null, object) → `null`
- Invalid or unparseable ISO 8601 string → `null`
- `>= 1 hour remaining` → `Xh Ym` (e.g. `4h 12m`)
- `< 1 hour remaining` → `Xm` (e.g. `45m`)
- `<= 0` (past reset time) → `now`

### 3c. Dedicated theme colors

`jsonFile` gets its own color keys, replacing the borrowed `env` colors. The color threshold
feature (warning/critical based on field value) is **deferred to a follow-up** — color
switching adds interface complexity that is premature before the basic display is validated.

**TypeScript changes required** (follow the pattern of any existing color key, e.g. `env`):

1. `src/themes/index.ts` — add to `ColorTheme` interface:
   ```typescript
   jsonFile: ColorPair;
   ```

2. `src/themes/index.ts` — add to `PowerlineColors` interface:
   ```typescript
   jsonFileBg: string;
   jsonFileFg: string;
   ```

3. `src/themes/index.ts` — add to `getThemeColors()` mapping:
   ```typescript
   jsonFileBg: theme.jsonFile.bg,
   jsonFileFg: theme.jsonFile.fg,
   ```

4. All six theme files (`src/themes/dark.ts`, `light.ts`, `nord.ts`, `tokyo-night.ts`,
   `rose-pine.ts`, `gruvbox.ts`) — add `jsonFile` color pair using theme-appropriate values.
   Default neutral values (same across all themes until styled individually):
   ```typescript
   jsonFile: { bg: "#3a3a4a", fg: "#c0c0e0" },
   ```

5. `src/segments/renderer.ts` — update `renderJsonFile()` to use `colors.jsonFileBg` /
   `colors.jsonFileFg` instead of `colors.envBg` / `colors.envFg`.

### 3d. Tests

Jest unit tests added to `test/segments.test.ts` covering:

| Case | Expected |
|---|---|
| File not found | `null` (hidden) |
| Invalid JSON | `null` |
| Missing field path | `null` |
| Dot-notation traversal | correct value extracted |
| Number formatting (`decimalPlaces`) | formatted string |
| `fieldType: "timeUntil"` — >1h remaining | `Xh Ym` format |
| `fieldType: "timeUntil"` — <1h remaining | `Xm` format |
| `fieldType: "timeUntil"` — past reset time | `now` |
| `fieldType: "timeUntil"` — bad string | `null` |
| `fieldType: "timeUntil"` — non-string value (number) | `null` |
| `fieldType: "timeUntil"` — non-string value (null) | `null` |
| Array config — two entries, both valid | two segments rendered |
| Array config — two entries, one null | one segment rendered |

---

## Component 4: Statusline Config

A new line in `~/.claude/claude-powerline.json` with a `jsonFile` array containing four
entries — utilization % and time-until-reset for each window:

```
⚡ 5h 3%  ↻ 4h 12m    ⚡ 7d 42%  ↻ 2d 3h
```

Config addition to `display.lines` (new line object):

```json
{
  "segments": {
    "jsonFile": [
      {
        "enabled": true,
        "path": "~/.claude/usage.json",
        "field": "five_hour.utilization",
        "prefix": "⚡ ",
        "suffix": "% 5h",
        "decimalPlaces": 0
      },
      {
        "enabled": true,
        "path": "~/.claude/usage.json",
        "field": "five_hour.resets_at",
        "fieldType": "timeUntil",
        "prefix": "↻ "
      },
      {
        "enabled": true,
        "path": "~/.claude/usage.json",
        "field": "seven_day.utilization",
        "prefix": "⚡ ",
        "suffix": "% 7d",
        "decimalPlaces": 0
      },
      {
        "enabled": true,
        "path": "~/.claude/usage.json",
        "field": "seven_day.resets_at",
        "fieldType": "timeUntil",
        "prefix": "↻ "
      }
    ]
  }
}
```

Color addition to `colors.custom`:

```json
"jsonFile": { "bg": "#3a3a4a", "fg": "#c0c0e0" }
```

---

## Data Flow

```
Claude responds
    → Stop hook fires
        → refresh-usage.sh spawned (background)
            → mtime check: skip if <60s old
            → read token from ~/.claude/.credentials.json
            → GET https://api.claude.ai/api/usage
            → write ~/.claude/usage.json (atomic)
    → statusline renders
        → jsonFile segment array reads ~/.claude/usage.json (x4)
        → extracts five_hour.utilization, five_hour.resets_at,
          seven_day.utilization, seven_day.resets_at
        → formats and displays
```

---

## Error Handling

| Failure | Behaviour |
|---|---|
| `~/.claude/.credentials.json` missing | poller exits 0 silently, old file kept |
| Token field missing/null | poller exits 0 silently, old file kept |
| API request fails | tmp file not created, rename skipped, old file kept |
| `~/.claude/usage.json` missing | all four segments return `null`, hidden from statusline |
| JSON parse error | all four segments return `null`, hidden |
| Field path missing | that segment returns `null`, hidden; others unaffected |
| `resets_at` value is non-string type | segment returns `null`, hidden |
| `resets_at` value is unparseable string | segment returns `null`, hidden |

---

## Out of Scope

- Color threshold switching on `jsonFile` (warning/critical by value) — deferred follow-up
- Dedicated `claudeUsage` segment (Option B) — revisit if display isn't satisfying
- Token refresh / re-auth if OAuth token is expired
- Displaying `extra_usage`, `seven_day_opus`, or other fields (not currently populated)
