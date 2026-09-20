# 夜語り (Yogatari) 開発引継ノート

最終更新: 2026-03-28 / 現在バージョン: v110

-----

## 1. バージョン履歴 (v80→v110)

|v        |主な変更内容                                                                |
|---------|----------------------------------------------------------------------|
|v80      |ベース。波状並列fetch・MediaSession・SILENT_MP3・prefetch世代管理・キュー管理等             |
|v81〜v87  |なろうサブタイトル取得の試行錯誤（v82: titleタグ分割→失敗、v85: preChapTitle導入）               |
|v88      |extractTitle をh1優先に変更（v61の動作に回帰）。カクヨムリピートバグ修正                         |
|v89      |extractTitle で .novel_subtitle → h1 の順に変更                             |
|v90      |DOM削除前に .novel_subtitle / h1 を preChapTitle として取得。titleタグ fallback追加  |
|v91      |titleタグをDOMParser経由でなくraw HTML文字列から正規表現で取得                            |
|v92      |novelTitle も raw HTML regex fallback 追加                               |
|v93      |.novel_subtitle の raw HTML regex fallback 追加（/class=“novel_subtitle”/）|
|v94      |URL入力欄 type=“url” → type=“text” + inputmode=“url”（iOS貼り付け改善）          |
|v95      |displayPage内でsegs[0]〜[4]を走査して章タイトルを補完                                 |
|v96      |segsスキャン正規表現を半角数字+章にも対応（`(?:第)?[\d漢字]+章`）                             |
|v97      |segsスキャンで日付行・N/N行をスキップ                                                |
|v98      |idx = Math.min(startSeg, segs.length-1) を復活（v95で誤削除していた）              |
|v99      |splitTextに日付・話数行フィルター追加（`2022.9/9 更新分`、`1248/1710`等）                  |
|v100     |NEWバッジをhist-titleからhist-metaの先頭に移動（長い作品名で見切れる問題修正）                    |
|v101     |checkNewChapter でカクヨムは parseKakuyomuToc を使うよう分岐                       |
|v102     |使用量バーを3種類別（Standard/WaveNet/Neural2）に分離。設定説明文「合計900万文字」追加             |
|v103     |設定タブ: 「予算アラート」リンク削除（請求リンクと同URL重複）。TTS説明文の重複削除                         |
|v104     |README④「月100万文字まで無料」→「無料枠あり。詳細は設定タブで確認」に変更                            |
|v105〜v107|睡眠タイマー実装試行→絵文字サロゲートペア問題でファイル破損。全廃棄                                    |
|v108     |v104ベースで睡眠タイマーを安全に再実装（絵文字はUnicodeエスケープ）。README追加                      |
|v109     |読書統計実装（読書日数・作品数・話数・読み上げ時間）。設定タブ最上部に4カード                               |
|v110     |読書日数を起動時記録に変更。リセットを個別選択式（日数/作品数/話数/時間/全部）に変更                          |

-----

## 2. 実装アプローチと変更内容

### ファイル構成

```
単一HTMLファイル: web-novel-reader-v110.html
総行数: 2654行 / JS: 2142行 / サイズ: 約126KB
出力先: /mnt/user-data/outputs/web-novel-reader-vXX.html
作業ベース: /home/claude/vXX_out.html
```

### なろうサブタイトル取得フロー（v80→v110の最重要変更）

```
問題: プロキシによって返るHTMLの品質が異なる
  - 良いプロキシ: h1タグあり → iPadで成功
  - 悪いプロキシ: head含む全タグなし → iPhoneで失敗（sub=0 h1=0）
  - ただしtitleタグはraw文字列に存在する

解決策（多段fallback）:
1. doc.querySelector('.novel_subtitle')  ← DOM経由
2. doc.querySelector('h1')              ← DOM経由
3. raw HTML /class="novel_subtitle"/    ← 生文字列regex
4. raw HTML /<title>/                   ← 生文字列regex（titleタグ分割）
5. segs[0]〜[4]をスキャン               ← 本文から章見出し抽出
   - `第七十二章　糾える道...` → 「章」以降を取得
   - スキップ条件: 数字/数字, YYYY.M/D, 空行, 60文字超
```

