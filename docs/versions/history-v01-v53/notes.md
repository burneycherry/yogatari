# 夜語り (Yogatari) 開発引き継ぎノート

最終更新: 2026-03-19 / 現在バージョン: v53

-----

## 1. バージョン履歴

|v  |主な変更内容                                                                  |
|---|------------------------------------------------------------------------|
|v6 |初期完成版。Web Speech API(Kyoko)・Google TTS・CORSプロキシ・しおり・章ピッカー               |
|v7 |二重音声バグ修正。`_busy`フラグ導入、SS/Google完全分離                                     |
|v8 |WaveNet/Standard 400万文字対応。音声リストをoptgroupに整理。音声切替でキャッシュクリア               |
|v9 |設定タブにGoogleCloud使用量確認・予算アラートリンク追加                                       |
|v10|音声表示名を人間が読みやすい形に変換（`friendlyVoiceName`）。音声切替時に即反映                       |
|v11|次話プリフェッチキャッシュ導入。段落間の無音ギャップ解消                                            |
|v12|`_gVoice`を唯一の情報源に一本化。DOMは表示専用                                           |
|v13|音声リスト性別ラベル削除。試聴ボタン追加。デフォルト→Wavenet-A                                    |
|v14|`_tabSw`フラグ。タブ切替時の段落リピート防止。プロキシリスト刷新                                    |
|v15|プロキシを7種に増強。allorigins raw優先。tryProxy関数で整理                               |
|v16|touchstart改善（ボタン・スクロール操作時は無視）。手動スクロール中の追従停止。goNext後にAudioContext resume |
|v17|DOM `<audio>`要素固定化でiOSオーディオアンロック対応                                      |
|v18|設定・README・UI各所にiOS初回タップ説明を追加。アンロック成功でヒント非表示                             |
|v19|次話HTMLを読み進め中にバックグラウンド先読み。カウントダウンを5秒→3秒                                  |
|v20|速度・ピッチをlocalStorageに永続保存                                                |
|v21|予算アラートURLをbillingページに統一                                                 |
|v22|Google Fonts完全削除。Safari対応Safari対応フォールバック強化                              |
|v23|`var(--xxx)` CSS変数を全廃。直接値に置換（iOSコピペでのem-dash変換対策の根本解決）                  |
|v24|履歴読み込みをsessionStorageキャッシュで高速化。NEW+話数バッジのHTML追加                         |
|v25|`body{}` 3重定義を1つに統合。`.rdm-body`のドット欠落修正（READMEスクロール不可を解消）               |
|v26|設定タブに自己ダウンロードボタン追加（後にv30で削除）                                            |
|v27|バージョン番号のみ変更（ファイル名にv番号付与対応）                                              |
|v28|selfDownload()でen-dash自動修復ロジック追加                                        |
|v29|`var(--xxx)` CSS変数を全廃（コードレベルで完全除去）。:root{}ブロック削除                        |
|v30|ダウンロードボタンをUIから削除（selfDownload関数はv51まで残留→デッドコード化）                        |
|v31|←前話ボタン追加。■停止ボタン削除。前へ/目次/次へナビ行を本文から除去。履歴から自動再生しない                        |
|v32|`currentChapterUrl`変数の宣言欠落を修正（起動フリーズの原因）                                |
|v33|`splitText`の正規表現構文エラーを修正（起動Scriptエラーの原因）                                |
|v34|次話→「次話→」。残り時間表示追加。1文移動(`prevSent`/`nextSent`)。タップゾーン追加                  |
|v35|残り時間をピッチ行の下・音声の上に移動                                                     |
|v36|音声セレクトと残り時間を横並びに（voice-remain-row）                                      |
|v37|⏮⏭を1段落移動に変更。タップゾーンを5秒スキップに変更。しおり保存を10→5段落に変更。README更新                   |
|v38|README使い方を実際の動作に合わせて修正（自動→▶ボタンで開始など）                                    |
|v39|履歴から読み込み時のprevUrl設定。段落移動と5秒スキップの分離。スキップラベル画面表示                          |
|v40|タップゾーンのpointer-eventsをJSで動的制御（setBadge連動）                               |
|v41|NEWバッジCSS追加。tocURL自動推定（inferTocUrl）。起動3秒後バックグラウンド新着チェック。NEWの永続保存・既読クリア  |
|v42|最終話後のカウントダウンリピートをnextUrl=nullで止める。エラーメッセージを「最終話まで読み終わりました」に             |
|v43|v42の文字列リテラル内改行によるSyntaxError修正                                          |
|v44|なろうエラーページ検出追加。タップゾーンの高さ調整・ctrl-barにz-index:20                           |
|v45|背景エフェクトエンジン追加（Canvas星空・月・流れ星・雲・雨・雪・天気API連動）                             |
|v46|背景z-index:-1に修正。UI要素に半透明背景。bodyのbackgroundをJS側で切替                       |
|v47|リアル雲描画（ベジェ曲線）。月齢計算・月相描画・月の位置を時刻で変化                                      |
|v48|天気API・位置情報を完全削除。夜固定・ランダム天気・オーロラ追加。設定を1ボタンに簡略化                           |
|v49|`(?<=...)`lookbehind正規表現をSafari非対応のため書き換え。`AbortSignal.timeout`のSafari対応|
|v50|旧天気連動の残骸setTimeoutブロック（wBtn/wRow/updBgStatus未定義）を削除                     |
|v51|`_speakLock`→`_busy`に置換。`selfDownload()`削除。`prevUrl`二重管理を統一             |
|v52|最終話誤検知修正（`存在しない`/`エラーが発生しました`を削除、200文字未満条件追加）                           |
|v53|goNextのエラー判定を「本当の最終話」と「プロキシ失敗」に分離。プロキシ失敗時はnextUrlを保持                    |

