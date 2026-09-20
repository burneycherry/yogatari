# 夜語り (Yogatari) 開発引継ノート

> 最終更新: v80 / 2026-03-23

-----

## 1. バージョン履歴サマリー

### 凡例

- ✅ バグ修正  🆕 新機能  🔧 リファクタ  🗑 削除

-----

### v54〜v56 (基盤修正期)

- ✅ `prevUrl` タイミングバグ修正 (`displayPage` に `prevUrlHint` 引数追加)
- ✅ `jumpQ()` のキャッシュクリア漏れ修正
- ✅ `splitSentences()` の空文生成バグ修正
- ✅ `startWatchdog()` タイムアウト改善（最低保証値・バッファ・上限）
- 🆕 背景エフェクト 30fps 上限 (`_bgLastFrame`)
- 🆕 `safeSessionSet()` — sessionStorage 書き込みラッパー（600KB制限・3件上限）
- 🆕 カクヨム・ハーメルン対応追加（v56時点）

### v57〜v59 (サイト対応期)

- ✅ ハーメルン `.html` URL対応
- ✅ カクヨム次話誤取得防止（同一作品エピソードのみ認識）
- 🆕 履歴2行表示（作品タイトル＋話タイトル・金色）
- 🆕 `_pickerNovelTitle` — ピッカー選択時に作品タイトルを保持

### v60〜v61 (話数・サイト整理期)

- ✅ 履歴話数ズレ修正 (`pickChapter` で `urlChapNum || item.num` 優先)
- ✅ `tryProxy()` Cloudflare検出強化（5000文字・追加パターン）
- 🗑 ハーメルン完全削除（Cloudflareで突破不可）
- 📋 対応サイト: なろう・カクヨムのみ（READMEも更新）

### v62 (話数根本対策)

- ✅ `resumeFrom` URLから話数を優先取得（旧誤保存値を無視）
- ✅ `saveBookmarkCurrent` 保存前に URL から話数を再検証
- ✅ `goPrev` / `jumpQ` でカクヨム話数0問題を修正

### v63 (sessionStorageキャッシュ衝突修正)

- ✅ **重大バグ**: `cacheKey = btoa(url).slice(0,40)` がカクヨム全作品で同一キーになる問題
- ✅ `urlHash()` (djb2) + `makePageCacheKey()` でURL全体ハッシュ化
- 🆕 起動時に旧バグキャッシュを全削除 (`cleanLegacyCache`)

### v64 (コード整理)

- 🗑 ハーメルン関連コード全削除（109行削減）
  - `parseHamelnToc()` / `SITE_RULES` の2エントリ / `chapNumFromUrl` の分岐など

### v65〜v66 (MediaSession実装)

- 🆕 **MediaSession API**: ロック画面コントロール対応
  - メタデータ（作品名・話タイトル・夜語りアイコン）表示
  - ▶/⏸・⏮/⏭（前話/次話）・±10秒スキップ
- ✅ `setPositionState()` を削除（段落終了ごとに `-0:00` になりiOSがセッション終了と誤判定）
- ✅ `msSetState('playing')` を各段落開始時・`el.onplay` で呼ぶ

### v67 (カクヨム次話取得フォールバック)

- ✅ カクヨムで `nextUrl=null` 時の多段フォールバック
1. 現在エピソードページを再fetch → nextUrl取得
1. 失敗時のみTOCをfetch
- ✅ `bNext` ボタンをカクヨムでは完全無効化しない

### v68 (長文停止・7話以降進めない修正)

- ✅ `el.onended` 直後に `msSetState('playing')` + `keepAudioSessionAlive()` 呼び出し
- ✅ TOCフォールバック「末尾=最終話」誤判定を削除

### v69 (並列fetch・速度改善)

- 🆕 **波状並列fetch**: 7プロキシを 2+2+2+1 で起動（最速応答を使用）
- 🆕 タイムアウト短縮: 20秒→8秒 / 12秒→7秒
- 🆕 prefetch遅延を1秒→3秒に延長（レート制限対策）
- ✅ `resumeFrom` でカクヨムはキャッシュなし時に `loadChapterUrl` へ委譲
- ⚠️ スクリプトエラー発生（`const` 二重宣言）→ v69修正版で対応

