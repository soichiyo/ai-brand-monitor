# Reporting

## Brand/Competitor Matching

After each adapter returns response text, apply these matching rules:

### Text Mention Detection

For each response text:
1. Search case-insensitively for `brand.name` and each entry in `brand.aliases[]`
2. If found: `brand_mentioned = true`
3. Extract context: 100 characters before and after the first match → `brand_context`
4. Determine `brand_rank`: count how many other service/product names appear before the brand mention. The brand_rank is 1-based (1 = mentioned first).

### Domain Citation Detection (AI Mode only)

For citation sidebar entries:
1. Check if `citation.url` or `citation.site_name` contains `brand.domain`
2. If match: `is_brand = true` on the citation record

### Competitor Matching

For each configured competitor in `config.competitors`:
1. Search response text for `competitor.name` (case-insensitive)
2. Check citation URLs for `competitor.domain`
3. Record as `entity_type: "configured_competitor"` in entity_mentions

### Entity Discovery

For services mentioned in responses that are NOT the brand and NOT configured competitors:
1. Look for proper nouns near keywords like "サービス", "アプリ", "カード", "ツール", or English service names (capitalized words)
2. Record as `entity_type: "discovered_competitor"` in entity_mentions
3. Extract mention_context (100 chars) and rank_in_response (order of appearance)

### Accuracy Evaluation

When the brand is mentioned, evaluate accuracy of the AI's description:

- `accurate`: The AI correctly identifies the brand's core product/service and key features
- `partially_inaccurate`: The AI identifies the brand but gets some details wrong (e.g., wrong pricing, wrong feature)
- `inaccurate`: The AI fundamentally misidentifies the brand (e.g., wrong product category, hallucinated facts)
- `n/a`: Brand is not mentioned

Record a one-line rationale in `accuracy_rationale`. Examples:
- "NFC型デジタル名刺サービスとして正しく説明。価格・機能とも正確"
- "デジタル名刺サービスだが、価格が異なる（実際は4,480円、AIは3,000円と記載）"
- "架空の銀行クレジットカードとして完全に誤認"

### Prompt Category Handling

If a prompt has `category: "brand_direct"`, note in the observation that this is a prompted recall test. The report will flag these with a caveat.

## Report Generation

After all observations and matching are complete, generate `{RUN_DIR}/reports/report.md`.

### Report Structure

```markdown
# AI Brand Monitor Report

Run: {run_id} | Date: {date} | Brand: {brand.name} ({brand.domain})

## Summary

- Platforms observed: {count of unique targets attempted}
- Successful observations: {count with status=success} / {total attempted}
- Skipped: {count with status!=success} ({list of skip reasons})
- Brand mention rate: {mentioned_count}/{success_count} ({percentage}%)
- Brand citation rate: {cited_count}/{citation_supported_count} ({percentage}%)
- Top competitor: {most mentioned competitor name} (mentioned in {count} observations)

## Platform Results

### Google AI Mode
- Queries observed: {count}
- Brand mentioned: {count}/{total} queries
- Citation sidebar appearances: {count} (average rank: {avg})
- Inline badge count: {total badges with brand as source}

| Query | Brand Mentioned | Brand Rank | Citation Rank | Badges | Top Competitor |
|-------|----------------|------------|---------------|--------|---------------|
| {query} | {yes/no} | {rank} | {sidebar_rank} | {count} | {name} |

### Perplexity
- Prompts observed: {count}
- Brand mentioned: {count}/{total}

| Prompt | Brand Mentioned | Brand Rank | Accuracy | Top Competitor |
|--------|----------------|------------|----------|---------------|

### ChatGPT
(same table format)

### Gemini Web
(same table format, with experimental warning)

### Claude
(same table format, with agent caveat)

### Google AIO
- Queries observed: {count}
- AIO displayed: {count}/{total}
- Brand in AIO: {count}
- Average organic rank: {avg}

| Query | AIO Status | Brand in AIO | Organic Rank | 1st Site |
|-------|-----------|-------------|-------------|---------|

## Mention Matrix

| Prompt/Query | AI Mode | Perplexity | ChatGPT | Gemini | Claude | AIO |
|-------------|---------|------------|---------|--------|--------|-----|
| {id}        | {rank or "-"} | ... | ... | ... | ... | ... |

## Competitor Map

| Service | Total Mentions | Mentioned By (platforms) | Primary Context |
|---------|---------------|------------------------|-----------------|
| {name}  | {count}       | {platform list}        | {context}       |

## Changes vs Previous Run

(If previous run data exists in output.dir)

| Observation | Previous | Current | Change |
|------------|----------|---------|--------|
| {target}_{input} | {mentioned/not} | {mentioned/not} | NEW / LOST / SAME |

## Caveats

- AI responses are nondeterministic. Single observations are snapshots, not rankings.
- Citation sidebar order varies by session and personalization.
- Prompts marked "brand_direct" (★) measure prompted recall, not natural visibility.
- Claude agent observation ≠ Claude.ai consumer web experience.
- Experimental targets (Gemini Web) may have incomplete data.
- All observations were taken from a single geographic location and browser profile.
```

### Generating the Report

1. Read all JSONL files from `{RUN_DIR}/raw/`
2. Parse each line as JSON
3. Group observations by target
4. Compute summary statistics
5. Build each table using the data
6. For "Changes vs Previous Run": find the most recent other run_id directory in `{output.dir}`, load its `observations.jsonl`, compare `brand_mentioned` and `response_text_hash` for matching observation_ids (same target + input_id)
7. Write the complete report using the Write tool

## CSV Export

After JSONL files are written, convert to CSV.

