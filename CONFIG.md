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

## プラグイン（2026-08-29 実測）

`plugins.allow` を設定すると allowlist が排他的になり、stock プラグインも列挙が必要。

| プラグイン | 種別 | 状態 | 用途 |
|---|---|---|---|
| `slack` (`@openclaw/slack`) | ClawHub 公式 | enabled | Slack チャンネル。`~/.openclaw/extensions/slack` |
| `duckduckgo` (`@openclaw/duckduckgo-plugin`) | stock（本体同梱） | enabled | Web 検索（キー不要）。cron ダイジェスト用に有効化 |

`openclaw config get plugins.allow` → `["slack","duckduckgo"]`

操作: `openclaw plugins list` / `plugins enable <id>` / `plugins inspect <id>` → 変更後 `openclaw daemon restart`

---

## 定期実行 / cron（2026-08-29 実測）

保存: SQLite（`~/.openclaw/state/openclaw.sqlite`）。スケジューラは `openclaw cron status` で `enabled`。

| id（先頭） | name | cron (JST) | 内容 | timeout |
|---|---|---|---|---|
| `445881d3` | `weather-jp-2x` | `0 9,18 * * *` | 日本の気象・防災まとめ | 1200s |
| `b18e0f97` | `econ-daily-2x` | `0 9,18 * * *` | 日本株・為替・世界の経済指標/イベント | 1200s |
| `db8a0d0b` | `ai-weekly-digest` | `0 9 * * 1` | 週次 AI 動向ダイジェスト | 1200s |

共通オプション: `--channel slack --to slack:U0XXXXXXXXX --announce --expect-final --tz Asia/Tokyo`
（`--announce` = エージェントが自力で送れない場合スケジューラが最終返信を代理配信）

操作:
```
openclaw cron list
openclaw cron get <id>                          # JSON 全体
openclaw cron edit <id> --model <provider/model># 使用モデル変更
openclaw cron edit <id> --timeout-seconds <n>
openclaw cron run <id> --wait --expect-final    # 手動テスト実行
openclaw cron runs --id <id>                    # 実行履歴
openclaw cron disable|enable|rm <id>
```

**課題**: 既定モデル `ollama/ornith-1.5:35b` は 1ラン 4〜10分・出力不安定。
→ Anthropic 追加後に `openclaw cron edit <id> --model anthropic/claude-sonnet-5` へ切替推奨。

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
選択したプロバイダ: なし（onboard は --auth-choice skip で通した）
                    → その後 Ollama を手動設定（openclaw が localhost:11434 のモデルを自動列挙）
既定モデル: ollama/ornith-1.5:35b
フォールバック: ollama/qwen3.8:27b-mlx
APIキーの保存先: 不要（ローカル Ollama）
環境変数: 不要
認証ストア: ~/.openclaw/agents/main/agent/openclaw-agent.sqlite （ollama:default=marker）
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
