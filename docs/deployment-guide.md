# デプロイメントガイド（日本語）

このガイドでは、OpenClaw を使ったAIエージェント社員を
Slack と LINE で稼働させるまでの全手順を説明します。

---

## 全体の流れ

```
1. 事前準備（Node.js, OpenClaw インストール）
2. AIモデルの APIキー取得
3. Slack ボットの作成
4. LINE ボットの作成
5. 設定ファイルの配置
6. ワークスペースの配置
7. ゲートウェイの起動
8. 動作確認
9. チームメンバーの登録
```

所要時間の目安: 1〜2時間（各サービスのアカウント作成含む）

---

## 1. 事前準備

### Node.js のインストール

OpenClaw には Node.js 22.12.0 以上が必要です。

```bash
# バージョン確認
node --version
# v22.12.0 以上であればOK
```

未インストールの場合は https://nodejs.org/ からLTS版をダウンロードしてください。

### OpenClaw のインストール

```bash
# 自動インストール（推奨）
curl -fsSL https://openclaw.ai/install.sh | bash

# または npm で手動インストール
npm install -g openclaw@latest

# インストール確認
openclaw --version
```

### LINE プラグインのインストール

LINE を使う場合、追加プラグインが必要です。

```bash
openclaw plugins install @openclaw/line
```

---

## 2. AIモデルの APIキー取得

エージェントの「頭脳」となるAIモデルを選び、APIキーを取得します。

### Anthropic（Claude）を使う場合（推奨）

1. https://console.anthropic.com/ にアクセス
2. アカウントを作成（またはログイン）
3. 左メニューの「API Keys」をクリック
4. 「Create Key」でAPIキーを生成
5. キーを安全な場所にメモ（`sk-ant-...` で始まる文字列）

### APIキーの設定

```bash
openclaw config set providers.anthropic.apiKey "sk-ant-あなたのAPIキー"
```

または環境変数で設定:

```bash
export ANTHROPIC_API_KEY="sk-ant-あなたのAPIキー"
```

---

## 3. Slack ボットの作成

### 3-1. Slack アプリの作成

1. https://api.slack.com/apps にアクセス
2. 「Create New App」→「From scratch」を選択
3. アプリ名を入力（例: 「社員アシスタント」）
4. ワークスペースを選択して「Create App」

### 3-2. Socket Mode の有効化

1. 左メニュー「Socket Mode」をクリック
2. 「Enable Socket Mode」をオンにする
3. App-Level Token の名前を入力（例: `openclaw-socket`）
4. スコープ `connections:write` を追加
5. 「Generate」して App Token をメモ（`xapp-...` で始まる）

### 3-3. Bot Token の取得

1. 左メニュー「OAuth & Permissions」をクリック
2. 「Bot Token Scopes」に以下を追加:
   - `app_mentions:read`
   - `channels:history`
   - `channels:read`
   - `chat:write`
   - `groups:history`
   - `groups:read`
   - `im:history`
   - `im:read`
   - `im:write`
   - `mpim:history`
   - `reactions:read`
   - `users:read`
3. 「Install to Workspace」をクリック
4. Bot User OAuth Token をメモ（`xoxb-...` で始まる）

### 3-4. イベント購読の設定

1. 左メニュー「Event Subscriptions」をクリック
2. 「Enable Events」をオンにする
3. 「Subscribe to bot events」に以下を追加:
   - `app_mention`
   - `message.channels`
   - `message.groups`
   - `message.im`
   - `message.mpim`
   - `reaction_added`

### 3-5. メンバーIDの確認方法

Slack でメンバーのプロフィールを開き、「...」→「メンバーIDをコピー」で取得できます。
`U` で始まる文字列です（例: `U01ABCDEF12`）。

---

## 4. LINE ボットの作成

### 4-1. LINE Developers アカウント