### v70 (話数抽出・レート制限対策)

- ✅ `goNext` で「末尾=最終話」誤判定削除
- ✅ `chapNumFromUrl` が 0 を返す場合タイトル `^第N話` から抽出
- ✅ `resumeFrom` / `goNext` でもタイトルから話数抽出
- ✅ prefetch開始を3秒遅延に変更（レート制限緩和）

### v71 (カクヨム次話取得根本改善)

- ✅ `goNext` フォールバック: TOCではなく現在エピソードページ再fetchを優先
  - TOCは最初7〜30話しか含まれないため curIdx=-1 問題が発生していた

### v72 (キュー上限管理)

- 🆕 `trimQueue()` — キュー上限10件（`QUEUE_MAX = 10`）
  - 完了済みエントリを優先削除、現在話は必ず保持
  - `queue.push` の全箇所に追加

### v73 (prefetch成功時nextUrl確定)

- ✅ **根本修正**: `prefetchNextChapter` 成功時に `nextUrl` を即確定
- ✅ カクヨムで `nextUrl=null` 時: displayPage直後に1秒後再fetchしてnextUrl取得

### v74 (UIロック競合修正)

- ✅ **競合修正**: `speakIdx` 末尾で `_nextHtmlCache.url` を `nextUrl` に補完
- ✅ `goNext` フォールバック: キャッシュ確認を最初に実行 → `disable(true)` 不要なケースを排除

### v75 (コードレビュー修正4件)

- ✅ `showSkipLabel` が常に±5秒固定だったのを動的秒数表示に修正
- 🗑 未使用の `confirmOverlay` / `showConfirm` を削除（40行削減）
- ✅ `resumeBookmark` に chapNum / novelTitle を渡すよう修正
- ✅ `displayPage` の `isKakuyomu` を `url` パラメータから判定

### v76 (goNext再入防止・MediaSession長文停止修正)

- ✅ `_goNextRunning` フラグで `goNext` 二重実行防止 (`try/finally` で確実解放)
- 🆕 `SILENT_MP3` (Base64埋め込み0.1秒無音MP3) で段落間ギャップを埋める
- ✅ `keepAudioSessionAlive()` — `el.onended` 直後に無音再生してiOSセッション維持

### v77 (MediaSession次話移動時の表示問題)

- ✅ `stopForChapterTransition()` 新設 — 章移動時は `msSetState` を呼ばない
  - `stop()` を呼ぶと `msSetState('none')` でロック画面が「再生停止中」になる問題を解消
- `goNext` / `goPrev` は `stopForChapterTransition()` を使用

### v78 (prefetch世代管理)

- ✅ **重大バグ**: 別作品に切り替え後も古いprefetchの `.then()` が実行されて `nextUrl` を上書きする問題
- 🆕 `_prefetchGeneration` カウンタ — `clearNextChapterCache()` でインクリメント
- ✅ `prefetchNextChapter` の `.then()` 内で世代チェック → 不一致なら無視

### v79〜v80 (キュー混在・メッセージ改善・スクリプトエラー修正)

- ✅ `loadChapterUrl` / `go()` で作品切り替え時にキューをリセット
- 🆕 カクヨムのプロキシ全失敗時のエラーメッセージを改善
  - 「連続読み込み後にプロキシ制限がかかる場合があります。しばらく時間をおいてから再試行してください。」
- ✅ Pythonで書き込んだ際の `\n` が実際の改行になり文字列リテラルが分断されたスクリプトエラーを修正

-----

## 2. 実装アプローチと設計方針

### ファイル構成

```
単一HTMLファイル (web-novel-reader-vXX.html)
├─ CSS (インライン style タグ)
├─ HTML (UI構造)
└─ JS (単一 script タグ)
```

### 音声エンジン

