# OpenClaw セルフホスト運用マニュアル

このMacに **OpenClaw**（セルフホスト型 AI アシスタント・ゲートウェイ）を導入し、
Slack から会話でき、気象・経済・AI の定期ダイジェストを Slack DM に自動配信する構成。

- 導入作業の詳細ログ: `~/projects/openclaw_setup/`（README / INSTALL / CONFIG / CHANNELS / PROGRESS / TROUBLESHOOTING）
  GitHub: <https://github.com/masauehr/openclaw_setup>（public）
- OpenClaw 公式: <https://openclaw.ai/> / <https://docs.openclaw.ai/>
- 導入日: 2026-08-29 / このマニュアル初版: 2026-08-30

---

## 全体像

```
Slack（ワークスペース openclaw-test）
   │  Socket Mode（xapp- / xoxb-）
   ▼
OpenClaw Gateway  ── launchd: ai.openclaw.gateway
   127.0.0.1:18789 / token auth / mode=local
   ├─ 対話モデル : ollama/qwen3.6:35b-mlx（ローカル・無料）  ← Ollama http://localhost:11434
   ├─ cron モデル : anthropic/claude-sonnet-5（Claude Pro サブスク、setup-token）
   └─ web_search : SearXNG  ── launchd: local.searxng
                     127.0.0.1:18899（自己ホスト・キー不要・上限なし）
```

- **すべて Mac 起動中のみ稼働**（スリープすると停止）。常用するなら `caffeinate` 等でスリープ抑止。

---

## コンポーネントとパス

### OpenClaw 本体

| 項目 | 値 |
|---|---|
| インストール | npm グローバル `openclaw@2026.7.1-2`（`/opt/homebrew/lib/node_modules/openclaw` → `/opt/homebrew/bin/openclaw`） |
| 設定ファイル | `~/.openclaw/openclaw.json`（`gateway.auth.token` は平文。共有禁止） |
| ワークスペース | `~/.openclaw/workspace/`（`SOUL.md` 等。git 管理下） |
| state DB | `~/.openclaw/state/openclaw.sqlite` |
| launchd | `~/Library/LaunchAgents/ai.openclaw.gateway.plist`（Label `ai.openclaw.gateway`、RunAtLoad/KeepAlive） |
| ログ | `~/Library/Logs/openclaw/gateway.log` |
| Gateway | `http://127.0.0.1:18789/`（ダッシュボード。`openclaw dashboard` でトークン付きURL） |

### エージェント（人格）

- 名前「クゥ」/ 黒猫 / 🐈‍⬛（`~/.openclaw/workspace/IDENTITY.md`）
- **応答言語は日本語固定**（`~/.openclaw/workspace/SOUL.md` の「## Language」セクション）
- オーナー: `commands.ownerAllowFrom = ["slack:U0XXXXXXXXX"]`（自分の Slack メンバーID）

### モデル

| 用途 | モデル | 認証 |
|---|---|---|
| 対話（Slack DM でクゥと会話） | `ollama/qwen3.6:35b-mlx`（fallback `ollama/qwen3.8:27b-mlx`） | 不要（ローカル） |
| cron ダイジェスト | `anthropic/claude-sonnet-5`（fallback `anthropic/claude-haiku-4-5`） | `claude setup-token` → `openclaw models auth login --provider anthropic`（「Anthropic setup-token」を選択） |

- 対話モデル選定の経緯: `ornith-1.5:35b` は「No response requested.」定型句を返す不具合、
  `nemotron-3.5-lightning:30b-mlx` はツールを大量ループして空応答。`qwen3.6:35b-mlx` が最良だった。
- ローカルモデルの `num_ctx` はモデル既定（qwen3.6 は 262144）だと KV キャッシュ肥大＋prompt処理が遅い。
  `agents.defaults.models.<ref>.params = { num_ctx: 32768, keep_alive: "60m" }` に設定。
  会話が長くなったら `openclaw sessions compact "<key>" --max-lines N` で圧縮。
- cron を Claude にした理由: ローカルモデルだと 1ラン 4〜16分・出力不安定・同時起動で AbortError が頻発。
  Claude だと 36〜90秒で安定。

### Slack チャンネル

