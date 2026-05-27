---
name: ai-brand-monitor
description: 主要なAI検索エンジン（Google AIモード/AIO, ChatGPT, Gemini, Perplexity, Claude）における自社ブランドの露出状況を監視し、言及・引用・競合の存在を追跡してレポートを生成します。`/ai-brand-monitor` で実行します。
---

# AI Brand Monitor (AIブランドモニター)

## トリガー

- `/ai-brand-monitor`
- `/ai-brand-monitor run`
- `/ai-brand-monitor init`
- "ブランドモニターを実行して"
- "AIでの認知度をチェックして"
- "AIブランド観測を開始"

## 前提条件

- **Playwright MCPプラグイン**: ブラウザ操作に必要です。
- **jq**: CSVエクスポートに使用します（`brew install jq` 等）。
- **Googleログイン**: Gemini Webを対象にする場合、PlaywrightブラウザでGoogleアカウントにログインしている必要があります。
- **gog CLI** (任意): Google Sheetsへのエクスポートを行う場合に必要です。

## コマンド

- `/ai-brand-monitor run`: 観測のフル実行
- `/ai-brand-monitor init`: 対話形式の設定セットアップ（`config.yaml` 作成）

## 実行ワークフロー

1. `config.yaml` を読み、`brand.name`、`brand.domain`、`queries`、`prompts`、`targets`、`devices` を検証する。
2. `run_id` を `YYYYMMDD-HHmm` で生成し、`{output.dir}/{run_id}/{raw,reports,csv,screenshots,raw_text}` を作る。
3. `raw/run.json` と `raw/manifest.json` を作る。日本語クエリをファイル名に入れない（`slot_id` 使用）。
4. `config.targets` と `config.devices` を巡回し、target別adapterを実行する。
5. 成功時は `raw/observations.jsonl`、`raw_text/{observation_id}.txt`、`response_text_hash`、必要に応じてscreenshot/citationを保存する。
6. 失敗時はエラー状態を記録し、次の観測へ進む。
7. ブランド/競合マッチング、Markdown report、CSV、必要時Google Sheets更新を行う。
8. `reporting.md` のRun Summary形式でユーザーへ報告する。

## Guardrails

- `output.dir` が配布先（`.claude/skills/...` 等）と一致する場合は書き込み拒否。必ずユーザーワークスペース配下へ解決する。
- device-aware targetは `google_aio` のみ。その他targetは `sp` 指定があってもdesktop固定で1回実行する。
- Google Sheetsの派生タブはraw JSONLのみを入力に毎run全置換し、手編集しない。
- AI Mode (`google_ai_mode`) は queries軸なので、プロンプト軸のタブとは分離する。
- Sheetsのキャノニカルタブは `read+merge+sort+replace` 方式を推奨。`append` だと空行や順序崩れが起きる。

## リファレンス

- `adapters.md` — 6つのプラットフォーム別のブラウザ操作・エージェント起動手順
- `data-model.md` — JSONL出力形式、フィールド定義、ディレクトリ構造
- `reporting.md` — ブランドマッチング、レポート生成、CSV出力、Sheets連携
- `config.schema.json` — 設定ファイルのJSONスキーマ
- `examples/config.example.yaml` — 設定ファイルのサンプル
