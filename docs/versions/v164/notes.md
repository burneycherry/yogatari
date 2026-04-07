# v164 リリースノート

## 変更点

### Google TTS 改善

- **APIキー送信方法の変更**: `?key=` URLクエリパラメータから `x-goog-api-key` HTTPヘッダーに変更した。ブラウザ履歴・URLログへのAPIキー露出を低減する
- **音声カテゴリ判定の修正**: `fetchGoogleAudio()` がリクエスト開始時点のカテゴリ (`voiceCategory()`) を固定し `addUsage()` に渡すよう変更した。APIリクエスト中にユーザーが音声を切り替えた場合に使用量が誤ったカテゴリに計上されるバグを修正
- **`addUsage()` に `cat` 引数を追加**: 呼び出し元から正確なカテゴリを受け取れるよう修正。引数なしの場合は従来通り `voiceCategory()` にフォールバック
- **Chirp3 音声名の表示対応**: `friendlyVoiceName()` に Chirp3 パターンを追加（例: `ja-JP-Chirp3-HD-Achernar` → `Chirp3-Achernar`）。従来の正規表現では Chirp3 音声がフルID表示になっていた

### 履歴読み込みの安定性改善

- **プロキシ過負荷の防止**: `renderHistList()` での `checkNewChapter()` 並列起動を廃止し、`startupNewCheck()` 経由のシーケンシャル実行に統一した。複数の新着確認が同時にプロキシを叩いてレート制限を引き起こし、直後の `resumeFrom()` が失敗する問題を修正
- **`startupNewCheck()` 重複起動防止**: `_startupCheckRunning` ガードを追加し、同一関数の並列実行を防止した
- **`_goNextImpl` と `resumeFrom` の競合状態修正**: `_navGeneration` カウンタを導入。`resumeFrom()` 開始時にインクリメントし、進行中の `_goNextImpl` が世代変化を検知して `displayPage()` / `startPlay()` を実行せずに終了する。この競合によりブックマークのノベルIDとチャプターURLが異なるサイトのものになるデータ破損を防止

### Canvas 高DPI対応（前バージョンより継続）

- `bgResize()` が `devicePixelRatio` を使用してキャンバスを物理ピクセル解像度で描画するよう修正済み

### Google TTS 実測CPM対応（前バージョンより継続）

- `_liveCpm` 変数による実測文字/分（EMA）を `updProg()` の残り時間計算に使用済み

## 決定事項

- Google TTS APIキーはブラウザ側で平文管理される構造上、完全な隠蔽は不可能だが、URLクエリ露出の排除により一般的なリスクを低減した
- Service Worker（`sw.js`）は引き続き作成しない

## 未実装・対応なし

| 項目 | 状況 |
|------|------|
| Service Worker（`sw.js`） | 対応予定なし。オフライン動作は設計上不要。 |
| オフラインキャッシュ | Service Worker未実装のため不可。 |