1. https://developers.line.biz/console/ にアクセス
2. LINE アカウントでログイン
3. プロバイダーを作成（例: 「あなたの会社名」）

### 4-2. Messaging API チャネルの作成

1. プロバイダー内で「Create a new channel」→「Messaging API」を選択
2. チャネル情報を入力:
   - チャネル名: 「社員アシスタント」
   - チャネル説明: 「社内AIアシスタント」
   - カテゴリ: 適切なものを選択
3. 「Create」をクリック

### 4-3. トークンの取得

1. 「Messaging API」タブを開く
2. **Channel access token** → 「Issue」で発行してメモ
3. 「Basic settings」タブを開く
4. **Channel secret** をメモ

### 4-4. Webhook の設定

1. 「Messaging API」タブで「Webhook URL」に以下を入力:

```
https://あなたのサーバーのアドレス/line/webhook
```

2. 「Use webhook」をオンにする
3. 「Verify」で接続テスト（ゲートウェイ起動後に実施）

**注意:** LINE の Webhook は HTTPS が必須です。
ローカル開発の場合は ngrok や Tailscale Funnel を使ってください。

### 4-5. 応答設定の変更

LINE Official Account Manager で以下を無効化:
- 「あいさつメッセージ」→ オフ
- 「応答メッセージ」→ オフ

（OpenClaw が応答するため、LINE 標準の自動応答は不要です）

---

## 5. 設定ファイルの配置

### 5-1. テンプレートをコピー

```bash
# 設定ディレクトリの作成
mkdir -p ~/.openclaw

# テンプレートをコピー
cp config/openclaw.example.json ~/.openclaw/openclaw.json
```

### 5-2. プレースホルダーの置き換え

`~/.openclaw/openclaw.json` を開き、以下の `CHANGE-ME` を実際の値に置き換えます:

| プレースホルダー | 置き換える値 |
|---|---|
| `CHANGE-ME-use-openssl-rand-base64-48` | ゲートウェイの認証トークン（下記コマンドで生成） |
| `CHANGE-ME-xapp-your-slack-app-token` | Slack の App Token (`xapp-...`) |
| `CHANGE-ME-xoxb-your-slack-bot-token` | Slack の Bot Token (`xoxb-...`) |
| `CHANGE-ME-slack-user-id-1` | 許可する Slack メンバーID (`U...`) |
| `CHANGE-ME-slack-user-id-2` | 許可する Slack メンバーID（不要なら行ごと削除） |
| `CHANGE-ME-slack-channel-id` | 許可する Slack チャネルID (`C...`) |
| `CHANGE-ME-line-channel-access-token` | LINE の Channel Access Token |
| `CHANGE-ME-line-channel-secret` | LINE の Channel Secret |

### 認証トークンの生成

```bash
openssl rand -base64 48
# 出力された文字列をそのまま gateway.auth.token に設定
```

---

## 6. ワークスペースの配置

```bash
# ワークスペースディレクトリの作成
mkdir -p ~/.openclaw/workspace

# ファイルをコピー
cp workspace/AGENTS.md ~/.openclaw/workspace/
cp workspace/SOUL.md ~/.openclaw/workspace/
cp workspace/MEMORY.md ~/.openclaw/workspace/
```

`~/.openclaw/workspace/MEMORY.md` を開いて、会社固有の情報を記入してください。

---

## 7. ゲートウェイの起動

### 手動起動（テスト用）

```bash
openclaw gateway run --bind loopback --port 18789
```

### バックグラウンド起動（本番用）

```bash
# バックグラウンドで起動
nohup openclaw gateway run --bind loopback --port 18789 --force > /tmp/openclaw-gateway.log 2>&1 &

# ログの確認
tail -f /tmp/openclaw-gateway.log
```

### 自動起動の設定（サーバー再起動時に自動で立ち上がるようにする）

```bash
# macOS の場合
openclaw onboard --install-daemon

# Linux の場合（systemd）
openclaw onboard --install-daemon
```