### extractPage の novelTitle/preChapTitle 取得タイミング

```javascript
// DOM削除「前」に取得（削除後は.novel_titleが消えるため）
var novelTitle = null;
var preChapTitle = null;
// .novel_title → .novel_subtitle → h1 → titleタグDOM → raw regex の順で取得

// DOM削除後
const title = extractTitle(doc, pageUrl, preChapTitle);
return { title, novelTitle, text: rawText, nextUrl: nextPageUrl };
```

### displayPage でのタイトル補完

```javascript
// segs生成後、chapterTitleが「第N話」形式のみならsegsスキャン
if (/^第\d+話?$/.test(currentChapterTitle)) {
  for (var i = 0; i < Math.min(5, segs.length); i++) {
    // 日付・N/N行をスキップ
    // 「第N章 タイトル」→ 章以降を取得
    // 〜～【「など特殊文字あり → そのまま取得
  }
}
```

### 使用量カウント（3種類別）

```javascript
function voiceCategory() {
  if (_gVoice.includes('Neural2')) return 'neural2';
  if (_gVoice.includes('Wavenet')) return 'wavenet';
  return 'standard';
}
function usageKey(cat) {
  return 'yogatari_usage_YYYY_M_' + (cat || voiceCategory());
}
// localStorage keys:
// yogatari_usage_2026_3_standard
// yogatari_usage_2026_3_wavenet
// yogatari_usage_2026_3_neural2
```

### 読書統計

```javascript
// localStorage key: yogatari_stats
// 構造: { days: {'2026-03-28': true, ...}, novels: {novelId: true, ...}, episodes: N, seconds: N }
// 記録タイミング:
//   読書日数: 起動時（即時実行）
//   作品数: goNext時（recordEpisode内）
//   話数: goNext時（手動・自動カウントダウン両方goNextを通る）
//   時間: startPlay〜togglePlay(停止)の差分
```

### 睡眠タイマー

```javascript
// 関数: toggleSleepRow() / setSleepTimer(mins) / cancelSleepTimer()
// UI: 右上🌙ボタン → ctrl-bar内 .sleep-row 表示/非表示
// 15分/30分/60分 選択 → setInterval でカウントダウン → togglePlay()で停止
// 注意: 絵文字はUnicodeエスケープ必須（\U0001f319等）→ Pythonサロゲートペア問題回避
```

-----

## 3. 具体的な実装タスク（未完了・将来候補）

|優先度 |タスク            |備考                                   |
|----|---------------|-------------------------------------|
|🟡 中 |① プロキシ成功履歴キャッシュ|前回成功したプロキシを優先順位先頭に。iPhone/iPad差異の根本緩和|
|🟡 中 |④ 複数しおり        |1作品で複数の保存位置                          |
|🟢 低 |⑤ フォントサイズ調整    |設定タブにスライダー1個                         |
|🟢 低 |章ピッカーUI強化      |現状は単純リスト                             |
|🔵 任意|失敗プロキシの一時スキップ  |レート制限対策                              |

-----

## 4. ブロッカーの記録

### 解決済み（v80〜v110）

|問題                |原因                          |解決バージョン            |
|------------------|----------------------------|-------------------|
|iPhoneでサブタイトルが取れない|プロキシがheadタグを削除したHTMLを返す     |v93（raw HTML regex）|
|iPhoneとiPadで動作が違う |波状並列fetchで勝つプロキシが端末により異なる   |v93（多段fallback）    |
|次話を押すと前の話の途中から始まる |v95でidx=Math.min()を誤削除      |v98（復活）            |
|睡眠タイマーで全操作が停止     |Pythonの絵文字サロゲートペア問題でファイル破損  |v108（Unicodeエスケープ） |
|カクヨムNEW検出が不正確     |parseSyosetuTocをカクヨムに使用していた |v101（分岐追加）         |
|iOS貼り付けが反応しない     |type=“url”がiOS Safari独自制御を発動|v94（type=“text”）   |

