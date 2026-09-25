# CLAUDE.md

このファイルはClaude Code（claude.ai/code）にこのリポジトリ内での作業指針を提供する。
詳細な仕様・実測事項は `docs/guide/` 配下に分けてある。**該当領域を変更する前に、対応する資料を必ず読むこと。**

| 資料 | 内容 |
|------|------|
| `docs/guide/audio.md` | 音声パイプライン、iOS・Android・Web Speechのバックグラウンド再生の実測挙動、デバッグモード |
| `docs/guide/sites.md` | テキスト処理、小説サイトパーサー、`SITE_RULES`、対応・非対応サイトの実測事項 |
| `docs/guide/proxy.md` | CORSプロキシ（自前2系統・公開プロキシ）、取得制御、Shift_JIS、削除済みプロキシ |
| `docs/guide/storage.md` | localStorage / sessionStorage のキー一覧 |

---

## 記述ルール

- 説明文・ガイドライン・メモ・ドキュメントは日本語で記述する
- コード本体・変数名・関数名・ファイル内コメントは英語で記述する。既存の英語コメントを日本語に書き換えない
- コミットメッセージは日本語で、形式は `種別: 内容`（例: `feat: スリープタイマーのUI改善` / `fix: iOS Safari でのバックグラウンド再生が途切れる問題を修正` / `docs: v164のバージョンドキュメント生成`）

---

## プロジェクト概要

**夜語り (Yogatari)** は日本語Web小説のテキスト読み上げPWA。小説家になろう・カクヨム・エブリスタ・青空文庫からCORSプロキシ経由で本文を取得し、Web Speech APIまたはGoogle Cloud TTSで読み上げる。アルファポリスとハーメルンは非対応（理由は `docs/guide/sites.md`）。

## 開発方法

- ビルドシステム・バンドラー・テスト・パッケージマネージャーは無い。`index.html` をブラウザで直接開くか、`python3 -m http.server` で配信する
- ただし `file://` で開くとプロキシのオリジン制限により小説の取得はできない（`docs/guide/proxy.md`）
- Context7 は、新しいライブラリの導入やGoogle Cloud APIなどの仕様確認で最新情報が必要な場合にのみ使う。単純なロジック・UI調整では使わない。出力は補助情報として扱い、既存コードとの整合性を優先する

## ファイル構成

| ファイル | 内容 |
|------|------|
| `index.html` | アプリ全体（`<head>` のPWAメタタグ / インライン `<style>` / 3タブ構成「読む・履歴・設定」の本体 / インライン `<script>`） |
| `manifest.json` | PWAマニフェスト |
| `icon-192.png` / `icon-512.png` | PWAアイコン（`purpose: any maskable`） |
| `icon.svg` | ブラウザのタブ・ブックマーク用アイコン（`<link rel="icon">`。非対応環境では `icon-192.png` を使う） |
| `README.md` | 利用者向けの説明（概要・使い方・対応サイト・開発方法） |
| `docs/guide/*.md` | 詳細資料（上表） |
| `docs/versions/vXXX/` | バージョン記録（`structure.json` + `notes.md`）。「バージョン管理プロトコル」実行時のみ操作する |
| `CLAUDE.md` | 本ファイル |

## 主要グローバル状態

| 変数 | 用途 |
|------|------|
| `segs` / `idx` | 現在の章のテキストセグメント配列 / 読み上げ中のインデックス |
| `playing` | 再生状態（Boolean） |
| `queue` | バッチ・キュー再生用の章URL配列 |
| `selVoice` | 選択中のWeb Speech API音声 |
| `_gApiKey` | Google Cloud TTS APIキー。**キーの有無であり、再生エンジンの判定ではない** |
| `_ttsEngine` | 再生エンジン（`'google'` / `'device'`）。`yogatari_engine` と同期 |
| `_prefetchCache` | Google TTS音声BLOBのキャッシュMap |
| `currentChapterTitle` / `currentNovelTitle` | 表示中のメタデータ |
| `nextUrl` / `prevUrl` | 章間ナビゲーション |
| `APP_VER` | アプリバージョン。`<title>` と `<span class="ver">` に同期。バージョン管理プロトコル実行時のみ更新する |
| `DATA_VERSION` | **未導入**。localStorageの破壊的変更を行う際に導入する想定（「後方互換性」参照） |

再生経路の判定には `useGoogleTts()`（`_ttsEngine === 'google' && hasGoogleKey()`）を使う。`hasGoogleKey()` は「APIキーが保存されているか」の判定にのみ使う。

---

## 壊しやすい箇所（変更前に詳細資料を読む）

