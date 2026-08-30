# OpenClaw 設定リファレンス

作成: 2026-08-29 ／ 実測更新: 2026-08-29（onboard 完了時）
OpenClaw バージョン: `2026.7.1-2 (0790d9f)`

---

## 実測サマリ（2026-08-29 onboard 完了）

| 項目 | 値 |
|---|---|
| 設定ファイル | `~/.openclaw/openclaw.json`（実体は JSON。`.bak` が自動生成される） |
| ワークスペース | `~/.openclaw/workspace` |
| エージェント | `main`（デフォルト）。dir: `~/.openclaw/agents/main/` |
| セッション | `~/.openclaw/agents/main/sessions/sessions.json` |
| 認証ストア | `~/.openclaw/agents/main/agent/openclaw-agent.sqlite` |
| state DB | `~/.openclaw/state/openclaw.sqlite`（+ `-shm` / `-wal`） |
| Gateway | mode=local / bind=loopback(127.0.0.1) / port=18789 / auth=token |
| Gateway トークン | `openclaw.json` の `gateway.auth.token` に**平文保存**（マスク管理。ここには載せない） |
| 既定モデル | `ollama/ornith-1.5:35b` |
| フォールバック | `ollama/qwen3.8:27b-mlx` |
| ダッシュボード | `http://127.0.0.1:18789/`（`openclaw dashboard` でトークン付きURLが開く） |
| ファイルログ | `/tmp/openclaw/openclaw-YYYY-MM-DD.log`（CLI）/ `~/Library/Logs/openclaw/gateway.log`（サービス） |

### 生成された `~/.openclaw/openclaw.json`（トークンはマスク）

```json
{
  "agents": { "defaults": {
    "workspace": "~/.openclaw/workspace",
    "model": {
      "primary": "ollama/ornith-1.5:35b",
      "fallbacks": ["ollama/qwen3.8:27b-mlx"]
    },
    "models": { "ollama/ornith-1.5:35b": {}, "ollama/qwen3.8:27b-mlx": {} }
  }},
  "gateway": {
    "mode": "local",
    "auth": { "mode": "token", "token": "***MASKED***" },
    "port": 18789,
    "bind": "loopback",
    "tailscale": { "mode": "off", "resetOnExit": false }
  },
  "session": { "dmScope": "per-channel-peer" },
  "tools": { "profile": "coding" },
  "skills": { "install": { "nodeManager": "npm" } },
  "hooks": { "internal": { "entries": { "session-memory": { "enabled": true } } } },
  "wizard": { "lastRunCommand": "onboard", "lastRunMode": "local", "lastRunVersion": "2026.7.1-2" }
}
```

> ⚠️ `gateway.auth.token` は平文。このファイルを共有・コミットしない（`.gitignore` 済み想定）。
> ローテーションは `openclaw configure --section gateway` か `openclaw config set gateway.auth.token <新値>` → `openclaw daemon restart`。

---

## 設定ファイル

| 項目 | 値（公式記載） | 備考 |
|---|---|---|
| パス | `~/.openclaw/openclaw.json` | |
| フォーマット | **JSON5**（コメント可・末尾カンマ可の JSON 拡張） | |
| ワークスペース | `~/.openclaw/` 配下（詳細要確認） | `rm -rf ~/.openclaw` でリセット |
| 何も書かない場合 | 同梱の「OpenClaw agent runtime」が使われる | |

設定できる項目（公式で言及があったもの）:
- チャンネルの allowlist（接続を許可するチャットアプリ）
- グループでのメンション必須設定（group mention requirements）

> 完全な設定スキーマは <https://docs.openclaw.ai/gateway/configuration> を参照。
> 実物の `~/.openclaw/openclaw.json` を取得したらこのファイルに貼る（**APIキーはマスクする**）。

---

## ネットワーク / ポート

| 用途 | 既定 | 備考 |
|---|---|---|
| Control UI ダッシュボード | `http://127.0.0.1:18789/` | ローカルのみ |
| リモートアクセス | Web surface / Tailscale 経由 | 公式に詳細あり。安易にポート開放しない |

- 衝突確認: `lsof -i :18789`

---

## プラグイン（2026-08-30 実測）

| プラグイン | id | 種別 | 用途 |
|---|---|---|---|
| `@openclaw/slack` | `slack` | ClawHub 公式 | Slack チャンネル。`~/.openclaw/extensions/slack` |
| `@openclaw/anthropic-provider` | `anthropic` | stock（同梱） | Anthropic プロバイダ。cron を Claude で回すため |
| `@openclaw/searxng-plugin` | `searxng` | ClawHub 公式 | Web 検索を自己ホスト SearXNG に接続（下記「Web検索」節） |
| `@openclaw/duckduckgo-plugin` | `duckduckgo` | stock（同梱） | 旧 Web 検索。ボット判定で不安定 → SearXNG に移行。loaded のまま放置（未使用） |

