# Usage API Statusline Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Display real Claude API utilization and time-until-reset for 5h and 7d windows in the powerline statusline, refreshed non-blocking on each Claude response via a Stop hook.

**Architecture:** A bash poller script writes `~/.claude/usage.json` atomically (60s mtime cache); a Stop hook spawns it in the background. The existing `jsonFile` segment is extended with `fieldType: "timeUntil"` and dedicated colors; `LineConfig.jsonFile` becomes an array so four segments can appear on one line. The user's `~/.claude/claude-powerline.json` gets a new line wiring it all together.

**Tech Stack:** Bash, TypeScript, Jest (`npx jest`), tsdown (`npx tsdown`). All commands run from `/home/keegan/git/vendor/claude-powerline-fork/` unless stated otherwise.

**Spec:** `docs/superpowers/specs/2026-03-26-usage-api-statusline-design.md`

---

## File Map

| File | Action | What changes |
|---|---|---|
| `~/bin/refresh-usage.sh` | Create | Poller script |
| `~/.claude/settings.json` | Modify | Add Stop hook |
| `src/themes/index.ts` | Modify | Add `jsonFile` to `ColorTheme` + `PowerlineColors`; wire in `getThemeColors` |
| `src/themes/dark.ts` | Modify | Add `jsonFile` to all 3 theme objects |
| `src/themes/light.ts` | Modify | Add `jsonFile` to all 3 theme objects |
| `src/themes/nord.ts` | Modify | Add `jsonFile` to all 3 theme objects |
| `src/themes/tokyo-night.ts` | Modify | Add `jsonFile` to all 3 theme objects |
| `src/themes/rose-pine.ts` | Modify | Add `jsonFile` to all 3 theme objects |
| `src/themes/gruvbox.ts` | Modify | Add `jsonFile` to all 3 theme objects |
| `src/segments/renderer.ts` | Modify | Add `fieldType` to interface; implement `timeUntil`; swap to `jsonFileBg/Fg` |
| `src/config/loader.ts` | Modify | Change `LineConfig.jsonFile` to array; normalize single-object configs |
| `src/powerline.ts` | Modify | `renderSegment` handles array jsonFile; `renderLine` spreads array results; `getThemeColors` wires jsonFile; `getSegmentBgColor` handles jsonFile |
| `test/segments.test.ts` | Modify | Add 13 test cases for jsonFile segment |
| `~/.claude/claude-powerline.json` | Modify | Add new line with 4 jsonFile array segments + jsonFile color |

---

## Task 1: Poller Script and Stop Hook

**Files:**
- Create: `~/bin/refresh-usage.sh`
- Modify: `~/.claude/settings.json`

- [ ] **Step 1: Create the poller script**

```bash
cat > ~/bin/refresh-usage.sh << 'EOF'
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
EOF
chmod +x ~/bin/refresh-usage.sh
```

- [ ] **Step 2: Smoke-test the script manually**

```bash
~/bin/refresh-usage.sh && cat ~/.claude/usage.json | jq '{five_hour, seven_day}'
```

Expected: JSON with `five_hour.utilization` and `seven_day.utilization` fields.

- [ ] **Step 3: Add the Stop hook to `~/.claude/settings.json`**

Use the `update-config` skill to add this hook. The hook command is:
```
(~/bin/refresh-usage.sh &)
```
Hook type: `Stop`. This fires after each Claude response and is non-blocking.

- [ ] **Step 4: Verify hook is wired correctly**

Open `~/.claude/settings.json` and confirm a `hooks` entry exists like:
```json
"hooks": {
  "Stop": [
    {
      "matcher": "",
      "hooks": [
        { "type": "command", "command": "(~/bin/refresh-usage.sh &)" }
      ]
    }
  ]
}
```

- [ ] **Step 5: Commit**

```bash
cd ~/bin && git add refresh-usage.sh
# Note: settings.json is tracked by chezmoi, not git — no commit needed for it
```

The `~/bin/` directory is inside `~/Projects` (tracked by Projects repo). Commit there:
```bash
cd ~/Projects && git add ../bin/refresh-usage.sh && git commit -m "feat: add claude usage API poller script"
```

---

## Task 2: Color System — Types, Themes, and getThemeColors

**Files:**
- Modify: `src/themes/index.ts`
- Modify: `src/themes/dark.ts`, `light.ts`, `nord.ts`, `tokyo-night.ts`, `rose-pine.ts`, `gruvbox.ts`
- Modify: `src/powerline.ts` (getThemeColors section only)

