# ストレージキー一覧（詳細）

> `CLAUDE.md` から参照される詳細資料。新しいキーは必ず `yogatari_` プレフィックスを付け、この一覧に追記すること。

## localStorageキー一覧

| キー | 内容 |
|------|------|
| `yogatari_bk` | しおりJSON（小説 → 章 + セグメントインデックス） |
| `yogatari_skip_filters` | スキップフィルターのON/OFF状態 |
| `yogatari_custom_words` | ユーザー定義スキップワード |
| `yogatari_gkey` | Google TTS APIキー |
| `yogatari_gvoice` | 選択中のGoogle TTS音声名 |
| `yogatari_engine` | 再生エンジンの選択（`'google'` / `'device'`）。未設定時はAPIキーの有無から導出する |
| `yogatari_stats` | 読書統計（日数・作品数・話数・秒数） |
| `yogatari_usage_<年>_<月>_<カテゴリ>` | Google TTS月間使用文字数 |
| `yogatari_proxy_hist` | プロキシ選択履歴 |
| `yogatari_rate` / `yogatari_pitch` | 読み上げ速度・ピッチ |
| `yogatari_reading_dict` / `yogatari_rdict_on` | 読み間違い辞書とそのON/OFF |
| `yogatari_bg` | 背景エフェクトのON/OFF |
| `yogatari_debug_mode` | デバッグモードのON/OFF（既定OFF） |
| `yogatari_debug_hud` | 計測ログ画面の開閉状態 |
| `yogatari_debug_log` | 計測ログの本文（最大220行） |

## sessionStorageキー一覧

| キー | 内容 |
|------|------|
| `yogatari_pc_<hash>` | 履歴からの再開（`resumeFrom()`）で保存するページHTMLキャッシュ。キーは `makePageCacheKey()` が生成するURLハッシュ。最大3件・1件600KBまで |

> ※ sessionStorageはタブセッション中のみ有効。ページを閉じると消去される。