以下はいずれも実機で原因を特定済みの挙動である。未使用・冗長に見えても削除・簡略化しない。理由と詳細は各資料にある。

### 音声（`docs/guide/audio.md`）

- iOSのバックグラウンド音声は `gAudioEl` 単独が保持する。中断後の拒否は `currentTime` が進まないことでしか検出できず、`checkAudioProgress()` / `markAudioSessionLost()` / `rebuildAudioElements()` が担う。復旧は▶押下時（ユーザー操作の内側）にのみ行う
- Web Speech APIはiOS・Androidともバックグラウンド再生できない（端末側の仕様）。`utt.onerror` が `interrupted` / `canceled` を無視するのは二重読み上げ防止のため
- Androidは5秒以下の音声を一時音として扱うため、`silentLoopEl` はMP3のデコード失敗時のみ実行時生成の6秒無音WAVに切り替える（`silentWavSrc()` / `_silentMode`）。判定はUAではなくデコード失敗で行う
- 一時停止時は無音ループも必ず止める。`stopSilentAudio()` で `src` を空にしない
- `unlockAudio()` の `volume` 復帰を成功時のみに書かない（Androidで全編無音になる）
- デバッグモード（設定タブ最下部、既定OFF）は削除・簡略化しない。OFFの間は計測処理を一切生成しない

### テキスト・サイト（`docs/guide/sites.md`）

- `splitText()` の「フィルター後に0段落なら元の行を返す」安全弁と、`_pendingEpTitle` の先頭3行限定は削除しない
- `SITE_RULES` の `rule.body` のカンマ区切りは優先順位リストとして先頭から順に試す。`querySelector()` にそのまま渡さない
- カクヨムの目次は作品ページの `__NEXT_DATA__`（Apolloストア）から構築し、並び順は参照リストから取る。作品IDはページ自身が名乗るものを使う
- アルファポリスのコンテンツ保護を迂回する実装は行わない

### 取得・プロキシ（`docs/guide/proxy.md`）

- 自前プロキシ（Cloudflare Workers / Deno Deploy）のコードは本リポジトリ外。両系統のホワイトリストと `ALLOWED_ORIGINS` は常に同じ内容に保つ。配信元を変えるときは `ALLOWED_ORIGINS` への追加が必須
- `own: true` の自前プロキシの404は最終話判定（`own404` / `isFinalErr()`）に使われる。プロキシ側で「ページ不在以外」の理由で404を返さない
- プロキシは1件ずつ500ms間隔で起動し、カクヨムには `HOST_MIN_INTERVAL_MS` による2秒間隔と12秒後の先読みを適用する。アクセス集中によるIP単位の拒否を避けるため
- Shift_JISのページ（青空文庫）は `arrayBuffer()` + `TextDecoder('shift_jis')` でデコードし、JSONで包むプロキシを対象から外す。`tryProxy()` / `fetchHtml()` の変更時は青空文庫を壊さないこと
- `fetchHtml()` 自体はキャッシュしない（`sessionStorage` のページキャッシュは `resumeFrom()` のみ）
- 削除済みプロキシ一覧にあるものを再追加しない。一時的な失敗を理由に既存プロキシを削除しない

---

## PWA

- `manifest.json`（`display: standalone`、テーマカラー `#0a0908`）とアイコン2種を導入済み。`viewport-fit=cover` と `env(safe-area-inset-top)` / `env(safe-area-inset-bottom)` でセーフエリアに対応
- Service Worker は使用しない。リアルタイム取得が前提でオフライン対応は不要なため

---

## 開発ルール

### ファイル構成

- ロジック(JS)・スタイル(CSS)・マークアップ(HTML)はすべて `index.html` 内に完結させる
- 新規作成を許可するファイルは次のみ：PWA動作に必要な `manifest.json` とタブ用アイコン `icon.svg`、バージョン管理プロトコルによる `/docs/versions/` 配下、`docs/guide/` 配下の詳細資料、`README.md`。サービスワーカー（`sw.js`）は作らない
- 詳細資料の事実が変わった場合は、同じコミットで `docs/guide/` の該当資料も更新する。対応サイト・使い方など利用者向けの内容が変わった場合は `README.md` も更新する

### 後方互換性

- `localStorage` の破壊的変更は行わない。構造を変える場合は `DATA_VERSION` を導入してマイグレーションを実装し、起動時に差分を検出して段階的に変換する。不明データ・旧バージョンのデータは削除しない
- 既存キーを変えず新しいキーを追加するだけならマイグレーションは不要。新しいキーには `yogatari_` プレフィックスを付け、`docs/guide/storage.md` に追記する

