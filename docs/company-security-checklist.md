# セキュリティチェックリスト（定期確認用）

> 非エンジニアの方でも確認できるよう、日本語で記載しています。
> 各項目の設定変更はエンジニアまたは管理者が行ってください。

## 初回セットアップ時

### ゲートウェイ（サーバー）の保護

- [ ] `gateway.bind` が `"loopback"` になっているか（外部からアクセス不可にする）
- [ ] `gateway.auth.token` に十分長い（32文字以上）ランダム文字列を設定したか
- [ ] `gateway.auth.mode` が `"token"` になっているか

### ツール制限

- [ ] `tools.exec.security` が `"deny"` になっているか（コマンド実行の禁止）
- [ ] `tools.elevated.enabled` が `false` になっているか（管理者権限昇格の無効化）
- [ ] `tools.fs.workspaceOnly` が `true` になっているか（ファイル操作の範囲限定）

### サンドボックス

- [ ] `agents.defaults.sandbox.mode` が `"all"` になっているか（全セッションを隔離実行）

### プラグイン

- [ ] `plugins.allow` に必要なプラグインのみ記載されているか（空配列 = 全プラグイン無効）

### チャネル（メッセージアプリ）

- [ ] 使用するチャネルのDMポリシーが `"pairing"` または `"allowlist"` になっているか
- [ ] グループチャットで `requireMention: true` が設定されているか（@メンション必須）

## 月次確認

- [ ] `openclaw security audit --deep` を実行して警告がないか確認
- [ ] 不要なチャネル接続（使っていないメッセージアプリ連携）がないか確認
- [ ] エージェントのセッションログに不審なやりとりがないか確認
- [ ] 認証トークンを定期的にローテーション（変更）しているか
- [ ] Node.jsのバージョンが22.12.0以上であることを確認（`node --version`）

## チャネル（メッセージアプリ）追加時

- [ ] DMポリシーを設定したか（`pairing` or `allowlist`）
- [ ] グループポリシーを設定したか（`allowlist` 推奨）
- [ ] `requireMention` を `true` に設定したか（グループの場合）
- [ ] `allowFrom` に許可するメンバーのIDのみ記載したか
- [ ] 追加後に `openclaw security audit` を再実行したか

## インシデント発生時

1. ゲートウェイを即座に停止する: `pkill -f openclaw-gateway`
2. セッションログを保全する: `~/.openclaw/agents/` 配下
3. 認証トークンを即座に変更する
4. `openclaw security audit --deep` で影響範囲を確認する
5. 必要に応じてチャネルのボットトークンも再発行する

## コマンド早見表

| やりたいこと | コマンド |
|---|---|
| セキュリティ監査（基本） | `openclaw security audit` |
| セキュリティ監査（詳細） | `openclaw security audit --deep` |
| セキュリティ問題の自動修復 | `openclaw security audit --fix` |
| チャネル接続状態の確認 | `openclaw channels status --probe` |
| サンドボックス状態の確認 | `openclaw sandbox explain` |
| 総合診断 | `openclaw doctor` |
