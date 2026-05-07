# BurnLog

A single-file browser tool for analyzing Claude Code session logs. Drop a `.jsonl` file, get an instant breakdown of cost, latency, token usage, and cache efficiency. No install, no backend, no uploads — everything runs locally.

## Usage

Open `inference_analyzer.html` in any Chromium-based browser (Chrome, Edge, Arc).

Three ways to load data:

**Connect to Claude Logs** — click the button and pick your `~/.claude/projects/` directory. The tool recursively finds all `.jsonl` files, parses them, and remembers the folder in IndexedDB so future sessions can reload with one click.

**Drag and drop** — drag one or more `.jsonl` files onto the drop zone. Multiple files accumulate into a single dataset.

**Paste** — paste raw JSONL lines into the text area and click Analyze.

> Firefox does not support the File System Access API. Use drag-and-drop instead — the Connect button is hidden automatically.

## Views

| Tab | What it shows |
|-----|---------------|
| By call type | Cost share, call count, avg latency, avg tokens per call type |
| By model | Per-model cost, latency, and call count |
| Model vs type | Heatmap of cost or latency across model × call type combinations |
| Timeline | Cost bucketed by UTC hour |
| Raw log | Last 30 records with full field detail |

**Metrics bar** (always visible):
- Total calls, total cost, avg latency
- Total tokens and model count
- Cache hit rate + estimated savings — shown only when cache data is present

## Supported formats

**Claude Code** (auto-detected) — native `~/.claude/projects/**/*.jsonl` session files. The tool derives call type from content blocks:

| Content | Call type |
|---------|-----------|
| `attributionSkill` present | `skill:<name>` |
| Single tool-use block | `tool:<ToolName>` |
| Multiple tool-use blocks | `tool:multi` |
| Thinking block (non-empty) | `thinking` |
| Text only | `text` |

Cost is calculated from the Anthropic pricing table (opus-4 / sonnet-4 / haiku-4) including cache read and cache write token rates.

**VR / game log format** (legacy) — original format with explicit `call_type`, `cost_usd`, etc. fields. Detected automatically when Claude Code fields are absent.

## Pricing table

| Model prefix | Input | Output | Cache read | Cache write |
|---|---|---|---|---|
| claude-opus-4 | $15/M | $75/M | $1.50/M | $18.75/M |
| claude-sonnet-4 | $3/M | $15/M | $0.30/M | $3.75/M |
| claude-haiku-4 | $0.80/M | $4/M | $0.08/M | $1.00/M |

