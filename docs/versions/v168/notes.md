# v168 リリースノート

## 背景

2026年9月上旬より、なろう小説の読み込みが全プロキシで失敗するようになった。調査の結果、原因はなろう側の対策強化ではなく、既存の無料公開CORSプロキシ側のインフラ限界（無料枠超過・サーバー不安定）であることが判明した。

- `api.allorigins.win`: Cloudflare Error 522（オリジンサーバー無応答）
- `cors.eu.org`: Cloudflare Error 1027（運営者のCloudflare Workers無料枠上限到達によるレート制限）
- `corsproxy.io`: HTTP 401（匿名アクセス拒否、認証必須化）

いずれも「誰でも無料で使える公開プロキシ」に多数のアプリが依存し、需要が供給を上回ったことによる構造的な問題と判断した。

## 変更点

### 自前のCloudflare Workersプロキシを追加
- `PROXIES` 配列の先頭に `https://yogatari-proxy.burneycherry.workers.dev/?url=<encodeURIComponent(対象URL)>` を追加
- Worker側で対象ホストを `ncode.syosetu.com` / `novel18.syosetu.com` / `kakuyomu.jp` にホワイトリスト制限し、汎用オープンプロキシとして悪用されるリスクを抑制
- Cloudflare Workers無料プラン（1日100,000リクエストまで、超過分は課金されず翌日リセットまでブロックされるのみ）で運用
- 既存の公開プロキシ（`allorigins.win` / `cors.eu.org` / `yacdn.org` / `codetabs.com` / `corsproxy.io` / `thingproxy.freeboard.io`）は削除せずフォールバックとして温存
- 動作確認できなかった `test.cors.workers.dev` は削除

## 未実装・対応予定なし

- Service Worker / オフライン対応: 引き続き対応予定なし
- SSMLポーズ最適化: 削除済み、復活予定なし
- 自前プロキシの悪用対策（レート制限・認証トークン等）: 現状の個人利用〜小規模公開の段階では未実装。将来的にアプリを公開し利用者が増えた場合は検討が必要
