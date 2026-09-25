# 音声パイプライン（詳細）

> `CLAUDE.md` から参照される詳細資料。音声制御（`gAudioEl` / `silentLoopEl` / Web Speech / Media Session / デバッグモード）を変更する前に読むこと。
> 実機ログ・実HTMLで確認した事実を記録する。推測で書き換えないこと。

## 音声パイプライン

2つのTTSモード：
1. **Web Speech API** (`speechSynthesis`): デフォルト。APIキー不要。`speakIdx()` で文単位再生。
2. **Google Cloud TTS**: 高品質。APIキー必須。`fetchGoogleAudio()` で取得し `_prefetchCache` にキャッシュ、`<audio id="gAudioEl">` で再生。

サイレント音声ループ（`<audio id="silentLoopEl">`）はセグメント間の無音区間を埋めるために使われる。Androidではこれに加えて、ページがオーディオフォーカスを保持し続けるための役割を持つ（下記「Androidバックグラウンド再生の実測挙動」参照）。

### iOSバックグラウンド再生の実測挙動

実機ログで確認済みの事実。推測で書き換えないこと。

- iOSはバックグラウンド移行時に `silentLoopEl` を**必ず一時停止する**。これは正常動作であり、中断の兆候ではない
- バックグラウンド再生のオーディオセッションを保持しているのは `gAudioEl` 単独である。無音ループはアンカーではない
- バックグラウンドでの `src` 差し替えと `play()` は正常に機能する
- 着信・他アプリの音声などで中断されると、iOSはこのページのバックグラウンド音声を拒否するようになる。**この拒否はJavaScriptから観測できない**
  - `play()` はresolveし、`play` / `playing` イベントも発火する
  - `error` は発生せず、`readyState` は4、`AudioContext` も `running` のまま
  - **`currentTime` だけが進まなくなる。これが唯一の検出手段である**
- 中断時の `AudioContext` の状態は `suspended` であり、`interrupted` ではない

この検出と復旧は `checkAudioProgress()` / `markAudioSessionLost()` / `rebuildAudioElements()` が担う。復旧は音声要素を作り直してセッションを取り直す方式で、▶ 押下時（ユーザー操作の内側）にのみ実行する。

### Web Speech APIのバックグラウンド挙動（実測）

実機ログで確認済みの事実。推測で書き換えないこと。

**Web Speech API は iOS・Android いずれでもバックグラウンド再生できない。** 中断の有無は無関係で、画面を消した時点で停止する。止まり方はプラットフォームで異なる。

> ※ 読み上げに使われる音声は端末側に依存する。iOSは `Kyoko`、Androidは端末搭載のTTSエンジン（Google音声など）。UI上の「Kyoko」表記はiOS前提の名称であり、Androidでは実際の音声名と一致しない。

| | 挙動 | アプリからの観測 |
|------|------|------|
| iOS | エンジンが無言で凍結。フォアグラウンド復帰時に未読部分を破棄して `end` を発火する | イベントが飛ばないため検出できない |
| Android | バックグラウンド移行の約30ms後に `error: interrupted` で打ち切られる | エラーは飛ぶが `utt.onerror` が `interrupted` を無視するため停止に気付けない |

- `utt.onerror` が `interrupted` / `canceled` を無視するのは、`SS.cancel()` でも同じエラーが出るため二重読み上げを防ぐ意図によるもの。この分岐を変更する場合は二重読み上げの再発に注意すること
- 復帰は `visibilitychange` ハンドラが担い、フォアグラウンドに戻ると `speakIdx()` から再開する
- Google TTSモードと異なり `<audio>` 要素が鳴っていないため、オーディオセッションを保持する主体が存在しない。無音ループを鳴らし直す対策は原理的に効かない
- `checkAudioProgress()` は `gAudioEl` の再生位置のみを見るため、Web Speechモードでは動作しない（誤検知もしない）

### Androidバックグラウンド再生の実測挙動（解決済み）

実機ログとChromiumのソースで確認済みの事実。推測で書き換えないこと。

**原因は「5秒以下の音声は一時音として扱われる」というChromiumの仕様。** `media/base/media_content_type.cc` に規定がある。

