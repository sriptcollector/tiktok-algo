# TikTok 47.1.4 — decompilation addendum

Follows the full static teardown. This covers the JADX decompile (**308,280 `.java` files** from the 56 dex) and what was recovered from readable source + surviving string literals.

## Obfuscation reality
TikTok ships heavy obfuscation: the overwhelming majority of classes are renamed and flattened into a single `X.` package with hashed names (e.g. `X.C1311505Bp`, `X.InterfaceC804503Cp`). Control flow and types are therefore **not** human-readable without deobfuscation — e.g. `FeedPostQueryParamInterceptor.intercept()` decompiles, but every collaborator it touches is an `X.*` hash. What survives cleanly:
- **String literals** — endpoint paths, header names, settings keys, event names.
- **Package-named business classes** under `com.ss.android.ugc.aweme.*` and `com.bytedance.*` (interfaces/models kept their names; implementations are mostly in `X.`).
- **Retrofit interface signatures** (thin; real annotations often on `X.*` types).

So the durable intelligence is the **endpoint + header + key catalog**, not decompiled algorithms. The Argus/Gorgon signing is native (`libmetasec_ov.so`) and was not reversed.

## Endpoint catalog — 1,288 versioned HTTP endpoints recovered
Full list in `endpoints.txt`. Highlights by function:

### Feed / recommendation (the mobile side of the web FYP we profiled)
- `/aweme/v1/feed/` — **the main For You feed** (mobile equivalent of web `/api/recommend/item_list/`).
- `/aweme/v1/aweme/stats/` — **engagement/watch stats reporting** (the signal pipeline that feeds the ranker; matches the `mcs…/v1/list` collector seen on web).
- `/aweme/v1/follow/feed/`, `/aweme/v1/friend/feed/`, `/aweme/v1/following/interest/feed/`, `/aweme/v1/follow/feed/recommend/` — the other feeds.
- `/aweme/v1/aweme/detail/`, `/aweme/v1/aweme/post/`, `/aweme/v1/aweme/favorite/`, `/aweme/v1/aweme/collect/` — item fetch/engagement.
- `/aweme/v1/general/search/` + `/search/stream/` + `/search/single/` — search.
- `/aweme/v1/signal/merge` — signal merging.
- Feed classes in source: `com.ss.android.ugc.aweme.feed.api.recommend.FeedPostQueryParamInterceptor` (adds query params/headers to feed requests), `FeedItemList`, `FeedAppLogParams`, `Feed0VVManager` (the `vv_count` first-view logic we saw as a web query param).

### Auth / crypto handshake
- `/api/v1/key_agreement` + `/api/v1/key_validation` — a **key-agreement handshake** (encrypted-channel / TTEncrypt setup).
- `/aweme/v1/settings`, `/aweme/v1/compliance/settings`, `/aweme/v1/rate/settings`, `/tiktok/v1/digital_wellbeing/settings/` — server-driven config/feature gating (the `libra` A/B system populates these).
- Account: `/tiktok/v1/account/confirm_change`, `/aweme/v1/register/check/login/name`, IM cloud tokens, Amazon ad-token generate/refresh.

### Commerce (TikTok Shop — a huge share of the surface)
- `/api/v1/shop/recommend/feed/*`, `/api/v1/mall/*`, `/api/v1/logistics/*`, `/api/v1/aftersale/*`, `/api/v1/affiliate/*`, `/ad/v3/order/*`, `/api/v1/android/validate_purchase(_v2)` (Play Billing). Commerce/ads endpoints outnumber content endpoints roughly 3:1 in the catalog.

### Ads
- `/ad/v1..v3/*` — splash stock, aggregated serving, order create/detail, IAB click/conversion reporting, in-ad agent "hints".

## Confirmed signing & telemetry classes (decompiled package names)
- **Request signing / anti-bot:** header literals `X-Argus`, `X-Ladon`, `X-Khronos`, `X-Gorgon`, `X-SS-REQ-TICKET`, `X-Tt-Token`, `req_sign`, `x-ss-stub` (native impl `libmetasec_ov.so`).
- **Risk/CAPTCHA:** `com.bytedance.bdturing.*` (`RiskControlService`, `TwiceVerifyService`, `VerifyTaskHandler`).
- **Telemetry:** `com.bytedance.applog.*` + `com.ss.android.common.applog.AppLog` — with native pieces `applog/filter/NativeFilterJNI`, `applog/log/NativeLogWrapper`, `CommonParamsWrapper`, `engine/Session`. Device IDs `device_id` / `install_id` / `openudid` / `cdid`, trace `x-tt-trace-id`.
- **Network:** `com.bytedance.frameworks.baselib.network…cronet.impl.SsCronetHttpClient` (TTNet over Chromium Cronet), with `TTNetSampleInterceptor`.

## Net assessment
The decompile confirms and grounds the structural teardown: TikTok Android is a ByteDance "super-app" shell (content + search + IM + live + **large commerce/ads**) over TTNet/Cronet, gated by native Argus signing + BDTuring risk control, instrumented by the AppLog telemetry pipeline, with server-driven config (Libra/settings) and a big remotely-updatable surface (Lynx/V8/Gecko/dynamic-features). The For You pipeline on mobile is `/aweme/v1/feed/` for pulls and `/aweme/v1/aweme/stats/` for the watch-time/engagement signals — the same watch-time-first loop documented on the web.

## Deliverables in `C:\Users\orion\Downloads\tiktok-re\`
- `endpoints.txt` — 1,288 endpoints
- `out/dex_hosts.txt` (229 hosts), `out/dex_urls_raw.txt` (1,072 URLs), `out/dex_strings.txt`
- `out/apktool/` — decoded manifest + resources
- `out/jadx-src/sources/` — 308,280 decompiled `.java` files (browsable)
- `evidence.json` — REA inventory; `tiktok-47.1.4-FULL-teardown.md`, `tiktok-47.1.4-teardown.md`
