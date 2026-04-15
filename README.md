# ai-brand-monitor

自社ブランドが、AI検索でどう扱われているかを定点観測する Claude Code スキルです。

ChatGPT・Gemini・Perplexity・Claude・Google AIモードに対して「おすすめのサービスは？」と聞いたとき、自社が紹介されているか、競合と比べてどの位置にいるかを一括でチェックし、レポートにまとめます。

## できること

- 6つのAIプラットフォームに対して、検索クエリやプロンプトを自動で投げる
- 自社ブランドが言及されているか、何番目に紹介されているかを記録
- Google AIモードの「引用元サイト」ランキング（右サイドバー）を取得
- 競合サービスの言及頻度を集計
- 前回の観測結果と比較して、変化を検出
- Markdown レポート・CSV・スクリーンショットを出力

## セットアップ

### 必要なもの

- Claude Code がインストール済みであること
- Claude Code の Playwright MCP プラグインが有効であること
- `jq` がインストール済みであること（CSV出力に使います）

### 手順

1. このリポジトリを `.claude/skills/` にクローンします:
```bash
cd ~/.claude/skills
git clone https://github.com/YOUR_USERNAME/ai-brand-monitor.git
```
（`YOUR_USERNAME` は自分のGitHubユーザー名に置き換えてください）

2. 設定ファイルをコピーして、自社の情報に書き換えます:
```bash
cd ai-brand-monitor
cp examples/config.example.yaml config.yaml
```

3. `config.yaml` を開いて、ブランド名・ドメイン・クエリ・プロンプトを編集します（後述）

4. Claude Code で実行します:
```
/ai-brand-monitor run
```

対話形式でゼロから設定ファイルを作ることもできます:
```
/ai-brand-monitor init
```

## 設定ファイルの書き方

`config.yaml` を編集します。`#` から始まる行はコメント（メモ）なので、自由に書き換えて大丈夫です。

```yaml
# --- ブランド情報 ---
brand:
  name: "自社サービス名"           # AI回答の中からこの名前を探します
  domain: "example.com"           # 引用元URLにこのドメインが含まれるかチェックします
  aliases: ["別名", "English Name"]  # 略称や英語名があれば追加

# --- 競合（任意） ---
competitors:
  - name: "競合A"
    domain: "competitor-a.com"
  - name: "競合B"
    domain: "competitor-b.com"

# --- 言語・地域 ---
locale:
  language: "ja"    # AIへの質問・回答の言語
  country: "JP"     # Google検索の対象国

# --- 検索クエリ（Google AIO・AIモード用） ---
queries:
  - "デジタル名刺"
  - "デジタル名刺 比較"
  - "デジタル名刺 おすすめ"

# --- LLMに聞くプロンプト ---
prompts:
  - id: P1
    text: "デジタル名刺のおすすめサービスを教えてください"
    category: purchase_intent       # 購買検討系
  - id: P2
    text: "NFC型のデジタル名刺でおすすめはどれですか"
    category: purchase_intent
  - id: P3
    text: "自社サービス名の評判を教えてください"
    category: brand_direct          # ブランド名指し（誘導された想起として注記されます）

# --- 観測対象 ---
targets:
  - google_ai_mode    # Google AIモード
  - google_aio        # Google通常検索のAI Overview
  - perplexity        # Perplexity
  - chatgpt_web       # ChatGPT
  - claude            # Claude（ブラウザ不要）
  - gemini_web        # Gemini Web版（Googleログインが必要、実験的）
```

### プロンプトのカテゴリについて

| カテゴリ | 意味 | 例 |
|---------|------|-----|
| `purchase_intent` | 購買検討・比較系 | 「おすすめを教えて」「比較して」 |
| `info_gathering` | 情報収集系 | 「メリット・デメリットは？」「手順を教えて」 |
| `brand_direct` | ブランド名を含む質問 | 「〇〇の評判は？」 |
| `neutral` | ブランド名を含まない一般的な質問 | 「選ぶポイントは？」 |

