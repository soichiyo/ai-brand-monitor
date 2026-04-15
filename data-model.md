# Data Model

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
  "accuracy_rationale": "NFC型デジタル名刺サービスとして正しく説明。価格・機能とも正確",
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
