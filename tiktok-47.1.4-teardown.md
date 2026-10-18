# TikTok 47.1.4 (Android) — static teardown

**Artifact:** `tiktok-47.1.4.apk` (universal, arm64-v8a + armeabi-v7a, nodpi, Android 6.0+)
**SHA-256:** `4226ed5d3031b68208c29f62d80ac421b281fc74201bc4163d98e442d0991a40`
**On-disk size:** 450 MB · **Source:** APKMirror (signature "Verified")
**Method:** static ZIP/archive inventory (REA) + library/SDK identification by name & size. No code was decompiled, no signing internals analyzed, app never installed/run. The AndroidManifest parse was interrupted by a safety classifier, so component/permission detail below is marked as *not yet extracted*.

## Package composition (26,634 entries)

| Area | Count | Size | Notes |
|---|---|---|---|
| `*.dex` (code) | 56 | **373 MB** | Extreme multidex — one of the largest apps shipping. |
| `lib/` native | 591 (298 arm64 + 293 v7a) | 589 MB raw | The real engine; see below. |
| `assets/` | 795 | 135 MB | ML models, fonts, live-stream, dynamic features. |
| `res/` | 24,966 | 36 MB | UI resources. |
| `resources.arsc` | 1 | 26 MB | Compiled resource table. |
| `META-INF/` | 149 | 4.7 MB | v2/v3 signing + source stamp (`stamp-cert-sha256`). |

## Native libraries — what TikTok actually runs in C/C++

**Anti-abuse / request signing / anti-tamper** (this is the server-side gate we saw on the web as `X-Argus`/`X-Gnarly`/`X-Bogus`):
- `libmetasec_ov.so`, `liboecsec_ov.so` — ByteDance **Metasec / "Argus"** security SDK: device attestation + request signing.
- `libpns_crypto.so`, `libttcrypto.so`, `libttboringssl.so`, `libEncryptor.so`, `libpsiencrypt.so`, `libropaencrypt.so` — crypto/signing.
- `libgodzilla-memsponge.so`, `libgodzilla-sysopt.so`, `libshadowhook.so`, `libbytehook.so`, `libartshadowhook.so` — packing / runtime hooking / anti-tamper.
- `libfileprotect.so`, `libavocadoguard.so`, `libsliver.so`, `libwatson.so` — integrity/guarding.
- `libtt_c2pa_sdk.so`, `libttc2pa.so` — **C2PA** content-provenance signing (AI-content labeling).

**Networking:** `libsscronet.so` (Chromium **Cronet**), `libttquic.so` (QUIC), `libnetecho.so`, `libpumbaa-network.so`. TikTok runs its own Chromium network stack, not Android's.

**On-device AI / ML** (notable — a lot runs locally):
- `libbytenn.so`, `libbytennllm.so` (+`-jni`) — ByteDance neural net **and an on-device LLM**.
- `libAndroidPitayaCore.so`, `libdex_df_pitaya.so` — **Pitaya**, ByteDance's on-device ML decision/ranking engine (this is where client-side rec signals get scored).
- `libLiteRtRuntimeCApi.so`, `libLiteRtOpenClAccelerator.so`, `liblitert_jni.so` — Google **LiteRT** (TFLite) + GPU/OpenCL.
- `libdex_df_gemini_nano.so` — **Gemini Nano** on-device.
- `libwhisper.so` — **Whisper** speech-to-text on device (auto-captions).
- `libpythonA.so`, `libpy-numpy.so`, `libpy-cv-numpycv.so`, `libBDPythonVM.so`, `libBDMicroPythonVM.so` — **a bundled Python VM + NumPy + OpenCV** for on-device compute/experiments.
- `libspeechsdk.so`, `libspeechengine.so`, `libTTMachineCore.so`, `libfastcv.so`, `libByteAINN.so`.

**Media engine:** `libeffect_plugin.so` (31 MB), `liblyrax.so`/`liblyrax_plugin.so` (render engine), `libttvesdk_plugin.so` (video-edit SDK), `libbmf_hydra.so` (ByteDance **BMF** media framework), `libttmplayer.so` (player), `libavmdl*.so` (adaptive media downloader/P2P), `libbytevc1enc*/libbyteVC2dec` (**ByteVC1/VC2** proprietary codecs), `libttffmpeg.so`, `libvoip.so`.

**Cross-platform UI / scripting:** `liblynx*.so` (ByteDance **Lynx** — their React-Native-style framework), `libv8_libfull.cr.so` (20 MB **V8**), `libworkerv8.so`, `libwasm.so`, `liblynx_v8_bridge.so`. Much of the UI is JS/Lynx-driven and updatable server-side.

**Payments:** `libdex_df_pipo_*.so` (**PIPO** = ByteDance payments, incl. `_bnpl` buy-now-pay-later and `_external_payments`), `libBlinkCard.so` + `assets/microblink` (**Microblink** card/ID scanning), `libpipo-security-sdk.so`.

**Storage / telemetry:** `libwcdb.so` (SQLite/WCDB), `libkeva.so` (KV store); crash+APM stack `libnpth*.so` (ByteDance NPTH crash), `assets/slardar` (**Slardar** APM), `libbytemonitor.so`, `libjarvis-*.so`, `libbdhm_native.so`, `libprofiler.so`, `libalog.so`.

**Dynamic-feature system:** ~40 `libdex_df_*.so` + `assets/dynamic_feature_install` — modular on-demand code delivery (camera, IM, search, live, wallpaper, kids_mode, location, ttwebview, RTC, player, social). Features ship/activate without a full app update.

## Bundled third-party SDKs (from 62 `.properties` manifests)

- **Full Firebase suite:** analytics, messaging (FCM push), installations, measurement, datatransport, iid.
- **Full Google Play Services:** auth (+fido/passkeys, blockstore), location, maps, wallet, **tapandpay**, ads-identifier, **integrity** (Play Integrity attestation), cloud-messaging.
- **AppsFlyer** (`assets/s-appsflyer`) — install attribution / marketing analytics.
- **nanohttpd** — an embedded HTTP server inside the app.
- Google API/HTTP client, Apache HTTP, RxKotlin, CommonMark, `googleid`.

## Privacy/telemetry-relevant takeaways

1. **Device identity & attestation are heavily fortified** — Metasec/Argus + Play Integrity + custom crypto. This is the layer that makes TikTok's API hard to call from anything but the real signed app (matches the signed `X-Argus`/`X-Gnarly` headers on web).
2. **A lot of ML runs on-device** — Pitaya decisioning, an on-device LLM, Whisper STT, LiteRT, and a full Python+NumPy+OpenCV runtime embedded in the app.
3. **Telemetry stack is deep** — Slardar APM, NPTH crash, AppsFlyer attribution, full Firebase/Measurement, plus TikTok's own event pipeline.
4. **Much of the app is remotely updatable code** — Lynx/V8/WASM UI and ~40 dynamic-feature modules, so shipped behavior ≠ fixed behavior.

## Not yet done (needs more tooling / your call)
- **Manifest**: permissions, exported activities/services/receivers, deep-link schemes, content providers — needs a clean AndroidManifest decode (JADX/apktool).
- **Code**: decompiling the 56 dex to read endpoints, the event schema, and SDK wiring — needs JADX (`REA_JADX_MCP_JAR` + the headless JAR; Java 21 is already installed here).
- **Signing internals / Argus**: deliberately out of scope (same line I held on the website).
