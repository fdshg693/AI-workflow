# playwright-app — AI に画面テストを効率よくさせるためのツール

人間が自然言語で書いたテストストーリーを、AIがブラウザを操作しながら実行し、再実行可能なPlaywrightテストコード（`.spec.ts`）として生成するツール。[docs/roadmap/](docs/roadmap/)のStep1〜8（縦の一本通し〜最低限の安全策）まで実装済み。

Playwright の研究用クローンから移設した実験アプリ。ブラウザ操作は `playwright-cli`（使い方は `using-playwright-cli` スキル）、検証用のテストサイトはアンブレラの `tools/browser/test-sites/`。

## ドキュメント

| ドキュメント                                                   | 内容                                                       |
| -------------------------------------------------------------- | ---------------------------------------------------------- |
| [AGENTS.md](AGENTS.md)                                         | 設計の前提・規約・課金に関わる操作の扱い                   |
| [MERMAID.md](MERMAID.md)                                       | 全体の流れ・構成を図解                                     |
| [QUICKSTART.md](QUICKSTART.md)                                 | セットアップから実行までの最短手順、just コマンド一覧      |
| [FEATURES.md](FEATURES.md)                                     | 現状実装済みの機能一覧                                     |
| [NEXT.md](NEXT.md)                                             | あると良いがまだ無い機能・部分対応の機能                   |
| [docs/roadmap/](docs/roadmap/)                                 | 実装ロードマップ（何を・どの順で作るか。全 Step 実装済み） |
| [docs/vertical-slice-runner.md](docs/vertical-slice-runner.md) | `scripts/vertical_slice/` の実装詳細                       |
| [docs/session-server.md](docs/session-server.md)               | `scripts/server/` の実装詳細                               |

## はじめかた

```sh
just setup && just env
```

以降は [QUICKSTART.md](QUICKSTART.md) を参照。
