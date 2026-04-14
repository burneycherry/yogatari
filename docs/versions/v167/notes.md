# v167 リリースノート

## 変更点

### Chirp3 HD「too long」エラー時の次セグメントスキップ
- `speakIdx()` の `.catch()` ブロックに `too long` エラーの検出条件を追加
- Google TTS APIが "sentences that are too long" エラーを返した場合、同一セグメントへの無限リトライを停止し、次セグメント（`speakIdx(idx+1)`）へ自動スキップする
- トースト表示なし（再生を継続させることを優先）

### silentLoopEl 音量変更
- バックグラウンド再生中のiOSオーディオセッション維持を強化するため、`silentLoopEl` の音量を `0.001` から `0.01` に変更

## 未実装・対応予定なし

- Service Worker / オフライン対応: 引き続き対応予定なし
- SSMLポーズ最適化: 削除済み、復活予定なし
- バックグラウンド中の `Fetch is aborted` 根本対策: iOSのネットワーク制限によるもので、音量変更では解決しない別問題として継続調査
