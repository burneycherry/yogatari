# 夜語り (Yogatari) 開発引継ノート

最終更新: 2026-03-30 / 現在バージョン: v139

-----

## 1. バージョン履歴 (v111→v139)

|v        |主な変更内容                                                                                                                                                                                           |
|---------|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
|v111     |プロキシ成功履歴キャッシュ（yogatari_proxy_hist）。getHostKey/getProxyCacheIdx/setProxyCacheIdx/clearProxyCacheIdx追加。fetchHtmlでcachedIdxはdelay=0で即時起動                                                            |
|v112     |省略設定UI追加。CONFIGURABLE_FILTERSにbasic/customグループ。splitTextをmap+filterに変更。yogatari_skip_filters                                                                                                     |
|v113     |省略設定を設定タブの独立セクションに。全8項目がON/OFF可能に。説明バナー追加                                                                                                                                                        |
|v114     |更新日時パターンのバグ修正。1314/1400等のページ数が誤って更新日時判定されていた問題。`/^\d{4}[/]\d{1,2}(…)(\s                                                                                                                          |
|v115     |v114の続き（`^(19                                                                                                                                                                                    |
|v116     |NEWバッジのtotalChap更新修正。saveBookmarkCurrentでmax(currentChapterNum, prevBk.totalChap, _newChapCache.total)に更新。読み進めたらNEWクリア                                                                           |
|v117     |区切り記号行フィルター追加（◇◆＊ーなど記号のみ行）。basicグループ                                                                                                                                                             |
|v118     |区切り記号フィルターをbasicグループに移動                                                                                                                                                                          |
|v119     |Google TTS音声にStudio/Chirp3 HD追加。使用量バーを3→5種別に（Standard/WaveNet/Neural2/Studio/Chirp3）。Chirp3 HD選択時は速度・ピッチスライダーグレーアウト。isChirp3()/updStepperState()追加。fetchGoogleAudioでChirp3はspeakingRate/pitch送らない|
|v120     |Studio（ja-JP非対応確定）を削除。使用量説明文修正                                                                                                                                                                   |
|v121〜v124|動的音声取得試行→Safari全壊バグ。全廃棄                                                                                                                                                                          |
|v125     |v119ベースに戻してloadVoicestry/catchラップ、初期プレースホルダー修正のみ                                                                                                                                                 |
|v126     |saveApiKey内にXHRで音声リスト取得追加（グローバル関数なし・fetch不使用）。実機でSafari全壊再発→廃棄                                                                                                                                   |
|v127     |v125ベース。saveApiKey内XHRを.then()チェーン方式で実装。async/awaitなし・constグローバルなし。実機テストでChirp3 HD 30種類取得確認。Studio ja-JP非対応確定                                                                                    |
|v128     |Chirp3 HDの全音声を実機確認済みリストで静的更新（30種類）。Studio完全削除。使用量バー4種別（Standard/WaveNet/Neural2/Chirp3 HD）、合計1000万文字                                                                                             |
|v129     |①説明文「1000万文字」に修正②stopForChapterTransition/stopにrecordTime追加（時間カウントバグ修正）③chaphead/errreportフィルター追加                                                                                                |
|v130     |splitTextを変換処理に変更。章タイトル行を省略しつつ同行に話タイトルがあれば話以降を残す                                                                                                                                                 |
|v131     |睡眠タイマー完了時にstopSilentAudio()+msSetState(‘none’)+黒オーバーレイ表示。iOSが自動ロックするよう改善                                                                                                                         |
|v132     |READMEに睡眠タイマーの黒画面・バッテリー節約説明追加                                                                                                                                                                    |
|v133     |bgTip（バナー）の文言削除。README「バックグラウンド再生」の誤った自動ロック設定案内を修正                                                                                                                                               |
|v134     |startSilentAudio()をAudioContext無音ループ→`<audio>`要素ループに変更。silentLoopElを追加。電源ボタンロック後も読み上げ継続が確認された重要バージョン                                                                                             |
|v135     |v133ベースでbgTip完全削除（v134の成果を確認前にv133に戻した過渡期）                                                                                                                                                       |
|v136     |v134ベースでbgTip完全削除。現在の安定ベース                                                                                                                                                                       |
|v137     |epheadフィルター追加（第X話/閑話/幕間/番外をサブタイトルに）。postmsgフィルター追加。_pendingEpTitleでサブタイトル自動反映                                                                                                                    |
|v138     |epheadを本文から削除しない設定に変更（読み上げ継続）                                                                                                                                                                    |
|v139     |chapheadとepheadを「章・話タイトル行」1項目に統合                                                                                                                                                                 |

-----

## 2. 実装アプローチと変更内容

### ファイル構成

```
単一HTMLファイル: web-novel-reader-v139.html
総サイズ: 135,448文字 / JS: 93,754文字
作業ベース: /home/claude/vXX_out.html
出力先: /mnt/user-data/outputs/web-novel-reader-vXX.html
```

### startSilentAudio の重要変更（v134）

**変更前（v133以前）：AudioContext無音ループ**

```javascript
// AudioContextのBufferSourceを無限ループ
// → iOSが「音楽再生中」とみなし画面消灯を妨げた
// → 電源ボタンでロックするとセッション切れて次話停止
```

**変更後（v134以降）：`<audio>`要素ループ**

```javascript
// silentLoopEl（<audio loop>）でSILENT_MP3をvolume=0.001で再生
// → iOSがミュージックアプリと同様に扱う
// → 電源ボタンロック後も読み上げ・自動進行が継続 ✅
// → AudioContextはresume()用に初期化のみ（closeしない）
```

実機確認済み：電源ボタンでロック → 読み上げ継続 → 次話自動進行継続

### 省略フィルター（CONFIGURABLE_FILTERS）

```javascript
const CONFIGURABLE_FILTERS = [
  // ===== 基本フィルター =====
  { key:'pagenum',    // 100/120形式のページ数行
  { key:'updatedate', // 2026年1月1日 更新分 等
  { key:'navi',       // 前へ/次へ/目次等のナビ行
  { key:'naroui',     // 応援する/ハートをクリック等
  { key:'chaphead',   // 第X章行省略＋第X話/閑話/幕間/番外をサブタイトルに
  { key:'divider',    // ◇◆＊ーなど記号のみの区切り行
  // ===== カスタムフィルター =====
  { key:'bookmark',   // ブックマーク・お気に入り依頼
  { key:'review',     // 感想・レビュー依頼
  { key:'rating',     // 評価・応援依頼（☆→★）
  { key:'activity',   // 活動報告
  { key:'errreport',  // 誤字脱字報告
  { key:'postmsg',    // 投稿・更新のあいさつ文
];
```

### splitText の変換処理（v130〜）

```javascript
// 章タイトルと話タイトルの処理
_chapRe: /^(?:第)?[数字・漢数字]+章/
_epRe:   /((?:第)?[数字・漢数字]+話.*)/
_epHeadRe: /^(?:第X話|閑話|幕間|番外)/

// chapheadがONの場合：
// 「第2章〇〇第49話〇〇」→「第49話〇〇」に変換（読み上げ継続）
// 「第一章　旅立ち」→ null（削除）
// 「第X話〇〇」→ _pendingEpTitleに記録＋そのまま読み上げ
// 閑話/幕間/番外 → _pendingEpTitleに記録＋そのまま読み上げ

var _pendingEpTitle = null; // displayPageでサブタイトルに反映
```

### 更新日時フィルターの最終パターン

```javascript
// 正しいパターン（v114以降）
/^\d{4}[\/]\d{1,2}([\/]\d{1,2})?(\s|$)  // 4桁/1-2桁 形式
|^\d{4}年                                  // 2026年〜
|^\d{4}\.\d{1,2}[\/]                       // 2022.9/ 形式
|^(更新分|更新日)/                          // 行頭のみ

// 解決した問題：1314/1400 等のページ数行が誤マッチしていた
// ページ数（N/M）は「/後が3桁以上」= updatedate不一致
// 日付（YYYY/M/D）は「/後が1-2桁」= updatedate一致
```

### Google TTS 音声リスト取得（v127）

```javascript
// saveApiKey()内でXHRを使用（async/awaitなし）
var _xhr = new XMLHttpRequest();
_xhr.open('GET', 'https://texttospeech.googleapis.com/v1/voices?languageCode=ja-JP&key='+key);
_xhr.onload = function() { /* 音声セレクト更新 */ };
_xhr.send();
// → 保存と同時に実在する音声リストを取得
// → Studio ja-JPは存在しないため表示されない（確認済み）
```

-----

## 3. 具体的な実装タスク（未完了・将来候補）

|優先度 |タスク                 |備考                                        |
|----|--------------------|------------------------------------------|
|🟡 中 |章・話タイトルフィルターの詳細リスト表示|省略される/されないの一覧をUI上に表示（v140として一度実装→廃棄、再実装待ち）|
|🟡 中 |複数しおり（1作品で複数保存位置）   |                                          |
|🟢 低 |フォントサイズ調整（設定タブスライダー）|                                          |
|🔵 任意|失敗プロキシの一時スキップ       |レート制限対策                                   |

-----

## 4. ブロッカーの記録

### 解決済み（v111〜v139）

|問題                     |原因                                      |解決バージョン                 |
|-----------------------|----------------------------------------|------------------------|
|電源ボタンロックで読み上げ停止        |AudioContextがiOSにサスペンドされる               |v134（`<audio>`ループに変更）   |
|ページ数行1314/1400が更新日時と誤判定|`^\d{4}[\./年]\d`が4桁数字/数字にマッチ            |v114（スラッシュ後の桁数で区別）      |
|Safari全壊（操作不能）         |動的音声取得でasync関数・constグローバル追加             |v125（v119ベースに戻す）        |
|Safari全壊（v126）         |XHRでfetch非同期処理がsaveApiKey外に関数追加         |v127（saveApiKey内に完全封じ込め）|
|Studio ja-JP音声エラー      |`ja-JP-Studio-Q`等が存在しない                 |v128（APIで確認→非対応確定・削除）   |
|読み上げ時間が累積されない          |stopForChapterTransitionでrecordTime未呼び出し|v129                    |
|第X章行に第X話が続く場合に全削除      |splitTextがフィルター処理を行単位で完結                |v130（変換処理に変更）           |
|bgTip赤いバナーが空枠として残る     |v133でテキスト削除したが要素は残存                     |v136（HTML・CSS・JS完全削除）   |

### 既知の未解決問題

|問題                       |状況                                                              |
|-------------------------|----------------------------------------------------------------|
|短い段落でオートロックタイマーがリセット     |`<audio>.play()`ごとにiOSがタイマーリセット。根本解決は困難（複数段落を1音声に結合すれば解決するがリスク高）|
|カクヨム連続30〜50話でプロキシレート制限   |エラーメッセージで対応中                                                    |
|MediaSession長文段落（20秒超）で停止|SILENT_MP3で改善中                                                  |

-----

## 5. 重要な決定事項

### 絶対厳守の制約

|制約                             |理由                               |
|-------------------------------|---------------------------------|
|CSS `var(--xxx)` 禁止            |iOSコピペ時に`--`→`–`(em-dash)変換でCSS破壊|
|Safari lookbehind `(?<=...)` 禁止|Safari非対応                        |
|`AbortSignal.timeout()` 禁止     |Safari非対応                        |
|`async function` グローバル追加禁止     |Safariでスクリプト全壊の実績あり（v121〜124）    |
|`const` グローバル追加は最小限            |初期化順序の干渉リスク                      |
|単一HTMLファイル出力                   |デプロイ・配布の簡便さ優先                    |
|バージョン番号は必ずインクリメント              |ファイル名にもv番号含める                    |
|Pythonで絵文字はUnicodeエスケープ        |サロゲートペア問題（\U0001f319等）           |

### startSilentAudioの役割（重要）

```
目的: iOSのAudioSessionを維持して次話自動進行を継続させる
方式: <audio loop>（silentLoopEl）でSILENT_MP3をvolume=0.001でループ
副作用（旧版）: AudioContextループは画面消灯を妨げていた（意図せぬ副作用）
現状（v134〜）: <audio>ループはiOSがミュージックアプリ扱い→画面ロック許可+セッション維持
```

### 動的音声取得の失敗パターン（v121〜v126）

Safari全壊を引き起こした共通原因：

1. `async function` をグローバルスコープに追加
1. `const VOICE_CACHE_KEY` 等のグローバル定数追加
1. 起動時のsetTimeout内からDOM操作（設定タブ外のselect要素）
1. スクリプト末尾への大きな関数追加（ネストした関数宣言含む）

**安全な追加方法：** 既存関数内に`.then()`チェーンで追記のみ

### Google TTS 音声（確定情報）

```
対応: Standard A-D / WaveNet A-D / Neural2 B-D / Chirp3 HD 30種類
非対応: Studio ja-JP（APIが返さない、確認済み）
Neural2-A: 存在しない（B・C・Dのみ）
Chirp3 HD: 速度・ピッチパラメータ非対応（送ると400エラー）
```

Chirp3 HDの全音声名（実機APIで確認済み）：

- 女性: Achernar, Aoede, Autonoe, Callirrhoe, Despina, Erinome, Gacrux, Kore, Laomedeia, Leda, Pulcherrima, Sulafat, Vindemiatrix, Zephyr
- 男性: Achird, Algenib, Algieba, Alnilam, Charon, Enceladus, Fenrir, Iapetus, Orus, Puck, Rasalgethi, Sadachbia, Sadaltager, Schedar, Umbriel, Zubenelgenubi

### localStorage キー一覧（v139時点）

```
yogatari_bk           : しおり JSON
yogatari_gkey         : Google TTS APIキー
yogatari_gvoice       : 選択中の音声名
yogatari_rate         : 再生速度
yogatari_pitch        : ピッチ
yogatari_bg           : 背景エフェクト ON/OFF
yogatari_usage_YYYY_M_standard : Standard使用文字数
yogatari_usage_YYYY_M_wavenet  : WaveNet使用文字数
yogatari_usage_YYYY_M_neural2  : Neural2使用文字数
yogatari_usage_YYYY_M_chirp3   : Chirp3 HD使用文字数
yogatari_stats        : 読書統計 JSON
yogatari_proxy_hist   : プロキシ成功履歴 {hostname: idx}
yogatari_skip_filters : 省略フィルター設定 {key: bool}
yogatari_voices_cache : 音声リストキャッシュ（XHR取得結果）
```

### 無料枠まとめ（v128以降）

|音声            |無料枠          |警告  |停止  |
|--------------|-------------|----|----|
|Standard A-D合計|400万文字/月     |360万|400万|
|WaveNet A-D合計 |400万文字/月     |360万|400万|
|Neural2 B-D合計 |100万文字/月     |90万 |100万|
|Chirp3 HD合計   |100万文字/月     |90万 |100万|
|**合計**        |**1000万文字/月**|    |    |

### 睡眠タイマー完了時の動作（v131〜）

```
togglePlay() → 再生停止
stopSilentAudio() → silentLoopEl停止
msSetState('none') → MediaSession解放
黒オーバーレイ表示（sleepOverlay）
↓
iOSが自動ロック設定に従って消灯
↓
タップで復帰（dismissSleepOverlay）
```

### 開発ワークフロー

```
1. ベースファイルをPythonで読み込み（str.replace優先）
2. 修正後にnode --checkでJS構文チェック（JSをscriptタグから抽出して実行）
3. brace diff（{の数 - }の数 = 0）確認
4. 重要な確認: 新規グローバル関数・async・constの追加がないか
5. cp → /mnt/user-data/outputs/web-novel-reader-vXX.html
6. GitHubにアップロード（コピペ厳禁・ファイルDL→アップロード）
7. iPhone/iPadの両方で動作確認
```

### Safari破壊チェックリスト（PR前に必ず確認）

```python
# 以下が増えていたら要注意
import re
old_funcs = set(re.findall(r'^function (\w+)', old_js, re.MULTILINE))
new_funcs = set(re.findall(r'^function (\w+)', new_js, re.MULTILINE))
print("新規グローバル関数:", new_funcs - old_funcs)  # 空であること

print("async追加:", new_js.count('async ') - old_js.count('async '))  # 0
print("fetch追加:", new_js.count('fetch(') - old_js.count('fetch('))  # 0
```