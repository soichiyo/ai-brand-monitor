---
name: ai-brand-monitor
description: Monitor brand visibility across AI search surfaces (Google AI Mode, AIO, ChatGPT, Gemini, Perplexity, Claude). Tracks mentions, citations, competitor presence, and generates reports. Run with /ai-brand-monitor.
---

# AI Brand Monitor

## Trigger

Activate when user says: /ai-brand-monitor, "run brand monitor", "check AI visibility", "AI brand observation"

## Prerequisites

- Playwright MCP plugin must be enabled in Claude Code (provides browser automation capabilities: page navigation, JavaScript execution, screenshots, viewport resizing)
- `jq` must be installed for CSV export (`brew install jq` or equivalent)
- For Gemini Web target: user must be logged into Google in the Playwright browser
- For Google Sheets export (optional): gog CLI (Google API CLI tool) must be installed

## Commands

- `/ai-brand-monitor run` — Run full observation
- `/ai-brand-monitor init` — Interactive config setup (creates config.yaml)

## Run Workflow

### Step 1: Load Config

1. Look for `config.yaml` in the skill directory (the directory containing this SKILL.md)
2. If not found, check if user provided a path. If neither exists, suggest running `/ai-brand-monitor init`
3. Read the YAML file using the Read tool
4. Validate required fields:
   - `brand.name` — must be non-empty string
   - `brand.domain` — must be non-empty string
   - `queries` — must be non-empty array
   - `prompts` — must be non-empty array
   - `targets` — must be non-empty array, values must be one of: google_aio, google_ai_mode, perplexity, chatgpt_web, gemini_web, claude
5. Set defaults for missing optional fields:
   - `brand.aliases` → []
   - `competitors` → []
   - `locale.language` → "ja"
   - `locale.country` → "JP"
   - `output.dir` → "./outputs"

### Step 2: Initialize Run

1. Generate run_id: format YYYYMMDD-HHmm (e.g., 20260415-1430)
2. Create output directory structure using Bash:
```bash
RUN_DIR="{output.dir}/{run_id}"
mkdir -p "$RUN_DIR"/{raw,reports,csv,screenshots,raw_text}
```
3. Write run metadata to `raw/run.json`:
```json
{
  "run_id": "{run_id}",
  "started_at": "{ISO datetime}",
  "config_hash": "{SHA256 of config.yaml}",
  "version": "0.1.0"
}
```

### Step 3: Execute Observations

For each target in `config.targets`, in order:

1. Determine input type:
   - `google_aio`, `google_ai_mode` → use `config.queries` (input_type: search_query)
   - `perplexity`, `chatgpt_web`, `gemini_web`, `claude` → use `config.prompts` (input_type: llm_prompt)

2. For each input (query or prompt):
   - Generate observation_id: `{run_id}_{target}_{input_id}`
     - For queries: input_id = sanitized query text (spaces→underscores, max 30 chars)
     - For prompts: input_id = prompt.id (e.g., purchase-digital-card)
   - Call the target adapter (詳細は adapters.md を参照)
   - The adapter returns a ParsedObservation or an error status
   - On success: write observation to JSONL (形式は data-model.md を参照), save artifacts
   - On failure: write observation with error status, log warning, continue

3. Report progress: "Running {target}... {n}/{total} inputs complete"

4. After each target completes: "{target}: {success_count} succeeded, {skip_count} skipped"

### Step 4: Post-Processing

After all targets complete:
1. Run brand/competitor matching on all observations (詳細は reporting.md を参照)
2. Generate Markdown report (詳細は reporting.md を参照)
3. Export CSV files (詳細は reporting.md を参照)
4. If previous run exists, add comparison section to report

### Step 5: Summary

実行結果をユーザーに報告する（出力フォーマットは reporting.md の「Run Summary Output」セクションを参照）

## Reference Files

The following files contain detailed instructions for each phase. Read them as needed during execution:

- `adapters.md` — 6つのプラットフォーム別観測手順（Playwright操作、Agent起動）
- `data-model.md` — JSONL出力形式、フィールド定義、出力ディレクトリ構造
- `reporting.md` — ブランドマッチング、レポート生成、CSV出力、init、Sheets連携
