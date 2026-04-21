# Adapters

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

### Execution Rules (MUST follow)

- Each prompt MUST be a separate Agent spawn. NEVER batch multiple prompts into one agent call. Batching reduces mention rates by allowing the agent to diversify answers across questions
- Run prompts sequentially (not parallel) to avoid context bleed between agents
- Save each agent's raw response to `{RUN_DIR}/raw_text/{observation_id}.txt`
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
  
  // Extract ONLY the latest model response, not sidebar/navigation
  const text = await page.evaluate(() => {
    // Gemini renders responses in message-content containers
    const responses = document.querySelectorAll(
      'message-content, .response-container, .model-response-text, [class*="response"]'
    );
    if (responses.length > 0) {
      return responses[responses.length - 1].innerText;
    }
    // Fallback: get main content area, excluding sidebar
    const main = document.querySelector('main, [role="main"], .conversation-container');
    if (main) return main.innerText;
    // Last resort: full body (will be flagged as potentially polluted)
    return document.body.innerText;
  });
  return text;
}
```

5. Take screenshot.
6. Save raw response text.

### Extraction Quality

- The extraction targets the latest model response container only, avoiding sidebar chat history
- If the selector falls back to `document.body.innerText`, mark `status_detail` with `extraction_fallback=body` to flag potential sidebar pollution
- Brand matches found only in sidebar/navigation text (not in the response body) should be excluded from mention counting

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
