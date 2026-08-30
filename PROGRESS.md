# OpenClaw 導入 進捗ログ

新しい項目は上に追記する（新しい日付が上）。
1 エントリ = 日付 / やったこと / 結果 / 次アクション。

---

## 現在の到達点サマリ（2026-08-30 更新）

正本マニュアル: [MANUAL.md](MANUAL.md)（= `pc_docs/manuals/automation/openclaw.md` のコピー）

| 項目 | 状態 |
|---|---|
| OpenClaw 本体 | `2026.7.1-2`（npm グローバル `/opt/homebrew/lib/node_modules/openclaw`） |
| Gateway | launchd 常駐 `ai.openclaw.gateway`（RunAtLoad/KeepAlive）／local・loopback・token auth・:18789 |
| **対話モデル** | `anthropic/claude-haiku-4-5`（fallback `ollama/qwen3.6:35b-mlx`）。ローカル勢は幻覚・中国語混入・遅延で断念（8/30） |
| **cron モデル** | `anthropic/claude-haiku-4-5`（fallback `anthropic/claude-sonnet-5`）。8/30 sonnet→haiku（コスト減） |
| Anthropic 認証 | `claude setup-token` → `openclaw models auth login --provider anthropic`（「Anthropic setup-token」選択）。プロファイル `anthropic:default (token)` |
| Web検索 | **自己ホスト SearXNG**（`127.0.0.1:18899`、launchd `local.searxng`）。`tools.web.search.provider=searxng`。キー不要・上限なし（8/30 DuckDuckGo から移行） |
| チャンネル | **Slack** 接続済み（WS `openclaw-test` / App「OpenClaw」/ Socket Mode / health:healthy） |
| オーナー | `commands.ownerAllowFrom = ["slack:U0XXXXXXXXX"]` |
| 応答言語 | 日本語固定（`~/.openclaw/workspace/SOUL.md` の Language セクション） |
| エージェント identity | 名前「クゥ」/ 黒猫 / 🐈‍⬛（`~/.openclaw/workspace/IDENTITY.md`） |
| プラグイン | `slack` / `anthropic` / `searxng` を enable。`duckduckgo` は loaded だが未使用。**`plugins.allow` は使わない**（排他リストが provider を巻き込むため 8/30 に `config unset`） |
| memory search | 無効（`memorySearch.enabled=false`） |
| ローカルモデル調整 | `agents.defaults.models."ollama/qwen3.6:35b-mlx".params = { num_ctx: 32768, keep_alive: "60m" }`（fallback 用。既定 262144 は遅い） |
| セッション | `agents.defaults.compaction = { mode: "safeguard", keepRecentTokens: 10000 }`。肥大時は `openclaw sessions compact "<key>" --max-lines N` |
| 補助スクリプト | `~/.openclaw/workspace/bin/jma_rank.py`（アメダス観測値ランキング。TOOLS.md で exec 指示） |

### 登録済み cron ジョブ（model=`anthropic/claude-haiku-4-5` / fallback `anthropic/claude-sonnet-5`）

| id (先頭) | name | schedule (JST) | 内容 |
|---|---|---|---|
| `445881d3` | `weather-jp-am` | `10 9 * * *`（毎日 09:10） | 日本の気象・防災まとめ |
| `b18e0f97` | `econ-jp-am` | `13 9 * * *`（毎日 09:13） | 日本株・為替・世界の経済指標/イベント |
| `db8a0d0b` | `ai-weekly-digest` | `16 9 * * 1`（月曜 09:16） | 週次 AI 動向ダイジェスト |

共通: `--channel slack --to slack:U0XXXXXXXXX --announce --expect-final --tz Asia/Tokyo --timeout-seconds 1200`。
操作: `openclaw cron list|get <id>|edit <id>|run <id> --wait --expect-final|rm <id>`。

- **8/30 変更**: sonnet→haiku ／ 夕方(18時台)廃止 ／ 朝を 09:00直撃→**09:10-09:16**
  （他自動実行を回避: `stock_analysis_local_daily` 毎日09:00・`econ_digest_ollama` 金09:00・`ai_news` 土09:00・
  `weather_digest_ornith` 日09:30・`stock_analysis` 毎日10:00〈Claude CLI〉）。
