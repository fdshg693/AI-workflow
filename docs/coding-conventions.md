---
type: Repo Convention
title: workflow/ のコーディング規約
description: "Coding conventions for the workflow/ submodule, where almost everything drives AI agents: models, providers and other non-secret run settings live in one config file per workflow (config.toml), only config.example.toml is committed and the real file is gitignored; secrets stay in .env with a committed .env.example; code refers to models by catalog alias and never hard-codes model ids; plus rules for fail-fast config validation, runaway limits (turns, timeouts, cost), least-privilege permissions, prompt placement, structured output, logging, state, and tests that never hit real APIs. Use when adding or changing a workflow under workflow/, adding a model or provider, or reviewing code here."
tags: [workflow, conventions, config]
generated: { by: claude-code/claude-opus-5-5, at: 2026-11-14T00:00:00Z }
status: draft
---

# workflow/ のコーディング規約

`workflow/` のコードはほぼすべて AI エージェントを動かす。モデルの世代交代、プロバイダの切り替え、API キーの管理、暴走やコストの抑止が繰り返し問題になるので、それをコードの書き方で防ぐための規約をまとめる。

設定の読み込み順・`.env` の扱いなど、アンブレラ全体の共通ルールは [tool-conventions](../_umbrella/maintenance-docs/tool-conventions.md) に従う。この文書はその上に `workflow/` 固有のルールを足す。食い違うときはこの文書を優先する。

## 1. 設定ファイルでモデル・プロバイダを一元管理する

### 規則

1. **ワークフローごとに設定ファイルを 1 つ持つ。** 名前は `config.toml`。置き場所はワークフローのディレクトリ直下（例: `cursor-wrapper/config.toml`、`schedule/issue-agent/config.toml`）。
2. **Git 管理するのは見本の `config.example.toml` だけ。** 実ファイル `config.toml` は gitignore する（ルートの [`.gitignore`](../.gitignore) で一括指定済み）。見本はそのまま `cp` すれば動く値で埋め、各キーにコメントで意味・選べる値・既定値を書く。
3. **モデル ID・プロバイダ・エンドポイントはコードに直書きしない。** `MODEL = "gpt-..."` のような定数や、`os.environ.get("X_MODEL", "sonnet")` のフォールバック値も直書きに当たる。値は `config.toml` にだけ書き、コードは設定から受け取る。
4. **コードやラベル・TODO ファイルからは、カタログのエイリアスで指す。** 実 ID（`claude-opus-5-5` など）への対応は設定ファイルの `[models.<alias>]` に書く。モデルを差し替えるときは設定ファイル 1 か所を直せば済むようにする。
5. **既定モデルは設定ファイルの `default` で決める。** コード側に既定モデルを持たない。
6. **秘密情報は設定ファイルに書かない。** API キー・トークン・秘密鍵のパスは `.env`（見本は `.env.example`）か環境変数から読む。`config.toml` に秘密情報らしきキー（`*_key`、`*_token`、`*_secret`）があれば読み込み時にエラーで止める。
7. **`config.toml` が無いときは、黙って見本を読まずにエラーで止める。** メッセージには探したパスと `cp config.example.toml config.toml` の手順を出す。

### 設定の分け方

| 置き場所                     | 書くもの                                                                              | Git                        |
| ---------------------------- | ------------------------------------------------------------------------------------- | -------------------------- |
| `config.toml`                | モデルカタログ・既定モデル・プロバイダ・ベース URL・上限値（ターン数・タイムアウト等） | 無視（見本だけ管理）       |
| `.env`                       | API キー・トークンなど秘密情報だけ                                                    | 無視（`.env.example` を管理） |
| 環境変数・CLI フラグ         | 一時的な上書き                                                                        | ―                          |

解決順は **CLI フラグ → 環境変数 → `config.toml`**。コード内の既定値は、モデル以外の上限値（タイムアウト等）に限って持ってよい。

### 見本の形

```toml
# config.example.toml — cp config.example.toml config.toml して使う。
# config.toml は gitignore 済み。秘密情報はここに書かず .env へ。

[models]
default = "standard"          # 下の [models.<alias>] のどれか

[models.fast]
provider = "anthropic"
id = "claude-haiku-5-5"
note = "要約・分類など軽い処理"

[models.standard]
provider = "anthropic"
id = "claude-sonnet-5-5"
note = "通常のコーディングタスク"

[models.deep]
provider = "anthropic"
id = "claude-opus-5-5"
note = "設計・難しいデバッグ。コスト高"

[providers.anthropic]
api_key_env = "ANTHROPIC_API_KEY"   # キー本体ではなく、読む環境変数名を書く

[limits]
max_turns = 30
timeout_seconds = 1800
max_budget_usd = 5.0
```

- `provider` は `[providers.<name>]` のどれかを指す。ベース URL・API の種類（`wire_api` 等）・キーを読む環境変数名はプロバイダ側に書く。
- 実 ID がアカウントや時期で変わるもの（SDK の `models.list()` で解決するもの等）は、カタログに要求する ID を書き、実際に解決した ID はログに残す（後述）。

### 実装の目安

- 設定の読み込みはワークフローごとに 1 モジュール（`config.py` / `lib/config.mjs`）にまとめ、型付きのオブジェクト（`dataclass` 等）を返す。ほかのモジュールは環境変数や TOML を直接読まない。
- Python は標準の `tomllib`、Node は `smol-toml` を使う。
- 複数のスクリプトが同じカタログを使うなら、ワークフロー内で 1 ファイルを共有する（例: `claude-wrapper/` 配下は `claude-wrapper/config.toml` を 1 つ）。

