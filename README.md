# ai-brand-monitor

Monitor your brand's visibility across AI search surfaces.

Track how your brand appears when users ask ChatGPT, Gemini, Perplexity, Claude, and Google AI about your product category. Know where you're mentioned, where you're missing, and who beats you.

## What This Does

This Claude Code Skill runs observations across 6 AI platforms and generates a structured report showing:

- Whether your brand is mentioned in AI-generated responses
- Your position relative to competitors
- Which sources cite your website (Google AI Mode citation sidebar)
- How accuracy of AI descriptions of your brand
- Changes between observation runs

## Quick Start

### Prerequisites

- [Claude Code](https://docs.anthropic.com/en/docs/claude-code) installed
- [Playwright MCP plugin](https://github.com/anthropics/claude-code-plugins) enabled in Claude Code

### Setup (5 minutes)

1. Clone into your Claude Code skills directory:
   ```bash
   cd ~/.claude/skills
   git clone https://github.com/anthropics/ai-brand-monitor.git
   ```

2. Copy and edit the config:
   ```bash
   cd ai-brand-monitor
   cp examples/config.example.yaml config.yaml
   # Edit config.yaml with your brand details
   ```

3. Run your first observation:
   ```
   /ai-brand-monitor run
   ```

Or use the interactive setup:
   ```
   /ai-brand-monitor init
   ```

## Configuration

Edit `config.yaml` with your brand details:

```yaml
brand:
  name: "Your Brand"           # Exact name to search for
  domain: "yourbrand.com"      # Domain for citation matching
  aliases: ["YourBrand"]       # Alternative names/spellings

competitors:                    # Optional: known competitors
  - name: "Competitor A"
    domain: "competitor-a.com"

locale:
  language: "ja"               # AI response language
  country: "JP"                # Google search country

queries:                        # Google search queries
  - "your category"
  - "your category comparison"

prompts:                        # Questions to ask LLMs
  - id: P1
    text: "What are the best services for [your category]?"
    category: purchase_intent
  - id: P2
    text: "Tell me about [Your Brand]"
    category: brand_direct      # Flagged as prompted recall

targets:                        # Platforms to monitor
  - google_ai_mode             # Google AI Mode (stable)
  - google_aio                 # Google AIO/SERP (stable)
  - perplexity                 # Perplexity (stable)
  - chatgpt_web                # ChatGPT (stable, no login)
  - claude                     # Claude (stable, no browser)
  - gemini_web                 # Gemini (experimental, login required)
```

### Prompt Categories

| Category | Description | Note |
|----------|-------------|------|
| `purchase_intent` | Buying/comparison queries | Core visibility test |
| `info_gathering` | Educational queries | Tests awareness |
| `brand_direct` | Queries containing your brand name | Flagged as "prompted recall" |
| `neutral` | Generic category queries | Most natural test |

## Platforms

### Stable

| Platform | Method | Login | What It Captures |
|----------|--------|-------|-----------------|
| Google AI Mode | Playwright | No | AI response, citation sidebar ranking, inline badges |
| Google AIO | Playwright | No | AIO display, brand citation, organic rank, PAA |
| Perplexity | Playwright | No | AI response, brand mention, competitor analysis |
| ChatGPT | Playwright | No* | AI response, brand mention analysis |
| Claude | Agent spawn | No | AI response from training data (no web search) |

*ChatGPT works without login but has query limits.

### Experimental

| Platform | Method | Login | What It Captures |
|----------|--------|-------|-----------------|
| Gemini Web | Playwright | Yes | AI response with Google Search grounding |

Experimental targets may fail due to login requirements, rate limits, or UI changes. Failures are skipped without blocking the run.

## Output

Each run creates a timestamped directory:

```
outputs/
  20260415-1430/
    reports/report.md         # Human-readable report
    raw/observations.jsonl    # Raw observation data
    raw/citations.jsonl       # Citation sidebar data
    raw/entity_mentions.jsonl # Competitor mentions
    csv/observations.csv      # Spreadsheet-ready data
    csv/citations.csv
    screenshots/              # Page screenshots
    raw_text/                 # Raw AI response text
```

### Google Sheets (Optional)

Set `output.sheets.enabled: true` in config. Requires [gog CLI](https://github.com/...) for Google Sheets API access.

## Measurement Caveats

This tool provides observation snapshots, not definitive rankings:

1. AI responses are nondeterministic -- the same query can produce different results
2. Citation sidebar order may vary by session and personalization
3. Prompts with `brand_direct` category measure prompted recall, not natural visibility
4. Claude agent observations test training data knowledge, not Claude.ai web search
5. Experimental targets may have incomplete data
6. Observations reflect a single geographic location and browser profile

For reliable trend analysis, run monthly and compare across multiple observations.

## FAQ

Q: How often should I run this?
A: Monthly is recommended. The tool supports comparing against previous runs.

Q: Why is Gemini experimental?
A: Gemini Web requires Google login and is sensitive to account state. The web version (with Google Search grounding) gives much better results than the API version.

Q: ChatGPT shows different results than expected?
A: Without login, ChatGPT may use a smaller model. Logged-in observations with GPT-4o + Browse mode will differ.

Q: My brand isn't mentioned anywhere. What should I do?
A: Focus on creating authoritative comparison/review content that AI models can cite. Structured data and third-party mentions help AI models learn about your brand.

## Contributing

Issues and PRs welcome. This project follows standard GitHub workflow.

## License

MIT