- cron の `--model` は `agents.defaults.models` allowlist に登録が必要
  （`config patch --stdin` で `anthropic/claude-haiku-4-5` / `claude-sonnet-5` 追加済み）。

## 未解決の課題・確認事項

- [ ] **09:10-09:16 の初回定時実行を Slack で確認**（初回は翌 09:10。`openclaw cron runs --id <id>` で `status`/`model` 点検）
- [ ] cron ジョブの `--model` がまれに既定へ巻き戻る事象（8/30 (11) で一度発生、原因未特定＝要監視）
- [ ] Claude Pro サブスク枠の消費（対話＋cron＋他プロジェクトの `*_haiku` と共有）。枠切れ時は fallback（対話→qwen3.6 / cron→sonnet）
- [ ] スリープ運用（OpenClaw も SearXNG も Mac 起動中のみ。常用するなら `caffeinate` / 常時起動機）
- [ ] 平文シークレット: `~/.openclaw/openclaw.json`（Gateway/Slack token）、認証ストア sqlite（Anthropic token）、
      `~/searxng/settings.yml`（secret_key）。いずれもリポジトリ外。SecretRef 化は未対応
- [ ] ポート 18789 の常時開放の是非（現状 loopback のみ。リモートは Tailscale 検討）
- [ ] doctor: Skills の Missing requirements 33 件（`openclaw doctor --fix` で未使用整理）
- [x] ~~pc_docs にマニュアル化~~ → `pc_docs/manuals/automation/openclaw.md` 作成済み（[MANUAL.md](MANUAL.md) に同梱）

---

## ログ

### 2026-08-30 (16) — cron を haiku 化・夕方廃止・09:10-09:16 へ

- **経緯**: ローカル断念決定を受け、定期実行のコスト削減と他自動実行との重複回避
- **やったこと**:
  - cron 3ジョブを `--model anthropic/claude-haiku-4-5 --fallbacks anthropic/claude-sonnet-5`
    （Ollama fallback は GPU 競合の元なので外した）
  - 夕方(18時台)の実行を廃止 → 気象・経済は朝1回に
  - 当初 07:30台にしたが「7時台はネット未接続」との指摘 → **09:10 / 09:13 / 09:16 (月)** に確定
  - このMacの 09:00-12:00 の他 launchd を全列挙して隙間を特定
    （09:00 `stock_analysis_local_daily` 毎日ほか、10:00 `stock_analysis` Claude CLI、10:30/11:00/12:00 各種）
  - ジョブ名を `weather-jp-am` / `econ-jp-am` に改名
- **結果**: 実行回数 5/日 → 2〜3/日。09:10台は毎日空いており、10:00 の Claude CLI ジョブとも時間が離れている
- **次アクション**: 翌 09:10 の初回定時実行を Slack で確認

### 2026-08-30 (15) — 「簡単な質問が遅い」の原因＝セッション肥大、を修正

- **症状**: Slack でクゥに簡単な質問 → 3分超（タイムアウト）
- **原因**: Slack DM セッション（`agent:main:slack:direct:…`）の履歴が **~87,000 トークン**に肥大
  （2日分のテスト＋ダイジェスト配信テキスト）。毎ターン全履歴を再プロンプト処理。
  加えて ollama がモデル既定 `num_ctx=262144` で読み込み KV キャッシュ肥大。keep_alive 5分で毎回コールドロード
- **やったこと**:
  - `openclaw sessions compact "agent:main:slack:direct:<uid>" --max-lines N`（269KB → 6KB。旧履歴は `.bak` 退避）
  - `agents.defaults.models."ollama/qwen3.6:35b-mlx".params = { num_ctx: 32768, keep_alive: "60m" }`（qwen3.8 も）
  - `agents.defaults.compaction = { mode: "safeguard", keepRecentTokens: 10000 }`
  - `openclaw sessions cleanup`
- **結果**: qwen3.6 で簡単な質問が 3分 → **1〜3 秒**。ただし観測値ランキング等では依然幻覚（→ (14) で haiku に）

