# セットアップ手順（手動作業ガイド）

このドキュメントは、Agentが自動化できないプラットフォーム固有の作業手順をまとめたものです。
**初回セットアップ時に一度だけ実行してください。**

---

## Step 1: Google Apps Script プロジェクトの作成

1. [Google Apps Script](https://script.google.com/) を開く
2. 「新しいプロジェクト」をクリック
3. プロジェクト名を `Menu` に変更
4. 「ツール」→「スクリプトのプロパティ」で以下を追加:
   | プロパティ名 | 値 |
   |---|---|
   | `LINE_CHANNEL_SECRET` | ← Step 2 で取得 |
   | `LINE_CHANNEL_ACCESS_TOKEN` | ← Step 2 で取得 |
   | `LIFF_CHANNEL_ID` | ← Step 3 で取得 |
5. スクリプトIDを確認: ブラウザURLの `/projects/<ScriptID>/edit` から取得

6. プロジェクトルートの `.clasp.json` を作成（`clasp.json.sample` をコピー）:
   ```json
   {
     "scriptId": "YOUR_SCRIPT_ID",
     "rootDir": "./gas"
   }
   ```

7. ターミナルで実行:
   ```powershell
   clasp login
   .\deploy.ps1
   ```

8. GASエディタで `initialize()` 関数を手動実行してシートを初期化する

---

## Step 2: LINE Messaging API チャンネルの作成

1. [LINE Developers Console](https://developers.line.biz/console/) を開く
2. 「プロバイダーを作成」または既存プロバイダーを選択
3. 「新しいチャンネルを作成」→「Messaging API」
4. チャンネル名: `Menu` など適宜
5. 作成後、「チャンネル基本設定」から以下を取得:
   - **チャンネルシークレット** → GAS Script Properties の `LINE_CHANNEL_SECRET` に設定
6. 「Messaging API設定」タブから:
   - **チャンネルアクセストークン（長期）** を発行 → `LINE_CHANNEL_ACCESS_TOKEN` に設定
   - **Webhook URL** に GAS Web App URL を設定:
     `https://script.google.com/macros/s/<ScriptID>/exec`
   - 「Webhookの利用」を ON にする
   - 「応答メッセージ」を OFF にする（GASが返信するため）

---

## Step 3: LIFF アプリの作成

1. LINE Developers Console で上記チャンネルを開く
2. 「LIFF」タブ → 「追加」
3. サイズ: `Full`
4. エンドポイントURL: GitHub Pages の URL
   - 例: `https://<username>.github.io/<repo>/`
   - GitHub Pages が有効化されていない場合は Step 4 先行
5. 作成後「LIFF ID」をコピーして `lib/config.dart` の `liffId` を更新

---

## Step 4: GitHub Secrets の設定

[GitHub リポジトリ設定](https://github.com/) → Settings → Secrets and variables → Actions

| シークレット名 | 値 |
|---|---|
| `GAS_URL` | GAS Web App URL（デプロイ後に取得） |
| `LIFF_ID` | Step 3 で取得した LIFF ID |

> **注**: `PAT_DEPLOY_TOKEN` は現在の CI 構成では不要になりました（同リポジトリへの push を使用）。

---

## Step 5: GitHub Pages の有効化

1. GitHub リポジトリ → Settings → Pages
2. Source: `Deploy from a branch`
3. Branch: `main` / `docs/`
4. 「Save」をクリック
5. 公開 URL が発行される（LIFF エンドポイントに設定）

---

## Step 6: 動作確認チェックリスト

- [ ] `.\deploy.ps1` が成功する
- [ ] GAS Web App URL に `?action=listMenus` でアクセスして `{"ok":true,"data":[]}` が返る
- [ ] `git push` → GitHub Actions が成功 → GitHub Pages が更新される
- [ ] LINE チャットにメニュー名を含むメッセージを送信 → Bot が返信する
- [ ] LIFF アプリを LINE から開いてメニュー一覧が表示される

---

## トラブルシューティング

| 症状 | 原因と対処 |
|---|---|
| `clasp push` が失敗 | GAS のバージョンが 200 個の上限に到達している。GASエディタ「プロジェクトの履歴」→ 古いバージョンを一括削除 |
| LIFF が `liff.init()` でエラー | エンドポイントURLが HTTPS でない、またはドメイン不一致。GitHub Pages の URL を正確に設定 |
| LINE Webhook が届かない | GAS Web App のアクセス設定が「全員（匿名を含む）」になっているか確認 |
| GitHub Actions が失敗 | `GAS_URL` / `LIFF_ID` シークレットが未設定の場合はデフォルト値でビルドされるが、実運用には設定が必要 |
