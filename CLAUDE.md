# ai-brand-monitor

AI検索プラットフォーム（Google AIモード、AIO、ChatGPT、Gemini、Perplexity、Claude）でのブランド可視性を定点観測する Claude Code スキル。

## リポジトリ構成

- `SKILL.md` — スキル本体（オーケストレーター、5ステップ実行フロー）
- `adapters.md` — 6プラットフォームの観測手順（Playwright操作、Agent起動）
- `data-model.md` — JSONL出力形式、フィールド定義
- `reporting.md` — レポート生成、CSV出力、init、Sheets連携
- `config.schema.json` — 設定ファイルのJSONスキーマ
- `examples/config.example.yaml` — 設定ファイルのサンプル（架空ブランド「アクメカード」）
- `README.md` — セットアップ・使い方ガイド（日本語）

## 開発方針

- 対象ユーザーは外部のマーケター（プロジェクト固有の文脈を持たない）
- Claude Code のスキルとして動作する（コードベースではない）
- SKILL.md は 500 words 以内に保ち、詳細は参照ファイルに分離する
- ドキュメントは自然な日本語で書く（マーケティング的な誇張や過度な簡略化はしない）
- git の知識は前提とし、MCP/スキルの基礎説明は不要

## 編集時の注意

- SKILL.md を肥大化させない。アダプター追加は `adapters.md`、出力形式変更は `data-model.md`、レポート変更は `reporting.md` に書く
- `config.schema.json` と `examples/config.example.yaml` は常に同期させる
- 特定のツール名（browser_navigate 等）ではなく、機能の説明で記述する（例: 「ブラウザでページを開く」）
- Playwright MCP のツール名はバージョンで変わるため、ハードコードしない
- 利用規約に関する注意書き（README の「利用規約について」セクション）は維持する
