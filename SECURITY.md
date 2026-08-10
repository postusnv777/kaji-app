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
- Blazeプラン（従量課金）に変更する場合は、下記「Blazeプラン移行時の必須対応」に従うこと
- `.env` ファイルは `.gitignore` に含めること

### Blazeプラン（従量課金）移行時の必須対応

Sparkプラン（無料）では不正課金リスクはないが、Blazeに変更すると
APIキーを悪用した大量リクエストで課金が発生する可能性がある（データは守られるがお金だけ取られる）。

**移行前に必ず以下を実施すること：**

1. **予算アラートを設定する**
   - Firebase Console → 使用量と請求 → 予算アラート
   - 月額上限（例: 500円）を超えたら通知が届くようにする

2. **App Check を有効化する**（最重要）
   - 正規アプリからのリクエストだけをFirebaseが受け付ける仕組み
   - 不正リクエストはFirebase側でブロック → 課金されない
   - Firebase Console → App Check から設定
   - reCAPTCHA Enterprise をプロバイダとして使用

3. **リポジトリのPrivate化を検討する**（任意）
   - Private化すればAPIキーが非公開になり不正利用リスクがさらに下がる
   - ただしGitHub Pages利用にはGitHub Pro（月$4）が必要
   - App Check で防御できるなら Public のままでも可

## 今後の機能追加時のルール
- 新しいファイル（CSV・画像・テキスト等）をアップロードする機能を追加する際は、
  リモートに送信するデータを事前に明示し、個人情報・不要データを除外してから送信すること
- サーバーに送るフィールドを必ず列挙してユーザーに確認を取ること
