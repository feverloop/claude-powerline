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
- Checks mtime of `~/.claude/usage.json`; exits immediately if file is younger than 60 seconds
- Reads OAuth access token from `~/.claude/.credentials.json` via `jq`
- GETs `https://api.claude.ai/api/usage` with Bearer auth
- Writes to `~/.claude/usage.json.tmp`, then renames to `~/.claude/usage.json` (atomic swap
  prevents the statusline reading a partial write)
- Fails silently — if token is missing or request fails, the old file is left intact and no
  error is surfaced

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
[ -z "$TOKEN" ] && exit 1

curl -sf \
  -H "Authorization: Bearer $TOKEN" \
  -H "Accept: application/json" \
  "https://api.claude.ai/api/usage" \
  -o "$USAGE_FILE.tmp" && mv "$USAGE_FILE.tmp" "$USAGE_FILE"
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

The existing partial implementation is extended with three improvements:

### 3a. `fieldType: "timeUntil"` support

New optional field on `JsonFileSegmentConfig`:

```typescript
fieldType?: "timeUntil";  // treat field value as ISO 8601 timestamp, render as countdown
```

Rendering rules:
- `>= 1 hour remaining` → `4h 12m`
- `< 1 hour remaining` → `45m`
- `<= 0` → `now`
- Invalid/unparseable timestamp → `null` (segment hidden)

### 3b. Dedicated theme colors

`jsonFile` gets its own color key in the theme config (`colors.jsonFile`), replacing the
borrowed `env` colors. Warning and critical threshold variants follow the same pattern as
`context`:

```json
"jsonFile":         { "bg": "#3a3a4a", "fg": "#c0c0e0" },
"jsonFileWarning":  { "bg": "#92400e", "fg": "#fbbf24" },
"jsonFileCritical": { "bg": "#991b1b", "fg": "#fca5a5" }
```

Threshold logic: if the field is numeric and a `warningThreshold` / `criticalThreshold` is
configured, color switches accordingly.

### 3c. Tests

Jest unit tests added to `test/segments.test.ts` covering:

| Case | Expected |
|---|---|
| File not found | `null` (hidden) |
| Invalid JSON | `null` |
| Missing field path | `null` |
| Dot-notation traversal | correct value extracted |
| Number formatting (`decimalPlaces`) | formatted string |
| `fieldType: "timeUntil"` — >1h | `Xh Ym` |
| `fieldType: "timeUntil"` — <1h | `Xm` |
| `fieldType: "timeUntil"` — past | `now` |
| `fieldType: "timeUntil"` — bad timestamp | `null` |

---

## Component 4: Statusline Config

Four `jsonFile` segments added to `~/.claude/claude-powerline.json` on a new line, showing
utilization and countdown for both windows:

```
⚡ 5h 3%  ↻ 4h 12m    ⚡ 7d 42%  ↻ 2d 3h
```

Config additions to `display.lines`:

```json
{
  "segments": {
    "jsonFile_5h_util": {
      "enabled": true,
      "path": "~/.claude/usage.json",
      "field": "five_hour.utilization",
      "prefix": "⚡ ",
      "suffix": "% 5h",
      "decimalPlaces": 0
    },
    "jsonFile_5h_reset": {
      "enabled": true,
      "path": "~/.claude/usage.json",
      "field": "five_hour.resets_at",
      "fieldType": "timeUntil",
      "prefix": "↻ "
    },
    "jsonFile_7d_util": {
      "enabled": true,
      "path": "~/.claude/usage.json",
      "field": "seven_day.utilization",
      "prefix": "⚡ ",
      "suffix": "% 7d",
      "decimalPlaces": 0
    },
    "jsonFile_7d_reset": {
      "enabled": true,
      "path": "~/.claude/usage.json",
      "field": "seven_day.resets_at",
      "fieldType": "timeUntil",
      "prefix": "↻ "
    }
  }
}
```

Color config additions (under `colors.custom`):

```json
"jsonFile":         { "bg": "#3a3a4a", "fg": "#c0c0e0" },
"jsonFileWarning":  { "bg": "#92400e", "fg": "#fbbf24" },
"jsonFileCritical": { "bg": "#991b1b", "fg": "#fca5a5" }
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
        → jsonFile segment reads ~/.claude/usage.json
        → extracts five_hour.utilization, five_hour.resets_at,
          seven_day.utilization, seven_day.resets_at
        → formats and displays
```

---

## Error Handling

| Failure | Behaviour |
|---|---|
| `~/.claude/.credentials.json` missing | poller exits silently, old file kept |
| Token field missing/null | poller exits silently, old file kept |
| API request fails | tmp file not created, rename skipped, old file kept |
| `~/.claude/usage.json` missing | segment returns `null`, hidden from statusline |
| JSON parse error | segment returns `null`, hidden |
| Field path missing | segment returns `null`, hidden |
| Bad `resets_at` timestamp | segment returns `null`, hidden |

---

## Out of Scope

- Dedicated `claudeUsage` segment (Option B) — revisit if display isn't satisfying
- Token refresh / re-auth if OAuth token is expired
- Displaying `extra_usage`, `seven_day_opus`, or other fields (not currently populated)