### 2026-08-30 (14) — 対話モデルもローカルを断念 → claude-haiku-4-5／ランキング用スクリプト追加

- **背景**: 「簡単な質問が遅い」の対処（セッション圧縮＋num_ctx）後も、ローカルモデルが
  観測値ランキングで**幻覚**を連発（qwen3.6「気温5℃以上の地点ゼロ」/ ornith「風速5m/s以上ゼロ」
  = 実際は高松33℃・奥尻10m/s）。中国語混入（风速・メスダ）、TOOLS.md の手順も守らない、20〜30秒
- **切り分け**（同じ「風速ランキング」質問）:
  | モデル | 時間 | 結果 |
  |---|---|---|
  | `ornith-1.5:35b`（clean session + num_ctx） | 31s | 幻覚「風速ゼロ」・日付も誤り |
  | `qwen3.6:35b-mlx` | — | 幻覚＋中国語混入 |
  | **`anthropic/claude-haiku-4-5`** | 8〜10s | **正確**（奥尻10.3m/s…、スクリプト実行、綺麗な日本語） |
- **やったこと**:
  - `~/.openclaw/workspace/bin/jma_rank.py` 追加（アメダス map JSON を集計する専用スクリプト。
    `temp|wind|precip1h|precip3h|precip24h` 等）。TOOLS.md に「ランキングは JSON 自前パースせず
    このスクリプトを exec」と明記
  - **対話の既定を `anthropic/claude-haiku-4-5`** に（fallback `ollama/qwen3.6:35b-mlx` = 課金枠切れ時の劣化用）
  - Slack DM セッションを `--max-lines 1` まで圧縮（ローカルモデル時代の幻覚ログを除去）
- **結果**: 気温/風速ランキングとも 8〜10 秒で正確な実数値。日本語も自然
- **コスト影響**: 対話・cron ともに Claude（Pro サブスク枠を共有）。多用すると枠に当たる可能性 →
  その場合 fallback の qwen3.6 に自動降格（品質は落ちる）
- **教訓**: ローカルモデル（ornith/qwen3.6/nemotron）は日本語のエージェント的ツール利用に不向き。
  雑談以外は Claude が必要

### 2026-08-30 (13) — SearXNG を自己ホストして web_search を安定化

- **やったこと**:
  - この Mac に Docker/podman が無いので **native インストール**:
    `git clone` → `uv venv --python 3.12`（隔離CPython、system非依存）→
    `uv pip install -r requirements*.txt` → `uv pip install -e . --no-build-isolation`
    （※`setup.py` が `msgspec` を import するため no-build-isolation 必須）
  - `~/searxng/settings.yml`: `127.0.0.1:18899` / `search.formats:[html,json]` / `limiter:false` / secret_key
  - launchd `~/Library/LaunchAgents/local.searxng.plist`（`local.searxng`、RunAtLoad/KeepAlive）で常駐化
  - OpenClaw: `openclaw plugins install clawhub:@openclaw/searxng-plugin` → enable →
    `config patch` で `tools.web.search.provider=searxng` ＋
    `plugins.entries.searxng.config.webSearch.baseUrl=http://127.0.0.1:18899`（categories=general,news / language=ja）
- **結果**:
  - SearXNG JSON API 直: 200・20件（日本語ソース）。ボット判定なし
  - OpenClaw `web_search` 経由: 23秒で実結果
  - **econ ダイジェスト（今朝 16分で error だったもの）→ テスト実行 40秒・status ok・Slack 配信・実データ入り**
  - キー不要・クエリ上限なし。DuckDuckGo からの移行完了（duckduckgo プラグインは loaded のまま未使用）
- **注意/残**:
  - SearXNG も Mac 起動中のみ稼働（OpenClaw と同じスリープ制約）
  - 一部エンジン（wikidata 等）は init 失敗するがコア検索は動作（非致命）
  - `~/searxng/settings.yml` に secret_key。リポジトリ外なので OK だが取り扱い注意
  - SearXNG のバージョン更新は `cd ~/searxng/src && git pull && uv pip install -e . --no-build-isolation` → kickstart

