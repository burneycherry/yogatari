# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

**夜語り (Yogatari)** is a Japanese web novel text-to-speech PWA. It fetches novel content from Japanese fiction platforms (小説家になろう, カクヨム) via CORS proxies and reads them aloud using either the Web Speech API or Google Cloud TTS.

## Development

There is no build system, bundler, test suite, or package manager. The entire app is a single static file: `index.html`. To develop:

- Open `index.html` directly in a browser, or serve it with any static HTTP server (e.g. `python3 -m http.server`)
- No compilation, transpilation, or install steps required
- Version is tracked manually in the `<title>` tag and `APP_VER` constant — increment when making notable changes

## Architecture

The entire application lives in `index.html` (~1,800 lines):

- **Lines 1–11**: HTML head / meta tags (PWA, viewport, theme)
- **Lines 12–240**: All CSS (inline `<style>` block)
- **Lines 241–591**: HTML body — 3 tabs: 読む (read), 履歴 (history), 設定 (settings)
- **Lines 592–end**: All JavaScript (inline `<script>` block)

### Key Global State

| Variable | Purpose |
|----------|---------|
| `segs` | Array of text segments for the current chapter |
| `idx` | Current segment index being read |
| `playing` | Boolean playback state |
| `queue` | Array of chapter URLs for batch/queue playback |
| `selVoice` | Selected Web Speech API voice |
| `_gApiKey` | Google Cloud TTS API key |
| `_prefetchCache` | Map for caching Google TTS audio blobs |
| `currentChapterTitle` / `currentNovelTitle` | Displayed metadata |
| `nextUrl` / `prevUrl` | Navigation between chapters |

### Audio Pipeline

Two TTS modes:
1. **Web Speech API** (`speechSynthesis`): default, no API key needed. Sentence-by-sentence via `speakIdx()`.
2. **Google Cloud TTS**: high-quality, requires API key. Audio fetched via `fetchGoogleAudio()`, cached in `_prefetchCache`, played via `<audio id="gAudioEl">`.

Silent audio loop (`<audio id="silentLoopEl">`) keeps AudioContext alive for background playback on iOS.

### Text Processing

- `splitText(html)` → converts DOM nodes to plain text segments (paragraphs)
- `splitSentences(text)` → splits on Japanese punctuation: `。！？…」』`
- `domToText(node)` → recursive DOM-to-text with inline element handling

### Novel Site Parsers

- `parseSyosetuToc(doc, url)` — parses 小説家になろう table of contents
- `parseKakuyomuToc(doc, url)` — parses カクヨム table of contents
- `inferTocUrl(url)` — detects which site and constructs the TOC URL
- `toNovelId(url)` — extracts the novel ID from a URL

### CORS Proxy System

`PROXIES` array lists 7 fallback services. `tryProxy(proxy, url)` wraps fetch with timeout. `fetchHtml(url)` rotates through proxies and caches results in `sessionStorage`. Proxy success history stored in `localStorage` key `yogatari_proxy_hist`.

### localStorage Keys

| Key | Contents |
|-----|---------|
| `yogatari_bk` | Bookmarks JSON (novel → chapter + segment index) |
| `yogatari_skip_filters` | Skip filter on/off state |
| `yogatari_custom_words` | User-defined skip words |
| `yogatari_gkey` | Google TTS API key |
| `yogatari_gvoice` | Selected Google TTS voice name |
| `yogatari_stat_*` | Reading statistics (days, novels, episodes, seconds) |
| `yogatari_proxy_hist` | Proxy selection history |
