# システム構築ナレッジ・地雷回避ガイド

本ドキュメントは、過去のプロジェクト（`REGALIA` / `simplescore_web`）で得られた「システム構築時の設定要件」と「一度踏んだ地雷（ハマりどころ）」を集約したナレッジベースです。新規プロジェクト立ち上げ時に参照し、同じミスを繰り返さないためのチェックリストとして機能します。

## 1. アプリケーション・フロントエンド (Flutter / LIFF)

### 1.1 Flutter Web/iOS 共通開発
- **Webホスティング**: GitHub Pages を用いる場合、ビルド出力先を `build/web/` から `docs/` に変更するか、gh-pages ブランチにデプロイする GitHub Actions を組むのが標準。
- **ファイルI/Oの抽象化**: Web環境（`dart:html`）とモバイル環境（`dart:io`）は共存できないため、プラットフォーム固有のインポートは conditional import (`if (dart.library.html) ...`) を用いて抽象化レイヤー（Stub）を作成すること。

### 1.2 LINE LIFF と GAS の連携における重大な地雷
- **LIFFエンドポイントURLの制限**: GAS の Web App URL (`script.google.com`) を直接 LIFF エンドポイントに設定してはいけない。GASは内部で `googleusercontent.com` にリダイレクトするため、`liff.init()` がドメイン不一致で失敗する。
- **解決策**: UI（HTML/JS）は GitHub Pages 等の静的ホスティングで配信し、そこからバックエンド API として GAS を `fetch` で呼び出すアーキテクチャに分離すること。

### 1.3 Google API 連携 (OAuth, Sign-in, Drive, YouTube)
- **iOS でのスコープ要求のハング**: `google_sign_in` パッケージにおいて、iOS 環境で `requestScopes()` を呼ぶとハングする地雷がある。スコープが不足している場合は catch して `signIn()` で再度フルスコープを要求するワークアラウンドが必要。
- **Google Drive バックアップ**: AppData フォルダへの保存には `spaces: 'appDataFolder'` のスコープが必要。ユーザーには不可視の領域だが、アプリ固有の設定・バックアップ保存に最適。

## 2. バックエンド・GAS (Google Apps Script)

### 2.1 Clasp によるデプロイ運用
- **バージョン上限の地雷**: GAS は保持できるコードバージョンが **200個まで**。これを越えると `clasp push` やデプロイ更新スクリプトが謎のエラーで失敗する。定期的に GAS エディタの「プロジェクトの履歴」から不要バージョンを一括削除する運用が必要。
- **デプロイIDの変動**: バージョン上限回避などでデプロイを新規作成した場合、発行される Web App URL が変わる。これに伴い、LINE Webhook URL やフロントエンドの環境変数 (`GAS_CONFIG_URL` 等) の更新を忘れると API 通信が死ぬ。

### 2.2 データ操作と同時実行制御
- GAS でスプレッドシートを DB 代わりに使う場合、更新（UPSERT）時は決定論的なハッシュ（ID）を持たせ、差分のみを追記ログ（Audit log）として記録する設計にすると運用トラブル時の復旧が容易。

## 3. CI/CD・ビルド運用 (Codemagic & GitHub Actions)

### 3.1 iOS / App Store 審査・バージョン管理の地雷
- **【超重要】バージョン番号の再利用不可**: Apple の仕様上、「審査提出済み」または「審査承認済み」のバージョン番号（例: `1.0.12`）は、たとえビルド番号（`+n`）を上げても**同じバージョン名で再提出することは絶対にできない**。（エラー `ITMS-90062` などが発生）。
- **解決策**: 審査に一度でも提出したバージョンがあるなら、次は必ず `pubspec.yaml` のバージョン名をインクリメントする（例: `1.0.12` → `1.0.13`）。
- **ビルド番号の自動化**: `pubspec.yaml` のビルド番号を手動でいじらず、Codemagic 側で `$BUILD_NUMBER + N`（Nはオフセット）のように算出し、`flutter build ipa --build-number="..."` で上書き注入するのが最も安全。

