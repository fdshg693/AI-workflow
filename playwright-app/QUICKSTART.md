# QUICKSTART

最短で「ストーリーYAML → 実際にブラウザを操作するAI → 生成された `.spec.ts`」を1回動かすまでの手順。設計の前提は [AGENTS.md](AGENTS.md)、図解は [MERMAID.md](MERMAID.md)、現状の機能は [FEATURES.md](FEATURES.md) を参照。

> **注意**: 以下の `just slice` / `just resume` / サーバー経由の `/run` 実行は、いずれも実際に課金されるAI API呼び出しを伴う。実行前に必ず内容を確認すること（[AGENTS.md](AGENTS.md) 参照）。

## 0. 前提

- [uv](https://docs.astral.sh/uv/)（Python依存関係・実行）
- アンブレラ（fdshg693/AI）の `workflow/playwright-app` として clone されていること（uv・pnpm のワークスペースはアンブレラにある）
- [just](https://just.systems/)・[mise](https://mise.jdx.dev/)（node / pnpm / uv のバージョンはアンブレラの `mise.toml`）
- `playwright-cli` が `PATH` 上になければ、`npx --no-install playwright cli` にフォールバックする（`just setup` 後であれば動く）
- （自作テストサイトを使う場合のみ）アンブレラの `tools/browser/test-sites/`。起動はアンブレラのルートで `just test-sites`

以下のコマンドは `workflow/playwright-app` で実行する。アンブレラのルートからは `just --justfile workflow/playwright-app/justfile <recipe>`。

## 1. セットアップ

```bash
just setup            # uv sync + pnpm install（どちらもアンブレラのワークスペース）
just env              # .env を .env.example から作成（既存なら何もしない）
just install-browsers # Playwright用ブラウザバイナリをインストール
```

`.env` を開き、`OPENAI_API_KEY=` に自分のAPIキーを追記する（`sk-...`）。起動時に `load_dotenv()` がこのディレクトリの `.env`（cwd に依らず固定）を読むので、シェルで `export` する必要はない。必要なら `AI_MODEL=` でモデルを上書きできる（デフォルトは `scripts/vertical_slice/config.py` の `DEFAULT_MODEL`）。

## 2. （任意）自作テストページをローカル配信する

`scripts/stories/*.yaml` の一部（`custom-pages-demo.yaml`・`wizard-demo.yaml` 等）はアンブレラの `tools/browser/test-sites/custom_pages/pages/` を対象にしている。使う場合は別ターミナルで、アンブレラのルートから:

```bash
just test-sites   # テストサイトを http://localhost:8080 で配信（フォアグラウンド、Ctrl+Cで停止）
```

外部サイト（`playwright.dev` 等）だけを対象にした `search-demo.yaml` を試すだけならこの手順は不要。

## 3. 一番手軽な方法: CLIで1ストーリーを最初から最後まで実行する

サーバーを立てず、`scripts/vertical_slice/main.py` が1プロセスで完結させる（Step1のコアループ）。

```bash
just slice scripts/stories/search-demo.yaml
```

成功すると:

- `tests/generated/search-demo.spec.ts` — 生成されたPlaywrightテスト（自動で `npx playwright test` も実行される）
- `tests/generated/search-demo.history/` — 実行ごとの生ログ（`*.steps.jsonl`）とタスク単位の記録（`*.tasks.jsonl`）
- `tests/generated/search-demo.recordings/` — 各ステップ前後のスクリーンショット
- 途中で `blocked` になった場合は `tests/generated/search-demo.failure-notes.json` に理由が記録される

他のデモシナリオは `scripts/stories/` 配下を参照（`edge-*-demo.yaml` は意図的に失敗・blockedになる検証用フィクスチャ）。

## 4. 途中再開・分岐実行

過去の実行が残した `.tasks.jsonl` から、途中のステップまでを再現して続きを実行できる（Step5）。

```bash
just resume scripts/stories/search-demo-branch.yaml \
  tests/generated/search-demo.history/<run_id>__search-demo.tasks.jsonl 4
```

3 番目の引数（`before`）は「このID未満のステップは記録から復元する」という境界値。1 番目の `story` に分岐先だけを持つ別YAMLを渡せば、そこから別パターンで実行を分岐できる。

## 5. サーバー経由で動かす（HTTP API、Step2〜8全部入り）

セッションを複数リクエストにまたがって維持したい・外部から強制停止したい場合はこちら。

```bash
just server                      # 127.0.0.1:8000 で起動（just server <host> <port> で変更可）
```

別ターミナルから:

```bash
# セッション開始（対象URLへnavigation、ストーリーも紐付け）
curl -X POST localhost:8000/sessions \
  -H 'content-type: application/json' \
  -d '{"target_url": "https://playwright.dev", "story": "scripts/stories/search-demo.yaml"}'
# => {"session_id": "..."}

# 現在の画面snapshot（人間が直接確認したい場合）
curl localhost:8000/sessions/<session_id>/snapshot

# playwright-cliコマンドを1つ人間が直接叩く
curl -X POST localhost:8000/sessions/<session_id>/command \
  -H 'content-type: application/json' -d '{"command": "snapshot", "args": []}'

# ストーリーをAI主導で最後まで自動実行（実課金コール）
curl -X POST localhost:8000/sessions/<session_id>/run

# 実行中のセッションを強制停止（ステップ/リトライ境界で反映）
curl -X POST localhost:8000/sessions/<session_id>/stop

# セッションを閉じる（閉じ忘れるとアイドルタイムアウトで自動close）
curl -X DELETE localhost:8000/sessions/<session_id>
```

`ALLOWED_DOMAINS`（例: `ALLOWED_DOMAINS=localhost,*.example.com`）・`MAX_CONCURRENT_SESSIONS`・`IDLE_SESSION_TIMEOUT_SECONDS` の3つの環境変数で、Step8のガードレール（URL許可リスト・同時セッション数上限・アイドルタイムアウト）を調整できる（未設定時は無制限/デフォルト値）。

## 6. 生成されたテストの再実行

```bash
just test                 # pnpm exec playwright test（生成済みの tests/generated/*.spec.ts 全体）
just test tests/generated/search-demo.spec.ts
```

## 7. トークン消費・コストの確認

```bash
just cost                                    # tests/generated/ 配下の全run-historyをテキスト集計
just cost tests/generated/search-demo.history/
just cost-html                               # 自己完結HTMLダッシュボードを cost_dashboard.html に書き出す
just cost-html tests/generated/ report.html
```

## 8. 掃除

```bash
just clean   # __pycache__ / playwright-report / test-results を削除
```

## コマンド一覧

引数なしの `just`（`just --list`）でいつでも確認できる。