## 2. 設定は起動時に検証して早く落とす

- 未知のキー、型違い、カタログに無いエイリアス、`default` の指し先が無い、を読み込み時にエラーにする。エラーには設定ファイルの絶対パスとキー名を入れる。
- 必須の環境変数が無いときは、探した場所（環境変数名・`.env` の絶対パス）を出す（[tool-conventions](../_umbrella/maintenance-docs/tool-conventions.md) の規則 5）。
- エージェントが返したモデル指定（TODO の `model` 等）も、使う前にカタログで検証する。

## 3. 暴走とコストに上限を付ける

- エージェントを呼ぶ箇所には必ず **ターン数・タイムアウト** の上限を付ける。SDK が対応していれば **予算（USD・トークン）** の上限も付ける。値は `config.toml` の `[limits]` から読む。
- ループ・定期実行するものは、1 サイクルで処理する件数の上限を持つ（`max_issues_per_cycle` 等）。
- 同じ入力を何度も処理しないよう、試行の記録を持つ（`sandbox` / `issue-agent` の attempt store と同じ考え方）。

## 4. 権限は最小にする

- ツール許可は allowlist で明示する。`bypassPermissions` やそれに相当するモードは、使い捨てコンテナや専用 worktree の中でだけ使う。
- 書き込み可能な範囲（作業ディレクトリ）を明示し、ホームディレクトリやリポジトリ全体を渡さない。
- アプリが管理するファイル（`todo.json` 等）をエージェントに直接書かせない。結果は決まった形で返させ、アプリが検証してから書く。
- 外部から来る入力（ISSUE 本文・コメント）でエージェントを起動するときは、実行してよい投稿者を制限する。

## 5. プロンプトはコードから分ける

- プロンプト本文はロジックに埋め込まず、`prompts.py` か `prompts/*.md` にまとめる。差し込む値はテンプレート変数にする。
- プロンプトにモデル ID を書かない。モデル一覧を渡す必要があるときは、カタログから生成する（`claude-wrapper/util/models.py` の `prompt_block()` が例）。

## 6. エージェントの出力は構造化して検証する

- 後続の処理で使う結果は、自由文を正規表現で拾わず、JSON Schema・構造化出力・専用ツール呼び出し（「完了」ツール等）で受け取る。
- 受け取った値はスキーマで検証し、失敗したら再試行するか明示的に失敗扱いにする。黙って既定値で埋めない。

## 7. ログと状態

- 1 回の実行ごとに、**実際に使ったモデル ID（解決後）・プロバイダ・セッション ID・ターン数・コスト／トークン数・終了理由** をログに残す。
- ログに API キーやトークンを出さない。設定の表示でも秘密情報は有無だけを出す。
- ログは `logs/`、状態（SQLite 等）は `data/` に置く。どちらも gitignore 済み。相対パスはワークフローのディレクトリ基準で解決する。

## 8. テスト

- 単体テストは実 API を呼ばない。SDK の呼び出しは差し替え可能にし（関数を引数で受ける等）、テストではスタブを渡す。
- 設定の読み込みテストは、一時ディレクトリに書いた TOML を明示パスで読む。開発者の `config.toml` や `.env` に依存させない。
- `config.example.toml` がそのまま読み込めて検証を通ることをテストする（見本の腐敗を防ぐ）。
- 実 API を使う確認が要るときは別のテスト（`pytest -m live` 等）に分け、既定では実行しない。

## 9. ディレクトリとドキュメント

- ワークフローごとに `AGENTS.md`（または `README.md`）と、`@./AGENTS.md` だけを書いた `CLAUDE.md` を置く。
- `AGENTS.md` には、セットアップ手順（`cp config.example.toml config.toml`、`cp .env.example .env`）、実行コマンド、設定キーの説明への導線を書く。設定キーの詳細は見本ファイルのコメントを正とし、二重に書かない。
- 新しいワークフローを足したら [README](../README.md) の一覧に追加する。

## 既存コードの移行

この規約より前に書かれたワークフローは、次に手を入れるときに揃える。2026-11-14 時点で規約と食い違っている箇所:

| ワークフロー                  | 現状                                                                                   | 揃え方                                                                           |
| ----------------------------- | -------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------- |
| `claude-wrapper`              | カタログ `util/models.json` が gitignore されているのに見本が無く、clone 直後に動かない | `config.toml`（`[models]`）へ移し `config.example.toml` を管理する                |
| `codex-wrapper`               | `main.py` に `MODEL` を直書き                                                          | `config.toml` を新設                                                             |
| `cursor-wrapper`              | `main.py` に `REQUESTED_MODEL` とエイリアス表を直書き                                  | カタログを `config.toml` へ。解決後の ID をログに出す                            |
| `cline-wrapper`               | `lib/config.mjs` に既定プロバイダ・モデルを直書き                                      | `config.toml` を新設し `lib/config.mjs` で読む                                   |
| `copilot-wrapper`             | モデル・プロバイダ・ベース URL を `.env` で指定                                         | 秘密情報以外を `config.toml` へ移す                                              |
| `sandbox/orchestrator`        | `SANDBOX_MODEL` の既定 `"sonnet"` をコードに持つ                                       | 既定を `config.toml` へ                                                          |
| `schedule/issue-agent`        | `dispatch.py` に既定エイリアスと許可エイリアス一覧を直書き                             | `model:` ラベルはカタログのエイリアスで検証する                                  |
| `schedule/ai-schedule`        | 設定は `config/jobs.yaml`（見本あり）。モデルは扱わない                               | 対象外（既存の YAML のままでよい）                                               |