### 3.2 Codemagic での iOS 拡張（Live Activities等）
- **Xcodeプロジェクトの設定上書き問題**: `flutter build ipa` は内部でマイグレーション（pbxprojのアップグレード等）を行うため、その後に Ruby スクリプト等で Extension ターゲット（例: LiveActivities）を追加しないと設定が破壊される。
- **解決策**: `flutter build ios --config-only --no-codesign` を先に実行して Flutter のマイグレーションを済ませてから、Ruby スクリプト等で Xcode プロジェクトを編集し、最後に本ビルドを回す。

### 3.3 GitHub Actions での PAT (Personal Access Token) 管理
- 別リポジトリ（`gh-pages` 用リポジトリ等）に push するために設定した `PAT_DEPLOY_TOKEN` は有効期限が切れると突然デプロイが止まる。GitHub からの「期限切れ間近」メールを見逃さず、トークンを再発行して Secrets を更新する運用フローをドキュメント化しておくこと。

## 4. 自動化と Agent（AI）関与最大化の原則

**「Userの関与（マニュアル作業）を最小化し、AI Agent と自動化スクリプトで完結させるアーキテクチャ」** を徹底するための設計原則です。

### 4.1 Token ベース制御による GAS/DB の直接操作
- **GUI操作の排除**: User にブラウザで GAS エディタやスプレッドシートを開かせない。
- **Agent 用 API の提供**: GAS 側に Agent 専用の読み書きエンドポイント（例: `api/admin/db-read`, `api/admin/db-write`）を用意する。
- **Token認証**: 環境変数（例: `CLAUDE_TOKEN`）に事前に共有したシークレットを持たせ、Agent は HTTP リクエスト実行用スクリプト（Python / PowerShell）を介して安全にデータ操作・設定変更を行う。

### 4.2 コマンドライン（CLI）による一括反映
- **GASのデプロイ自動化**: ブラウザからのデプロイ更新を禁止する。`clasp push` と `clasp deploy` をラップしたデプロイスクリプト（例: `deploy.sh` や `deploy.ps1`）を用意し、Agent がコマンド一発でコード変更と環境適用を完結できるようにする。
- **ローカル設定の注入**: `.env` や `pubspec.yaml` のバージョン更新など、ローカルの変更もすべて Agent がスクリプトで書き換える。

### 4.3 CI/CD によるインフラ・配信の自動化
- **状態ベースのトリガー**: GitHub Actions や Codemagic のビルドを、指定ブランチへの `git push` をトリガーとして完全自動化する。
- **環境変数の CI 注入**: CI 上で必要な秘匿情報（App Store Connect キー、デプロイ用 PAT、Supabase URL 等）は GitHub Secrets / Codemagic Environment Variables で一元管理し、ソースコードには含めない。

### 4.4 マニュアル作業の局所化と明文化（Agentからの指示）
どうしても User が手でやらなければならないプラットフォーム固有の作業は、以下のタイミングで Agent から明確な指示（URL付き）を出すようにプロンプト・ドキュメントを設計する。
1. **LINE Developers Console**: Webhook URL の初回登録・変更時。
2. **GitHub Secrets**: デプロイ用 PAT の期限切れ時、または初期設定時。
3. **App Store Connect**: 審査提出時の GUI での「提出」ボタンクリック、スクリーンショットの手動アップロード時。
4. **GAS バージョン上限**: `clasp push` 失敗時に、GASエディタの履歴画面からバージョンを一括削除するよう依頼する時。

## 5. プロジェクトテンプレートに含めるべき構成要素

次回以降のプロジェクト構築時は、以下の要素を選択的に組み込めるテンプレートを用意する。

1. **プラットフォーム・フレームワーク**
   - [ ] Flutter (Web / iOS / Android)
   - [ ] GAS (Backend API)
   - [ ] Cloud Run (Python FastAPI 等の重い処理用)
2. **連携機能**
   - [ ] LINE Webhook / LIFF
   - [ ] Google Sign-In & Google API (Drive / YouTube etc.)
3. **CI/CD パイプライン**
   - [ ] GitHub Actions (Web デプロイ用)
   - [ ] Codemagic (`codemagic.yaml` 雛形: iOS証明書、バージョン自動取得、App Store連携)
4. **ドキュメント雛形**
   - `README.md`, `ARCHITECTURE.md`, `APP_STORE_SUBMISSION_CHECKLIST.md` などの標準セット