- [ ] **Step 1: Add `jsonFile` to `ColorTheme` interface in `src/themes/index.ts`**

In `src/themes/index.ts`, after line `env: SegmentColor;` (line 34), add:
```typescript
  jsonFile: SegmentColor;
```

- [ ] **Step 2: Add `jsonFileBg` and `jsonFileFg` to `PowerlineColors` in `src/themes/index.ts`**

After `envBg: string; envFg: string;` (lines 63-64), add:
```typescript
  jsonFileBg: string;
  jsonFileFg: string;
```

- [ ] **Step 3: Add `jsonFile` color entry to all theme objects in all 6 theme files**

In each of `dark.ts`, `light.ts`, `nord.ts`, `tokyo-night.ts`, `rose-pine.ts`, `gruvbox.ts` — add the following to **every** theme object exported (each file has 3: base, ansi256, ansi):

```typescript
jsonFile: { bg: "#3a3a4a", fg: "#c0c0e0" },
```

Add it after the `env` entry in each object.

- [ ] **Step 4: Wire `jsonFile` into `getThemeColors()` in `src/powerline.ts`**

In `getThemeColors()` (around line 775), after `const env = getSegmentColors("env");`, add:
```typescript
const jsonFile = getSegmentColors("jsonFile");
```

In the returned object (around line 804), after `envFg: env.fg,`, add:
```typescript
jsonFileBg: jsonFile.bg,
jsonFileFg: jsonFile.fg,
```

- [ ] **Step 5: Add `jsonFile` case to `getSegmentBgColor()` in `src/powerline.ts`**

In `getSegmentBgColor()` (around line 834), after `case "env": return colors.envBg;`, add:
```typescript
case "jsonFile":
  return colors.jsonFileBg;
```

- [ ] **Step 6: Typecheck — confirm no compile errors**

```bash
cd /home/keegan/git/vendor/claude-powerline-fork && npx tsc --noEmit
```

Expected: no errors. If TypeScript errors appear, all are about missing `jsonFile` field — check each theme file has it added to all 3 variant objects.

- [ ] **Step 7: Commit**

```bash
git add src/themes/ src/powerline.ts
git commit -m "feat: add jsonFile color to theme system"
```

---

## Task 3: Write Failing Tests

**Files:**
- Modify: `test/segments.test.ts`

- [ ] **Step 1: Add the jsonFile test suite to `test/segments.test.ts`**

Append the following at the end of the file, before any closing braces:

