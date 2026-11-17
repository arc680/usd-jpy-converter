# agent-workflow-template

複数 AI エージェントで開発するための、リポジトリ雛形。

考え方の要点は `AGENTS.md` に集約してある:

- 状態の中心は GitHub（Issue / ブランチ / PR）。入口はスマホでも自宅マシンでもクラウドでもよい
- 仕様策定 → ブラッシュアップ → 実装 → クロスレビュー → 検収・マージ を、別々のエージェントが担当する（人間の複数人チームと同じ構成）
- 認証は分離し、連携は GitHub など双方が公式サポートする経路だけを使う

## 使い方

1. GitHub でこのリポジトリをテンプレートとして有効化し（Settings → Template repository）、新プロジェクトは「Use this template」で作る
2. 新リポジトリの `PROJECT.md` に、そのプロジェクト固有の内容（目的・構成・データの置き場・設置先）を書く
3. 作業は Issue 起票から始める。仕様は `templates/SPEC.md`、受け渡しは `templates/HANDOFF.md`、レビュー観点は `templates/REVIEW.md` の形を使う

## ファイル構成

| ファイル | 役割 |
|---|---|
| `AGENTS.md` | 共通ルールの正本（全エージェントが読む） |
| `CLAUDE.md` / `GEMINI.md` | 各ツールへの橋渡し（`@AGENTS.md` の取り込みのみ） |
| `.gemini/settings.json` | Gemini CLI にも `AGENTS.md` を読ませる設定 |
| `PROJECT.md` | プロジェクト固有情報の置き場（雛形） |
| `docs/WORKFLOW.md` | 場所を移りながら進める運用の詳細 |
| `templates/` | 仕様・レビュー・受け渡しの雛形 |
| `.github/` | Issue / PR テンプレート |
