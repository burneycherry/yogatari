# v166 リリースノート

## 変更点

### 98%超過時のKyoko自動切替
- `speakIdx()` の分岐条件を `hasGoogleKey() && !isOverLimit()` に変更
- 98%超過時はGoogle TTSをスキップしてWebSpeech API（Kyoko）に直接フォールバックする
- `startPlay()` および `prefetchNext()` も同様に `isOverLimit()` チェックを追加し、超過時はAPI呼び出しを行わない

### 超過時のバッジ・モード表示切替
- `updModeDisplay()` に `isOverLimit()` 分岐を追加
- 超過時: バッジ → `🔊 Kyoko（制限中）`、設定タブ → `🔊 Kyoko（制限中フォールバック）`
- `addUsage()` から `updModeDisplay()` を呼び出すよう変更し、再生中に制限到達した際にリアルタイムで表示が切り替わる

### 98%エラーのトースト抑制
- catchブロックにエラーメッセージが「98%」を含む場合の分岐を追加
- トーストを表示せず静かに `speakIdx()` を再呼び出し（新条件によりKyoko経路へ進む）
- 下部警告バーのみで通知する設計を維持

### SSMLポーズ最適化機能の削除
- `isSsmlPauseOn()` 関数を削除
- `toggleSsmlPause()` 関数を削除
- `toSsml()` 内のbreakタグ挿入処理（句点・三点リーダー・感嘆符・疑問符）を削除
- 設定タブの「ポーズ最適化」UI行を削除
- `initRdict()` 内の ssmlPauseBtn 初期化コードを削除
- `fetchGoogleAudio()` の条件式から `isSsmlPauseOn()` を除去
- 会話文ピッチ変更（isSsmlDialogueOn）は引き続き有効

## 未実装・対応予定なし

- Service Worker / オフライン対応: 引き続き対応予定なし
- SSMLポーズ最適化: 削除済み、復活予定なし