| 項目 | 値 |
|---|---|
| プラグイン | `@openclaw/slack`（ClawHub、`~/.openclaw/extensions/slack`） |
| ワークスペース | `openclaw-test`（テスト用に新規作成。既存コミュニティは使わない） |
| App | 「OpenClaw」/ bot 表示名 `openclaw` / Socket Mode |
| トークン | Bot `xoxb-` + App-Level `xapp-`（`connections:write`）。`~/.openclaw/openclaw.json` の `channels.slack.*` に平文保存 |
| DM ポリシー | 既定 = pairing（承認制）。`openclaw pairing approve slack <code>` で承認済み |

### SearXNG（web_search バックエンド）

Docker 不使用の native 構成。DuckDuckGo（キー無し）がボット判定で不安定なため導入。

| 項目 | 値 |
|---|---|
| ソース | `~/searxng/src`（`git clone --depth 1 github.com/searxng/searxng`） |
| Python 環境 | `~/searxng/.venv`（`uv venv --python 3.12` の隔離 CPython。system/brew 非依存） |
| 導入手順 | `uv pip install -r requirements.txt -r requirements-server.txt` → `uv pip install -e . --no-build-isolation`（※`setup.py` が `msgspec` を import するため no-build-isolation 必須） |
| 設定 | `~/searxng/settings.yml`（`chmod 600`。`secret_key` あり＝コミット禁止。`127.0.0.1:18899` / `search.formats:[html,json]` / `limiter:false`） |
| 起動スクリプト | `~/searxng/run.sh` |
| launchd | `~/Library/LaunchAgents/local.searxng.plist`（Label `local.searxng`、RunAtLoad/KeepAlive） |
| ログ | `~/searxng/logs/searxng.log` / `searxng.err.log` |
| OpenClaw 側設定 | `tools.web.search.provider = "searxng"` ＋ `plugins.entries.searxng.config.webSearch.baseUrl = "http://127.0.0.1:18899"`（categories=general,news / language=ja）。プラグイン `@openclaw/searxng-plugin` |

- 一部エンジン（wikidata / ahmia 等）は init に失敗するがコア検索は動作（非致命）。

---

## 定期ダイジェスト（OpenClaw cron）

3ジョブ。すべて model=`anthropic/claude-sonnet-5` / fallback `anthropic/claude-haiku-4-5` /
`--channel slack --to slack:U0XXXXXXXXX --announce --expect-final --tz Asia/Tokyo --timeout-seconds 1200`。
実行時刻を数分ずつずらして同時実行を回避。

| id（先頭） | name | スケジュール(JST) | 内容 |
|---|---|---|---|
| `445881d3` | `weather-jp-2x` | 毎日 09:00 / 18:00 | 日本の気象・防災（警報注意報・災害・台風・地震・津波・見通し） |
| `b18e0f97` | `econ-daily-2x` | 毎日 09:05 / 18:05 | 日本株（日経・TOPIX）・為替・世界の経済指標/イベント |
| `db8a0d0b` | `ai-weekly-digest` | 毎週月曜 09:10 | 週次 AI 動向ダイジェスト（幅広く俯瞰） |

- cron の `--model` override は `agents.defaults.models` allowlist に登録済みのモデルしか使えない。
  未登録だと即エラー `payload.model '…' rejected by agents.defaults.models allowlist`。
  追加: `printf '%s' '{ "agents": { "defaults": { "models": { "<ref>": {} } } } }' | openclaw config patch --stdin`

---

## 運用コマンド早見表