### 既知の未解決問題

|問題                       |状況                           |
|-------------------------|-----------------------------|
|カクヨム連続30〜50話でプロキシレート制限   |根本解決困難。エラーメッセージで対応中          |
|MediaSession長文段落（20秒超）で停止|SILENT_MP3で改善中。完全解消ならず       |
|カクヨムTOCは最初7〜30話分のみ       |Next.jsの仕様。currentEP再fetchで対応|

-----

## 5. 重要な決定事項

### 絶対厳守の制約

|制約                               |理由                                     |
|---------------------------------|---------------------------------------|
|CSS `var(--xxx)` 禁止              |iOSコピペ時に`--`→`–`(em-dash)に変換されCSSが破壊される|
|Safari lookbehind `(?<=...)` 禁止  |Safari非対応                              |
|`AbortSignal.timeout()` 禁止       |Safari非対応                              |
|単一HTMLファイル出力                     |デプロイ・配布の簡便さ優先                          |
|バージョン番号は必ずインクリメント                |ファイル名にもv番号を含める                         |
|Pythonで絵文字を文字列に含める場合はUnicodeエスケープ|サロゲートペア問題（\U0001f319等）                 |

### アーキテクチャ決定

- **APIキー非埋め込み**: ユーザーが設定画面で入力。リポジトリはpublic
- **対応サイト確定**: なろう・カクヨムのみ（ハーメルン・アルファポリス・エブリスタは非対応）
- **プロキシ方式**: 波状並列fetch（7プロキシを2+2+2+1で起動、最速応答を使用）
- **titleタグ取得**: DOMParser経由ではなく生HTML文字列に正規表現（プロキシがheadを削除する問題への対応）

### localStorage キー一覧

```
yogatari_bk                    : しおり JSON
yogatari_gkey                  : Google TTS APIキー
yogatari_gvoice                : 選択中の音声名
yogatari_rate                  : 再生速度
yogatari_pitch                 : ピッチ
yogatari_bg                    : 背景エフェクト ON/OFF ('1'/'0')
yogatari_usage_YYYY_M_standard : Standard使用文字数
yogatari_usage_YYYY_M_wavenet  : WaveNet使用文字数
yogatari_usage_YYYY_M_neural2  : Neural2使用文字数
yogatari_stats                 : 読書統計 JSON
```

### 開発ワークフロー

```
1. ベースファイルをPythonで読み込み（str.replace優先、スライス置換で対応）
2. 修正後に node --check でJS構文チェック
3. brace diff（{の数 - }の数 = 0）で括弧バランス確認
4. cp → /mnt/user-data/outputs/web-novel-reader-vXX.html
5. GitHubにアップロード（コピペ厳禁・ファイルダウンロード→アップロード）
6. iPhoneとiPadの両方で動作確認

注意:
- str.replace失敗時はスライス（html[:start] + new + html[end:]）で対応
- 絵文字は必ずUnicodeエスケープ（\U0001f319等）
- CSS変数var(--xxx)は絶対使用禁止
- Safari非対応構文: lookbehind / AbortSignal.timeout / ?.()オプショナルチェーン（closest等）
```

### 無料枠まとめ

|音声種別          |無料枠         |警告  |停止  |
|--------------|------------|----|----|
|Standard A〜D合計|400万文字/月    |360万|400万|
|WaveNet A〜D合計 |400万万文字/月   |360万|400万|
|Neural2 B〜D合計 |100万文字/月    |90万 |100万|
|**3種合計**      |**900万文字/月**|    |    |

### 現在の実装状況（v110時点）

**動作確認済みの機能:**

- なろう・カクヨムのサブタイトル2行表示（iPhone/iPad両対応）
- 睡眠タイマー（🌙ボタン / 15・30・60分 / カウントダウン表示）
- 読書統計（日数・作品数・話数・時間 / 個別リセット）
- 使用量3種別バー（Standard/WaveNet/Neural2）
- NEWバッジ位置修正（hist-meta先頭）
- splitTextフィルター（日付行・N/N行除外）
- URL入力欄貼り付け改善（type=“text”）