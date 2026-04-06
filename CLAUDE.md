# CLAUDE.md

このファイルはClaudeコード（claude.ai/code）にこのリポジトリ内での作業指針を提供する。

---

## 記述ルール（最優先）

- **説明文・ガイドライン・メモはすべて日本語で記述する**
- **コード本体・変数名・ファイル内コメントは英語を維持する**
- `/edit` やコード生成時も、このルールを厳守すること

---

## プロジェクト概要

**夜語り (Yogatari)** は日本語Web小説のテキスト読み上げPWA。小説家になろう・カクヨムなどのプラットフォームからCORSプロキシ経由でコンテンツを取得し、Web Speech APIまたはGoogle Cloud TTSで読み上げる。

---

## 開発方法

ビルドシステム・バンドラー・テスト・パッケージマネージャーは一切ない。アプリ全体は `index.html` 1ファイルで完結する。

- ブラウザで `index.html` を直接開く、またはHTTPサーバー（例: `python3 -m http.server`）で配信する
- コンパイル・トランスパイル・インストール不要
- バージョンは `<title>` タグと `APP_VER` 定数で手動管理する — 大きな変更時はインクリメントすること

---

## アーキテクチャ

アプリ全体は `index.html`（約1,800行）に収録：

- **1〜11行目**: HTMLヘッド / メタタグ（PWA・ビューポート・テーマ）
- **12〜240行目**: 全CSS（インライン `<style>` ブロック）
- **241〜591行目**: HTMLボディ — 3タブ: 読む / 履歴 / 設定
- **592行目〜末尾**: 全JavaScript（インライン `<script>` ブロック）

### 主要グローバル状態

| 変数 | 用途 |
|------|------|
| `segs` | 現在の章のテキストセグメント配列 |
| `idx` | 現在読み上げ中のセグメントインデックス |
| `playing` | 再生状態（Boolean） |
| `queue` | バッチ/キュー再生用の章URL配列 |
| `selVoice` | 選択中のWeb Speech API音声 |
| `_gApiKey` | Google Cloud TTS APIキー |
| `_prefetchCache` | Google TTS音声BLOBのキャッシュMap |
| `currentChapterTitle` / `currentNovelTitle` | 表示中のメタデータ |
| `nextUrl` / `prevUrl` | 章間ナビゲーション |

### 音声パイプライン

2つのTTSモード：
1. **Web Speech API** (`speechSynthesis`): デフォルト。APIキー不要。`speakIdx()` で文単位再生。
2. **Google Cloud TTS**: 高品質。APIキー必須。`fetchGoogleAudio()` で取得し `_prefetchCache` にキャッシュ、`<audio id="gAudioEl">` で再生。

サイレント音声ループ（`<audio id="silentLoopEl">`）がiOSのバックグラウンド再生でAudioContextを維持するために使われる。

### テキスト処理

- `splitText(html)` → DOMノードをプレーンテキストセグメント（段落単位）に変換
- `splitSentences(text)` → 日本語句読点（`。！？…」』`）で分割
- `domToText(node)` → インライン要素を考慮した再帰的DOM→テキスト変換

### 小説サイトパーサー

- `parseSyosetuToc(doc, url)` — 小説家になろう目次の解析
- `parseKakuyomuToc(doc, url)` — カクヨム目次の解析
- `inferTocUrl(url)` — サイト判定とTOC URL構築
- `toNovelId(url)` — URLから小説IDを抽出

### CORSプロキシシステム

`PROXIES` 配列に7つのフォールバックサービスを列挙。`tryProxy(proxy, url)` はタイムアウト付きfetchをラップ。`fetchHtml(url)` はプロキシをローテーションしながら結果を `sessionStorage` にキャッシュする。プロキシ選択履歴は `localStorage` キー `yogatari_proxy_hist` に保存。

### localStorageキー一覧

| キー | 内容 |
|------|------|
| `yogatari_bk` | しおりJSON（小説 → 章 + セグメントインデックス） |
| `yogatari_skip_filters` | スキップフィルターのON/OFF状態 |
| `yogatari_custom_words` | ユーザー定義スキップワード |
| `yogatari_gkey` | Google TTS APIキー |
| `yogatari_gvoice` | 選択中のGoogle TTS音声名 |
| `yogatari_stat_*` | 読書統計（日数・作品数・話数・秒数） |
| `yogatari_proxy_hist` | プロキシ選択履歴 |

---

## PWA対応状況

現在のPWA実装は**メタタグのみ**による簡易対応で、`manifest.json` および Service Worker は**未実装**。

### 実装済みのPWA要素（`index.html` ヘッド部）

```html
<meta name="apple-mobile-web-app-capable" content="yes">
<meta name="apple-mobile-web-app-status-bar-style" content="black-translucent">
<meta name="theme-color" content="#0a0908">
<meta name="viewport" content="width=device-width, initial-scale=1.0, viewport-fit=cover">
```

- iOS Safari でホーム画面追加した場合、全画面アプリとして動作する（`apple-mobile-web-app-capable`）
- ステータスバーは半透明黒（`black-translucent`）
- `viewport-fit=cover` でノッチ・セーフエリアに対応。CSS で `env(safe-area-inset-top)` / `env(safe-area-inset-bottom)` を使用

### 未実装の要素

| 要素 | 状況 | 備考 |
|------|------|------|
| `manifest.json` | 未実装 | アイコン・ショートカット名・display等を定義するファイル |
| Service Worker | 未実装 | オフラインキャッシュ・バックグラウンド同期が不可 |
| Web App Manifest `<link>` | 未実装 | `<link rel="manifest" href="manifest.json">` タグなし |

音声キャッシュは `_prefetchCache`（Map）と `sessionStorage` で実装されているが、Service Workerによるオフライン対応はない。

---

## 今後の開発の進め方

### 基本方針

- **単一ファイル原則を維持する**: 新機能追加でも `index.html` 以外にJSやCSSファイルを追加しない
- **後方互換性を壊さない**: `localStorage` のキー名・データ構造を変更する場合は既存データの移行処理を入れる
- **iOS/Android両対応を意識する**: 音声再生・タッチ操作・セーフエリアは各デバイスで動作確認が必要

### 機能追加時の注意点

- **バージョン更新**: 顕著な変更を加えた場合は `<title>` の `vXXX` と `APP_VER` 定数を同時にインクリメントする
- **新しいlocalStorageキー**: 必ず `yogatari_` プレフィックスを付ける
- **スキップフィルター拡張**: `yogatari_skip_filters` と `yogatari_custom_words` の両方への影響を考慮する
- **プロキシ追加**: `PROXIES` 配列に追記するだけでフォールバックチェーンに組み込まれる

### PWAを本格対応する場合

1. `manifest.json` を作成し、`<link rel="manifest">` を `index.html` に追加する
2. Service Worker（`sw.js`）を作成してオフラインキャッシュを実装する
3. アイコン画像（192×192、512×512）を用意する

### コーディングスタイル

- 説明コメント・ドキュメントは日本語、コード・変数名・インラインコメントは英語で統一する
- 新しい関数は既存のグループ（再生制御・テキスト処理・UI更新など）の近くに配置する
- グローバル変数の追加は最小限に留め、既存の状態構造に組み込めないか先に検討する
