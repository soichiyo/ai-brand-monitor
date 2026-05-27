# Data Model

## ID Conventions

Use stable, ASCII-safe slot IDs in filenames and IDs. Never put Japanese query text or prompt text in file paths.

- `slot_id`:
  - For queries: `q01`, `q02`, ... in the order they appear in `config.yaml#queries`
  - For prompts: the `prompt.id` from `config.yaml#prompts` (e.g., `p1-recommend`)
- `observation_id` = `{run_id}_{target}_{slot_id}` (e.g., `20260415-1430_google_ai_mode_q01`)
- `raw_response_path` = `raw_text/{observation_id}.txt`
- `screenshot_path` = `screenshots/{observation_id}.png`

The text↔slot mapping is recorded in `raw/manifest.json` (see below). Reports, CSV, and Sheets consumers MUST resolve `slot_id` → human-readable text via this file.

## Run Manifest (`raw/manifest.json`)

```json
{
  "run_id": "20260415-1430",
  "queries": [
    {"slot": "q01", "text": "デジタル名刺"},
    {"slot": "q02", "text": "デジタル名刺 比較"}
  ],
  "prompts": [
    {"slot": "p1-recommend", "text": "デジタル名刺のおすすめサービスを教えてください", "category": "purchase_intent"}
  ]
}
```

Write this in Step 2 alongside `raw/run.json`. Consumers (report, CSV, Sheets) resolve `slot_id` → `text` via this file.

## JSONL Output Format

Each observation is one JSON line in `raw/observations.jsonl`:

```json
{
  "observation_id": "20260415-1430_google_ai_mode_q01",
  "run_id": "20260415-1430",
  "target": "google_ai_mode",
  "surface": "web_ui",
  "input_id": "q01",
  "input_text": "デジタル名刺",
  "input_type": "search_query",
  "prompt_category": "n/a",
  "device": "desktop",
  "locale": "ja-JP",
  "status": "success",
  "status_detail": "",
  "brand_mentioned": true,
  "brand_rank": 1,
  "brand_context": "アクメカード: デザイン性が高く...",
  "accuracy": "accurate",
  "accuracy_rationale": "NFC型デジタル名刺サービスとして正しく説明。価格・機能とも正確",
  "inaccuracy_detail": "",
  "web_access_enabled": true,
  "prior_context_present": false,
  "response_text_hash": "abc123...",
  "screenshot_path": "screenshots/20260415-1430_google_ai_mode_q01.png",
  "raw_response_path": "raw_text/20260415-1430_google_ai_mode_q01.txt"
}
```

Citations go to `raw/citations.jsonl`:
```json
{
  "citation_id": "20260415-1430_google_ai_mode_q01_c1",
  "observation_id": "20260415-1430_google_ai_mode_q01",
  "rank": 1,
  "site_name": "アクメカード",
  "article_title": "デジタル名刺ならアクメカード...",
  "url": "https://acme-card.example.com/",
  "normalized_domain": "acme-card.example.com",
  "is_brand": true,
  "is_competitor": false
}
```

Entity mentions go to `raw/entity_mentions.jsonl`:
```json
{
  "mention_id": "20260415-1430_google_ai_mode_q01_m0",
  "observation_id": "20260415-1430_google_ai_mode_q01",
  "entity_name": "Eight",
  "entity_type": "discovered_competitor",
  "mention_context": "Eight: 日本で利用率の高い名刺管理...",
  "rank_in_response": 2
}
```

Inline badges go to `raw/inline_badges.jsonl` (AI Mode only):
```json
{
  "badge_id": "20260415-1430_google_ai_mode_q01_b0",
  "observation_id": "20260415-1430_google_ai_mode_q01",
  "source_name": "アクメカード",
  "additional_refs": 3,
  "section_label": "NFCカード型"
}
```

To write JSONL, use Bash:
```bash
echo '{"observation_id":"...","run_id":"..."}' >> "$RUN_DIR/raw/observations.jsonl"
```

## Optional Fields (Synthetic Raw / Provenance)

When importing data from existing manually-operated spreadsheets (one-time migration), observations may carry these fields:

- `provenance`: `"sheet_import"` — marks data derived from a spreadsheet rather than a live run.
- `_sheet_aio_status`: Preserves the original sheet's AIO display status column value.
- `_sheet_brand_cited`: Preserves the original sheet's brand citation presence column value.
- `_sheet_llm_version`: Preserves the original LLM version column text.
- `_sheet_note`: Preserves the original notes column text.

These serve as fallbacks when canonical columns are absent. Do not add them to new native-run observations.
