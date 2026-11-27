---
name: "needle-in-service-worker"
description: "Run a Cactus Needle model (wasm) inside a PWA service worker instead of a page-owned Web Worker. Use when the user wants Needle hosted in sw.js, shared across tabs, or answering fetch/message requests."
---

# Needle as a service worker

First decide whether a service worker is the right host. The proven setup in these repos is a page-owned **dedicated worker** (`needle/needle.worker.js` + `needle-client.js`): it lives as long as the page, the 35-63 MB model stays in memory, and `importScripts` works. A service worker is a different environment, so confirm the reason before building (shared across tabs? works with no page open? intercepts requests?). If the reason is only offline caching, the existing `sw.js` cache-first shell plus the dedicated worker already covers it.

## What differs in a service worker (verify on the target browser; do not assume)
- **Lifetime.** The browser stops an idle service worker after a short time (tens of seconds) and caps long-running events. Stopping discards the wasm heap, so the loaded model is lost and the next request pays a full load (about 35-63 MB read from Cache API plus `cactus_load`, `cactus_init`). Design for cold start as the normal case, not the exception.
- **Keep-alive is not guaranteed.** Wrap inference in `event.waitUntil(promise)` so a request in flight is not killed, and show a loading state for cold starts. Do not rely on ping tricks to hold the worker open; treat them as best effort.
- **`importScripts` timing.** In a service worker `importScripts` is only allowed during initial script evaluation / install, not later. Call `importScripts('./cactus.js')` (and `validate.js`, `guard.js` if used) at the top level of `sw.js`; a lazy `importScripts` inside a message handler will fail. Prefer a classic (non-module) service worker for this reason.
- **Memory.** One instance for all tabs is the benefit, but the same wasm heap limits apply. Keep the existing rule: never free the model buffer after `cactus_load`. Allocate the output buffer once.
- **Threads/GPU.** Do not assume SharedArrayBuffer (needs cross-origin isolation headers on the page and worker responses) or WebGPU are available in a service worker. Use the CPU wasm path and measure.
- **Serialization.** Calls to `cactus_complete` must run one at a time; keep the promise-queue pattern from `needle.worker.js` (including `cactus_reset` before each request, raw user text only, one completion per request).

## Steps
1. **Reuse, do not fork.** Extract the engine part of `needle.worker.js` (load assets, init tools, complete, parse with `validate.js`) into a module-free file that both the dedicated worker and `sw.js` can `importScripts`. One implementation, two hosts. Keep the call sequence: `createCactus({wasmBinary})` -> `_malloc` + `HEAPU8.set` + `_cactus_load(ptr, BigInt(len))` -> `cactus_init("", toolsJson, 0)` -> `cactus_complete(text, 0, 0, maxTokens, out, cap)`.
2. **Assets.** Store `cactus.js`, `cactus.wasm` and the `.cact` in a dedicated Cache API bucket (separate from the app-shell cache) and verify SHA-256 once on download. Retrained models need a content hash in the manifest URL (`?v=<sha12>`) because caches keyed on URL + size will serve a stale model. Never let the app-shell `fetch` handler intercept `/needle/` model URLs or large ranges; return early for them as the current `sw.js` already does for `/needle/`.
3. **Activation.** Do not download the model in `install` unless offline-first is required: a 60 MB install failure blocks the whole service worker update. Download lazily on first request, or in `install` with failure tolerated (as pwa-vti does with `cache.add(...).catch`), and report progress to clients via `clients.matchAll()` + `postMessage`.
4. **Client API.** Mirror `NeedleClient` (`init()`, `complete(text)`) but talk to `navigator.serviceWorker.controller` over a `MessageChannel` per request so replies are matched to the caller. Handle `controller === null` (first load before claim: `clients.claim()` in `activate`) and the worker being restarted mid-request (timeout, then retry once after re-init).
5. **Version/cache hygiene.** Bump the cache name when the model or tools change; delete old caches in `activate`. Keep `skipWaiting`/`claim` deliberate: swapping workers mid-request drops that request.
6. **Tools.** Register exactly the training-time tool set (flat, local markers stripped). The service worker must not apply an app-side guard unless the app does; keep raw vs guarded behaviour identical to the dedicated worker so results stay comparable.

## Test plan
- Cold start: stop the worker in devtools, send a request, record load time and first-answer latency; compare to the dedicated worker's ~0.8 s median warm latency.
- Idle kill: leave it idle, confirm the next request recovers without a page reload.
- Two tabs, simultaneous requests: answers are not mixed and run one at a time.
- Offline after first load, and after a model update (old model not served).
- Run the dev and hold-out eval cases through the service-worker path and compare with the dedicated-worker results; they should match.

## When to stop
If cold-start load time makes interactions feel broken (several seconds per request after idle), keep the model in a dedicated or shared worker owned by the page and use the service worker only for caching and message relay. Report the measured numbers rather than guessing.