```cpp
const int kMinimumContentDurationSecs = 5;
// duration > 5秒 → kPersistent（完全なオーディオフォーカス・バックグラウンド維持）
// duration ≤ 5秒 → kTransient（通知音と同じ一時音扱い・維持しない）
```

- 本アプリは段落ごとに別の音声ファイルを再生するため、**段落ごとに新しいオーディオフォーカス要求が発生する**。段落の多くは5秒以下のため一時音に分類され、バックグラウンド再生権が維持されなかった
- 症状は**画面消灯から15.5〜17.7秒で停止**。停止は必ず**新しい段落の開始時**（フォーカスを要求し直す瞬間）に起きる。段落の途中で止まることはない
- 停止の現れ方はiOSと同じ2種類。`G:playing` 直後に `currentTime` が凍結するか、`pause` イベントが `ended=false` で飛ぶ
- 検出は `checkAudioProgress()` がそのまま機能する。一方 **`rebuildAudioElements()` による▶復旧はAndroidでは効かない**（フォーカスの問題であり要素の問題ではないため）

**対策**: `silentLoopEl` の音源を、MP3のデコードに失敗した環境でのみ**実行時生成の6秒無音WAV**へ切り替える（`silentWavSrc()` / `_silentMode`）。5秒を超えるためChromeが `kPersistent` と判定し、ページが完全なオーディオフォーカスを保持し続ける。

- WAVはPCMのためコーデックを要さず、MP3が再生できない環境でも確実に鳴る
- 判定はUA判定ではなく**デコード失敗の検知**で行う。MP3が再生できるiOSは従来どおりMP3を使い、挙動は一切変わらない
- しきい値は「5秒**超**」なので、ちょうど5秒では不足する
- 対策後、Androidで停止しなくなり、章をまたぐ自動進行も動作。**Chromeのメディア通知が表示されるようになる**（`kPersistent` として認識された証拠）

### デバッグモード（計測用）

設定タブ最下部のトグルで切り替える。バックグラウンド再生の不具合調査で使った計測機能であり、**既定はOFF**。

- ONのときだけ、音声要素の `play` / `pause` のラッパー、メディアイベントの購読、計測タイマーを生成する。**OFFの間はこれらを一切生成しないため、再生経路に計測処理が入らない**
- ONにすると、トップバーのバージョン表記のタップで計測ログ画面を開閉できる。OFFのときタップしても反応しない
- ログは `currentTime` の進行、アプリ起点の操作（`G.play() APP` 等）とプラットフォーム起点のイベント（`G:pause` 等）の区別、Media Sessionの各ハンドラ呼び出しを記録する
- **この機能を削除・簡略化しないこと。** 画面に出ないため未使用コードに見えるが、iOS・Androidのバックグラウンド再生の問題はいずれもこのログでしか原因を特定できなかった

### その他の実測事項

- Android Chrome では `SILENT_MP3` のデータURIがデコードできない（`MEDIA_ERR_SRC_NOT_SUPPORTED`）。このため `keepAudioSessionAlive()` が段落間を埋めるために `gAudioEl` へ同じ音源を渡す処理は、Android では行わない（`_silentMode !== 'mp3'` で抑止）
- **一時停止時は無音ループも必ず止めること。** 鳴らしたままだとページは音を出し続けている扱いになり、プラットフォームはメディアセッションを再生中のままにする。ロック画面のボタンは見た目が ▶ でも実体は一時停止のままで、押すと `play` ではなく `pause` ハンドラが呼ばれ再生できない
- `stopSilentAudio()` で `src` を空にしてはならない。空srcはページURLの読み込みとして失敗し、そのエラーが `startSilentAudio()` のデコード判定に届いて `_silentMode` を誤って切り替える
- **`volume` はiOSでは読み取り専用で無視されるが、Androidでは有効**。`unlockAudio()` が解除用の無音再生のために `volume=0` にする箇所があり、この復帰処理を成功時のみに書くとAndroidで全編無音になる。`startPlay()` は `unlockAudio()` の直後に `stopGoogleAudioFull()` を呼ぶため、解除用の `play()` は必ず中断され `AbortError` で終わる。iOSでは `volume` が無視されるためこの不具合は表面化しない
