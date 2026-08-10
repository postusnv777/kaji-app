# kaji-app セキュリティ制約（2026-08-10設定）

## 原則
外部サーバー（Firestore等）に送信するデータは必要最小限に絞ること。
アップロードされたファイルをそのままサーバーに保存しない。

## 家計簿CSVの場合
- CSVはブラウザのメモリ上でパースして即破棄
- Firestoreに保存するのは以下の5項目のみ
  - date（日付）
  - name（利用内容・店名）
  - amount（金額）
  - category（カテゴリ）
  - id（内部管理用ランダムID）
- 氏名・会員番号・カード番号・住所などの個人情報は読み取り自体しない

## APIキー・シークレット管理

### 方針
- Firebase のフロントエンド用 `apiKey` はクライアント公開前提のキーであり、秘匿対象ではない
- セキュリティはキーの秘匿ではなく、以下の多層防御で担保する：
  1. **APIキーのリファラー制限**（許可ドメイン: `https://postusnv777.github.io/*`, `https://kaji-app-dbd61.firebaseapp.com/*`, `https://accounts.google.com/*`）
  2. **APIキーのAPI制限**（Identity Toolkit API / Cloud Firestore API / Token Service API のみ）
  3. **Firebase Authentication**（ALLOWED_EMAILS によるログイン制限）
  4. **Firestore セキュリティルール**（許可メールアドレスのみ読み書き可）
- GitHub Secret Scanning のアラートが出た場合は「False positive」としてクローズする

### 2026-08-10 対応記録
- GitHub Secret Scanning により `index.html` 内の Google API Key 漏洩アラートを受信
- Google Cloud Console でAPIキーをローテーション（旧キー無効化 + 新キー発行）
- 新キーにリファラー制限・API制限を設定
- プランがSpark（無料）かつFirestoreルールが厳格なため、実害リスクはなかった
- 今後キーをローテーションした場合は、`index.html` の `firebaseConfig.apiKey` を更新してプッシュすればよい

### 今後の注意事項
- サーバーサイド用のシークレット（サービスアカウントキー等）は絶対にコミットしない
- Blazeプラン（従量課金）に変更する場合は、APIキー制限の設定を再確認すること
- `.env` ファイルは `.gitignore` に含めること

## 今後の機能追加時のルール
- 新しいファイル（CSV・画像・テキスト等）をアップロードする機能を追加する際は、
  リモートに送信するデータを事前に明示し、個人情報・不要データを除外してから送信すること
- サーバーに送るフィールドを必ず列挙してユーザーに確認を取ること
