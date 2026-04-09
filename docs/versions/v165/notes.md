# v165 リリースノート

## 変更点

### iOSバックグラウンド再生の改善

- **stopGoogleAudio() / stopGoogleAudioFull() の分割**: 段落間遷移では `el.src=''` を実行しないことでiOSオーディオセッションの断絶を防止。ユーザーによる明示的停止時のみ `stopGoogleAudioFull()` を使用
- **keepAudioSessionAlive() の改善**: `silentLoopEl` の再起動処理を追加。サイレントループが途切れた際も復旧するよう対応

### カクヨム履歴読み込みエラーの修正

- **`_resumeActive` フラグの追加**: `resumeFrom()` 実行中に `startupNewCheck()` が並走するとプロキシ競合でリクエストが失敗する問題を修正。フラグでシーケンシャル実行を保証
- **`renderHistList()` からの新着確認トリガー削除**: 履歴タブ表示のたびに新着確認が起動していた設計を廃止。起動3秒後の1回のみに統一

### iOSホーム画面アイコンの修正

- **相対パスへの統一**: GitHub Pagesサブディレクトリ配信での `/manifest.json` → `manifest.json`、`/icon-*.png` → `icon-*.png` に変更
- **manifest.json の purpose 分割**: `"purpose": "any maskable"` を仕様通り `any` と `maskable` の別エントリに分割
- **apple-touch-icon に `sizes="180x180"` を追加**

### データ管理（バックアップ／リストア）機能の追加

- **バックアップ**: `yogatari_*` キー（APIキー除く）を日付付きJSONファイルにエクスポート
- **リストア**: JSONファイルから読み込み、ページリロードで反映
- **セキュリティ**: `yogatari_gkey`（APIキー）はバックアップ対象から除外

### Google TTS APIキー管理の改善

- **キー切替時のキャッシュクリア**: `saveApiKey()` / `clearApiKey()` 両方で `clearPrefetchCache()` を呼ぶよう統一
- **期限切れエラーの再試行**: catch正規表現を絞り込み、一時的なエラーはリトライ、認証エラーのみ停止するよう修正

### Google TTS 使用量カウンターの改善

- **3カテゴリへの統合**: Standard と WaveNet が同一SKU（9D01-5995-B545）であることに合わせ、アプリ内カウントを統合（合計600万文字表示に修正）
- **起動時マイグレーション**: `yogatari_usage_*_wavenet` キーの値を `standard` に統合して wavenet キーを 0 にする一回限りの処理を追加
- **98%でKyoko自動切替**: 100%到達による課金リスクを回避するため、自動切替しきい値を98%に変更
- **持続的警告バー**: 画面下部に固定表示する警告バーを追加。90%超過・98%到達をバックグラウンド再生中でも見落とさないよう常時表示

### 設定タブUIの整理

- **Google音声を選ぶをカウンター上に移動**: 音声選択とカウント確認を近接配置
- **音声エンジン表示セクションを削除**: 選択中の音声はドロップダウンから直接確認可能なため削除
- **試聴ボタン下の重複説明文を削除**: カウンター下の説明と内容が重複していたため削除

### 確認ダイアログのバグ修正

- **openConfirm() のコールバック未実行バグ**: `closeConfirm()` が先に `_cfmCallback = null` を実行するため、その後の `if(_cfmCallback)` チェックが常に偽になりコールバックが実行されなかった。コールバックを退避してから `closeConfirm()` を呼ぶよう修正。カウントリセット・統計リセット等、全ての確認ダイアログに影響していた

### getUsage() 累積バグの修正

- **addUsage() でwavenet値が毎回累積する問題**: `getUsage('standard')` がwavenet値を加算した結果を `standard` キーに書き戻す設計により、セグメントごとにwavenet分が累積し短時間で数百万文字が水増しされるバグを修正

---

## 決定事項

- Service Worker（`sw.js`）は引き続き作成しない（CLAUDE.md で明示禁止）
- APIキーはURLに含めずヘッダー（`x-goog-api-key`）送信を維持
- Standard / WaveNet は同一SKUとして1カテゴリで管理

## 未実装・対応なし

| 項目 | 状況 |
|------|------|
| Service Worker（`sw.js`） | 対応予定なし。CLAUDE.mdで使用禁止。オフライン動作は設計上不要。 |
| オフラインキャッシュ | Service Worker未実装のため不可。 |

---

# v165.md 積み残し事項の処理記録

v164 リリース後の積み残し課題（`docs/versions/v165.md`）の対応状況。

## 要対応 → 対応完了

### 1. v164 のバージョンドキュメントが古い状態

`docs/versions/v164.md`（作業完了記録）を `docs/versions/v164/notes.md` に統合した上でファイルを削除。v164 ドキュメントは正式な状態に更新済み。

### 2. アイコンのデザイン確認

実機（iOS Safari）でのホーム画面追加時の表示をユーザーが確認。アイコンが正常に表示されることを確認済み。v165 で絶対パス→相対パスの修正を実施し、GitHub Pages環境でも正しく読み込まれるよう対応。

### 3. `docs/versions/v164/structure.json` の `sw.js` エントリ

v165 の `structure.json` から `sw.js` エントリを削除。代わりに「オフライン動作」フィーチャーに「Service Worker不使用（CLAUDE.mdで禁止）」として明記。

## 検討事項 → 対応完了

### 4. Google TTS APIキーのテスト

ユーザーによる実機テストで動作確認済み。APIキー保存・音声一覧取得・試聴・読み上げの一連フローが正常に動作することを確認。

### 5. CLAUDE.md の APP_VER 説明との整合

ユーザーが手動で確認・修正済み。