-----

## 2. 実装アプローチと技術仕様

### ファイル構成

- **単一HTMLファイル** `/mnt/user-data/outputs/web-novel-reader-v53.html`（約2585行）
- 外部依存なし（Google Fonts削除済み・APIキー不要で動作）
- GitHub公開リポジトリ: `web-novel-reader`（APIキー未埋め込みのため公開のまま運用）

### 音声エンジン

```
優先順位: Google TTS（APIキーあり）> Kyoko（Web Speech API）

Google TTS:
  - DOM固定<audio>要素でiOSアンロック対応
  - プリフェッチキャッシュ（Map）で段落間のギャップ解消
  - 次話HTML・音声を読み進め中に先読み
  - _busy フラグで二重発話防止

Kyoko:
  - SpeechSynthesisUtterance
  - iOSバックグラウンド対策: AudioContext + visibilitychange + touchstart

音声変数:
  _gVoice が唯一の情報源（DOMは表示専用）
  localStorage: yogatari_gvoice
```

### CORSプロキシ（順番に試行）

```
1. https://api.allorigins.win/raw?url=  (タイムアウト:20秒)
2. https://api.allorigins.win/get?url=  (JSON, 20秒)
3. https://cors.eu.org/
4. https://yacdn.org/serve/
5. https://api.codetabs.com/v1/proxy?quest=
6. https://corsproxy.io/?url=
7. https://thingproxy.freeboard.io/fetch/
```

### localStorage キー一覧

```
yogatari_bk       : しおり（{novelId: {novelId,novelTitle,chapterUrl,...,tocUrl,totalChap,hasNew,newDiff}}）
yogatari_gkey     : Google TTS APIキー
yogatari_gvoice   : 選択中の音声名
yogatari_rate     : 再生速度
yogatari_pitch    : ピッチ
yogatari_usage_YYYY_M : 月次TTS使用文字数
yogatari_bg       : 背景エフェクトON/OFF ('1'/'0')
```

### CSS変数廃止（重要）

v29以降、`var(--xxx)` 形式を**完全廃止**。iOSのコピペでem-dash変換されるため。
すべて直接値（`#0a0908`等）で記述。`:root{}`ブロックなし。

-----

## 3. 現在の機能一覧

### 読むタブ

- URL入力（なろう目次URL対応・章ピッカー表示）
- fetchHtmlでCORSプロキシ経由取得
- 本文抽出（SITE_RULESで各サイト対応）
- しおりバナー（前回の続きから再開）
- 進捗バー・段落番号表示

### 再生コントロール

- ←前話 / ⏮(1段落戻る) / ▶(再生/停止) / ⏭(1段落進む) / 次話→
- 速度（0.5〜2.5）・ピッチ（0.5〜2.0）ステッパー
- 残り時間表示（250文字/分 × 速度で計算）
- 音声選択（右横に横並び）
- 5秒スキップ（画面左右1/3タップ、再生中のみ有効）
- スキップ時に画面中央に「+5秒/-5秒」ラベル表示

### 履歴タブ

- 自動しおり（5段落ごと）
- 起動3秒後にバックグラウンド新着チェック（1時間ごと）
- NEW+話数バッジ（赤・点滅）
- 既読クリア（作品を開いた時点でNEW消去）
- 履歴から読み込み→自動再生しない（▶で開始）

