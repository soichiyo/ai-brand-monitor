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

## Adapter: google_ai_mode

Target: Google AI Mode (udm=50)
Surface: web_ui
Input type: search_query (uses config.queries)
Requires login: No
Supports citations: Yes (sidebar)
Supports inline badges: Yes
Stability: stable

### Execution Steps

For each query in config.queries:

1. Resize the Playwright browser to 1280x900 (use the browser_resize tool).

2. Navigate the Playwright browser to AI Mode (use the browser_navigate tool):
   URL: `https://www.google.co.jp/search?q={encoded_query}&hl={config.locale.language}&gl={config.locale.country}&udm=50`

3. Wait for AI response generation (5 seconds).

4. Extract main content and brand detection. Execute JavaScript in the browser page (use the browser_run_code tool):
```javascript
async (page) => {
  await page.waitForTimeout(5000);
  const bodyText = await page.evaluate(() => document.body.innerText);
  
  // Check brand mentions (will be matched in post-processing)
  // Extract inline badges
  const badges = [];
  const badgeRegex = /([^\n]{2,40}?)\s+\+(\d+)/g;
  let match;
  while ((match = badgeRegex.exec(bodyText)) !== null) {
    badges.push({ source_name: match[1].trim(), additional_refs: parseInt(match[2]) });
  }
  
  return JSON.stringify({
    bodyText: bodyText.substring(0, 8000),
    badges
  }, null, 2);
}
```

5. Extract citation sidebar (right panel source ranking). Execute JavaScript in the browser page (use the browser_run_code tool):
```javascript
async (page) => {
  const rightTexts = await page.evaluate(() => {
    const walker = document.createTreeWalker(document.body, NodeFilter.SHOW_TEXT);
    const texts = [];
    let node;
    while (node = walker.nextNode()) {
      const range = document.createRange();
      range.selectNode(node);
      const rect = range.getBoundingClientRect();
      if (rect.left > 700 && node.textContent.trim().length > 2) {
        texts.push({ text: node.textContent.trim(), top: Math.round(rect.top) });
      }
    }
    return texts.sort((a, b) => a.top - b.top);
  });
  
  // Parse citation cards from right panel text
  // Pattern: "N 件のサイト" header, then repeating: title → description → site_name
  const totalSitesMatch = rightTexts.find(t => t.text.match(/(\d+)\s*件のサイト/));
  const totalSites = totalSitesMatch ? parseInt(totalSitesMatch.text.match(/(\d+)/)[1]) : 0;
  
  // Extract citation entries (skip the header and "すべて表示")
  const citationTexts = rightTexts.filter(t => 
    !t.text.match(/件のサイト/) && t.text !== 'すべて表示' && t.text.length > 3
  );
  
  return JSON.stringify({ totalSites, citationTexts });
}
```

6. Take a screenshot of the page (use the browser_take_screenshot tool). Save as PNG, fullPage, to `{RUN_DIR}/screenshots/{observation_id}.png`.

7. Save raw response text to `{RUN_DIR}/raw_text/{observation_id}.txt` using Write tool.

### Parsing Results

From the extracted data, construct:

Observation record:
- target: "google_ai_mode"
- surface: "web_ui"
- input_type: "search_query"
- device: "desktop"
- status: "success" (or "no_ai_answer" if bodyText is empty/error)
- web_access_enabled: true
- prior_context_present: false
- response_text_hash: SHA256 of bodyText (use `echo -n "text" | shasum -a 256 | cut -d' ' -f1`)

Citation records:
Parse the citationTexts array into citation entries. The pattern in the sidebar is:
- Line with article title (longer text, often with "..." truncation)
- Line with date or description snippet
- Line with site name (shorter, like "プレーリーカード", "SKYPCE", "Eight")

Group every 2-3 lines as one citation card. For each:
- rank: position (1-based)
- site_name: the short line (site identifier)
- article_title: the title line
- url: not directly available from text; set to "" (URL extraction requires clicking which is fragile)
- normalized_domain: extract from site_name if recognizable, otherwise ""
- is_brand: check if site_name contains any of brand.name or brand.aliases
- is_competitor: check if site_name matches any competitor.name

Inline badge records:
From the badges array:
- source_name: badge label
- additional_refs: the +N number
- section_label: try to determine from context (best effort, can be "")

### Error Handling

- If page fails to load: status = "timeout", save empty observation
- If AI Mode response is empty: status = "no_ai_answer"
- If parsing fails: status = "parse_failed", still save screenshot and raw text
- Never block the run — always continue to next query

## Adapter: perplexity

Target: Perplexity
Surface: web_ui
Input type: llm_prompt (uses config.prompts)
Requires login: No
Supports citations: No
Supports inline badges: No
Stability: stable