```typescript
describe("renderJsonFile", () => {
  const config = { display: { style: "capsule" as const, lines: [] } };
  const symbols = {} as any;
  const colors = {
    jsonFileBg: "\x1b[48;2;58;58;74m",
    jsonFileFg: "\x1b[38;2;192;192;224m",
  } as any;

  let renderer: SegmentRenderer;
  let tmpFile: string;

  beforeEach(() => {
    renderer = new SegmentRenderer(config, symbols);
    tmpFile = join(tmpdir(), `usage-test-${Date.now()}.json`);
  });

  afterEach(() => {
    try { rmSync(tmpFile); } catch {}
  });

  it("returns null when file does not exist", () => {
    const result = renderer.renderJsonFile(colors, {
      enabled: true,
      path: "/nonexistent/usage.json",
    });
    expect(result).toBeNull();
  });

  it("returns null when file contains invalid JSON", () => {
    writeFileSync(tmpFile, "not json");
    const result = renderer.renderJsonFile(colors, {
      enabled: true,
      path: tmpFile,
    });
    expect(result).toBeNull();
  });

  it("returns null when dot-notation field path is missing", () => {
    writeFileSync(tmpFile, JSON.stringify({ five_hour: {} }));
    const result = renderer.renderJsonFile(colors, {
      enabled: true,
      path: tmpFile,
      field: "five_hour.utilization",
    });
    expect(result).toBeNull();
  });

  it("extracts nested value via dot-notation", () => {
    writeFileSync(tmpFile, JSON.stringify({ five_hour: { utilization: 42 } }));
    const result = renderer.renderJsonFile(colors, {
      enabled: true,
      path: tmpFile,
      field: "five_hour.utilization",
    });
    expect(result).not.toBeNull();
    expect(result!.text).toContain("42");
  });

  it("formats numbers with decimalPlaces", () => {
    writeFileSync(tmpFile, JSON.stringify({ val: 3.14159 }));
    const result = renderer.renderJsonFile(colors, {
      enabled: true,
      path: tmpFile,
      field: "val",
      decimalPlaces: 0,
    });
    expect(result!.text).toContain("3");
    expect(result!.text).not.toContain(".");
  });

  it("uses jsonFileBg and jsonFileFg colors", () => {
    writeFileSync(tmpFile, JSON.stringify({ val: "hello" }));
    const result = renderer.renderJsonFile(colors, {
      enabled: true,
      path: tmpFile,
      field: "val",
    });
    expect(result!.bgColor).toBe(colors.jsonFileBg);
    expect(result!.fgColor).toBe(colors.jsonFileFg);
  });

  describe("fieldType: timeUntil", () => {
    it("formats countdown as Xh Ym when more than 1 hour remains", () => {
      const future = new Date(Date.now() + 4 * 60 * 60 * 1000 + 12 * 60 * 1000);
      writeFileSync(tmpFile, JSON.stringify({ resets_at: future.toISOString() }));
      const result = renderer.renderJsonFile(colors, {
        enabled: true,
        path: tmpFile,
        field: "resets_at",
        fieldType: "timeUntil",
      });
      expect(result!.text).toMatch(/\d+h \d+m/);
    });

    it("formats countdown as Xm when less than 1 hour remains", () => {
      const future = new Date(Date.now() + 45 * 60 * 1000);
      writeFileSync(tmpFile, JSON.stringify({ resets_at: future.toISOString() }));
      const result = renderer.renderJsonFile(colors, {
        enabled: true,
        path: tmpFile,
        field: "resets_at",
        fieldType: "timeUntil",
      });
      expect(result!.text).toMatch(/^\d+m$/);
    });

    it("returns 'now' when reset time is in the past", () => {
      const past = new Date(Date.now() - 60 * 1000);
      writeFileSync(tmpFile, JSON.stringify({ resets_at: past.toISOString() }));
      const result = renderer.renderJsonFile(colors, {
        enabled: true,
        path: tmpFile,
        field: "resets_at",
        fieldType: "timeUntil",
      });
      expect(result!.text).toContain("now");
    });

    it("returns null for an unparseable timestamp string", () => {
      writeFileSync(tmpFile, JSON.stringify({ resets_at: "not-a-date" }));
      const result = renderer.renderJsonFile(colors, {
        enabled: true,
        path: tmpFile,
        field: "resets_at",
        fieldType: "timeUntil",
      });
      expect(result).toBeNull();
    });

    it("returns null when field value is a number (not a string)", () => {
      writeFileSync(tmpFile, JSON.stringify({ resets_at: 1234567890 }));
      const result = renderer.renderJsonFile(colors, {
        enabled: true,
        path: tmpFile,
        field: "resets_at",
        fieldType: "timeUntil",
      });
      expect(result).toBeNull();
    });

    it("returns null when field value is null", () => {
      writeFileSync(tmpFile, JSON.stringify({ resets_at: null }));
      const result = renderer.renderJsonFile(colors, {
        enabled: true,
        path: tmpFile,
        field: "resets_at",
        fieldType: "timeUntil",
      });
      expect(result).toBeNull();
    });
  });
});
```

Also add these imports at the top of `test/segments.test.ts` if not already present:
```typescript
import { writeFileSync } from "fs";
import { join } from "path";
import { tmpdir } from "os";
```

(Check existing imports first — `join`, `tmpdir`, `mkdirSync`, `rmSync` may already be imported.)

- [ ] **Step 2: Run tests — confirm the new suite fails**

```bash
cd /home/keegan/git/vendor/claude-powerline-fork && npx jest test/segments.test.ts --testNamePattern="renderJsonFile" 2>&1 | tail -20
```

Expected: FAIL. The `timeUntil` cases fail because `fieldType` is not yet implemented. The color test fails because `renderJsonFile` still uses `envBg/Fg`.

---

## Task 4: Implement `renderJsonFile` — timeUntil and dedicated colors

**Files:**
- Modify: `src/segments/renderer.ts`

- [ ] **Step 1: Add `fieldType` to `JsonFileSegmentConfig` interface**

In `src/segments/renderer.ts`, update the `JsonFileSegmentConfig` interface (around line 85):

```typescript
export interface JsonFileSegmentConfig extends SegmentConfig {
  path: string;
  field?: string;
  prefix?: string;
  suffix?: string;
  decimalPlaces?: number;
  fieldType?: "timeUntil";
}
```