> ⚠️ **`plugins.allow` は使わない**。設定すると allowlist が**排他的**になり、
> stock の provider プラグイン（anthropic 等）まで巻き込んでブロックする
> （`models auth login` が `No provider plugins found` で失敗した）。
> 一度 `openclaw config unset plugins.allow` で解除済み。非バンドル plugin の
> 起動時 warning は無害。個別に `openclaw plugins enable <id>` すれば足りる。

---

## Web 検索（SearXNG 自己ホスト）— 2026-08-30 実測

DuckDuckGo（キー無し）がボット判定で不安定なため、**ローカルで SearXNG を立てて**
`web_search` のバックエンドにした。キー不要・クエリ上限なし。

### SearXNG 本体（OpenClaw とは別プロセス）

| 項目 | 値 |
|---|---|
| ソース | `~/searxng/src`（`git clone --depth 1 github.com/searxng/searxng`） |
| Python 環境 | `~/searxng/.venv`（`uv venv --python 3.12` の隔離 CPython。system/brew に非依存） |
| 導入 | `uv pip install -r requirements.txt -r requirements-server.txt` → `uv pip install -e . --no-build-isolation`※ |
| 設定 | `~/searxng/settings.yml`（`chmod 600`。`secret_key` あり＝**コミット禁止**、リポジトリ外） |
| 待受 | `http://127.0.0.1:18899`（loopback のみ）。`search.formats: [html, json]` を有効化（JSON API 必須） |
| `limiter` | `false`（valkey/redis 不要にするため。単一ユーザー loopback なので可） |
| 起動 | `~/searxng/run.sh`（`SEARXNG_SETTINGS_PATH` を渡して `searxng-run` を exec） |
| 常駐 | launchd `~/Library/LaunchAgents/local.searxng.plist`（Label `local.searxng`、RunAtLoad/KeepAlive） |
| ログ | `~/searxng/logs/searxng.log` / `searxng.err.log` |

※ SearXNG の `setup.py` がビルド時に `msgspec` を import するため、先に requirements を入れて
`--no-build-isolation` で editable install する必要がある。

操作:
```
launchctl list | grep searxng
launchctl kickstart -k gui/$(id -u)/local.searxng      # 再起動
launchctl bootout gui/$(id -u)/local.searxng           # 停止・解除
# JSON API 動作確認（curl 不使用）:
python3 -c "import urllib.request,urllib.parse,json;print(json.loads(urllib.request.urlopen('http://127.0.0.1:18899/search?'+urllib.parse.urlencode({'q':'test','format':'json'})).read()).get('results',[])[:1])"
```
- 一部エンジン（wikidata / ahmia 等）は init に失敗するが**コア検索は動く**（非致命）。
- `~/searxng` を丸ごと消せばアンインストール（＋ plist を bootout・削除）。

### OpenClaw 側の接続設定

```json5
{
  tools: { web: { search: { enabled: true, provider: "searxng" } } },
  plugins: { entries: { searxng: { config: { webSearch: {
    baseUrl: "http://127.0.0.1:18899",
    categories: "general,news",
    language: "ja"
  } } } } }
}
```
- プラグイン: `openclaw plugins install clawhub:@openclaw/searxng-plugin` → `openclaw plugins enable searxng` → `openclaw daemon restart`
- 確認: `openclaw config get tools.web.search` → `{enabled:true, provider:"searxng"}`
- テスト: `openclaw agent … -m "web_searchで…を検索して"` で実結果が返る／
  `openclaw cron run b18e0f97-… --wait --expect-final` で econ ダイジェストが 40 秒で配信

操作: `openclaw plugins list [--json]` / `plugins enable <id>` / `plugins inspect <id>` → `openclaw daemon restart`

---

## 定期実行 / cron（2026-08-29 実測）

保存: SQLite（`~/.openclaw/state/openclaw.sqlite`）。スケジューラは `openclaw cron status` で `enabled`。

| id（先頭） | name | cron (JST) | 内容 |
|---|---|---|---|
| `445881d3` | `weather-jp-am` | `10 9 * * *`（毎日 09:10） | 日本の気象・防災まとめ |
| `b18e0f97` | `econ-jp-am` | `13 9 * * *`（毎日 09:13） | 日本株・為替・世界の経済指標/イベント |
| `db8a0d0b` | `ai-weekly-digest` | `16 9 * * 1`（月曜 09:16） | 週次 AI 動向ダイジェスト |

