@./README.md

## このフォルダだけを開いたとき

アンブレラ（fdshg693/AI）の `workflow/` として clone されている前提。共通の手順（Junction・フックの作り方、`_umbrella/` の扱い、2 段階コミット）は [repo-layout](../maintenance/docs/repo-layout.md#サブモジュールを単体で開く)。

- `uv run`・pnpm はアンブレラの workspace を使う。このフォルダ直下に `pyproject.toml`・`package.json`・`pnpm-workspace.yaml` を置かない。単体 clone ではワークフローごとに `uv run --project <name>` で動かす。
- `.env`・秘密鍵などのローカル設定はワークフローごとのディレクトリに置く（gitignore 済み）。