### observations.csv

Header: observation_id,run_id,target,input_id,input_text,input_type,prompt_category,status,brand_mentioned,brand_rank,brand_context,accuracy,screenshot_path

Generate using Bash:
```bash
echo 'observation_id,run_id,target,input_id,input_text,input_type,prompt_category,status,brand_mentioned,brand_rank,brand_context,accuracy,screenshot_path' > "{RUN_DIR}/csv/observations.csv"
cat "{RUN_DIR}/raw/observations.jsonl" | while IFS= read -r line; do
  echo "$line" | jq -r '[.observation_id, .run_id, .target, .input_id, .input_text, .input_type, .prompt_category, .status, .brand_mentioned, .brand_rank, .brand_context, .accuracy, .screenshot_path] | @csv'
done >> "{RUN_DIR}/csv/observations.csv"
```

### citations.csv

Header: citation_id,observation_id,rank,site_name,article_title,url,normalized_domain,is_brand,is_competitor

Same jq pattern for citations.jsonl.

### entity_mentions.csv

Header: mention_id,observation_id,entity_name,entity_type,mention_context,rank_in_response

Same jq pattern for entity_mentions.jsonl.

Note: CSV export requires `jq` to be installed (Prerequisites参照).

## Interactive Init (/ai-brand-monitor init)

When user runs `/ai-brand-monitor init`:

1. Ask: "What is your brand name?" → brand.name
2. Ask: "What is your brand's domain?" → brand.domain
3. Ask: "Any alternative names or spellings? (comma-separated, or skip)" → brand.aliases
4. Ask: "Who are your main competitors? (name:domain format, comma-separated, or skip)" → competitors
5. Ask: "What language should AI responses be in?" → locale.language (default: ja)
6. Ask: "What country for Google search?" → locale.country (default: JP)
7. Ask: "観測したい検索クエリを入力してください（1行に1つ、空行で終了）" → queries
8. Ask: "LLMに聞くプロンプトを入力してください。形式: カテゴリ|プロンプト文（1行に1つ、空行で終了）" → prompts
9. Ask: "どのプラットフォームを観測しますか？" → targets（全6つを推奨、安定/実験的の違いを説明）
10. config.yaml をスキルのディレクトリに保存
11. 確認: "設定を保存しました！`/ai-brand-monitor run` で最初の観測を始められます。"

## Google Sheets Export

When `output.sheets.enabled: true` in config:

1. Check if `gog` CLI is available: `which gog`
2. If not available: warn "Google Sheets出力にはgog CLI（Google API向けCLIツール）が必要です。スキップします。"
3. If available:
   - If `output.sheets.spreadsheet_id` is set: append to existing spreadsheet
   - If not set: create new spreadsheet titled "AI Brand Monitor - {brand.name}"
   - Write observations to "Observations" tab
   - Write citations to "Citations" tab
   - Write entity mentions to "Entity Mentions" tab
4. Report spreadsheet URL to user

### Write Protocol for Canonical Tabs

When maintaining canonical observation tabs in Sheets (especially if you later expand beyond the default 3 tabs), use **read+merge+sort+replace** instead of simple append:

1. Read all existing rows from the target tab.
2. Merge new observations with existing rows, using a composite key (e.g., `(date, query or prompt, device)`). Newer rows win on key collision.
3. Sort the merged dataset chronologically (date ascending, then query/prompt ascending).
4. Clear the tab and rewrite the entire sorted dataset.

**Why not append?** Append-based updates fail when blank rows exist from prior manual editing or when timestamps are out of order. A full read+merge+sort+replace guarantees chronological integrity and eliminates drift.

### Axis Projection Guidance

Design your Sheets tabs to project onto a single axis:

- **Search-query axis** (Google AIO, Google AI Mode): These are responses to `config.queries`. Keep them in a "Search Observations" tab.
- **Prompt axis** (Perplexity, ChatGPT, Gemini, Claude): These are responses to `config.prompts`. Keep them in a "LLM Observations" tab.

Do not mix query-axis observations (e.g., AI Mode) into prompt-axis tabs. `google_ai_mode` belongs with search observations because it is query-driven, even though it uses an LLM to generate the answer.

### Label Mapping Discipline

If you are migrating from an existing manually-operated spreadsheet, separate your internal enum values from user-facing display labels:

1. Define a canonical label map in your skill or script (e.g., `perplexity` → `Perplexity`, `claude` → `Claude`).
2. When reading existing rows, normalize old or experimental labels (e.g., `Claude (agent)`, `Gemini`) to canonical forms before merging.
3. When writing, always output canonical labels. Do not let ad-hoc labels from manual editing pollute downstream tabs.
4. Store the label map alongside your config or in a dedicated normalization script so it is versioned with the skill.

This prevents "label drift" where the same platform appears under multiple names across runs.

### Run Summary Output

At the end of a run, output to the user:

```
=== AI Brand Monitor Run Complete ===

Run ID: {run_id}
Duration: {minutes}m

Platform Results:
  ✅ google_ai_mode: {n} observations
  ✅ perplexity: {n} observations
  ✅ claude: {n} observations
  ✅ chatgpt_web: {n} observations
  ⚠️ gemini_web: skipped (login_required)
  ✅ google_aio: {n} observations

Brand Mention Rate: {x}/{y} ({z}%)
Brand Citation Rate: {a}/{b} ({c}%)
Top Competitor: {name} ({count} mentions)

📄 Report: {output_dir}/{run_id}/reports/report.md
📊 CSV: {output_dir}/{run_id}/csv/
📸 Screenshots: {output_dir}/{run_id}/screenshots/
```