|エンジン                  |条件       |特徴                             |
|----------------------|---------|-------------------------------|
|Google Cloud TTS      |APIキー設定済み|高品質・段落ごとに音声生成                  |
|Web Speech API (Kyoko)|フォールバック  |iOSネイティブ・MediaSessionが効かないことがある|

### CORSプロキシ (波状並列fetch)

```
wave0 (0ms): allorigins/raw, allorigins/get  (ms:8000)
wave1 (400ms): cors.eu.org, yacdn.org         (ms:7000)
wave2 (800ms): codetabs, corsproxy.io         (ms:7000)
wave3 (1200ms): thingproxy.freeboard.io       (ms:7000)
最初に成功したものを使用。全失敗で reject。
```

### 3段階先読みキャッシュ

```
_nextHtmlCache   → 次話のHTMLテキスト (fetchHtml結果)
_nextPageCache   → 次話のextractPage結果
_nextAudioCache  → 次話の最初2段落の音声データ
```

### prefetch世代管理

```javascript
_prefetchGeneration  // グローバルカウンタ
// clearNextChapterCache() でインクリメント
// prefetchNextChapter().then() 内で世代チェック
// 不一致 = 別作品に切り替わったので nextUrl を更新しない
```

### chapNum (話数) 取得優先順

```
1. chapNumFromUrl(url)  // URLの末尾数字 (なろう: /1248/)
2. タイトルから抽出 (^第N話)  // カクヨム・タイトル正規化後
3. bk.chapterNum       // 保存済み値（旧バグ値の可能性あり）
4. 1                   // フォールバック
```

> カクヨムは `chapNumFromUrl` が 0 を返す（18桁IDを話数として使えないため）

### nextUrl 設定フロー（カクヨム）

```
extractPage() → SITE_RULES セレクタ → NEXT_TEXTS スキャン → null
     ↓ null の場合
prefetchNextChapter.then() で nextUrl を確定（世代チェック付き）
     ↓ それでも null で goNext が呼ばれた場合
① _nextHtmlCache.url を使う（UIロックなし・即時）
② 現在エピソード再fetch → extractPage → nextUrl
③ TOCfetch → curIdx+1 のURL
```

-----

## 3. 具体的な実装タスク（未完了・将来候補）

|優先度 |タスク                        |備考                      |
|----|---------------------------|------------------------|
|🟡 中 |なろう・カクヨムの長時間連続読み込み安定性確認    |v80でキュー混在は修正済み          |
|🟡 中 |MediaSession長文停止の完全解消確認    |SILENT_MP3+keepAliveで改善中|
|🟢 低 |章ピッカーUI強化                  |現状は単純リスト                |
|🟢 低 |履歴再開時の▶誘導UI改善              |バナー表示はあるが分かりにくい         |
|🔵 任意|プロキシ成功履歴のキャッシュ（前回成功プロキシを優先）|レート制限対策強化               |
|🔵 任意|失敗プロキシの一時スキップ機能            |同上                      |

-----

## 4. ブロッカー（技術的制約）

### ① カクヨムのプロキシブロック

- **状況**: 連続読み込み（目安30〜50話）後に全プロキシがレート制限される
- **症状**: 全プロキシ失敗エラー、リロードしても解消しない
- **回避**: 5〜10分待つと解除される
- **対応済**: カクヨム向けに判別したエラーメッセージを表示

### ② iOS の MediaSession 制約

- **状況**: audio要素が停止するとiOSがセッションを終了する
- **対応済**: `SILENT_MP3` で段落間ギャップを埋める / `keepAudioSessionAlive()`
- **残課題**: 長文段落（20秒超）でまだ停止するケースあり（改善中）

### ③ カクヨム TOC の件数制限

- **状況**: `__NEXT_DATA__` に含まれるエピソードリストは最初7〜30件のみ
- **対応済**: TOC依存をやめて現在エピソードページからnextUrlを取得

### ④ Web Speech API (Kyoko) の MediaSession 非対応

- **状況**: Kyoko使用時はロック画面ボタンが効かない場合がある
- **根本原因**: iOSはHTML5 audio要素が再生中の場合のみMediaSessionを認識
- **回避**: Google TTSキーを設定して使用

-----