---

## 8. 動作確認

以下のコマンドを順番に実行して、すべて正常であることを確認します。

```bash
# ゲートウェイの状態確認
openclaw health

# チャネル接続状態の確認
openclaw channels status --probe

# セキュリティ監査
openclaw security audit --deep

# 総合診断
openclaw doctor
```

### 期待される結果

- `openclaw health` → 「Gateway is running」と表示される
- `openclaw channels status --probe` → Slack と LINE が「connected」と表示される
- `openclaw security audit --deep` → 重大な警告がないこと

---

## 9. チームメンバーの登録

### Slack（allowlist 方式の場合）

設定ファイルの `channels.slack.allowFrom` にメンバーの Slack ID を追加します:

```json
"allowFrom": ["U01ABCDEF12", "U02GHIJKL34", "U03MNOPQR56"]
```

設定変更後、ゲートウェイを再起動:

```bash
# 実行中のゲートウェイを停止
pkill -f openclaw-gateway

# 再起動
nohup openclaw gateway run --bind loopback --port 18789 --force > /tmp/openclaw-gateway.log 2>&1 &
```

### LINE（pairing 方式の場合）

1. メンバーが LINE でボットにメッセージを送ります
2. ボットが認証コード（ペアリングコード）を返します
3. 管理者がターミナルで承認します:

```bash
# 保留中のペアリングリクエスト一覧
openclaw pairing list line

# 承認
openclaw pairing approve line コード番号
```

---

## トラブルシューティング

### ゲートウェイが起動しない

```bash
# ログを確認
tail -100 /tmp/openclaw-gateway.log

# ポートが使用中でないか確認
ss -ltnp | grep 18789

# 設定ファイルの構文チェック
openclaw doctor
```

### Slack に接続できない

- App Token (`xapp-`) と Bot Token (`xoxb-`) が正しいか確認
- Socket Mode が有効になっているか確認
- Event Subscriptions が設定されているか確認

### LINE に接続できない

- Channel Access Token と Channel Secret が正しいか確認
- Webhook URL が HTTPS で始まっているか確認
- 「Use webhook」がオンになっているか確認
- LINE プラグインがインストールされているか確認:
  ```bash
  openclaw plugins list
  ```

### メッセージを送っても反応がない

```bash
# チャネル状態の確認
openclaw channels status --probe

# ログをリアルタイムで監視しながらメッセージを送ってみる
tail -f /tmp/openclaw-gateway.log
```

### セキュリティ監査で警告が出る

```bash
# 自動修復を試す
openclaw security audit --fix

# それでも解決しない場合は手動で設定を確認
openclaw config get
```

---

## コマンド早見表

| やりたいこと | コマンド |
|---|---|
| ゲートウェイ起動 | `openclaw gateway run --bind loopback --port 18789` |
| ゲートウェイ停止 | `pkill -f openclaw-gateway` |
| 状態確認 | `openclaw health` |
| チャネル確認 | `openclaw channels status --probe` |
| セキュリティ監査 | `openclaw security audit --deep` |
| 設定表示 | `openclaw config get` |
| 設定変更 | `openclaw config set キー名 値` |
| ペアリング一覧 | `openclaw pairing list チャネル名` |
| ペアリング承認 | `openclaw pairing approve チャネル名 コード` |
| プラグイン一覧 | `openclaw plugins list` |
| ログ確認 | `tail -f /tmp/openclaw-gateway.log` |
| 総合診断 | `openclaw doctor` |
| アップデート | `npm install -g openclaw@latest` |

---

## バックアップ

定期的に以下をバックアップしてください:

```bash
# 設定ファイル
cp ~/.openclaw/openclaw.json ~/backups/

# ワークスペース（エージェントの知識・記憶）
cp -r ~/.openclaw/workspace/ ~/backups/

# 認証情報
cp -r ~/.openclaw/credentials/ ~/backups/
```
