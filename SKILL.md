---
name: ai-brand-monitor
description: Monitor brand visibility across AI search surfaces (Google AI Mode, AIO, ChatGPT, Gemini, Perplexity, Claude). Tracks mentions, citations, competitor presence, and generates reports. Run with /ai-brand-monitor.
---

# AI Brand Monitor

## Trigger

Activate when user says: /ai-brand-monitor, "run brand monitor", "check AI visibility", "AI brand observation"

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
   - `output.markdown` → true
   - `output.csv` → true
   - `output.jsonl` → true
   - `output.screenshots` → true

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
   - Call the target adapter (see adapter sections below)
   - The adapter returns a ParsedObservation or an error status
   - On success: write observation to JSONL, save artifacts
   - On failure: write observation with error status, log warning, continue

3. Report progress: "Running {target}... {n}/{total} inputs complete"

4. After each target completes: "{target}: {success_count} succeeded, {skip_count} skipped"

### Step 4: Post-Processing

After all targets complete:
1. Run brand/competitor matching on all observations (see matching section)
2. Generate Markdown report (see reporting section)
3. Export CSV files (see CSV section)
4. If previous run exists, add comparison section to report

### Step 5: Summary

Report to user:
- Run ID and output directory path
- Per-target status summary
- Brand mention rate
- Path to report file

## JSONL Output Format

Each observation is one JSON line in `raw/observations.jsonl`:

```json
{
  "observation_id": "20260415-1430_google_ai_mode_デジタル名刺",
  "run_id": "20260415-1430",
  "target": "google_ai_mode",
  "surface": "web_ui",
  "input_id": "デジタル名刺",
  "input_text": "デジタル名刺",
  "input_type": "search_query",
  "prompt_category": "n/a",
  "device": "desktop",
  "locale": "ja-JP",
  "status": "success",
  "status_detail": "",
  "brand_mentioned": true,
  "brand_rank": 1,
  "brand_context": "プレーリーカード: デザイン性が高く...",
  "accuracy": "accurate",
  "inaccuracy_detail": "",
  "web_access_enabled": true,
  "prior_context_present": false,
  "response_text_hash": "abc123...",
  "screenshot_path": "screenshots/20260415-1430_google_ai_mode_デジタル名刺.png",
  "raw_response_path": "raw_text/20260415-1430_google_ai_mode_デジタル名刺.txt"
}
```

Citations go to `raw/citations.jsonl`:
```json
{
  "citation_id": "20260415-1430_google_ai_mode_デジタル名刺_c1",
  "observation_id": "20260415-1430_google_ai_mode_デジタル名刺",
  "rank": 1,
  "site_name": "プレーリーカード",
  "article_title": "デジタル名刺ならプレーリーカード...",
  "url": "https://prairie.cards/",
  "normalized_domain": "prairie.cards",
  "is_brand": true,
  "is_competitor": false
}
```

Entity mentions go to `raw/entity_mentions.jsonl`:
```json
{
  "mention_id": "20260415-1430_google_ai_mode_デジタル名刺_m0",
  "observation_id": "20260415-1430_google_ai_mode_デジタル名刺",
  "entity_name": "Eight",
  "entity_type": "discovered_competitor",
  "mention_context": "Eight: 日本で利用率の高い名刺管理...",
  "rank_in_response": 2
}
```

Inline badges go to `raw/inline_badges.jsonl` (AI Mode only):
```json
{
  "badge_id": "20260415-1430_google_ai_mode_デジタル名刺_b0",
  "observation_id": "20260415-1430_google_ai_mode_デジタル名刺",
  "source_name": "プレーリーカード",
  "additional_refs": 3,
  "section_label": "NFCカード型"
}
```

To write JSONL, use Bash:
```bash
echo '{"observation_id":"...","run_id":"..."}' >> "$RUN_DIR/raw/observations.jsonl"
```

## Adapter Sections (Placeholder)

The following adapter sections will be added in subsequent tasks:
- google_ai_mode adapter
- perplexity adapter
- claude adapter
- chatgpt_web adapter
- gemini_web adapter
- google_aio adapter

Each adapter section will contain step-by-step Playwright/Agent instructions.

## Brand Matching (Placeholder)

Will be added in Task 10.

## Report Generation (Placeholder)

Will be added in Task 11.

## CSV Export (Placeholder)

Will be added in Task 12.
