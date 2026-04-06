# v162 リリースノート

## 決定事項

- プロジェクト全体を単一ファイル（`index.html`）に維持する方針を継続する
- Claude Code向け作業指針ファイル `CLAUDE.md` をリポジトリに追加した
- CLAUDE.md の記述ルール: 説明・ガイドラインは日本語、コード・変数名は英語で統一する
- バージョン管理ドキュメント構造（`/docs/versions/vXXX/`）を導入した

## 実装済み機能

- Web Speech API による日本語テキスト読み上げ（`speechSynthesis`）
- Google Cloud TTS による高品質読み上げ（APIキー必須）
- 小説家になろう（ncode.syosetu.com）の目次・章解析
- カクヨム（kakuyomu.jp）の目次・章解析
- 7サービスのCORSプロキシフォールバック
- しおり・履歴・読書統計のlocalStorage永続化
- スキップフィルター（定義済みパターン＋カスタムワード）
- スリープタイマー（15・30・60分）
- iOSバックグラウンド再生（silentLoopEl + Media Session API）
- 章キュー再生・自動進行
- Google TTS音声プリフェッチキャッシュ

## 未実装・今後の課題

| 項目 | 状況 |
|------|------|
| `manifest.json` | 未作成。PWAインストールに必要。 |
| Service Worker（`sw.js`） | 未作成。オフライン動作不可。 |
| Web App Manifest `<link>` | `index.html` に `<link rel="manifest">` タグなし。 |

## 設計意図

- **単一ファイル構成**: 配信インフラを不要にするため、HTML・CSS・JSをすべて `index.html` に収録する
- **プロキシフォールバック**: 外部サービスの可用性に依存しないよう、複数CORSプロキシを順番に試行する
- **iOS対応の優先**: 自動再生制限・バックグラウンド再生の制約が最も厳しいiOS Safariを基準に実装する
