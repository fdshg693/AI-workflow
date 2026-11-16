@./README.md

## 崩してはいけない前提

このプロジェクトは「人間が自然言語で書いたテストストーリー → AIがブラウザを操作 → 再実行可能なPlaywrightテストコード」を自動生成する仕組み。移設元にあった設計書（`SPEC.md`）は持ち込んでいない。設計の正はこの節で、ロードマップや実装詳細の文書に残る「元設計書（未収録）N章」はその名残（要点はこの節と各ロードマップに書いてある）。

- **ブラウザセッションは永続、AIコンテキストはステップごとに初期化**という2つを混同しないこと。サーバー（またはそれに相当するコード）側がセッションを1本維持しつつ、AI呼び出しのたびに `snapshot` を取り直して渡すことで「フレッシュなコンテキストでも今の画面がわかる」状態を作る。
- **1タスク＝1ステップ**が粒度の単位。「1シナリオ＝1セッション」という playwright-cli の plan→generate→heal のセッション単位と、AIコンテキストを初期化する単位は別物として扱う。
- 各ステップへの入力は「残りのストーリー」と「現在の画面snapshot」だけに絞る。過去の操作履歴に依存しないと解けないシナリオはスコープ外という設計上の割り切り。
- 安全性・認証は元々スコープ外。Step8 で入れたのは URL 許可リスト（`ALLOWED_DOMAINS`）・最大同時セッション数（`MAX_CONCURRENT_SESSIONS`）・アイドルセッションタイムアウト（`IDLE_SESSION_TIMEOUT_SECONDS`）の最低限の3つだけ（[docs/roadmap/08-safety-guardrails.md](docs/roadmap/08-safety-guardrails.md)）。

### Playwright CLI を選んだ理由

Playwright MCP ではなく playwright-cli を採用している。MCPは閉じたループの中で完結するには楽だが、その中間にタスク管理・記録・再開制御などの独自ロジックを挟もうとするとMCPの作法に合わせるアダプトコストが発生するため。CLIは薄いプロセス呼び出しなので独自ロジックを挟むコストが小さい。コマンドの仕様は `using-playwright-cli` スキル（`skills/plugins/browser/`）を読んでから使う。

## 構成

- `scripts/vertical_slice/` — 1ストーリーを1プロセスで最初から最後まで実行するランナー（Step1・4〜6・8）。詳細は [docs/vertical-slice-runner.md](docs/vertical-slice-runner.md)
- `scripts/server/` — セッションを維持する FastAPI サーバー（Step2・3・7・8）。詳細は [docs/session-server.md](docs/session-server.md)
- `scripts/internal/` — コスト集計（下記）
- `scripts/stories/` — ストーリー YAML。書式と落とし穴は [scripts/stories/README.md](scripts/stories/README.md)
- `tests/generated/` — 生成物（`.spec.ts`・run history・録画・failure notes）。gitignore 済み

## 実行と課金

- コマンドは `justfile`（このディレクトリを cwd にして実行する。`python -m scripts...` がトップレベルの `scripts` を cwd から解決するため）。アンブレラのルートからは `just --justfile workflow/playwright-app/justfile <recipe>`。
- **`just slice`・`just resume`・サーバーの `POST /sessions/{id}/run` は実課金の AI API 呼び出し**。エージェントが動作確認のために実行するときは、実行前に必ずユーザーへ確認する。課金なしで確認できるのは `just server` の起動と `/snapshot`・`/command`、`just test`（生成済みテストの再実行）、`just cost`。
- 実 AI での完走確認の手順: (1) アンブレラのルートで `just test-sites` を起動（`localhost:8080` を使うストーリーの場合）、(2) ユーザーの了承を得て `just slice scripts/stories/<name>.yaml`、(3) `tests/generated/<name>.spec.ts` が生成され自動実行の `npx playwright test` が通ったこと、`<name>.failure-notes.json` の有無、`just cost tests/generated/<name>.history/` のコストを報告する。
- `.env` はこのディレクトリの `.env` に固定（`Path(__file__)` 基準。cwd に依らない）。`.env.example` をコピーして `OPENAI_API_KEY` を入れる。
- テストサイトはアンブレラの `tools/browser/test-sites/` にある（`just test-sites` で `http://localhost:8080`）。このサブモジュールだけを開いたときは使えない。ストーリーの `seed_url`（`http://localhost:8080/...`）はその配信位置に合わせてあるので変えない。

## コスト集計（scripts/internal/）

- `cost_summary.py`はCLIエントリポイントのみ（argparse配線）。実処理は3モジュールに分割されている: `cost_log_discovery.py`（ファイル/ディレクトリ引数の解決・時間範囲フィルタ）、`cost_aggregate.py`（`<out>.steps.jsonl`のusage集計・価格計算）、`cost_report.py`（出力整形）。
- 引数を1ファイルだけ渡すとそのファイルのステップ内訳（`format_single_file_report`）を出す。複数ファイル/ディレクトリを渡すと実行ごとの行（時刻昇順）＋末尾に累計コストの「run history」表（`format_run_history_report`）に切り替わる。
- 複数ファイルの累計コストは、トークンを合算してから1回課金するのではなく、ファイルごとに自分のモデルで価格計算した金額（ドル）を合算する（ファイルによってモデルが異なりうるため）。
- ディレクトリを渡すと配下の`*.steps.jsonl`のみ再帰的に対象にする（`*.tasks.jsonl`は`usage`フィールドを持たないため対象外）。
- 実行時刻は`scripts/vertical_slice/run_id.py`の`parse_run_id_prefix`でファイル名プレフィックス（`{run_id}__{stem}`）から復元し、無ければファイルの`mtime`にフォールバックする（run history表に`(mtime fallback)`と注記される）。
- 壊れた/空のJSON行はファイル単位でスキップし警告を`stderr`に出す（`cost_aggregate.load_usages`）。usageが1件も無いファイルやモデルが一意に決まらないファイルは、複数ファイルモードでは警告を出してそのファイルだけスキップし、他ファイルの集計は継続する（単一ファイルモードでは従来どおりエラー終了する）。
- 該当ファイルが0件（ディレクトリにマッチ無し／時間範囲で全滅／全ファイルがusage無しでスキップ）の場合はエラーにせず「0件でした」という旨のメッセージを出して正常終了する。
- `--html [PATH]`（`nargs="?"`＋`const="cost_dashboard.html"`）を指定すると、run-history集計結果（`RunCostRow`のリスト）を`cost_html.py`の`render_html_report`でHTML化し、指定パス（省略時は`cost_dashboard.html`）へ書き込む。標準出力には書き込み先パスと概要1行のみ出す。`--html`指定時はファイルが1件しかなくても常にrun-history経路（`rows`のリスト）を使い、単一ファイル用のステップ内訳（`format_single_file_report`）は使わない。生成されるHTMLは外部JS/CSSライブラリ・CDN参照の無い自己完結ファイル（インラインCSS、インラインSVGの棒グラフ）で、rows由来の文字列（パス・モデル名）は`html.escape`でエスケープ済み。`--html`未指定時の既存の出力（単一ファイルのステップ内訳・複数ファイルのrun-historyテキスト表）は変更されない。

## 文書の更新

- 機能を足したら `FEATURES.md`、未対応を見つけたら `NEXT.md`、サブシステムの実装詳細が変わったら `docs/vertical-slice-runner.md`・`docs/session-server.md` を更新する。
- 新しい大きな作業はアンブレラの `plans/` の運用で計画する（移設元の `.claude/plan/` の書式は使わない）。