### 2026-08-30 (12) — 対話用モデルの入れ替え検討＋DuckDuckGo 検索ブロック判明

- **背景**: Slack でクゥに質問しても「No response requested.」しか返らない
  → 原因は対話既定の `ollama/ornith-1.5:35b` が返信不要の定型句を吐く/中断する挙動（小型ローカルモデルの誤発火）
- **モデル比較テスト**（`openclaw agent` で簡単な質問 / 散文推論 / web_fetch）:
  | モデル | 簡単 | 散文推論 | ツール(web_fetch) | 「No response」バグ |
  |---|---|---|---|---|
  | `ornith-1.5:35b` | △ | ○ | ○ | ❌ 発生 |
  | `nemotron-3.5-lightning:30b-mlx` | ○ | ○ | ❌ 30回ループ→空応答(2.5分) | 出ない |
  | **`qwen3.6:35b-mlx`** | ○ | ✅ 9秒・良質 | ✅ 6秒・正確・完結 | 出ない |
  → **対話既定を `ollama/qwen3.6:35b-mlx` に変更**（fallback は `ollama/qwen3.8:27b-mlx` のまま）
- **重要な発見: DuckDuckGo `web_search` がボット判定でブロック**
  `{"status":"error","tool":"web_search","error":"DuckDuckGo returned a bot-detection challenge."}` が連発。
  今日のテスト連打が引き金。**cron の Claude ダイジェストにも影響する**（web_fetch で粘れば部分的に可）。
  DDG キー無しは負荷で不安定 = 恒久運用に不向き
- **未決**: web_search の恒久対策
  - A: 検索APIキー（Brave 無料枠 月2000 / Tavily 無料枠）→ `openclaw config` で provider 指定
  - B: ダイジェストのプロンプトを「検索」→「固定URL(JMA/アメダス/Yahoo) を web_fetch」に書き換え
  - C: `jma-mcp` を認証設定（気象のみ。経済/AIは別途）
  - D: DDG 回復待ち（数時間〜1日。恒久策にならず）
- **次アクション**: 上記 A〜C を決めて web_search を安定化。決めるまで気象ダイジェストは精度が落ちる可能性

### 2026-08-30 (11) — 朝の定時実行が失敗 → 2原因を修正

- **症状**: 8/30 の定時実行で weather が `ornith-1.5:35b` で走り「No response requested.」、
  econ 9:05 が `AbortError`（※econ は 9:21 に claude-sonnet-5 で自動リトライ成功していた）
- **原因1**: **weather ジョブの `payload.model` が `ollama/ornith-1.5:35b` に戻っていた**
  （econ / AI は `anthropic/claude-sonnet-5` を保持。weather だけ。前夜 20:16 は sonnet で成功していたので
  20:16〜翌9:04 の間に何かで巻き戻った。確たる原因は未特定＝**要監視**）
  → `openclaw cron edit 445881d3… --model anthropic/claude-sonnet-5 --fallbacks anthropic/claude-haiku-4-5` で再設定。
  daemon restart を挟んでも保持されることを確認
- **原因2**: cron 実行の診断に
  `web_search tool requested in toolsAllow but no web search provider is selected`。
  duckduckgo プラグインは `loaded` でも、**`tools.web.search.provider` の明示選択が別途必要**だった
  （プラグインの loaded ≠ web_search プロバイダとして選択済み）
  → `openclaw config patch` で `tools.web.search = { enabled: true, provider: "duckduckgo" }`
- **結果**: weather をテスト実行 → status ok / claude-sonnet-5 / 90秒 / Slack 配信。
  内容も実データ（福井の大雨特別警報レベル5・荒川氾濫、台風22/23号、地震一覧）
- **メモ**: 気象庁の警報 JSON API（`bosai/warning/data/warning/*.json`）が一部 CDN キャッシュで
  古い日付（5月）を返す事象をエージェントが報告。概況テキスト API ＋ 報道で補完している
- **次アクション**: 明朝 9:00/9:05 の定時実行を Slack で確認（model 巻き戻りが再発しないか）

