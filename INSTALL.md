# OpenClaw インストール手順

作成: 2026-08-29
出典: <https://github.com/openclaw/openclaw> / <https://docs.openclaw.ai/>
（この2つは要約経由で取得。**実行前に必ず公式サイト最新版を確認**すること）

---

## 0. 前提の確認

```bash
# OS
sw_vers

# Node.js（22.22.3+ / 24.15+ / 25.9+ が必要）
node -v
which -a node          # nvm と Homebrew で複数入っている場合、どれが使われるか確認

# npm（11.15+ 目安。--allow-scripts フラグの仕様が版で違う）
npm -v
```

このMac（2026-08-29）:
- Node.js `v25.9.0`（要件 25.9+ を辛うじて満たす。念のため公式の下限表記を再確認）
- npm `11.12.1` → **11.15 未満**。下記いずれかで対応:
  - `npm install -g npm@latest` で npm を更新してから進める（推奨）
  - もしくは npm 11.15 以前向けの記法（`--allow-scripts=openclaw` を **付けない**）でインストール
- Node/npm 更新の判断は nvm の有無も踏まえて行う（`nvm install --lts` 等）

---

## 1. インストール（いずれか1つ）

### 方法A: 公式インストールスクリプト（推奨・最短）

```bash
# macOS / Linux / WSL2
curl -fsSL https://openclaw.ai/install.sh | bash
```

> ⚠️ パイプ実行（`curl … | bash`）はスクリプト内容を確認してから流すのが安全。
> 一度ファイルに落として中身を読む:
> ```bash
> curl -fsSL https://openclaw.ai/install.sh -o /tmp/openclaw-install.sh
> less /tmp/openclaw-install.sh
> bash /tmp/openclaw-install.sh
> ```

### 方法B: npm グローバルインストール

```bash
# npm 11.16+ / 12+
npm install -g openclaw@latest --allow-scripts=openclaw

# npm 11.15 以前
npm install -g openclaw@latest
```

`--allow-scripts=openclaw` は postinstall スクリプトの実行許可（新しい npm の
デフォルト無効化に対応するため）。npm を更新した場合はこちらを付ける。

### 方法C: Docker / pnpm / Bun

公式は pnpm・Bun・Docker もサポートと記載。Docker 版の詳細は
<https://docs.openclaw.ai/> で確認（本書では未検証）。

---

## 2. オンボーディング

```bash
openclaw onboard --install-daemon
```

このコマンドが行うこと（公式記載）:
- モデルアクセスの確認（APIキー等）
- ワークスペースの作成
- Gateway の設定
- `--install-daemon`: バックグラウンドサービス（デーモン）としての登録

> モデルプロバイダ / APIキー: 公式は「選んだプロバイダの API キー」「最新世代の最強モデル推奨」
> としか書いておらず、対応プロバイダ（Anthropic / OpenAI / Ollama 等）の明記なし。
> **onboard 実行時のプロンプトで確認**し、結果を [CONFIG.md](CONFIG.md) に記録する。
> 何も設定しなければ「同梱の OpenClaw agent runtime」が使われる、とのこと。

---

## 3. 起動確認

```bash
# Gateway の状態
openclaw gateway status

# Control UI ダッシュボード（既定 http://127.0.0.1:18789/ ）
openclaw dashboard
```

- ダッシュボードでテストメッセージを送り、応答が返れば疎通OK。
- ポート `18789` が他プロセスと衝突していないか（`lsof -i :18789`）。

---

## 4. チャットアプリ接続

→ [CHANNELS.md](CHANNELS.md) に、接続したアプリごとの手順・トークン取得先・許可設定を記録する。
公式のチャンネル別手順: <https://docs.openclaw.ai/channels>

---

## 5. アンインストール / やり直し

（要確認。一般的には以下）

```bash
# デーモン停止・削除（コマンド名は onboard 出力で確認）
openclaw gateway stop
# npm 版
npm uninstall -g openclaw
# 設定・ワークスペース
rm -rf ~/.openclaw
```

---

## チェックリスト（2026-08-29 時点）

- [x] Node.js 要件を満たす（Node v25.9.0 でOK。更新不要だった）
- [x] npm 更新は不要だった（11.12.1 のまま `npm install -g openclaw@latest` 成功）
- [x] npm でインストール完了（`openclaw --version` → `2026.7.1-2`）
- [x] `openclaw onboard` 完了（`--auth-choice skip --install-daemon`。モデルは後から手動で Ollama 設定）
      → 設定・パス・plist は [CONFIG.md](CONFIG.md) に記録済み
- [x] `openclaw gateway status` が healthy（Runtime: running / Connectivity probe: ok）
- [x] エージェント疎通OK（`openclaw agent` で `ollama/ornith-1.5:35b` が応答）
- [ ] `openclaw dashboard` で Control UI からもテストメッセージ往復
- [ ] チャットアプリ 1 つ接続（Slack。[CHANNELS.md](CHANNELS.md) に手順準備済み）
- [x] launchd 自動起動確認（`~/Library/LaunchAgents/ai.openclaw.gateway.plist` / label `ai.openclaw.gateway`）
- [ ] （任意）Anthropic / Claude Code サブスクをモデルに追加（`claude setup-token` 経由）
- [ ] pc_docs にマニュアル化

### 実施時の注意（実測で判明）

- `--auth-choice claude-cli` は非推奨 → `anthropic-cli`。ただし macOS の Claude Code は
  認証をキーチェーン保管しており openclaw が読めず失敗する。`claude setup-token` で橋渡しが必要。
- `--auth-choice ollama` は既定モデル `gemma4`（十数GB）を**無条件ダウンロード**する。
  既存ローカルモデルを使うなら `--auth-choice skip` で onboard → `openclaw models set <provider/model>`。
- `--non-interactive` には `--accept-risk` が必須。
