# GimJector Standalone — README

Oct 15, 2026

## Overview

**GimJector Standalone** (GimJectorStandalone.html) is a zero-dependency, single-file version of GimJector — a Gimkit bot flooder — that runs entirely in the browser with no server, no Node.js, and no installation required. Drop the HTML file anywhere (a local folder, a CDN, or open it as a `file://` URL) and it works immediately.

The original project (`GimJector-main`) is a **Node.js bridge server**: the bot logic runs on the server and all Gimkit connections are made server-side; your browser talks to your own local server over a WebSocket, which then manages all the Gimkit connections on your behalf. The standalone version eliminates the server entirely by re-implementing the same bot logic in client-side JavaScript, using [libcurl.js](https://cdn.jsdelivr.net/npm/libcurl.js@0.7.1/libcurl.js) to route all network traffic through a Wisp proxy — bypassing the CORS restrictions that would otherwise block a browser page from opening direct connections to Gimkit's servers.

Version bump: original is **v6.0.0**, standalone is **v6.0.1**.

## Architecture: Standalone vs. Original

The original project is a **Node.js bridge server**. Your browser opens a WebSocket to `localhost:3000`, and that server — running as a Node.js process on your machine — handles everything: fetching the JID, calling the matchmaker API, opening WebSocket connections to Gimkit, and keeping heartbeats alive. The browser frontend (`public/index.html`) is a thin dashboard that sends control messages (`flood`, `stop`, `spam`) to the server and receives status updates back. There are no user accounts — one person runs the server locally and that's the only user.

The standalone file **collapses the entire server into the browser tab**. There is no `server.js`, no `npm install`, no `localhost`. The `Bot` class, `spawnBot`, `getJID`, and the blueboat encoder all run as JavaScript directly in the page. This is made possible by libcurl.js — a WebAssembly build of libcurl that intercepts `fetch()` and WebSocket calls and routes them through a Wisp proxy, making it possible to reach arbitrary HTTPS/WSS endpoints from inside a browser without triggering CORS errors.

The trade-off is that everything lives in memory for the lifetime of the tab. There is also no persistent WebSocket bridge between the UI and the bot workers — both live in the same JS thread.

## Key Technical Changes and Fixes

**Script loading via `<script src>` instead of `fetch()` + blob URL** The original server version loads JavaScript dynamically using `fetch()`. In a standalone HTML file opened from `file://`, the page's origin is `null`, and `fetch()` calls to CDN URLs are blocked by CORS because a null origin is treated as opaque. The fix in GimJectorStandalone.html (the "fixed" in the filename refers to this) loads both `libcurl.js` and `lzutf8` by dynamically creating `<script src="…">` tags and appending them to `<head>`. Script tags bypass CORS for cross-origin scripts entirely, so the libraries load correctly regardless of whether the file is served over HTTP or opened directly as `file://`.

**`libcurl.CurlWebSocket` instead of `ws.WebSocket`** The original server uses Node.js's `ws` package for WebSocket connections. The standalone file replaces every `new WebSocket(…)` call with `new libcurl.CurlWebSocket(…)`, which routes traffic through the configured Wisp proxy. The API shape is intentionally compatible — `onmessage`, `onerror`, `onclose`, and `.send()` all behave the same way — so the bot logic required minimal changes beyond the constructor call.

**`libcurl.fetch` instead of Node's built-in `fetch`** All HTTP requests (JID lookup, matchmaker calls, catalog fetches) use `libcurl.fetch()` instead of the native `fetch()` API. This is necessary for the same Wisp-tunnel reason: native browser `fetch()` from a `file://` page cannot reach external URLs, while libcurl routes those requests through the proxy.

**LZUTF8 global alias** The browser build of lzutf8 exposes itself as `window.LZUTF8` (uppercase). The original server code references `lzutf8` (lowercase). The standalone adds a one-liner alias after loading: `if (typeof LZUTF8 !== 'undefined') window.lzutf8 = LZUTF8;` to avoid renaming every call site.

**Self-contained blueboat encoder** The original project imports its blueboat encoder from `network/blueboat.js` in the Node module tree. The standalone file embeds the entire encoder inline, verbatim, as a set of plain functions at the top of the script block. No module bundler needed.

**StegCloak ported to browser WebCrypto** The original server uses the `stegcloak` npm package (which relies on Node's `crypto` module). The standalone re-implements `stegcloak.hide()` using the browser's native `window.crypto.subtle` (PBKDF2 key derivation + AES-256-CTR encryption) and the inline lzutf8 library for compression. The zero-width-character encoding logic is ported verbatim.

## Feature Comparison

| Feature | Original (`GimJector-main` v6.0.0) | Standalone (GimJectorStandalone.html v6.0.1) |
| --- | --- | --- |
| Deployment | `node server.js` on your machine | Single HTML file — open directly in browser |
| User accounts | None — single local user | None |
| Bot transport | Native `ws` WebSocket (Node.js, no CORS) | `libcurl.CurlWebSocket` via Wisp proxy |
| HTTP transport | Native `node-fetch` | `libcurl.fetch` via Wisp proxy |
| Wisp URL | Not needed (server connects directly) | User-configurable in UI |
| Bot count limit | Up to 500 | Up to 500 |
| Name prefix | Configurable | Configurable |
| Random names | Yes | Yes |
| Stop button | Yes — disconnects all bots instantly | No — close/refresh tab to stop |
| Auto-reconnect | Yes | Yes |
| Spawn delay | Configurable | Configurable |
| Chat spam | Yes — all bots send a message on an interval | No — removed in standalone |
| Live bot grid | Yes — per-bot status events streamed to UI | No — status bar shows totals only |
| Browser notifications | No | Yes — Web Notifications API |
| Blueboat support | Yes | Yes (encoder inlined) |
| Colyseus support | Yes | Yes |
| StegCloak | `stegcloak` npm package + Node `crypto` | Browser WebCrypto + lzutf8 (ported inline) |
| JID caching | In-memory, 2-minute TTL | In-memory, per tab session |
| Dependencies | Node.js + 5 npm packages | None installed — CDN-loaded at runtime |

## How to Use

1. **Open the file.** Double-click GimJectorStandalone.html to open it in your browser, or serve it from any static host. No build step, no `npm install`.
2. **Wait for the libraries to load.** The status bar at the bottom of the card shows `Loading libcurl.js…` → `Loading WASM…` → `Loading lzutf8…` → `Ready — libcurl.js vX.X.X`. The Flood button is disabled until all three finish loading. This typically takes a few seconds on a normal connection.
3. **Enter the game code.** Paste or type the Gimkit game code into the Game Code field.
4. **Configure bots.** Set a name prefix (or toggle Random names), bot count, and spawn delay.
5. **Optionally configure auto-reconnect and the Wisp URL.** The default Wisp server (`wss://wisp.mercurywork.shop/`) is pre-filled. If it is down or slow, swap in any other public Wisp endpoint.
6. **Click 🚀 Flood.** Bots spawn one by one at the configured delay. The status bar tracks alive/failed/total in real time. Browser notifications fire for joins, drops, and failures if you grant permission when prompted.
7. **To stop**, close or refresh the tab. There is no stop button — closing the tab triggers `beforeunload`, which disconnects all bots cleanly.

## Configuration Options

| Option | Default | Notes |
| --- | --- | --- |
| Game Code | *(required)* | The Gimkit join code for the active game |
| Random names | Off | When on, generates names like `CoolHawk42`; hides the prefix field |
| Name Prefix | `Bot` | Bots are named `Bot1`, `Bot2`, … when random names is off |
| Bot Count | 10 | Maximum 500 |
| Delay (ms) | 300 | Milliseconds between spawning each successive bot |
| Auto-reconnect | Off | When a bot drops mid-game, it re-spawns automatically after 2 seconds |
| Wisp URL | `wss://wisp.mercurywork.shop/` | Any Wisp-compatible proxy server works here |

## Known Differences and Limitations

**No Stop button.** The original server handles a `stop` message from the browser, immediately terminates all bot WebSockets, and clears state. The standalone has no equivalent — once `go()` is called the only way to halt is to close or refresh the tab, which fires `beforeunload` and disconnects bots cleanly.

**No chat spam.** The original `server.js` includes a full chat spam feature: all alive bots send a configurable message at a configurable interval, controllable live from the dashboard without restarting the flood. This was not ported to the standalone.

**No live bot grid.** The original streams per-bot status events (`botUpdate`) to the browser, powering a grid that shows each bot's individual state (connecting / alive / dropped / failed). The standalone shows a single status bar with aggregate totals (`Alive: X | Failed: Y | Total: Z`) but no per-bot breakdown.

**Wisp dependency.** The original server makes direct outbound connections from Node.js with no CORS restrictions. The standalone must route everything through a Wisp proxy. If the default endpoint (`wss://wisp.mercurywork.shop/`) is down or rate-limited, you need to supply an alternative in the Wisp URL field.

**CDN dependency at runtime.** The file loads `libcurl.js`, `libcurl.wasm`, and `lzutf8` from jsDelivr on every open. An offline environment will not work without self-hosting those assets and editing the URLs in the script block.

**Browser notifications instead of a log panel.** The original streams a live log of every event (joins, drops, retries, errors) to the dashboard. The standalone fires browser notifications for key events and surfaces the last error in the status bar, but there is no scrollable event log.

## Under the Hood

The standalone file mirrors the structure of `server.js` section by section. Each logical unit is ported with a comment noting its origin.

**`bbEncode` / blueboat encoder** — Ported verbatim from `network/blueboat.js`. In the original, this is a proper ES module imported with `import { encode as blueboatEncode } from './network/blueboat.js'`. In the standalone, the entire encoder is inlined as plain functions at the top of the script block — no module bundler needed.

**`stegcloakHide(message, password, cover)`** — Re-implements `new StegCloak(true, false).hide(jid, 'BSKA', 'Gimkit Web Client V3.1')` from the original without the npm package. The original uses the `stegcloak` package which depends on Node's `crypto` module. The standalone replaces it with browser-native `window.crypto.subtle` (PBKDF2 key derivation + AES-256-CTR) plus lzutf8 for compression, keeping the same zero-width-character encoding pipeline.

**`getJID(force)`** — Mirrors `server.js getJID`. Fetches `https://www.gimkit.com/join`, extracts the `int:jid` meta tag content, reverses it, and encodes it with `stegcloakHide`. The original uses `node-html-parser` to extract the tag; the standalone uses `libcurl.fetch` to get the page then parses it with a regex instead (no DOM parser available in the same way). Result is cached in `_jidCache`; `force = true` bypasses the cache.

**`class Bot`** — Mirrors the `Bot` class in `server.js` exactly:

- `spawn(code, jid)` — three-step flow: `find-info-from-code` → `matchmaker/join` → connect via `_blueboat()` or `_colyseus()` depending on `join.source`.
- `_blueboat(serverUrl, roomId, intentId)` — connects via `libcurl.CurlWebSocket` instead of `new WebSocket`. Sends the encoded join payload on open, then sends heartbeat `'2'` every 25 seconds.
- `_colyseus(serverUrl, roomId, intentId)` — fetches a `sessionId` from `matchmake/joinById`, then connects via `libcurl.CurlWebSocket`. No heartbeat (matches server behavior).
- `disconnect()` — clears the heartbeat interval and closes the WebSocket.
- `sendChat()` — present in the original, **not ported** to the standalone.

**`spawnBot(name, retries = 2)`** — Wraps `Bot.spawn()` with the same retry loop as the server (up to 3 attempts, 500ms × attempt backoff) and the same `onDrop` auto-reconnect callback, minus the `toBrowser()` status event calls since there is no WebSocket bridge to the UI.

**`go()`** — The entry point called by the Flood button. Reads all form values, sets the Wisp URL on libcurl, pre-warms the JID cache, then loops through bot names spawning each with the configured delay. Replaces the server's `browserWs.on('message')` → `case 'flood':` handler.