### 2026-08-29 (10) — Anthropic 追加・cron を claude-sonnet-5 に切替（ダイジェスト実運用化）

- **経緯**: 18:00 の初回定時実行で weather / econ が両方 `AbortError`（同時起動＋ローカルモデルが重い）
- **やったこと**:
  - cron スケジュールをずらして同時実行を回避: weather `0 9,18`、econ `5 9,18`、AI `10 9 * * 1`
  - Anthropic 認証（ユーザーが自端末で）:
    `claude setup-token` → `openclaw models auth login --provider anthropic`（対話）→ **「Anthropic setup-token」**を選択
    （「Claude CLI」は onboard 同様 keychain を読めず不可。API key は残高不足）
    → `anthropic:default (anthropic/token)` プロファイル登録
  - 途中の詰まり:
    1. `models auth login` が `No provider plugins found` → `plugins.allow=["slack","duckduckgo"]` の排他リストが
       anthropic プロバイダまでブロックしていた → **`openclaw config unset plugins.allow`**（排他リスト自体をやめた）
       ＋ `openclaw plugins enable anthropic`
    2. cron 実行が即エラー `payload.model '…' rejected by agents.defaults.models allowlist` →
       `agents.defaults.models` に使うモデルの登録が必要 →
       `openclaw config patch --stdin` で `anthropic/claude-sonnet-5` と `anthropic/claude-haiku-4-5` を追加
  - 3ジョブを `--model anthropic/claude-sonnet-5 --fallbacks anthropic/claude-haiku-4-5` に変更
- **結果**: weather / econ をテスト実行 → **status ok / Slack 配信成功 / 所要 36〜42秒**（ローカル比で桁違いに速い）。
  内容も実データ入りの実用的なダイジェストに。**定期ダイジェスト実運用化 完了**
- **次アクション**: 明朝 9:00 / 9:05 の自動実行を Slack で確認 / クゥ（Slack側エージェント）に cron 設定済みを共有 /
  対話用の既定モデルは Ollama のまま（cron のみ Anthropic）

### 2026-08-29 (9) — git 管理化・GitHub public リポジトリへ push

- **やったこと**:
  - 公開前スキャン: 全 `.md` に実トークン/APIキー/本名/メール無しを確認
  - 個人識別子（Slack ワークスペース名 / Slack メンバーID / home パスのユーザー名）を
    プレースホルダ（`openclaw-test` / `U0XXXXXXXXX` / `~`）に置換
  - `git init -b main` → 初回コミット `8e92bfd`（8ファイル）
  - `gh repo create openclaw_setup --public --source=. --push`
- **結果**: https://github.com/masauehr/openclaw_setup （Public、`main`）
- **注意**: 今後 `openclaw.json` 実体や秘密情報をこのディレクトリに置かない（`.gitignore` 済み）。
  以降の変更は `git add` → `commit` → `push`（CLAUDE.md ルール）

### 2026-08-29 (8) — 定期情報収集（cron）を手動セットアップ

- **経緯**: Slack で クゥ に「AI週次／気象2回／経済2回のダイジェストを DM に。設定後テスト実行」と指示 →
  クゥは「設定します」と返答しただけでツール未実行。`openclaw cron list` = 0 件で**何も動いていなかった**
  （ローカルモデル `ornith-1.5:35b` が宣言のみで実処理しない failure mode）
- **やったこと**（Claude Code 側で CLI 直接設定）:
  - `openclaw cron add` で3ジョブ登録（すべて `--channel slack --to slack:U0XXXXXXXXX --announce --expect-final --tz Asia/Tokyo`）
    | name | cron | 内容 |
    |---|---|---|
    | `ai-weekly-digest` | `0 9 * * 1` | 毎週月曜9時 AIダイジェスト |
    | `weather-jp-2x` | `0 9,18 * * *` | 毎日9/18時 日本の気象・防災 |
    | `econ-daily-2x` | `0 9,18 * * *` | 毎日9/18時 日本株・世界経済指標 |
  - Web 検索が無効（onboard の `--skip-search`）と判明 → バンドルの `duckduckgo` プラグインを有効化
    （`plugins.allow` を `["slack","duckduckgo"]` に。単体 agent turn で DuckDuckGo 検索の動作確認OK）
  - テスト実行（`openclaw cron run <id> --wait --expect-final`）: weather ジョブが Slack DM に配信成功を確認
