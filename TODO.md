# BurnLog — Phase 1: Claude Code JSONL Parser

## Context

`inference_analyzer.html` is a working single-file tool for analyzing VR-project inference logs. The 90-day plan turns it into a public Claude Code session analyzer. Phase 1 (weeks 1-3) adds native parsing of `~/.claude/projects/**/*.jsonl` without touching the existing VR format or internal data model.

---

## Internal Schema (keep as canonical, extend only with cache fields)

```
{ timestamp, call_type, model, source, input_tokens, output_tokens,
  latency_ms, cost_usd, status,
  cache_read_tokens,        // NEW — for cost accuracy + cache-hit view
  cache_write_tokens }      // NEW — for cost accuracy
```

---

## Tasks

### Step 1 — Pricing table
- [ ] Add `PRICING` constant with claude-opus-4, claude-sonnet-4, claude-haiku-4 rates (input/output/cache_read/cache_write per million tokens)
- [ ] Add `calcCost(model, usage)` helper — includes cache read (0.1x) and cache write (1.25x) costs

### Step 2 — Format detector
- [ ] Add `detectFormat(text)` — scan up to first 10 valid JSON lines; return `'claude-code'` on the first that has both `sessionId` + `type`, else `'vr'` (tolerates logs that open with metadata or summary events)
- [ ] Wire into `readFiles()` and `parsePaste()` before dispatching to parser

### Step 3 — `parseClaudeCodeJSONL(text)`
- [ ] Parse all lines, skip non-JSON and non-assistant events
- [ ] For each `type === 'assistant'` with `message.usage`:
  - **latency_ms**: scan backward for nearest `type === 'user'` timestamp; store raw delta (no cap here — cap at render time)
  - **call_type**: `attributionSkill` → `skill:{name}`, tool_use blocks → `tool:{name}` or `tool:multi`, thinking block → `thinking`, else `text`
  - **cost_usd**: via `calcCost()`
  - **cache fields**: map `message.usage.cache_read_input_tokens → cache_read_tokens`, `message.usage.cache_creation_input_tokens → cache_write_tokens` (normalize to canonical schema names at parser boundary)
- [ ] Two-pass tool error association:
  - **Pass 1**: collect all `tool_result` blocks from `type === 'user'` events into a `Map<tool_use_id, is_error>`
  - **Pass 2**: when processing each assistant event's tool_use blocks, check if any `id` is in the error map → `status: 'failed'`
- [ ] Edge cases:
  - Multi-block messages (collect all tool names)
  - Empty thinking blocks (skip — empty string means hidden)
  - Idle gaps (latency_ms raw — cap to 300,000 ms only in render/display path)
  - `compact_boundary` events — skip silently
  - Files with no assistant messages — return `[]`

### Step 4 — Wire into existing pipeline
- [ ] `readFiles()`: detect format, dispatch to `parseClaudeCodeJSONL()` or existing `parseLines()`
- [ ] `parsePaste()`: same
- [ ] No changes to `processData()`, `aggBy()`, or any render function except `renderMetrics()` (see Step 5)

### Step 6 — "Connect to Claude Logs" (File System Access API)
- [ ] Add "Connect to Claude Logs" button to UI
- [ ] On click: call `window.showDirectoryPicker()`, recursively walk the directory for all `.jsonl` files, pipe them through the existing `readFiles()` pipeline
- [ ] Persist the directory handle to IndexedDB; on next page load, if a handle exists show "Re-use last folder?" button instead of requiring re-navigation
- [ ] Graceful fallback: if browser doesn't support `showDirectoryPicker` (Firefox), hide the button and show a tooltip explaining drag-drop is the alternative

### Step 5 — Cache hit rate metric
- [ ] In `renderMetrics()`: add cache hit rate card (`cache_read / (input + cache_read)`) — only show if any record has cache reads
- [ ] Show estimated savings: cost delta if all cache reads were billed at full input rate

---

## Verification Checklist

- [ ] Drop real `~/.claude/projects/<hash>/<session>.jsonl` — records appear, cost > $0, latency > 0
- [ ] "By call type" shows `tool:Read`, `tool:Edit`, `tool:Bash`, `text`, `thinking`, `skill:*`
- [ ] Timeline shows activity in correct UTC hours
- [ ] Raw log: timestamps/models/tokens match JSONL source
- [ ] VR-format file still parses correctly (regression check)
- [ ] File with thinking blocks — no crash, classified as `thinking`
- [ ] File with compact_boundary events — no extra records, no crash
- [ ] Multi-file load — records accumulate correctly

---

## Out of Scope (Phase 2)

- MCP timing breakdown (tool duration from hook `durationMs`)
- Session-level stacked timeline bar
- WSL2 diagnostics
- Cost anomaly flagging
- Persistent history / backend
