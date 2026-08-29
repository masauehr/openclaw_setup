# OpenClaw チャットアプリ連携メモ

作成: 2026-08-29
公式のチャンネル別手順: <https://docs.openclaw.ai/channels>

OpenClaw が対応（公式記載）: WhatsApp / Telegram / Slack / Discord / Google Chat /
Signal / iMessage / Microsoft Teams / Matrix / Zalo など。

接続したものから下に記録していく。**トークン・シークレットはこのファイルに直書きしない**
（`~/.openclaw/openclaw.json` 側に置き、ここには「取得先」と「設定した事実」だけ書く）。

---

## 最初に接続するアプリの検討

| 候補 | 手軽さ | メモ |
|---|---|---|
| **Telegram** | ◎ | BotFather で `/newbot` → トークン取得のみ。サーバー不要。まずこれが最短 |
| Slack | ○ | 既存ワークスペースあり。App 作成・スコープ設定・イベント購読が必要 |
| Discord | ○ | Developer Portal で Bot 作成・トークン・intent 設定 |
| iMessage | △ | macOS ネイティブだが権限（フルディスクアクセス等）が要る場合あり |
| Signal / WhatsApp | △ | 電話番号紐付け・リンクデバイス手順が要る |

→ **決定（2026-08-29）: 最初の接続先は Slack**（既存ワークスペースを使う）。Telegram は後回し。

---

## Slack 接続手順（Socket Mode）— ユーザー実施用

OpenClaw 2026.7.1-2 実測メモ。Socket Mode（公開URL不要・ローカルPC可）で進める。

### 0. Slack チャンネルプラグインを導入（本体非同梱）✅ 2026-08-29 実施済み

Slack は Telegram と違い本体に同梱されていないので、公式プラグインを入れる。

```bash
openclaw plugins install clawhub:@openclaw/slack
openclaw plugins list | grep -i slack     # Status が enabled になる
openclaw daemon restart
```

実施結果: `@openclaw/slack@2026.7.1` を `~/.openclaw/extensions/slack` に導入。
`plugins list` で `Slack | slack | enabled | global:slack/dist/index.js | 2026.7.1`。official のためリスク確認プロンプトなし。

### 1. Slack App を作成（マニフェストから）

1. <https://api.slack.com/apps> → **Create New App** → **From a manifest**
2. 対象ワークスペースを選択
3. 下記マニフェスト（YAML）を貼り付け（`display_information.name` は好みで変更可）

```yaml
display_information:
  name: OpenClaw
  description: Self-hosted AI assistant gateway
features:
  bot_user:
    display_name: openclaw
    always_online: true
  slash_commands:
    - command: /openclaw
      description: Talk to OpenClaw
      should_escape: false
oauth_config:
  scopes:
    bot:
      - app_mentions:read
      - channels:history
      - channels:read
      - chat:write
      - commands
      - groups:history
      - groups:read
      - im:history
      - im:read
      - im:write
      - mpim:history
      - mpim:read
      - users:read
      - reactions:read
      - reactions:write
      - files:read
      - files:write
settings:
  event_subscriptions:
    bot_events:
      - app_mention
      - message.im
      - message.channels
      - message.groups
      - message.mpim
  interactivity:
    is_enabled: true
  socket_mode_enabled: true
  org_deploy_enabled: false
```

4. **Create** → **Install to Workspace** で認可

### 2. トークンを2種類取得

| トークン | 取得場所 | 形式 |
|---|---|---|
| Bot User OAuth Token | 左メニュー **OAuth & Permissions** | `xoxb-...` |
| App-Level Token（`connections:write` スコープ付き） | **Basic Information → App-Level Tokens → Generate Token and Scopes** | `xapp-...` |

### 3. OpenClaw に登録（**自分のターミナルで**・トークンは環境変数経由）

```bash
export SLACK_BOT_TOKEN=xoxb-...
export SLACK_APP_TOKEN=xapp-...
openclaw channels add --channel slack --bot-token "$SLACK_BOT_TOKEN" --app-token "$SLACK_APP_TOKEN"
openclaw daemon restart
openclaw channels status --probe        # 接続確認
```

