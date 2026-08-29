# OpenClaw トラブルシューティング

作成: 2026-08-29
遭遇した問題と対処をここに追記する（新しいものを上に）。

---

## 想定される詰まりどころ（事前メモ）

### Node のバージョンが要件未満 / 複数入っている

- `which -a node` で Homebrew 版と nvm 版のどちらが使われているか確認。
- nvm 側が古い場合: `nvm install --lts && nvm alias default lts/*`
- Homebrew 版更新: `brew upgrade node`
- OpenClaw が要求: 22.22.3+ / 24.15+ / 25.9+

### npm の postinstall スクリプトがブロックされる

- 新しい npm は依存パッケージの install スクリプトを既定で実行しない。
- `npm install -g openclaw@latest --allow-scripts=openclaw` を使う（npm 11.16+ / 12+）。
- npm 自体が古い（このMacは 11.12.1）: `npm install -g npm@latest` で更新。

### `curl … | bash` を実行したくない

- スクリプトをファイルに落として中身を確認してから実行（INSTALL.md 方法A の注記参照）。

### `openclaw: command not found`（インストール後）

- グローバル bin が PATH に入っているか: `npm bin -g` / `npm prefix -g`。
- Homebrew node なら通常 `/opt/homebrew/bin` に symlink される。`hash -r` でシェルのキャッシュ更新。

### ポート 18789 が使えない

- `lsof -i :18789` で占有プロセス確認。
- 設定でポート変更（`~/.openclaw/openclaw.json`、キー名は公式 configuration 参照）。

### Gateway が起動しない / status が unhealthy

- `openclaw gateway status` の出力全文を控える。
- ログ（`~/.openclaw/logs/` 想定）を確認。
- モデルプロバイダの APIキー未設定・無効の可能性 → `openclaw onboard` をやり直す。

### モデル呼び出しが失敗する

- Anthropic API キー（`~/.anthropic_env`）は**残高不足で使用不可**。API を選ばない。
- ローカル Ollama を使う場合は `ollama ps` でサーバー稼働を確認。

### デーモンが再起動後に上がらない（macOS launchd）

- `launchctl list | grep -i openclaw` でロード状態確認。
- plist を `launchctl bootout` → `launchctl bootstrap` し直す。
- ネットワーク未確立で失敗するなら起動遅延を入れる（他プロジェクトの launchd 対策と同様）。

---

## 実際に遭遇した問題

### 2026-08-29 — onboard の `--auth-choice anthropic-cli` が「Claude CLI auth が必要」で失敗

- 症状: `openclaw onboard --auth-choice anthropic-cli` が
  `requires Claude CLI auth on this host. Run claude auth login first` で終了。
- 原因: macOS の Claude Code は認証をキーチェーン保管し `~/.claude/.credentials.json` を作らない。
  openclaw はそのファイルを見るため未認証と判定する。`claude auth status` は loggedIn:true。
- 対処: 今回は `--auth-choice skip` で onboard を通し、モデルは後から Ollama を手動設定。
  Anthropic を使う場合は `claude setup-token` で `sk-ant-oat01-…` を発行し
  `openclaw models auth login --provider anthropic --method setup-token`。
- 補足: `--auth-choice claude-cli` は非推奨エイリアス（→ `anthropic-cli`）。

### 2026-08-29 — `--auth-choice ollama` が既定モデル gemma4 を無条件ダウンロード

- 症状: onboard が `Downloading gemma4...` で数分停止（partial 9.6GB まで進行）。
- 原因: quickstart の Ollama フローは既存ローカルモデルを見ず、ハードコードの `gemma4` を pull する。
- 対処: onboard を中断（`kill`）→ `~/.ollama/models/blobs/*<digest>*-partial*` を削除
  → `--auth-choice skip` で onboard → `openclaw models set ollama/<既存モデル>`。

### 2026-08-29 — Slack Socket Mode が `not_allowed_token_type` で接続失敗

- 症状: gateway ログに
  `socket-mode: Failed to retrieve a new WSS URL (error: not_allowed_token_type)` と
  `slack auth.test failed at boot (auth.test returned no user_id)` が繰り返し。bot が無反応。
- 原因: `openclaw channels add` に渡した bot/app トークンの取り違え or app-level token が
  `xapp-`（`connections:write`）でなかった。
- 対処: OAuth & Permissions の `xoxb-` と Basic Information → App-Level Tokens の `xapp-`
  （`connections:write` 付き）を取り直し、`openclaw channels add --channel slack --bot-token … --app-token …`
  で再登録 → `openclaw daemon restart`。ログに `slack socket mode connected` が出れば OK。
- 補足: `openclaw config get channels.slack.botToken` は値を `__OPENCLAW_REDACTED__` にするので、
  先頭確認は `grep -oE '"(botToken|appToken)": *"[A-Za-z]+-' ~/.openclaw/openclaw.json`。

### 2026-08-29 — bot が「access not configured / pairing code」を返す

- 症状: DM で話しかけると本文の代わりにペアリングコードと承認コマンドを返す。
- 原因: **正常動作**。DM アクセスポリシー既定 = pairing（承認制）。
- 対処: host で `openclaw pairing approve slack <code>`。あわせて
  `openclaw config set commands.ownerAllowFrom '["slack:U0XXXXXXXXX"]'` → `openclaw daemon restart`。

### 2026-08-29 — Slack App の DM 入力欄が「このアプリへのメッセージ送信はオフ」

- 原因: App の Messages Tab 未有効化。
- 対処: api.slack.com → App → Features → App Home → Show Tabs で **Messages Tab = On**、
  「Allow users to send Slash commands and messages from the messages tab」にチェック。
  もしくはチャンネルに `/invite @openclaw` して `@openclaw …` で使う。

### 2026-08-29 — エージェントに指示したのに定期処理が動いていない

- 症状: Slack でオーナーが「ダイジェストを定期実行して」と指示、クゥは「設定します」と返答。
  だが `openclaw cron list` が 0 件。
- 原因: ローカルモデル `ornith-1.5:35b` が宣言だけして cron ツールを実行しない
  （小型ローカルモデルにありがちな narrate-without-executing）。
- 対処: `openclaw cron add` で人間が直接登録。恒久対策は capable なモデル（Anthropic 等）に切替。
- 確認: `openclaw cron list` / `openclaw cron runs --id <id>` / `openclaw cron run <id> --wait --expect-final`。

### 2026-08-29 — cron ダイジェストで Web 検索が使えない（`web_search 無効`）

- 症状: cron 実行結果に「web_search: 無効（プロバイダー未設定）」。静的 fetch だけで内容が薄い。
- 原因: `openclaw onboard --skip-search` で検索プロバイダ未設定。
- 対処: 同梱の `duckduckgo` プラグイン（キー不要）を有効化。
  `openclaw config set plugins.allow '["slack","duckduckgo"]'`（allow を設定済みなので列挙が必要）
  → `openclaw plugins enable duckduckgo` → `openclaw daemon restart`。
  単体 `openclaw agent … -m "…web検索して…"` で結果が返れば有効。
- 補足: `plugins.allow` を一度でも設定すると allowlist が排他になり、stock プラグインも明記が要る。

### 2026-08-29 — cron エージェントジョブが 10 分でタイムアウト（status: error）

- 症状: `openclaw cron run <id> --wait --wait-timeout 10m` が `status: error` / `durationMs: 600067`。
- 原因: ローカルモデル + Web 検索多用で 1 ラン 10 分超。ジョブ既定 timeout 600s を超過。
- 対処: `openclaw cron edit <id> --timeout-seconds 1200`。根本的には高速なモデルへ。

### （テンプレ）YYYY-MM-DD — 症状の1行要約

### （テンプレ）YYYY-MM-DD — 症状の1行要約

- 症状:
- 原因:
- 対処:
- 再発防止:
