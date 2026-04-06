# v163 リリースノート

## 変更点

### PWA外観対応（manifest導入）

- `manifest.json` をルートに新規作成した
  - `display: standalone` — ブラウザUIを非表示にしてアプリ風表示を実現
  - `orientation: portrait` — 縦向き固定
  - `theme_color` / `background_color`: `#0a0908`（アプリ背景色と統一）
  - アイコン2種（192×192、512×512）を `purpose: any maskable` で登録
- `index.html` の `<head>` に以下を追記した
  - `<link rel="manifest" href="/manifest.json">`
  - `<link rel="apple-touch-icon" href="/icon-192.png">`
- `apple-mobile-web-app-status-bar-style` を `black-translucent` から `default` に変更した
- `icon-192.png` / `icon-512.png` を新規生成した（背景 #0a0908・中央に「夜」文字 #c8991e）

## 決定事項

- Service Worker（`sw.js`）は作成しない。本アプリはウェブからリアルタイムでコンテンツを取得する設計であり、オフライン対応は不要と判断した
- アイコンデザインはアプリの既存カラースキーム（ダーク背景・金色文字）に合わせた

## 未実装・対応なし

| 項目 | 状況 |
|------|------|
| Service Worker（`sw.js`） | 対応予定なし。オフライン動作は設計上不要。 |
| オフラインキャッシュ | Service Worker未実装のため不可。 |