### モバイル・ファースト

- iOS / Android の Safari / Chrome を前提とし、`env(safe-area-inset-*)` を使う
- 音声再生は必ずユーザー操作（クリック等）を起点とする（自動再生禁止）

### ユーザー確認が必要な変更

以下は実装を進める前にユーザーに確認する：データ構造の変更（localStorage含む）、UIレイアウトの大幅変更（タブ構成の変更・主要セクションの追加削除・既存コントロールの移動など）、音声制御ロジックの変更、状態管理フローの変更、ファイル構成の変更。

### 変更の範囲

- 明示されていない機能・変更は実装しない。指示された目的を達成するための最小の変更にとどめる
- 既存コードの意図・挙動を変えない。正常に動作している処理をリファクタリング目的で書き換えない
- `CLAUDE.md` はユーザーの明示的な指示がある場合のみ変更する

### コーディングスタイル

- 新規コードは既存の機能単位（Audio / Data / UI など）に従って配置し、同一責務のコードは近接させる
- **既存コードの再利用（DRY）**: 同一責務・類似機能の処理が既にある場合は、新規実装せず既存の関数を再利用または拡張する。特にデータ保存 / 取得処理・UI更新処理・音声制御処理は重複しやすいため、実装前に確認する
- スキップフィルターを拡張する場合は `yogatari_skip_filters` と `yogatari_custom_words` の両方への影響を考慮する
- プロキシは `PROXIES` 配列に追記するだけでフォールバックチェーンに組み込まれる

### CSS

- **CSS変数（`--xxx`）は使用しない。** iOS環境でのテキストコピー時にスタイル崩れ（レイアウト破綻）を起こすリスクがあるため
- スタイルの再利用は共通クラス（例：`.btn-primary`）やUtilityクラス（例：`.flex-center`, `.mt-8`）で行う。単純なコピペは避ける
- インラインスタイルはJavaScriptから値を動的に変える場合にのみ使う

### Git運用

- 通常の開発は `main` ブランチで行う。Claude Code on the web が自動生成する `claude/...` ブランチは許容するが、手動でのブランチ作成やプルリクエストの提案は行わない
- 作業完了後は遅滞なくコミット・プッシュする

---

## バージョン管理プロトコル

`/docs/versions/` は安定状態のスナップショットであり、作業ログではない。途中・失敗状態は記録せず、細かな履歴はGitに委ねる。プロジェクトの構造と変更履歴は `/docs/versions/` を唯一の参照元とし、最大の vXXX を最新とする。`structure.json` を構造的事実、`notes.md` を補足として扱い、不明な情報は推測で補完しない。

### 実行トリガー

「vXXXとして記録」「vXXXに上げて」「この状態をvXXXとして保存」のように、ユーザーがバージョン番号を明示して指示した場合にのみ実行する。それ以外の理由（コード修正に伴う自動更新、AI側の判断など）でバージョンを作成・更新しない。通常のコード修正・コミットでは実行しない。

### 停止してユーザーに確認する条件

- 指定された番号が既存の最大バージョン以下である（番号の飛びは許可し、ユーザーの意図として扱う）
- 同一番号のディレクトリが既に存在する（新規作成・上書き・変更は一切行わない）
- `git status` に未コミットの変更がある（HEAD状態のみを記録対象とする）
- 直前バージョンとの比較で、意図しないファイルの消失など不明瞭な差分がある（推測で補完・修正しない）

### 実行手順

1. `/docs/versions/` 内の最大バージョンを確認し、上記の停止条件に該当しないことを確かめる
2. `index.html` の3か所をすべて vXXX に書き換える：`<title>` 内の文字列 / トップバーの `<span class="ver">` / `APP_VER` 定数
3. `/docs/versions/vXXX/` を作成する
4. `structure.json` を生成する
   - 階層を深くしない、実在するファイルのみのシンプルなファイルリストとする
   - 各ファイル・機能に `status`（`implemented` / `partial` / `not_implemented`）を付ける
   - `features` フィールドで主要機能の実装状態を記載する（直前バージョンの `structure.json` を元に更新する）
5. 直前バージョンの `structure.json` と比較し、構造差分に矛盾がないか確認する
6. `notes.md` を日本語で作成する。変更点 / 決定事項 / 未実装を、技術的事実のみで簡潔に記述する
7. `index.html` と `docs/versions/vXXX/` の変更をまとめ、「docs: vXXXのバージョンドキュメント生成」のようなメッセージで作業ブランチへコミット・プッシュする