> トークンは `~/.openclaw/openclaw.json` の `channels.slack.botToken` / `appToken` に保存される
> （env 参照方式にするなら公式の SecretRef を使う）。このファイルは共有・コミットしない。

### 4. command owner（オーナー権限）を設定

`/config` や exec 承認など特権コマンドを使えるのは owner だけ。DM ペアリングだけでは owner にならない。

```bash
# 自分の Slack ユーザーID（Uxxxx）を調べる
openclaw directory self --channel slack        # または channels resolve
openclaw config set commands.ownerAllowFrom '["slack:UXXXXXXXX"]'
openclaw daemon restart
```

### 5. 疎通確認

- Bot をチャンネルに招待、または DM で話しかける → 応答が返れば OK
- `openclaw channels logs` / `~/Library/Logs/openclaw/gateway.log` でトラブル確認

### ハマりどころ

- チャンネルの allowlist は**チャンネルID指定**（名前ではマッチしない）
- DM ポリシー（`pairing` / `allowlist` / `open` / `disabled`）の既定に注意
- Socket Mode はアウトバウンド WSS が通れば良い（インバウンド公開URL不要）

---

## 連携記録

### （テンプレ）アプリ名 — YYYY-MM-DD

- 取得したもの: Bot トークン（保存先: `~/.openclaw/openclaw.json` の xxx）
- 設定手順:
  1.
  2.
- allowlist 設定: （どのユーザー/チャンネルからの入力を許可したか）
- 動作確認: テストメッセージ「…」→ 応答「…」 OK / NG
- 注意点:

---

### Telegram — （未）

- [ ] BotFather で Bot 作成（`/newbot`）
- [ ] トークンを OpenClaw に設定
- [ ] 自分のユーザーIDを allowlist に追加
- [ ] Bot にメッセージ → 応答確認

### Slack — 接続完了・疎通OK（2026-08-29）

- [x] Slack チャンネルプラグイン導入（`@openclaw/slack@2026.7.1`、enabled）
- [x] テスト用ワークスペース `openclaw-test` を新規作成（既存コミュニティは汚さない方針）
- [x] Slack App「OpenClaw」作成 → App Manifest 画面に YAML を貼って Save（From scratch 経由でも後貼りOK）
- [x] App-Level Token（`xapp-`, `connections:write`）生成 ＋ Install to Workspace で Bot Token（`xoxb-`）取得
- [x] `openclaw channels add --channel slack --bot-token … --app-token …`（env 経由）→ `openclaw daemon restart`
- [x] 初回はトークン取り違えで `not_allowed_token_type` / `auth.test failed` → 正しい値で入れ直し
      → ログに `slack socket mode connected` 確認
- [x] `openclaw pairing approve slack <code>` で自分を承認 ＋ `commands.ownerAllowFrom=["slack:U0XXXXXXXXX"]`
- [x] **疎通OK**（Slack から話しかけて `ollama/ornith-1.5:35b` が応答）
- [ ] Bot トークンの**再発行**（セットアップ中にスクショで露出したため。Install App → Reinstall to openclaw-test）
- [ ] App Home の Messages Tab 有効化（DM で使いたい場合。チャンネル運用なら不要）

保存先: `~/.openclaw/openclaw.json` の `channels.slack.botToken` / `channels.slack.appToken`（平文。共有・コミット禁止）
ワークスペース名: `openclaw-test` / App 名: `OpenClaw`（bot 表示名 `openclaw`）

**自分の Slack メンバーID**: `U0XXXXXXXXX`（＝ `slack:U0XXXXXXXXX`。秘密情報ではない）
→ `commands.ownerAllowFrom` に設定するオーナーID。config リセット時はこれを再投入する。

#### トラブル: bot が「access not configured / pairing code」を返す

正常動作（DM ポリシー既定 = pairing 承認制）。bot が Slack に出したコマンドを host で実行:
```bash
openclaw pairing approve slack <ペアリングコード>
openclaw config set commands.ownerAllowFrom '["slack:U0XXXXXXXXX"]'
openclaw daemon restart
```
承認制をやめたい場合は DM ポリシーを緩める（`openclaw configure --section channels` で Slack の DM policy を変更）。
