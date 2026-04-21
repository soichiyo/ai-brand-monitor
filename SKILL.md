---
name: ai-brand-monitor
description: 主要なAI検索エンジン（Google AIモード/AIO, ChatGPT, Gemini, Perplexity, Claude）における自社ブランドの露出状況を監視し、言及・引用・競合の存在を追跡してレポートを生成します。`/ai-brand-monitor` で実行します。
---

# AI Brand Monitor (AIブランドモニター)

## トリガー

ユーザーが以下のように発言した時に起動します：
- `/ai-brand-monitor`
- "ブランドモニターを実行して"
- "AIでの認知度をチェックして"
- "AIブランド観測を開始"

## 前提条件

- **Playwright MCPプラグイン**: ブラウザ操作（ページ遷移、JS実行、スクリーンショット、ビューポート変更）のために必須です。
- **jq**: CSVエクスポートに使用します（`brew install jq` 等でインストール済みであること）。
- **Googleログイン**: Gemini Webを対象にする場合、PlaywrightブラウザでGoogleアカウントにログインしている必要があります。
- **gog CLI** (任意): Google Sheetsへのエクスポートを行う場合に必要です。

## コマンド

- `/ai-brand-monitor run` — 観測のフル実行
- `/ai-brand-monitor init` — 対話形式の設定セットアップ（`config.yaml` の作成）

## 実行ワークフロー

### ステップ 1: 設定の読み込み

1. スキルディレクトリ（この `SKILL.md` がある場所）から `config.yaml` を探します。
2. 見つからない場合は、ユーザーが指定したパスを確認します。いずれもない場合は `/ai-brand-monitor init` の実行を提案します。
3. YAMLファイルを読み込み、以下の必須項目をバリデートします：
   - `brand.name` — 空でない文字列
   - `brand.domain` — 空でない文字列
   - `queries` — 空でない配列
   - `prompts` — 空でない配列
   - `targets` — `google_aio`, `google_ai_mode`, `perplexity`, `chatgpt_web`, `gemini_web`, `claude` のいずれかを含む空でない配列

### ステップ 2: 実行の初期化

1. `run_id` を生成します（形式: `YYYYMMDD-HHmm`）。
2. 出力ディレクトリ構造を Bash で作成します：
   `{output.dir}/{run_id}/{raw,reports,csv,screenshots,raw_text}`
3. 実行メタデータを `raw/run.json` に書き込みます。

### ステップ 3: 観測の実行

`config.targets` に指定された各ターゲットに対して順番に実行します：

1. 入力タイプの決定：
   - `google_aio`, `google_ai_mode` → `config.queries` を使用
   - `perplexity`, `chatgpt_web`, `gemini_web`, `claude` → `config.prompts` を使用

2. 各入力（クエリまたはプロンプト）の処理：
   - 各ターゲットのアダプターを呼び出します（詳細は `adapters.md` を参照）。
   - 成功時：以下の3つを必ず保存します：
     a. 観測レコードを `raw/observations.jsonl` に追記
     b. 生テキストを `raw_text/{observation_id}.txt` に保存（Write tool）
     c. `response_text_hash` に SHA256 を記録（`echo -n "text" | shasum -a 256 | cut -d' ' -f1`）
   - 失敗時：エラー状態を記録し、ログ警告を出して次へ進みます。

3. 進捗報告: 「{target} を実行中... {n}/{total} 完了」と表示します。

### ステップ 4: 後処理とレポート生成

全てのターゲットが完了した後：
1. 全ての観測結果に対してブランド/競合のマッチングを実行します（詳細は `reporting.md` を参照）。
2. Markdownレポートを生成します。
3. CSVファイルをエクスポートします。
4. 前回の実行結果がある場合は、比較セクションを追加します。

### ステップ 5: 実行サマリーの表示

ユーザーに最終結果を報告します（詳細は `reporting.md` の「Run Summary Output」を参照）。

## リファレンスファイル

各フェーズの詳細な手順については、必要に応じて以下のファイルを読み込んでください：

- `adapters.md` — 6つのプラットフォーム別のブラウザ操作・エージェント起動手順
- `data-model.md` — JSONL出力形式、フィールド定義、ディレクトリ構造
- `reporting.md` — ブランドマッチング、レポート生成、CSV出力、Sheets連携