- [ ] **Step 2: Replace `renderJsonFile` with updated implementation**

Replace the existing `renderJsonFile` method (lines 864–904) with:

```typescript
renderJsonFile(
  colors: PowerlineColors,
  config: JsonFileSegmentConfig,
): SegmentData | null {
  try {
    const filePath = config.path.startsWith("~")
      ? config.path.replace("~", os.homedir())
      : config.path;

    const content = fs.readFileSync(filePath, "utf-8");
    const data = JSON.parse(content) as Record<string, unknown>;

    let value: unknown = data;
    if (config.field) {
      for (const key of config.field.split(".")) {
        if (value === null || typeof value !== "object") {
          return null;
        }
        value = (value as Record<string, unknown>)[key];
      }
    }

    if (value === null || value === undefined) return null;

    if (config.fieldType === "timeUntil") {
      if (typeof value !== "string") return null;
      const target = new Date(value);
      if (isNaN(target.getTime())) return null;
      const msLeft = target.getTime() - Date.now();
      if (msLeft <= 0) {
        const text = `${config.prefix ?? ""}now${config.suffix ?? ""}`;
        return { text, bgColor: colors.jsonFileBg, fgColor: colors.jsonFileFg };
      }
      const totalMinutes = Math.floor(msLeft / 60000);
      const hours = Math.floor(totalMinutes / 60);
      const minutes = totalMinutes % 60;
      const countdown = hours >= 1 ? `${hours}h ${minutes}m` : `${minutes}m`;
      const text = `${config.prefix ?? ""}${countdown}${config.suffix ?? ""}`;
      return { text, bgColor: colors.jsonFileBg, fgColor: colors.jsonFileFg };
    }

    let displayValue: string;
    if (typeof value === "number" && config.decimalPlaces !== undefined) {
      displayValue = value.toFixed(config.decimalPlaces);
    } else {
      displayValue = String(value);
    }

    const parts: string[] = [];
    if (config.prefix) parts.push(config.prefix);
    parts.push(displayValue);
    if (config.suffix) parts.push(config.suffix);

    return { text: parts.join(""), bgColor: colors.jsonFileBg, fgColor: colors.jsonFileFg };
  } catch {
    return null;
  }
}
```

- [ ] **Step 3: Run the tests — confirm jsonFile suite passes**

```bash
cd /home/keegan/git/vendor/claude-powerline-fork && npx jest test/segments.test.ts --testNamePattern="renderJsonFile" 2>&1 | tail -20
```

Expected: all 13 tests PASS.

- [ ] **Step 4: Run full test suite — confirm no regressions**

```bash
npx jest 2>&1 | tail -15
```

Expected: all pre-existing tests still pass.

- [ ] **Step 5: Commit**

```bash
git add src/segments/renderer.ts test/segments.test.ts
git commit -m "feat: add fieldType timeUntil and dedicated colors to jsonFile segment"
```

---

## Task 5: Array Support for Multiple jsonFile Segments

**Files:**
- Modify: `src/config/loader.ts`
- Modify: `src/powerline.ts`

- [ ] **Step 1: Change `LineConfig.jsonFile` to array type in `src/config/loader.ts`**

In `src/config/loader.ts`, line 36, change:
```typescript
jsonFile?: JsonFileSegmentConfig;
```
to:
```typescript
jsonFile?: JsonFileSegmentConfig | JsonFileSegmentConfig[];
```

- [ ] **Step 2: Add normalization in `loadConfig()` in `src/config/loader.ts`**

At the end of `loadConfig()`, just before `return config;`, add:

```typescript
// Normalize jsonFile: ensure each line's jsonFile is always an array
for (const line of config.display?.lines ?? []) {
  if (line.segments.jsonFile && !Array.isArray(line.segments.jsonFile)) {
    (line.segments as any).jsonFile = [line.segments.jsonFile];
  }
}
```

- [ ] **Step 3: Update `renderSegment` jsonFile dispatch in `src/powerline.ts`**

In `renderSegment()` (around line 553), replace:
```typescript
if (segment.type === "jsonFile") {
  return this.segmentRenderer.renderJsonFile(
    colors,
    segment.config as JsonFileSegmentConfig,
  );
}
```
with:
```typescript
if (segment.type === "jsonFile") {
  const configs = Array.isArray(segment.config)
    ? (segment.config as JsonFileSegmentConfig[])
    : [segment.config as JsonFileSegmentConfig];
  const results = configs
    .map((cfg) => this.segmentRenderer.renderJsonFile(colors, cfg))
    .filter((r): r is SegmentData => r !== null);
  return results.length > 0 ? results : null;
}
```

