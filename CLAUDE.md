# CLAUDE.md — openclaw_setup プロジェクト専用指示

## このプロジェクトについて

OpenClaw（セルフホスト型 AI アシスタント・ゲートウェイ）をこのMacに導入し、
運用に乗せるまでの作業記録。運用が安定したら pc_docs にマニュアル化して役割終了。

- OpenClaw 公式: <https://openclaw.ai/> / <https://docs.openclaw.ai/> / <https://github.com/openclaw/openclaw>
- OpenClaw は 2026年1月末に改称された新しめのソフト。手順は**必ず公式の最新版と照合**する。
  本リポジトリの記述で「要確認」とある箇所は特に鵜呑みにしない。

## 作業ルール

- **破壊的操作・外部実行の前に確認**:
  - `curl … | bash` はファイルに落として中身を読んでから実行（`less` で確認）。
  - グローバル npm インストール、`--install-daemon`（launchd 登録）、`brew upgrade node`、
    nvm の default 変更 は事前にユーザーへ相談。
- **秘密情報を平文で置かない**: Bot トークン・APIキーは `~/.openclaw/openclaw.json` 側に置き、
  本リポジトリの md には「取得先」と「設定した事実」だけ書く。設定ファイルを貼る時はマスク。
- **進捗は必ず [PROGRESS.md](PROGRESS.md) に追記**（日付・やったこと・結果・次アクション）。
  詰まったら [TROUBLESHOOTING.md](TROUBLESHOOTING.md)、設定が判明したら [CONFIG.md](CONFIG.md)。
- 既存の `~/projects` 群には影響を与えない（このプロジェクトは独立）。

## 環境メモ（2026-08-29）

- macOS 26.6.2 / Node v25.9.0（Homebrew）/ npm 11.12.1 / nvm あり
- 使えるモデル: Claude Code CLI（サブスク）/ Ollama（ローカル）。Anthropic API は残高不足で不可。

## ドキュメント構成

| ファイル | 役割 |
|---|---|
| `README.md` | 概要・OpenClaw とは・全体の段取り・前提環境 |
| `INSTALL.md` | インストール手順（前提確認〜疎通確認〜チェックリスト） |
| `PROGRESS.md` | 進捗ログ（最新が上）＋未解決課題リスト |
| `CONFIG.md` | 設定ファイル・ポート・モデルプロバイダ・デーモンの記録 |
| `CHANNELS.md` | チャットアプリ連携の手順と記録 |
| `TROUBLESHOOTING.md` | 詰まりどころと対処 |

## Git

- 独立リポジトリにするかは運用開始後に判断（`git init` はユーザー確認後）。
- `.openclaw/` 実体や秘密情報はコミットしない。