```bash
# --- Gateway / デーモン ---
openclaw gateway status              # ランタイム状態（pid, listening, capability）
openclaw daemon restart              # 設定変更の反映に必要
openclaw daemon status | stop | start
openclaw doctor                      # 健康診断（--fix で自動修正提案）
tail -f ~/Library/Logs/openclaw/gateway.log

# --- モデル ---
openclaw models status               # 既定/fallback/認証
openclaw models list [--provider ollama|anthropic] [--all]
openclaw models set <provider/model> # 既定モデル変更
openclaw models auth login --provider anthropic   # 対話メニューで setup-token

# --- チャンネル ---
openclaw channels status [--deep] [--probe]
openclaw channels logs
openclaw pairing approve slack <code>

# --- cron ---
openclaw cron list
openclaw cron get <id>                       # JSON 全体
openclaw cron edit <id> --model <ref> --fallbacks <ref> --cron "<expr>" --tz Asia/Tokyo --timeout-seconds <n>
openclaw cron run <id> --wait --expect-final # 手動テスト実行
openclaw cron runs --id <id>                 # 実行履歴（entries[].model / status を点検）
openclaw cron disable | enable | rm <id>

# --- プラグイン ---
openclaw plugins list [--json]
openclaw plugins install clawhub:@openclaw/<name>
openclaw plugins enable <id>

# --- SearXNG ---
launchctl list | grep searxng
launchctl kickstart -k gui/$(id -u)/local.searxng    # 再起動
launchctl bootout   gui/$(id -u)/local.searxng       # 停止・解除
# JSON API 確認（curl 不使用）:
python3 -c "import urllib.request,urllib.parse,json;print(json.loads(urllib.request.urlopen('http://127.0.0.1:18899/search?'+urllib.parse.urlencode({'q':'test','format':'json'})).read()).get('results',[])[:1])"
# 更新: cd ~/searxng/src && git pull && VIRTUAL_ENV=~/searxng/.venv uv pip install -e . --no-build-isolation && launchctl kickstart -k gui/$(id -u)/local.searxng
```

---

## トラブルシューティング（要点）

| 症状 | 原因 / 対処 |
|---|---|
| Slack で「No response requested.」しか返らない | 対話モデルの誤発火。`openclaw models set ollama/qwen3.6:35b-mlx` |
| bot が pairing コードを返す | 正常（DM 承認制）。`openclaw pairing approve slack <code>` |
| Slack DM 入力欄が「送信オフ」 | App Home → Show Tabs → Messages Tab を On ＋「Allow users to send…」にチェック |
| Socket Mode `not_allowed_token_type` | `xoxb-`/`xapp-` の取り違え。`xapp-` は Basic Information → App-Level Tokens（`connections:write`） |
| cron が `AbortError` | 同時起動 or ローカルモデルが重い。スケジュールをずらす／Claude モデルに |
| cron が `rejected by agents.defaults.models allowlist` | `openclaw config patch` でモデルを allowlist に追加 |
| `models auth login` が `No provider plugins found` | `plugins.allow` の排他リストが原因。`openclaw config unset plugins.allow` ＋ `openclaw plugins enable anthropic` |
| web_search が `DuckDuckGo returned a bot-detection challenge` | SearXNG へ移行済み。SearXNG が落ちていれば `launchctl kickstart -k gui/$(id -u)/local.searxng` |
| cron の model がいつのまにか ollama に戻る | 原因未特定・要監視。`openclaw cron get <id>` で確認し `--model` 再設定 |
| クゥの気象ランキングに数値(mm/℃)が出ない・地点名が化ける | HTMLスクレイプ＋偽MCP＋幻覚。ランキングは **アメダス map JSON を自前集計**（`amedas/data/map/{stamp}.json` の `precipitation1h/3h/24h` を `amedastable.json` の `kjName` と結合してソート）。ルールは `~/.openclaw/workspace/TOOLS.md` に記載 |
| クゥの簡単な質問への応答が数分かかる | Slack DM セッションの履歴肥大（~87k tokens）。`openclaw sessions compact "agent:main:slack:direct:<uid>" --max-lines 8` ＋ `num_ctx` を 32768 に縮小（`agents.defaults.models.<ref>.params`）＋ `compaction.mode:safeguard`。1〜3秒に短縮 |

詳細は `~/projects/openclaw_setup/TROUBLESHOOTING.md`。

---

## 既知の制約

- Mac がスリープすると OpenClaw も SearXNG も止まる。スマホから使うなら起動維持が必要。
- 対話は無料（ローカル）だが、cron ダイジェストは Claude Pro のサブスク枠を消費（1日約5ラン）。
- `~/.openclaw/openclaw.json`（Gateway token・Slack token）と `~/searxng/settings.yml`（secret_key）は平文。
  いずれもリポジトリ外。SecretRef 化は未対応。
- 秘密情報は `openclaw_setup` リポジトリにコミットしない（`.gitignore` 済み）。

---

## アンインストール / やり直し

```bash
# OpenClaw
openclaw daemon uninstall
npm uninstall -g openclaw
rm -rf ~/.openclaw
rm ~/Library/LaunchAgents/ai.openclaw.gateway.plist

# SearXNG
launchctl bootout gui/$(id -u)/local.searxng
rm ~/Library/LaunchAgents/local.searxng.plist
rm -rf ~/searxng
```
