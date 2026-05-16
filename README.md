# family-chat

Vue3 + Firebase Authentication + Realtime Database で構成した家族用チャットです。

## セットアップ

1. 依存関係をインストール
```bash
npm install
```
2. `.env.example` をコピーして `.env.local` を作成し、Firebase値を設定
3. 開発起動
```bash
npm run dev
```

## GitHub Pages デプロイ

1. GitHub リポジトリの `Settings > Pages` で `Build and deployment` を `GitHub Actions` に設定
2. `Settings > Secrets and variables > Actions` に以下を登録
- `VITE_FIREBASE_API_KEY`
- `VITE_FIREBASE_AUTH_DOMAIN`
- `VITE_FIREBASE_DATABASE_URL`
- `VITE_FIREBASE_PROJECT_ID`
- `VITE_FIREBASE_STORAGE_BUCKET`
- `VITE_FIREBASE_MESSAGING_SENDER_ID`
- `VITE_FIREBASE_APP_ID`
3. `main` ブランチへ push

## Firebase Realtime Database Rules 例

```json
{
  "rules": {
    ".read": "auth != null",
    ".write": "auth != null"
  }
}
```