- [ ] **Step 4: Update `renderSegment` return type in `src/powerline.ts`**

Find the `private async renderSegment(` signature (around line 452). Change its return type from `Promise<SegmentData | null>` to `Promise<SegmentData | SegmentData[] | null>`.

- [ ] **Step 5: Update `renderLine` to spread array results in `src/powerline.ts`**

In `renderLine()` (around line 427), replace:
```typescript
if (segmentData) {
  renderedSegments.push({
    type: segment.type,
    text: segmentData.text,
    bgColor: segmentData.bgColor,
    fgColor: segmentData.fgColor,
  });
}
```
with:
```typescript
if (segmentData) {
  const dataArray = Array.isArray(segmentData) ? segmentData : [segmentData];
  for (const data of dataArray) {
    renderedSegments.push({
      type: segment.type,
      text: data.text,
      bgColor: data.bgColor,
      fgColor: data.fgColor,
    });
  }
}
```

- [ ] **Step 6: Typecheck**

```bash
npx tsc --noEmit
```

Expected: no errors.

- [ ] **Step 7: Run full test suite**

```bash
npx jest 2>&1 | tail -15
```

Expected: all tests pass.

- [ ] **Step 8: Commit**

```bash
git add src/config/loader.ts src/powerline.ts
git commit -m "feat: support array of jsonFile segments per statusline line"
```

---

## Task 6: Build and Verify

- [ ] **Step 1: Run full build**

```bash
cd /home/keegan/git/vendor/claude-powerline-fork && npx tsdown
```

Expected: build succeeds with no errors.

- [ ] **Step 2: Smoke-test the built binary against a real usage.json**

First ensure `~/.claude/usage.json` exists (run `~/bin/refresh-usage.sh` if not):
```bash
~/bin/refresh-usage.sh && cat ~/.claude/usage.json
```

Then test the built binary directly with a minimal config snippet to confirm the jsonFile array renders:
```bash
CLAUDE_POWERLINE_CONFIG=/dev/stdin node dist/index.js << 'EOF'
{
  "theme": "dark",
  "display": {
    "style": "capsule",
    "lines": [{
      "segments": {
        "jsonFile": [
          { "enabled": true, "path": "~/.claude/usage.json", "field": "five_hour.utilization", "prefix": "5h: ", "suffix": "%", "decimalPlaces": 0 },
          { "enabled": true, "path": "~/.claude/usage.json", "field": "five_hour.resets_at", "fieldType": "timeUntil", "prefix": "↻ " }
        ]
      }
    }]
  }
}
EOF
```

Expected: a rendered statusline line showing `5h: 3%` and `↻ 4h 12m` (values will vary).

---

## Task 7: Wire Statusline Config

**Files:**
- Modify: `~/.claude/claude-powerline.json`

- [ ] **Step 1: Add the jsonFile color to `colors.custom` in `~/.claude/claude-powerline.json`**

In the `colors.custom` object, add:
```json
"jsonFile": { "bg": "#3a3a4a", "fg": "#c0c0e0" }
```

- [ ] **Step 2: Add a new line to `display.lines` in `~/.claude/claude-powerline.json`**

Append a new object to the `display.lines` array:
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

- [ ] **Step 3: Update `~/.claude/settings.json` to point to the fork's built binary**

The `statusLine.command` currently uses `npx -y @owloops/claude-powerline@latest`. Update it to use the local fork's built output so your changes are live:

```json
"statusLine": {
  "type": "command",
  "command": "node /home/keegan/git/vendor/claude-powerline-fork/dist/index.js --style=powerline"
}
```

Use the `update-config` skill to make this change.

- [ ] **Step 4: Trigger a Claude response and verify the new statusline line appears**

Send any message to Claude. After the response, the statusline should show a new line like:
```
⚡ 3% 5h  ↻ 4h 12m  ⚡ 42% 7d  ↻ 2d 3h
```

If `~/.claude/usage.json` doesn't exist yet (first run), the jsonFile segments are hidden — send another message after the Stop hook fires to see them.

- [ ] **Step 5: Commit fork changes**

```bash
cd /home/keegan/git/vendor/claude-powerline-fork && git add -A && git commit -m "feat: wire usage API statusline — full implementation complete"
```