- **結果 / 課題**:
  - ✅ 3ジョブ登録済み、Slack DM への配信は機能。次回自動実行は本日18:00（weather/econ）、月曜9:00（AI）
  - ✅ DuckDuckGo Web 検索 有効化・動作確認
  - ⚠️ **ローカルモデルが遅い**（1ラン 4〜10分超）。検索多用ランで 10分タイムアウト → `error` 1件。
    暫定で全ジョブ `--timeout-seconds 1200`（20分）に延長
  - ⚠️ 出力品質が不安定（良いサマリの回と「取得できませんでした」の回が混在）
  - → **推奨: Anthropic 追加（`claude setup-token`）してダイジェストを Claude で回す**（option B）。
    ローカルのままなら精度・速度は妥協が必要
- **次アクション**: 18:00 の自動実行結果を Slack で確認 / 必要なら `openclaw cron edit <id> --model <...>` でモデル変更 /
  クゥ側にも「cron は設定済み」と共有（重複登録を避ける）

### 2026-08-29 (7) — 日本語を既定の応答言語に設定

- **やったこと**:
  - `~/.openclaw/workspace/SOUL.md` に「## Language」セクション追加（常に日本語で応答、コード等は原文維持）
  - `~/.openclaw/workspace/USER.md` を初期化（TZ=Asia/Tokyo、言語=日本語、扱うプロジェクトの文脈）
  - workspace（`~/.openclaw/workspace/.git`）にコミット → `openclaw daemon restart`
- **結果**: 新セッションで英語プロンプトに対しても日本語で応答することを CLI で確認（`ornith-1.5:35b`）
  - 注: workspace の md は**セッション開始時に読み込まれる**。既存 Slack セッションは古い文脈が残るので、
    確認は新スレッド／新メッセージで行う
- **残タスク**: スリープ運用の方針、（任意）Anthropic 追加、pc_docs マニュアル化

### 2026-08-29 (6) — Slack 疎通完了（エンドツーエンド）

- **やったこと**:
  - トークン取り違えで Socket Mode が `not_allowed_token_type`、bot `auth.test failed` → 正しい `xoxb-`/`xapp-` で入れ直し
    → ログに `slack socket mode connected`
  - bot が pairing コードを返す（DM ポリシー既定 = 承認制）→ `openclaw pairing approve slack <code>`
  - 自分の Slack メンバーID `U0XXXXXXXXX` が判明 → `commands.ownerAllowFrom` に設定
  - `openclaw config set plugins.allow '["slack"]'`（非バンドルプラグインの明示信頼、警告解消）
- **結果**: **Slack から話しかけて `ollama/ornith-1.5:35b` が応答。連携完成**
- **残タスク（仕上げ）**:
  1. Bot トークン再発行（スクショ露出のため。Install App → Reinstall to openclaw-test → `channels add` で上書き）
  2. 日本語応答の既定化（workspace の SOUL.md / システムプロンプト）
  3. スリープ運用の方針（Mac 起動中のみ稼働。常用するなら `caffeinate` or 常時起動機）
  4. （任意）Anthropic / Claude Code サブスクをモデル追加（`claude setup-token`）
  5. 運用が安定したら pc_docs にマニュアル化

### 2026-08-29 (5) — Slack 接続完了

- **やったこと**（ユーザーが自端末＋ブラウザで実施）:
  - テスト用 Slack ワークスペース `openclaw-test` を新規作成
  - Slack App「OpenClaw」を作成（From scratch で作成後、App Manifest 画面に CHANNELS.md の YAML を貼って Save）
  - App-Level Token（`xapp-`, `connections:write`）生成、Install to Workspace で Bot Token（`xoxb-`）取得
  - `openclaw channels add --channel slack --bot-token … --app-token …`（env 経由）→ `openclaw daemon restart`