`brand_direct` のプロンプトは、レポート上で「ブランド名を直接聞いているので、自然な想起とは異なります」と注記されます。

## 観測対象のプラットフォーム

| プラットフォーム | ログイン | 安定性 | 取得できる情報 |
|---------------|---------|-------|-------------|
| Google AIモード | 不要 | 安定 | AI回答テキスト、引用元サイトランキング、インライン引用バッジ |
| Google AIO | 不要 | 安定 | AI Overview表示状況、ブランド言及、オーガニック順位、PAA |
| Perplexity | 不要 | 安定 | AI回答テキスト、ブランド言及、競合分析 |
| ChatGPT | 不要* | 安定 | AI回答テキスト、ブランド言及分析 |
| Claude | 不要 | 安定 | AI回答テキスト（訓練データからの回答。Web検索なし） |
| Gemini Web | 必要 | 実験的 | AI回答テキスト（Google検索による情報補完あり） |

*ChatGPTはログインなしでも使えますが、回数に制限があります。

Gemini Webは実験的な対象です。Googleログインが必要で、ログインできない場合はスキップされます（他の観測は止まりません）。

## 出力されるもの

観測のたびに、タイムスタンプ付きのフォルダが作られます:

```
outputs/
  20260415-1430/
    reports/report.md          # 観測レポート（人間が読む用）
    raw/observations.jsonl     # 観測データ（機械処理用）
    raw/citations.jsonl        # AIモードの引用元データ
    raw/entity_mentions.jsonl  # 競合サービスの言及データ
    csv/observations.csv       # スプレッドシートに取り込める形式
    csv/citations.csv
    screenshots/               # 各プラットフォームのスクリーンショット
    raw_text/                  # AIの回答テキスト原文
```

### Google Sheetsへの出力（任意）

`config.yaml` で `output.sheets.enabled: true` にすると、Google Sheetsにも出力できます。gog CLI（Google API向けCLIツール）のインストールが別途必要です。

## 注意事項

この観測は「ある時点のスナップショット」であり、確定的なランキングではありません:

1. AIの回答は毎回変わります。同じ質問でも違う結果が出ることがあります
2. Google AIモードの引用元ランキングは、セッションやパーソナライゼーションで変動します
3. `brand_direct` カテゴリのプロンプトは、ブランド名を直接聞いているため、自然な想起テストではありません
4. Claudeの観測はClaude Codeのエージェント経由です。Claude.aiのWeb版とは異なる結果になることがあります
5. 実験的な対象（Gemini Web）はデータが不完全になることがあります

信頼性のあるトレンド分析には、月1回の定期観測をおすすめします。

### 利用規約について

このツールはブラウザの自動操作を通じて各AIプラットフォームの回答を取得します。各プラットフォームの利用規約によっては、自動化されたアクセスが制限されている場合があります。ご利用の際は各プラットフォームの利用規約を確認の上、自己責任でお使いください。月に数回・数クエリ程度の個人利用を想定しており、大量のリクエストを送る用途には対応していません。

## よくある質問

Q: どのくらいの頻度で観測すべき？
A: 月1回がおすすめです。前回との差分比較に対応しています。

Q: Geminiが「実験的」なのはなぜ？
A: Gemini Web版はGoogleログインが必要で、ログイン状態に依存します。ただし、Web版（Google検索による情報補完あり）はAPI版（情報補完なし）とは全く異なる結果を返すため、観測する価値は高いです。

Q: ChatGPTの結果が期待と違う？
A: ログインなしの場合、小さいモデルが使われることがあります。ログイン状態でGPT-4o + Browse modeをONにした場合とは結果が異なります。

Q: 自社ブランドがどこにも出てこない場合は？
A: まずは、AIが引用しやすい比較記事・まとめ記事を作成することが有効です。構造化データの整備や、第三者サイトでの言及を増やすことも効果的です。

## コントリビューション

Issue や PR を歓迎します。

## ライセンス

MIT
