# workflow/

単なる CLI 利用を超えた、手元で組み替えて使うカスタマイズ可能なワークフローの置き場。AI コーディングツールの CLI ラッパー、サンドボックス、スケジュール実行などを置く。

`tools/` の配布 CLI と違い、ここに置くものは `registry/repo-tools.yaml`・`ai-tools.yaml` に登録せず、付属スキル（`_plugin`）も持たない。

## 中身

移行前は `tools/` にあった（AI-tools から履歴を移さずにコピー）。ディレクトリの構成と深さは `tools/` の時と同じ。

### AI コーディングツールの CLI ラッパー

- [claude-wrapper](claude-wrapper/AGENTS.md) — Claude Agent SDK のラッパーと、TODO 駆動の反復ランナー
- [cline-wrapper](cline-wrapper/AGENTS.md)
- [codex-wrapper](codex-wrapper/AGENTS.md)
- [copilot-wrapper](copilot-wrapper/AGENTS.md)
- [cursor-wrapper](cursor-wrapper/AGENTS.md)

### エージェントの自動実行

- [sandbox](sandbox/README.md) — GitHub ISSUE のラベルを検知し、Docker の使い捨てコンテナでエージェントを動かす
- [schedule/ai-schedule](schedule/ai-schedule/README.md) — Windows タスクスケジューラにジョブを登録する
- [schedule/issue-agent](schedule/issue-agent/README.md) — `ai-schedule` から定期実行し、git worktree 上でエージェントに ISSUE を処理させる