- **結果**:
  - `openclaw channels status --probe` →
    `Slack default: enabled, configured, running, bot:config, app:config, works` ＝ 接続成功
  - トークンは `~/.openclaw/openclaw.json` の `channels.slack.*` に平文保存
- **次アクション**:
  1. Bot トークン再発行（セットアップ中にスクショで露出。Install App → Reinstall to openclaw-test）
  2. `openclaw directory self --channel slack` で自分のIDを確認 → `commands.ownerAllowFrom` 設定 → restart
  3. Slack で `@openclaw` を招待 or DM して疎通テスト
  4. 日本語既定化（SOUL.md 等）

### 2026-08-29 (4) — Slack チャンネルプラグイン導入

- **やったこと**: `openclaw plugins install clawhub:@openclaw/slack`（ユーザーが自端末で実行）
- **結果**:
  - Slack は本体非同梱と判明（同梱チャンネルは Telegram のみ）。公式プラグインを ClawHub から導入
  - `@openclaw/slack@2026.7.1` → `~/.openclaw/extensions/slack`。`plugins list` で enabled
  - `openclaw daemon restart` 済み。official のためリスク確認プロンプトは出ず
  - 注: `openclaw channels capabilities --channel slack` や自動でのプラグイン導入は
    対話プロンプト／ネット取得のため CLI エージェント側からは実行不可（ユーザー実施）
- **次アクション**: Slack App 作成（マニフェスト）→ `xoxb-`/`xapp-` 取得 →
  `openclaw channels add --channel slack ...` → command owner 設定 → 疎通確認（[CHANNELS.md](CHANNELS.md)）

### 2026-08-29 (3) — onboard 完了・Gateway 常駐化・Ollama で疎通OK

- **やったこと**:
  - ユーザーが Qiita 記事 (nogataka/34cfbee988a9cd873c91) を参考に `npm install -g openclaw@latest` 実施済みと確認
    → openclaw `2026.7.1-2` 導入済み・PATH OK・npm 更新通知は無視で問題なし
  - `openclaw onboard` を非対話実行。認証の紆余曲折:
    - `--auth-choice claude-cli` → 非推奨エラー（`anthropic-cli` に改名）
    - `--auth-choice anthropic-cli` → 「Claude CLI auth が必要」で失敗。
      原因: この Mac の Claude Code 認証は **macOS キーチェーン**保管で `~/.claude/.credentials.json` が無く、
      openclaw が読めない。橋渡しは `claude setup-token`（対話・秘密情報）
    - `--auth-choice ollama` → 既定モデル `gemma4` を無条件 DL 開始（partial 9.6GB まで進行）。
      既存ローカルモデルで十分なので**中断**、partial blob を削除
    - **`--auth-choice skip`** で再実行 → 成功
  - onboard 成功後、`openclaw models set ollama/ornith-1.5:35b` /
    `openclaw models fallbacks add ollama/qwen3.8:27b-mlx` で既定モデル設定
  - `openclaw daemon restart` → `openclaw agent --agent main --session-key smoketest -m "..."` で疎通テスト
- **結果**:
  - `~/.openclaw/openclaw.json` 生成（gateway: local / loopback / token auth / port 18789）
  - **LaunchAgent 登録**: `~/Library/LaunchAgents/ai.openclaw.gateway.plist`
    （RunAtLoad=true, KeepAlive=true。ログ `~/Library/Logs/openclaw/gateway.log`）
  - `openclaw gateway status` → Runtime: running (pid 動的) / Connectivity probe: ok
  - **疎通テスト成功**: エージェントが `ollama/ornith-1.5:35b` で応答（14.5s、フォールバック未使用）
  - 詳細は [CONFIG.md](CONFIG.md) に記録（トークンはマスク）
- **次アクション**:
  1. （ユーザー）Slack App 作成 → `openclaw channels add --channel slack ...`（[CHANNELS.md](CHANNELS.md) 参照）
  2. （ユーザー）`commands.ownerAllowFrom` に自分の Slack ユーザーIDを設定
  3. （任意・ユーザー）`claude setup-token` → `openclaw models auth login --provider anthropic --method setup-token`
     で Anthropic を追加、必要なら `openclaw models set` で主モデル切替
  4. `openclaw dashboard` で Control UI からもテスト
  5. 日本語既定化（workspace の SOUL.md / システムプロンプト）