共通オプション:
`--model anthropic/claude-haiku-4-5 --fallbacks anthropic/claude-sonnet-5`

> 8/30 変更: モデル sonnet→haiku（コスト減）／夕方(18時台)廃止／朝 09:00直撃→09:10-09:16
> （09:00 は他の自動実行 `stock_analysis_local_daily`(毎日)・`econ_digest_ollama`(金)・`ai_news`(土)・
>  `weather_digest_ornith`(日 09:30) と重なる。うち local な stock_analysis は Ollama GPU を使うため、
>  OpenClaw digest の Ollama fallback は外し fallback を sonnet にした）。
`--channel slack --to slack:U0XXXXXXXXX --announce --expect-final --tz Asia/Tokyo --timeout-seconds 1200`
（`--announce` = エージェントが自力で送れない場合スケジューラが最終返信を代理配信）
実行時刻は数分ずつずらして同時実行を回避（1台の Ollama で複数ローカル推論が詰まった対策の名残）。

操作:
```
openclaw cron list
openclaw cron get <id>                          # JSON 全体
openclaw cron edit <id> --model <provider/model># 使用モデル変更
openclaw cron edit <id> --cron "<expr>" --tz Asia/Tokyo
openclaw cron edit <id> --timeout-seconds <n>
openclaw cron run <id> --wait --expect-final    # 手動テスト実行
openclaw cron runs --id <id>                    # 実行履歴
openclaw cron disable|enable|rm <id>
```

**モデル切替時の注意**: `--model` の値は `agents.defaults.models` allowlist に登録済みである必要がある。
未登録だと即エラー `payload.model '…' rejected by agents.defaults.models allowlist`。追加は:
```
printf '%s' '{ "agents": { "defaults": { "models": { "anthropic/claude-sonnet-5": {} } } } }' | openclaw config patch --stdin
```

**実績**: `anthropic/claude-sonnet-5` で weather / econ をテスト実行 → status ok・Slack 配信成功・36〜42 秒。
ローカル `ollama/ornith-1.5:35b`（1ラン 4〜10分・不安定・同時実行で AbortError）から切替済み。

---

## モデルプロバイダ / APIキー

- 公式は「選んだプロバイダの API キーが必要」「最新世代の最強モデルを推奨（品質と安全性）」
  としか書いておらず、対応プロバイダ名の明記が見当たらない。
- **`openclaw onboard` 実行時に出る選択肢をここに記録する。**
- このMacの手持ち:
  - Claude Code CLI（Pro/Max サブスク） … `~/.local/bin/claude`
  - Anthropic API キー … `~/.anthropic_env`（**残高不足で使用不可**）
  - ローカル LLM … Ollama（`http://localhost:11434`、qwen3.6:35b-mlx 他）
  - OpenAI/Gemini 等 … 未確認

### 記録欄（onboard 後に記入）— 2026-08-29 実測

```
対話（既定）: anthropic/claude-haiku-4-5（fallback ollama/qwen3.6:35b-mlx）
              ※ 8/30: ローカル勢（ornith/qwen3.6/nemotron）は観測値ランキングで幻覚・中国語混入・
                 20〜30秒。claude-haiku-4-5 は 8〜10秒で正確。fallback の qwen3.6 は課金枠切れ時の劣化用
cron（ダイジェスト）: anthropic/claude-sonnet-5（fallback anthropic/claude-haiku-4-5）

Anthropic 認証: claude setup-token → openclaw models auth login --provider anthropic
               → 対話メニューで「Anthropic setup-token」を選択（「Claude CLI」は keychain 読めず不可）
               → プロファイル anthropic:default (anthropic/token)
認証ストア: ~/.openclaw/agents/main/agent/openclaw-agent.sqlite
           （ollama:default=marker / anthropic:default=token）
APIキー環境変数: 不要（token はストアに保存。openclaw.json には平文 token は入らないが sqlite にはある）
```

補足:
- `--auth-choice ollama` は既定モデル `gemma4` を無条件ダウンロードする（約十数GB）。
  既存ローカルモデルを使いたい場合は **`--auth-choice skip` で onboard → 後から `openclaw models set`** が正解。
- `claude-cli/*` モデル（claude-opus-4-6/4-7/4-8, claude-sonnet-4-6, claude-sonnet-5）は
  `openclaw models list` 上で Auth=yes と表示される。ただし onboard の `--auth-choice anthropic-cli` は
  keychain 認証を検出できず失敗した。正式に足すなら:
  ```
  claude setup-token                 # sk-ant-oat01-... が出る（秘密情報・対話）
  openclaw models auth login --provider anthropic --method setup-token
  openclaw models set claude-cli/claude-sonnet-5   # 主モデルにする場合
  ```