### 設定タブ

- 夜空アニメーション ON/OFF
- 音声エンジン表示（Kyoko / Google TTS）
- 今月の使用量バー（WaveNet: 400万文字、Neural2: 100万文字）
- Google Cloud Console リンク3種
- Google TTS APIキー入力・音声選択（WaveNet/Standard/Neural2）
- 試聴ボタン

### 背景エフェクト（Canvas, z-index:-1）

- 夜空グラデーション固定
- 月相（朔望月29.53日で実際の月齢計算）・時刻で東→西に移動
- 星200個（瞬き）・流れ星（ランダム出現）
- オーロラ（緑〜シアン縦縞、ゆらぎ）
- 雲（ベジェ曲線の積雲、夜空色）
- 雨・雪
- 天気はランダム（晴れ/曇り/雨/雪/オーロラを30〜90秒で切替）

-----

## 4. ブロッカーの記録

### 解決済み

|問題                 |原因                           |解決バージョン         |
|-------------------|-----------------------------|----------------|
|em-dashでCSS破壊      |iOSコピペで`--`→`–`変換            |v29（CSS変数廃止）    |
|Safariで全体が白/崩れ     |`body{}`3重定義・`.rdm-body`ドット欠落|v25             |
|起動フリーズ             |`currentChapterUrl`変数宣言欠落    |v32             |
|起動Scriptエラー        |splitTextの正規表現構文エラー          |v33             |
|SafariでScript error|lookbehind正規表現 `(?<=...)` 非対応|v49             |
|SafariでScript error|wBtn/wRow/updBgStatus未定義の残骸  |v50             |
|最終話の誤検知            |`存在しない`が本文にも含まれる             |v52             |
|プロキシ失敗→最終話扱い       |isFetchErrの条件が広すぎた           |v53             |
|二重音声・オーム返し         |Google TTS+Kyokoが同時起動        |v7（_busyフラグ）    |
|タブ切替で段落リピート        |`_tabSw`フラグ未実装               |v14             |
|次ページ自動再生失敗         |iOSオーディオアンロック未対応             |v17（DOM audio固定）|

### 既知の未解決・注意点

- なろう以外のサイトは動作保証なし（サイトルール追加で対応可能）
- ロック画面のメディアコントロール未対応（AVSession相当）
- バックグラウンド再生は不安定（iOSの制限）
- なろうが主要プロキシをブロックし始めると取得失敗（プロキシ更新で対応）

-----

## 5. 重要な決定事項

### アーキテクチャ

- **単一HTMLファイル**を維持（外部ファイル・サーバー不要）
- `var(--xxx)` CSS変数は使用禁止（iOSコピペ問題の根本対策）
- `(?<=...)` lookbehind正規表現は使用禁止（Safari非対応）
- `AbortSignal.timeout()`は`typeof`チェック後に使用

### 音声管理

- `_gVoice`が唯一の情報源、DOMは表示専用
- `_busy`フラグで二重発話防止（`_speakLock`は廃止済み）
- 次話プリフェッチは`_nextHtmlCache`/`_nextPageCache`/`_nextAudioCache`の3段階

### UI/UX方針

- バージョン番号は修正のたびに必ず上げる
- ファイル名にもv番号を含める（`web-novel-reader-v53.html`）
- CSS変数の代わりに直接値を使用
- 履歴からの読み込みは自動再生しない（ユーザーが▶を押して開始）
- タップゾーンは再生中のみ有効（setBadge連動でpointer-events切替）

### 廃止した機能・コード

- `selfDownload()` → 削除済み（v51）
- `_speakLock` → `_busy`に統一（v51）
- 天気API（Open-Meteo）・位置情報（geolocation）→ 削除済み（v48）
- `prevUrl`の二重管理 → displayPage経由に統一（v51）
- `wBtn`/`wRow`/`updBgStatus`/`_bgWeather`/`_bgWeatherData` → 削除済み（v50）

### コスト・無料枠

- WaveNet/Standard: 月400万文字無料
- Neural2: 月100万文字無料
- アプリ内カウンター: WaveNet→360万警告/400万停止、Neural2→90万警告/100万停止

-----

## 次の開発候補タスク

- ロック画面メディアコントロール対応（MediaSession API）
- 履歴から開いた際の「▶ボタンで開始」UIの改善
- 背景エフェクトのフレームレート制御（バッテリー配慮）
- 使い方説明の簡略化
- なろう以外のサイトの動作検証・ルール追加