### Execution Steps

For each prompt in config.prompts:

1. Navigate the Playwright browser to Perplexity (use the browser_navigate tool):
   URL: `https://www.perplexity.ai/search?q={encoded_prompt_text}`

2. Wait 10 seconds for response generation.

3. Extract response:
```javascript
async (page) => {
  await page.waitForTimeout(10000);
  const answerText = await page.$$eval(
    '[class*="prose"], [class*="answer"], article, main p',
    els => els.map(e => e.textContent).join('\n')
  );
  return answerText;
}
```

4. Take screenshot (fullPage).
5. Save raw response text.

### Parsing

- target: "perplexity"
- surface: "web_ui"
- input_type: "llm_prompt"
- prompt_category: from prompt.category in config
- web_access_enabled: true (Perplexity always searches web)
- prior_context_present: false
- status: "success" if answerText is non-empty, "no_ai_answer" if empty

### Error Handling

- Empty response after timeout: status = "no_ai_answer"
- Navigation failure: status = "timeout"
- Always continue to next prompt

## Adapter: claude

Target: Claude
Surface: agent
Input type: llm_prompt (uses config.prompts)
Requires login: No
Supports citations: No
Supports inline badges: No
Stability: stable

### Execution Steps

For each prompt in config.prompts:

1. Spawn a fresh Agent subagent with NO project context (use the Agent tool with subagent_type: general-purpose):
   - description: "Brand observation: {prompt.id}"
   - prompt: "You are a fresh assistant with NO prior context about any brand or service. Do NOT read any project files. Answer the following question naturally in {config.locale.language}.\n\nQuestion: {prompt.text}\n\nProvide a helpful, detailed answer."

2. Capture the agent's response text.
3. No screenshot needed (no browser).
4. Save raw response text to file.

### Parsing

- target: "claude"
- surface: "agent"
- input_type: "llm_prompt"
- prompt_category: from prompt.category in config
- web_access_enabled: false (Claude agent has no web access)
- prior_context_present: false (clean agent)
- status: "success" if response is non-empty

### Notes

- Run prompts sequentially (not parallel) to avoid context bleed between agents
- Agent responses are text-only, no screenshots
- This tests Claude's training data knowledge, NOT Claude.ai web search
- Add caveat in report: "Claude agent observation ≠ Claude.ai consumer web experience"

## Adapter: chatgpt_web

Target: ChatGPT Web
Surface: web_ui
Input type: llm_prompt (uses config.prompts)
Requires login: No (limited queries without login)
Supports citations: No
Supports inline badges: No
Stability: stable

### Execution Steps

For each prompt in config.prompts:

1. Navigate the Playwright browser to ChatGPT (use the browser_navigate tool):
   URL: `https://chatgpt.com/`

2. Wait 3 seconds for page load.

3. Find input and submit prompt:
```javascript
async (page) => {
  await page.waitForTimeout(3000);
  const input = await page.$('#prompt-textarea');
  if (!input) return JSON.stringify({ error: 'input_not_found' });
  
  await input.click();
  await page.waitForTimeout(500);
  await page.keyboard.type('{prompt_text}', { delay: 30 });
  await page.waitForTimeout(500);
  await page.keyboard.press('Enter');
  
  // Wait for response
  await page.waitForTimeout(20000);
  
  const responseText = await page.evaluate(() => {
    const msgs = document.querySelectorAll('[data-message-author-role="assistant"]');
    if (msgs.length > 0) return msgs[msgs.length - 1].innerText;
    const articles = document.querySelectorAll('article, [class*="markdown"]');
    if (articles.length > 0) return articles[articles.length - 1].innerText;
    return '';
  });
  
  return responseText;
}
```

4. Take screenshot.
5. Save raw response text.

### Parsing

- target: "chatgpt_web"
- surface: "web_ui"
- input_type: "llm_prompt"
- prompt_category: from prompt.category in config
- web_access_enabled: false (without login, Browse mode is off)
- prior_context_present: false (new chat each time)

### Error Handling

- Login wall detected (page contains "ログイン" prominently): status = "login_required"
- Input not found: status = "parse_failed"
- Empty response after 20s: status = "no_ai_answer"
- Rate limit: status = "blocked", status_detail = "rate_limited"
- Always navigate to fresh chatgpt.com/ for each prompt (prevents conversation context)

## Adapter: gemini_web

Target: Gemini Web
Surface: web_ui
Input type: llm_prompt (uses config.prompts)
Requires login: Yes (Google account)
Supports citations: No
Supports inline badges: No
Stability: experimental

### Execution Steps

For each prompt in config.prompts:

1. Navigate the Playwright browser to Gemini (use the browser_navigate tool):
   URL: `https://gemini.google.com/app`

