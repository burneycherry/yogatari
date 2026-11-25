# 開発予定（検討中）

> 実装前の検討事項を記録する。**ここに書かれた内容はいずれも未実装・未検証**であり、実機で確認した事実ではない。
> 実装に着手する際は、`CLAUDE.md` の「ユーザー確認が必要な変更」に従い、具体的な仕様をユーザーと確認してから進める。
> 実装・検証が済んだ項目は、該当する詳細資料（`proxy.md` / `audio.md` など）へ事実として移し、本資料からは削除する。

## 1. Deno Deployプロキシの移行（期限: 2027年4月頃）

背景は `proxy.md` の「Deno Deployは2027年4月頃にサービスを終了する」を参照。カクヨム本文はCloudflare経由で403になるため、公式の移行先であるCloudflare Workersは使えない。

移行先の候補:

| 候補 | 概要 | 懸念 |
|------|------|------|
| Mac mini（自宅サーバー） | 自宅回線の家庭用IPから取得するため、カクヨムに拒否されにくいと見込む | 常時稼働が必要。自宅IPが拒否されると自宅の全端末でカクヨムが見られなくなる恐れ |
| Vercel / Netlify 等（Node.js系） | Cloudflare以外かつDeno非依存 | カクヨムが受け付けるかは未確認 |

Val TownはDenoベースのため、長期的にはDeno開発終了の影響を受けうる。

### Mac mini案の構成

- プロキシのコードは `Deno.serve()` の入口を持つため、MacにDenoを入れればほぼそのまま動く見込み（Node.jsでも可）
- アプリは `https://burneycherry.github.io` から配信されるため、HTTPSの公開URLが必須（混在コンテンツはブロックされる）。Cloudflare Tunnel または Tailscale Funnel を使う
  - Cloudflare Tunnelを経由するのは「アプリ→Mac mini」の区間のみで、「Mac mini→カクヨム」は自宅回線から直接出るため、カクヨムの403は回避できる見込み
- スリープ無効化・停電後の自動起動・launchd等によるプロキシの自動起動を設定する
- ホワイトリストは必ず有効にする（`ALLOWED_ORIGINS` はブラウザ以外からは偽装可能なため、悪用を防ぐ実質的な防御はホワイトリスト）

### 手順

1. 移行先にプロキシを設置する
2. カクヨム本文が `?debug=1&url=<encoded>` で200になることを確認する
3. `PROXIES` の2番目（Deno Deploy）のURLを差し替える
4. `proxy.md` の自前プロキシの記述を更新する

## 2. Mac miniでのローカル音声合成（Google TTSの代替）

Mac mini上で音声AIを動かし、Google Cloud TTSの代わりに使う。文字数課金と使用量上限（`isOverLimit()`）、APIキーの管理が不要になる。

### エンジン候補

| エンジン | 日本語の質（評判ベース・未試聴） | 備考 |
|---------|------|------|
| Kokoro-82M | 中 | 軽量・高速。英語中心で、日本語は読み・アクセントが不自然になりやすいとされる。日本語の声は5種類程度 |
| VOICEVOX | 良 | 日本語専用。Mac版あり、HTTP APIを内蔵。キャラクターごとの利用規約でクレジット表記（「VOICEVOX:〇〇」）が必要なものが多い |
| AivisSpeech（Style-Bert-VITS2系） | 高 | 抑揚が自然で朗読向き。VOICEVOX互換のHTTP API |

VOICEVOXとAivisSpeechはAPIが同じ形式のため、両方に対応させやすい。まずMac上で実際に聞き比べて決める。

### 構成

```
iPhone（夜語り） ──HTTPS──▶ Cloudflare Tunnel ──▶ Mac mini
                                                 ├ 小説取得プロキシ（項目1）
                                                 └ 音声AI
```

### アプリ側の変更（想定）

- 現在のGoogle TTSは `fetchGoogleAudio()` が「テキスト → 音声のdata URL」を返し、`getAudioCached()` / `_prefetchCache` で先読みし、`gAudioEl` で再生している。送信先をMac miniに替えるだけで、iOSのバックグラウンド再生・先読み・中断復旧の仕組みはそのまま使える見込み
- `_ttsEngine` に `'local'` を追加する（現状は `'google'` / `'device'`）
- 設定タブにMac miniのURLと声の選択欄を追加する
- 新しい `localStorage` キー（例: `yogatari_local_tts_url`）を追加する。既存キーは変更しない（`storage.md` に追記する）
- VOICEVOX系はWAVを返す。iOS・Androidとも再生可能だが、MP3より大きい

### 懸念

- 生成速度：M1以降なら実時間に間に合う見込みだが、長文や重いモデルでは待ちが出うる。先読みである程度吸収する
- Mac停止時は端末音声（Web Speech）に頼ることになるが、Web Speechはバックグラウンド再生できない
- 音声APIをそのまま公開すると誰でも使えるため、オリジン確認や簡易トークンで保護する

### 手順

1. Mac miniにVOICEVOX / AivisSpeechを入れ、音質と生成速度を確認する
2. 小説取得プロキシと合わせてTunnelで公開する
3. 仕様をユーザーと確認したうえで、アプリ側の `'local'` エンジンを実装する
