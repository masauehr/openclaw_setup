# openclaw_setup

OpenClaw（セルフホスト型の個人用 AI アシスタント・ゲートウェイ）を
このMacに導入し、運用に乗せるまでの作業をまとめるプロジェクト。

- **運用マニュアル（現行構成の全体像）** → [MANUAL.md](MANUAL.md)
- **運用開始までの進捗ログ** → [PROGRESS.md](PROGRESS.md)
- **インストール手順** → [INSTALL.md](INSTALL.md)
- **設定リファレンス** → [CONFIG.md](CONFIG.md)
- **チャットアプリ連携** → [CHANNELS.md](CHANNELS.md)
- **トラブルシューティング** → [TROUBLESHOOTING.md](TROUBLESHOOTING.md)

> [MANUAL.md](MANUAL.md) は `pc_docs/manuals/automation/openclaw.md` のコピー（自己完結のため同梱）。両方を更新すること。

> ⚠️ このプロジェクトは 2026-08-29 開始。OpenClaw は 2026年1月末に現名称へ改称された
> 新しめのソフトで、公式ドキュメントの内容が変わりやすい。各手順は必ず
> <https://docs.openclaw.ai/> の最新版と照合すること。本リポジトリの記述で「要確認」と
> 付いている箇所は特に。

---

## OpenClaw とは

- チャットアプリ（Slack / Telegram / WhatsApp / Discord / Signal / iMessage / Google Chat など）を
  ローカルで動く AI コーディングエージェントに橋渡しする **セルフホスト型ゲートウェイ**。
- 1 つの「Gateway」プロセスを自分のマシンで動かし、会話をまたいでコンテキストを保持し、
  マシン上で実際に操作を実行できる。
- 完全オープンソース（MIT、OpenClaw Foundation という非営利団体が開発）。サブスク不要。
- 作者は Peter Steinberger。旧名 Warelay →（Anthropic の商標指摘で）Moltbot → OpenClaw。
- 参考: 公式サイト <https://openclaw.ai/> / GitHub <https://github.com/openclaw/openclaw>

## このプロジェクトで何をするか

1. 前提ツール（Node.js / npm）の確認と必要なら更新
2. OpenClaw のインストール（インストールスクリプト or npm グローバル）
3. `openclaw onboard` でオンボーディング（モデルアクセス確認・ワークスペース作成・デーモン登録）
4. Control UI ダッシュボードで疎通確認
5. チャットアプリを 1 つ接続して実運用へ
6. 手順が固まったら `pc_docs/manuals/automation/` にマニュアル化

## このMacの前提環境（2026-08-29 時点）

| 項目 | 値 | OpenClaw 要件 | 判定 |
|---|---|---|---|
| OS | macOS 26.6.2 (25G83) | macOS / Linux / WSL2 / Windows | ✅ |
| Node.js | v25.9.0（Homebrew: `/opt/homebrew/bin/node`） | 22.22.3+ / 24.15+ / 25.9+ | ✅ ぎりぎり満たす（要確認：25.9.0 ちょうどで可か） |
| npm | 11.12.1 | 11.15+ / 11.16+ / 12+（`--allow-scripts` 絡み） | ⚠️ 要更新の可能性（[INSTALL.md](INSTALL.md) 参照） |
| nvm | `~/.nvm` あり | — | 別 node が優先される場合あり。`which -a node` で確認 |

## ステータス

現在（2026-08-29）: **基本セットアップ完了・運用調整中**。詳細・スナップショットは [PROGRESS.md](PROGRESS.md) の「現在の到達点サマリ」。

- ✅ インストール（npm `openclaw@2026.7.1-2`）／`onboard`（`--auth-choice skip`）
- ✅ Gateway を launchd 常駐化（`ai.openclaw.gateway`、loopback:18789、token auth）
- ✅ モデル: ローカル `ollama/ornith-1.5:35b`（fallback `qwen3.8:27b-mlx`）
- ✅ Slack 接続（WS `openclaw-test`、health:healthy）、オーナー = `slack:U0XXXXXXXXX`
- ✅ 日本語を既定応答に（`SOUL.md`）／エージェント名「クゥ」🐈‍⬛
- ✅ 定期ダイジェスト cron 3本（気象/経済=毎日7:30台、AI=月曜7:40、Slack DM 配信）＋ DuckDuckGo 検索有効化
- ✅ Anthropic 追加（`claude setup-token`）。cron は `anthropic/claude-sonnet-5`、対話の既定は `ollama/qwen3.6:35b-mlx`
- ✅ Web検索は自己ホスト **SearXNG**（`127.0.0.1:18899`、launchd 常駐）。DuckDuckGo のボット判定ブロックを回避
- ✅ 対話の既定は `anthropic/claude-haiku-4-5`（ローカル勢は幻覚・中国語混入で断念、fallback は qwen3.6）。ランキングは `bin/jma_rank.py` を exec させる
- ⏳ スリープ運用、平文シークレット（Gateway token / SearXNG secret_key）の扱い、pc_docs マニュアル化