2. Wait 3 seconds. Check if logged in:
```javascript
async (page) => {
  await page.waitForTimeout(3000);
  const pageText = await page.evaluate(() => document.body.innerText.substring(0, 500));
  const hasEditor = await page.$('.ql-editor, [contenteditable="true"]');
  const isLoggedIn = hasEditor && !pageText.includes('ログイン') && !pageText.includes('Sign in');
  return JSON.stringify({ isLoggedIn });
}
```

3. If NOT logged in: set status = "login_required", status_detail = "Gemini requires Google login. Log in via browser and retry.", skip this target entirely (don't try remaining prompts).

4. If logged in, submit prompt:
```javascript
async (page) => {
  const editor = await page.$('.ql-editor, [contenteditable="true"]');
  await editor.click();
  await page.waitForTimeout(300);
  await page.keyboard.type('{prompt_text}', { delay: 15 });
  await page.waitForTimeout(300);
  await page.keyboard.press('Enter');
  
  await page.waitForTimeout(25000); // Gemini is slower
  
  const text = await page.evaluate(() => document.body.innerText);
  return text;
}
```

5. Take screenshot.
6. Save raw response text.

### Parsing

- target: "gemini_web"
- surface: "web_ui"
- input_type: "llm_prompt"
- web_access_enabled: true (Gemini Web has Google Search grounding)
- prior_context_present: false (new chat per prompt)

### Important Notes

- This target is EXPERIMENTAL. Failures do not block the run.
- Gemini Web with grounding gives fundamentally different results from Gemini API without grounding.
- Navigate to fresh /app for each prompt to avoid conversation context.
- Report warning: "Gemini Web is experimental and requires Google login."

## Adapter: google_aio

Target: Google AIO (AI Overview in regular SERP)
Surface: web_ui
Input type: search_query (uses config.queries)
Requires login: No
Supports citations: No
Supports inline badges: No
Stability: stable

### Execution Steps

For each query in config.queries:

1. Navigate the Playwright browser to Google Search (use the browser_navigate tool):
   URL: `https://www.google.co.jp/search?q={encoded_query}&hl={config.locale.language}&gl={config.locale.country}`

2. Wait 3 seconds.

3. Extract AIO status, organic results, and PAA:
```javascript
async (page) => {
  await page.waitForTimeout(3000);
  const content = await page.content();
  
  // AIO detection
  const hasAIO = content.includes('AI による概要') || content.includes('AIによる概要') || content.includes('AI Overview');
  const aioFailed = content.includes('この検索では AI による概要を表示できません') || content.includes('AI による概要を生成できません');
  let aioStatus;
  if (!hasAIO) aioStatus = 'not_available';
  else if (aioFailed) aioStatus = 'generation_failed';
  else aioStatus = 'displayed';
  
  // AIO text (if displayed)
  let aioText = '';
  if (aioStatus === 'displayed') {
    const bodyText = await page.evaluate(() => document.body.innerText);
    const aioIdx = bodyText.indexOf('AI による概要');
    if (aioIdx >= 0) {
      aioText = bodyText.substring(aioIdx, Math.min(bodyText.length, aioIdx + 1000));
    }
  }
  
  // Organic results
  const results = await page.$$eval('a h3', els => els.slice(0, 10).map((h3, i) => ({
    rank: i + 1,
    title: h3.textContent || '',
    url: h3.closest('a')?.href || ''
  })));
  
  // PAA (People Also Ask)
  let paa = [];
  try {
    paa = await page.$$eval('[data-sgrd] [data-q]', els => els.map(el => el.getAttribute('data-q')));
  } catch(e) {}
  
  return JSON.stringify({ aioStatus, aioText, results, paa });
}
```

4. Take screenshot (fullPage).
5. Save raw response text.

### Parsing

- target: "google_aio"
- surface: "web_ui"
- input_type: "search_query"
- device: "desktop"
- web_access_enabled: true
- prior_context_present: false

For the observation:
- If aioStatus is "displayed": check aioText for brand mentions. Set brand_mentioned based on AIO text content.
- If aioStatus is "not_available" or "generation_failed": brand_mentioned = false for AIO, but still record organic rank.
- brand_rank: find brand.domain in organic results URLs. Record the organic position.
- status: "success" (page loaded). Use status_detail to record aioStatus value.

### Additional Fields (AIO-specific)

Store AIO-specific data in the observation's status_detail field:
```
status_detail: "aio_status={aioStatus}, organic_rank={rank}, paa_count={paa.length}"
```

Store PAA questions in raw_text file along with AIO text.

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

Note: CSV export requires `jq` to be installed. If jq is not available, skip CSV export and note in the run summary.

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

## Google Sheets Export (Optional)

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

## Run Summary Output

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