### モデル操作コマンド早見表

```
openclaw models status                       # 既定/フォールバック/認証の一覧
openclaw models list                         # 設定済みモデル
openclaw models list --provider ollama --all # Ollama の全ローカルモデル
openclaw models set <provider/model>         # 既定モデル変更
openclaw models fallbacks add <provider/model>
```

---

## デーモン（自動起動）

- `openclaw onboard --install-daemon` がバックグラウンドサービスを登録する。
- macOS では launchd の可能性が高い。**登録された plist の場所と label を確認**:
  ```bash
  ls -la ~/Library/LaunchAgents/ | grep -i openclaw
  launchctl list | grep -i openclaw
  ```
- 手動制御コマンドは `onboard` の出力 / `openclaw gateway --help` で確認:
  ```
  openclaw gateway status
  openclaw gateway stop
  openclaw gateway start / restart   （名称要確認）
  ```

### 記録欄（install-daemon 後に記入）— 2026-08-29 実測

```
plist パス: ~/Library/LaunchAgents/ai.openclaw.gateway.plist
label:      ai.openclaw.gateway
起動タイミング: RunAtLoad=true / KeepAlive=true（ログイン時に自動起動、落ちたら再起動）
ThrottleInterval=10 / ExitTimeOut=20 / ProcessType=Interactive
実行コマンド: /bin/sh <state>/service-env/ai.openclaw.gateway-env-wrapper.sh \
             <state>/service-env/ai.openclaw.gateway.env \
             /opt/homebrew/opt/node/bin/node \
             /opt/homebrew/lib/node_modules/openclaw/dist/index.js gateway --port 18789
WorkingDirectory: ~/.openclaw
環境ファイル: ~/.openclaw/service-env/ai.openclaw.gateway.env（OPENCLAW_GATEWAY_PORT=18789 等）
StandardOutPath: ~/Library/Logs/openclaw/gateway.log
StandardErrorPath: /dev/null
```

### デーモン操作コマンド

```
openclaw daemon status      # インストール状況 + 疎通/機能プローブ
openclaw daemon start | stop | restart
openclaw daemon uninstall   # LaunchAgent 削除（やり直し時）
openclaw gateway status     # より詳細なランタイム状態（pid, listening, capability）
launchctl list | grep openclaw
```

- `openclaw daemon restart` は `launchctl kickstart -k gui/<uid>/ai.openclaw.gateway` 相当。
  設定変更（モデル・トークン等）を反映するには restart が必要。

---

## ワークスペース（エージェントの人格・記憶ファイル）

`~/.openclaw/workspace/`（git 管理下）。セッション開始時に読み込まれる。

| ファイル | 役割 |
|---|---|
| `SOUL.md` | 人格・振る舞い。**「## Language」セクションで「常に日本語で応答」を指定済み**（2026-08-29） |
| `USER.md` | 人間側の情報（TZ=Asia/Tokyo、言語=日本語、扱うプロジェクト文脈）。名前は未記入 |
| `IDENTITY.md` | エージェント自身の名前・見た目・雰囲気（未設定） |
| `AGENTS.md` | 運用ルール（メモリ、group chat、heartbeat 等）。既定のまま |
| `TOOLS.md` | ツール固有メモ（カメラ名・SSH 等）。空 |
| `HEARTBEAT.md` | heartbeat 時のチェックリスト |
| `BOOTSTRAP.md` | 初回セットアップ用。役目を終えたら削除される想定 |
| `memory/YYYY-MM-DD.md` / `MEMORY.md` | 日次ログ／長期記憶（main セッションのみ） |

- 変更後は `openclaw daemon restart`。既存セッションには即時反映されないため、確認は新セッションで。
- 応答言語を戻す/変える場合は `SOUL.md` の「## Language」を編集。

## ログ（2026-08-29 実測）

| 種別 | パス |
|---|---|
| サービス（LaunchAgent）標準出力 | `~/Library/Logs/openclaw/gateway.log` |
| CLI / Gateway ファイルログ | `/tmp/openclaw/openclaw-YYYY-MM-DD.log` |
| チャンネルログ | `openclaw channels logs`（gateway ログから抽出） |

確認コマンド:
```
tail -f ~/Library/Logs/openclaw/gateway.log
openclaw gateway logs
openclaw doctor           # 健康診断（--fix で自動修正提案を適用）
openclaw status           # 総合ステータス（--usage で課金/クォータ内訳）
```