### 2026-08-29 (2) — 現状再把握: openclaw は既にインストール済みだった

- **やったこと**:
  - 環境の再確認（`which -a node` / `nvm ls` / `brew list` / `npm ls -g` / `openclaw` 系）
  - `openclaw --help` / `gateway status` / `doctor` / `onboard --help` を実行して現状把握
- **結果（判明したこと）**:
  - **openclaw `2026.7.1-2` が既に npm グローバルで導入済み**
    （`/opt/homebrew/lib/node_modules/openclaw` → `/opt/homebrew/bin/openclaw`、Aug 29 08:07）
    ※ 前回ログの「まだ何もインストールしていない」以降にインストールされていた
  - Node は Homebrew `v25.9.0` が有効（nvm default は v22.22.2 だが `system` 選択中）。
    npm は 11.12.1 のままだが **インストールは問題なく完了**。→ Node/npm 更新は不要と判断
  - `~/.openclaw/state/openclaw.sqlite` は生成済み。ただし **`~/.openclaw/openclaw.json` は未生成**
    （= `onboard` 未完了）。`gateway.mode` 未設定、auth トークン無し、command owner 無し
  - launchd サービス（`ai.openclaw.gateway`）は未登録。Gateway 未起動。ポート 18789 は空き
  - `openclaw onboard` は実在した（`openclaw setup` のエイリアス）。前回 INSTALL.md の「要確認」は解消
  - `onboard --auth-choice` の選択肢にこのMac向けが揃っている:
    `claude-cli` / `anthropic-cli`（Claude Code サブスク）, `ollama` / `ollama-cloud`, `anthropic-api-key`（残高不足で不可）
  - `onboard --flow`: quickstart|advanced|manual|import。`--non-interactive` には `--accept-risk` 必須
  - `doctor` 指摘: memory search provider が "openai" 設定でキー無し（要対応）
- **次アクション（次セッションはここから）**:
  1. ユーザーに 4 点確認（モデルプロバイダ / onboard 実行方式 / デーモン登録の可否 / 最初のチャットapp）
  2. `openclaw onboard` 実行（デーモン登録は要ユーザー確認）
  3. `openclaw gateway status` が healthy → `openclaw dashboard` で疎通
  4. 生成された `~/.openclaw/openclaw.json`（マスク）と launchd plist を CONFIG.md に記録

### 2026-08-29 — プロジェクト開始・ドキュメント整備

- **やったこと**:
  - `~/projects/openclaw_setup/` 作成。README / INSTALL / CONFIG / CHANNELS / TROUBLESHOOTING / CLAUDE.md を作成。
  - OpenClaw の概要と公式インストール手順を Web から収集（GitHub / docs.openclaw.ai）。
  - このMacの前提環境を確認:
    - OS: macOS 26.6.2 (25G83)
    - Node.js: v25.9.0（Homebrew `/opt/homebrew/bin/node`）
    - npm: 11.12.1
    - nvm: `~/.nvm` あり
- **結果**:
  - Node は要件を（表記どおりなら）満たす。npm は 11.15 未満のため要更新 or 旧記法。
  - まだ何もインストールしていない。
- **次アクション（次セッションはここから — Node.js/npm 更新が最初）**:
  1. 現状把握: `which -a node` / `node -v` / `npm -v` / `nvm ls`（あれば）
     → どの node が使われるか（Homebrew 版 or nvm 版）を確定
  2. node 管理の一本化方針を決めて更新（※実行前にユーザー確認）:
     - Homebrew に寄せる: `brew upgrade node`
     - nvm に寄せる: `nvm install --lts && nvm alias default lts/*`
  3. npm 更新: `npm install -g npm@latest`（11.16+ にする）
  4. ここまでの結果をこの PROGRESS.md に追記
  5. `curl -fsSL https://openclaw.ai/install.sh -o /tmp/openclaw-install.sh` で中身を読む → インストール
  6. `openclaw onboard --install-daemon` → `openclaw gateway status` → `openclaw dashboard`