## 5. 重要な決定事項

### アーキテクチャ

- **単一HTMLファイル**: デプロイ・配布の簡便さを優先。GitHub Pages で公開。
- **APIキー非埋め込み**: 設定画面からユーザーが入力。リポジトリはpublic。
- **CSS変数 `var(--xxx)` 禁止**: iOSでコピペ時にem-dashに変換されるバグのため。
- **Safari lookbehind正規表現 `(?<=...)` 禁止**: Safari非対応。

### 対応サイト（確定）

|サイト    |状態   |理由                 |
|-------|-----|-------------------|
|小説家になろう|✅ 対応 |                   |
|カクヨム   |✅ 対応 |プロキシ制限あり           |
|ハーメルン  |❌ 非対応|Cloudflare保護で本文取得不可|
|アルファポリス|❌ 非対応|robots.txt でブロック   |
|エブリスタ  |❌ 非対応|1話が複数ページに分割される構造   |

### MediaSession設計

- `stop()` → `msSetState('none')` を呼ぶ（明示的な停止）
- `stopForChapterTransition()` → `msSetState` を呼ばない（章移動中の表示維持）
- `setPositionState()` は使用しない（段落終了時に `-0:00` になりセッション終了と誤判定）

### goNext 設計

- `_goNextRunning` フラグで再入防止（`try/finally` で確実解放）
- カクヨムでは `nextUrl=null` でも `bNext` を無効化しない
- フォールバック3段階: キャッシュ → 現在EP再fetch → TOC

### queue 設計

- 上限10件 (`QUEUE_MAX`)、`trimQueue()` で古い完了済みを優先削除
- 作品切り替え時 (`currentNovelId !== novelId`) にキューをリセット

### prefetch設計

- `_prefetchGeneration` で世代管理 → 作品切り替え後の古い`.then()`を無効化
- displayPage後: カクヨムは0.5秒後、他は3秒後にprefetch開始
- nextUrl確定タイミング: prefetch成功時に即確定 (`nextUrl = url`)

-----

## 6. 主要変数・フラグリファレンス

```javascript
// 再生状態
segs[]          // 現在話の段落配列
idx             // 現在段落インデックス
playing         // 再生中フラグ
_busy           // TTS処理中フラグ（二重発話防止）
_tabSw          // タブ切り替えフラグ（600ms間オーディオコールバックをブロック）

// ナビゲーション
nextUrl         // 次話URL (null = 最終話 or 未取得)
prevUrl         // 前話URL
currentChapterUrl  // 現在話URL
currentNovelId     // 現在作品ID (toNovelId(url)の結果)
currentChapterNum  // 現在話数
_goNextRunning  // goNext再入防止フラグ

// キャッシュ
_prefetchCache  // Map<segIdx, Promise<audioSrc>> Google TTS音声
_nextHtmlCache  // {url, promise} 次話HTMLの先読み
_nextPageCache  // {url, page} 次話extractPage結果
_nextAudioCache // [{idx, promise}] 次話の最初2段落音声
_prefetchGeneration  // prefetch世代ID (clearNextChapterCache でインクリメント)

// MediaSession
_msSupported    // MediaSession API 対応フラグ
_msHandlersRegistered  // ハンドラ登録済みフラグ
```

## 7. LocalStorage キー

```
yogatari_bk           : しおり (JSON)
yogatari_gkey         : Google TTS APIキー
yogatari_gvoice       : 選択中の音声名
yogatari_rate         : 再生速度
yogatari_pitch        : ピッチ
yogatari_bg           : 背景エフェクト ON/OFF ('1' or '')
yogatari_usage_YYYY_M : 月次TTS使用文字数
```

## 8. sessionStorage キー

```
yogatari_pc_<urlHash>  : ページHTMLキャッシュ（上限3件・600KB以下）
                         ハッシュ = urlHash(url) + '_' + urlHash(url.reverse())
```

## 9. ファイル管理

```
出力先: /mnt/user-data/outputs/web-novel-reader-vXX.html
GitHub: web-novel-reader リポジトリ (public、APIキー非埋め込み)
```