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

### 2026-08-29 — cron ジョブ2本が同時刻起動で `AbortError: agent run aborted`

- 症状: `0 9,18 * * *` の weather / econ が同じ分に起動し、両方 `AbortError`（各6分前後で中断）。
- 原因: 1台の Ollama で 22GB モデルを2本同時推論しようとして詰まる。
- 対処: スケジュールをずらす（`openclaw cron edit <id> --cron "5 9,18 * * *" --tz Asia/Tokyo`）。
  根本的にはクラウドモデル（Anthropic）へ切替。

### 2026-08-29 — `openclaw models auth login` が `No provider plugins found`

- 症状: `openclaw models auth login --provider anthropic` が
  `Error: No provider plugins found. Install one via 'openclaw plugins install'.`
- 原因: 過去に `openclaw config set plugins.allow '["slack","duckduckgo"]'` で作った**排他 allowlist**が、
  stock の `@openclaw/anthropic-provider` まで無効化していた。
- 対処: `openclaw config unset plugins.allow`（排他リスト自体をやめる）＋ `openclaw plugins enable anthropic`
  → `openclaw daemon restart`。
- 補足: `--method setup-token` は無効名。`openclaw models auth login --provider anthropic` を**引数なし**で
  実行し、対話メニューで「Anthropic setup-token」を選ぶ。

### 2026-08-29 — cron が即エラー `payload.model '…' rejected by agents.defaults.models allowlist`

- 症状: `openclaw cron edit <id> --model anthropic/claude-sonnet-5` 後、実行が1秒で error。
- 原因: cron の `--model` override は `agents.defaults.models`（設定内のモデル allowlist）に
  載っているモデルしか使えない。onboard 後は Ollama 2つだけが登録されていた。
- 対処:
  ```
  printf '%s' '{ "agents": { "defaults": { "models": { "anthropic/claude-sonnet-5": {}, "anthropic/claude-haiku-4-5": {} } } } }' \
    | openclaw config patch --stdin
  openclaw daemon restart
  ```

### 2026-08-30 — cron の web_search が「no web search provider is selected」

- 症状: cron 実行の diagnostics に
  `web_search tool requested in toolsAllow but no web search provider is selected`。
  digest は出るが web_search 由来の鮮度が落ちる。
- 原因: `duckduckgo` プラグインが `loaded` でも、`tools.web.search.provider` の**明示選択**が別途必要。
  キー無しプロバイダは「available API keys からの自動検出」に引っかからない。
- 対処:
  ```
  printf '%s' '{ "tools": { "web": { "search": { "enabled": true, "provider": "duckduckgo" } } } }' | openclaw config patch --stdin
  openclaw daemon restart
  ```
  確認: `openclaw config get tools.web.search` → `{enabled:true, provider:"duckduckgo"}`

### 2026-08-30 — cron ジョブの model override がいつの間にか既定(ollama)へ巻き戻る

- 症状: `--model anthropic/claude-sonnet-5` を設定した weather ジョブが、翌朝の定時実行で
  `ollama/ornith-1.5:35b` で走っていた（同時に設定した econ/AI は保持）。
- 原因: 未特定。daemon restart 単体では再現せず。要監視。
- 対処/確認:
  ```
  openclaw cron get <id> | grep -A2 '"payload"'      # payload.model を確認
  openclaw cron edit <id> --model anthropic/claude-sonnet-5 --fallbacks anthropic/claude-haiku-4-5
  ```
  定時実行のたびに `openclaw cron runs --id <id>` の `entries[].model` を点検するのが安全。

### 2026-08-30 — Slack でクゥに質問すると「No response requested.」しか返らない

- 症状: DM で普通に質問しても本文が「No response requested.」だけ。前ターンで `stop=aborted` も。
- 原因: 対話既定の `ollama/ornith-1.5:35b` が、返信不要の定型句を出力したり中断する
  （小型ローカルモデルがエージェント・プロトコル＝「話すべきでないとき黙る」等を誤発火）。
- 対処: 既定モデルを替える。テストの結果 `ollama/qwen3.6:35b-mlx` が最良
  （`nemotron-3.5-lightning:30b-mlx` はツールを30回ループして空応答になった）。
  `openclaw models set ollama/qwen3.6:35b-mlx` → `openclaw daemon restart`。

### 2026-08-30 — web_search が「DuckDuckGo returned a bot-detection challenge.」

- 症状: `web_search` ツールが毎回
  `{"status":"error","tool":"web_search","error":"DuckDuckGo returned a bot-detection challenge."}`。
  エージェントが検索を諦めて `web_fetch`/`exec` を乱発しループ or 空応答。
- 原因: DuckDuckGo 側のボット判定。短時間に多数の自動検索を投げると発火（テスト連打が引き金）。
  キー無し DDG は負荷で不安定で、恒久運用には向かない。
- 対処（恒久策・いずれか）:
  - 検索 API キー: Brave Search API（新規は実質有料化）/ Tavily（無料枠 1,000/月）等を入れて
    `tools.web.search.provider` をそれに変更
  - **自己ホスト SearXNG**（採用。CONFIG.md「Web 検索」節）。キー不要・上限なし
  - ダイジェストのプロンプトを「検索」→「固定 URL（JMA/アメダス/Yahoo）を web_fetch」に書き換え
  - `jma-mcp` を認証設定（気象は検索不要になる）
- 一時対処: 数時間〜1日で DDG 判定は解除されることが多い（恒久策にならない）。
- **2026-08-30 対応済み**: SearXNG を `127.0.0.1:18899` に立て `provider=searxng` に変更。
  econ ダイジェストが 16 分 error → 40 秒 ok に改善。

### 2026-08-30 — クゥの「降水量ランキング」に数値(mm)が出ない／地点名が化ける

- 症状: Slack で降水量ランキングを頼むと「大量の雨」「（佐貫）」等の空欄・幻覚、地点名に中国語漢字混入。
  「jma-mcp 経由」と称するが実際は未接続。
- 原因: (1) JMA の「最新の気象データ」ランキングページは **JS 描画で生 HTML に数値が無い** のにスクレイプ、
  (2) `jma-mcp-render` MCP は未認証なのに使ったことにする、(3) 取れないので幻覚で埋める。
- 対処: ランキングは **アメダス map JSON を自前集計** する。
  `latest_time.txt` → `amedas/data/map/{YYYYMMDDHHMMSS}.json`（`precipitation1h/3h/24h` 等）→
  `amedas/const/amedastable.json` の `kjName` で地点名 → 降順ソート。
  `~/.openclaw/workspace/TOOLS.md` に「JMA データ取得のルール」として明記済み（HTMLスクレイプ禁止・偽MCP禁止）。
- 検証: 2026-08-30 17:50 の24h降水量 → 勝山 328.0mm / 美山 302.5mm / 春江 282.0mm（福井の大雨）を正しく取得。

### 2026-08-30 — Slack でクゥへの簡単な質問に数分かかる／タイムアウトする

- 症状: 「今日は何日？」レベルの質問に 3 分以上（タイムアウト）。
- 原因: Slack DM セッション（`agent:main:slack:direct:…`）の履歴が **~87,000 トークン**に肥大
  （2日分のテスト＋気象/経済ダイジェストの配信テキストが蓄積）。毎ターン全履歴をローカルモデルで
  再プロンプト処理 → prompt processing だけで 50 秒超。さらに ollama がモデル既定の
  `num_ctx=262144` で読み込み、KV キャッシュが巨大。keep_alive 5 分で毎回コールドロード。
- 対処:
  1. セッション圧縮: `openclaw sessions compact "agent:main:slack:direct:<uid>" --max-lines 8`
     （旧トランスクリプトは `.jsonl.bak.<ts>` に退避。269KB → 6KB に）
  2. `openclaw config patch` で
     `agents.defaults.models."ollama/qwen3.6:35b-mlx".params = { num_ctx: 32768, keep_alive: "60m" }`
     （fallback の qwen3.8 も同様）
  3. `agents.defaults.compaction = { mode: "safeguard", keepRecentTokens: 10000 }`（自動圧縮を早める）
  4. `openclaw sessions cleanup`（不要なテストセッションを整理）
- 結果: 簡単な質問が **1〜3 秒**に短縮。
- 再発防止: num_ctx を絞ったことで自動圧縮が早く効く。ダイジェストの配信先を DM から別チャンネルに
  分ければセッション汚染をさらに減らせる（未対応）。

### （テンプレ）YYYY-MM-DD — 症状の1行要約

- 症状:
- 原因:
- 対処:
- 再発防止:
