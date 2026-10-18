# TikTok 47.1.4 (Android) — full static teardown

**Package:** `com.zhiliaoapp.musically`  ·  **versionCode** 2024701040  ·  **minSdk** 23, **targetSdk** 36
**Artifact:** universal APK (arm64-v8a + armeabi-v7a), 450 MB
**SHA-256:** `4226ed5d3031b68208c29f62d80ac421b281fc74201bc4163d98e442d0991a40`
**Method:** static only — ZIP inventory (REA), `apktool` manifest/resource decode, dex string/URL extraction, native-lib & SDK identification. Full JADX source decompile was running separately. App never installed or run; signing/attestation internals not reversed.

---

## 1. Package composition (26,634 entries)
| Area | Count | Size |
|---|---|---|
| dex (code) | 56 files | 356 MB |
| native `lib/` | 591 (298 arm64 + 293 v7a) | 589 MB raw |
| assets | 795 | 135 MB |
| res + resources.arsc | 24,966 + 1 | 62 MB |
| signing (`META-INF`, v2/v3 + source stamp) | 149 | 4.7 MB |

## 2. Manifest surface
- **65 permissions**, **6 app-defined permissions**, **498 activities**, **87 services**, **52 receivers**, **12 content providers**, **67 explicitly exported** components.
- **Sensitive permissions:** CAMERA, RECORD_AUDIO, ACCESS_FINE/COARSE_LOCATION, ACCESS_MEDIA_LOCATION, READ_CONTACTS, READ_MEDIA_IMAGES/VIDEO/AUDIO, BLUETOOTH_SCAN/ADVERTISE/CONNECT, NFC, USE_BIOMETRIC/FINGERPRINT, SYSTEM_ALERT_WINDOW (overlay), the full FOREGROUND_SERVICE_* set incl. **MEDIA_PROJECTION** (screen capture) and PHONE_CALL, **DETECT_SCREEN_CAPTURE + DETECT_SCREEN_RECORDING**, ACCESS_ADSERVICES_AD_ID/ATTRIBUTION, BOOT_COMPLETED, INSTALL_REFERRER. (No QUERY_ALL_PACKAGES.)
- **App-defined perms:** `permission.PRF_INTERNAL`, `permission.WRITE_OTL_TOKEN` (one-tap-login token), `permission.RECEIVE_ADM_MESSAGE` (Amazon push), `DYNAMIC_RECEIVER_NOT_EXPORTED_PERMISSION`.
- **Deep-link schemes:** `tiktok://`, `musically://`, `snssdk1233://` (+ lite/otl/only/m variants — 1233 is TikTok's ByteDance app id), `musicsdk://`, OAuth callbacks for Google (`com.googleusercontent.apps.*`), Facebook (`fbconnect`), Kakao, and `com.zhiliaoapp.musically.braintree` (**Braintree/PayPal** payments).
- **Deep-link hosts:** `www.tiktok.com`, `m.tiktok.com`, `vm.tiktok.com`, `vt.tiktok.com`, `t.tiktok.com`, `tiktok.me`, `shop.tiktok.com`, `oec-api.tiktokv.com` (e-commerce), `pipopay`, `vt.tokopedia.com` (ByteDance's Indonesian e-commerce), and the odd short domains `tt.fun` / `tt.inc` / `tt.site`.

## 3. Backend surface (229 distinct hosts pulled from dex; 1,072 URLs)
Registrable-domain frequency: `tiktokcdn.com` 425, `tiktok.com` 156, `tiktokv.com` 146, `tiktokcdn-us.com` 68, `tiktokv.us` 11, `tiktok-minis.com` 12, `byteoversea.com`, `ibytedtos.com`, `isnssdk.com`, `tiktok-row.org/net`, `onelink.me` (AppsFlyer), `oraclecloud.com`.

**Core API clusters** (`*.tiktokv.com`): `api-normal`, `api-va`, `api-si`, `api-core`, `api` + regional template hosts `api16-normal-useast5`, `api16-normal-c-useast1a/useast2a`, `api16-normal-alisg` (Alibaba-Cloud Singapore), `api16-normal-alisg.helo-api.com`.

**By function:**
- **Telemetry/logging:** `log-va`, `rtlog`, `mon-va/-sg/-i18n/-boot`, `mon-va.byteoversea.com`, `mon16-normal-useast5.tiktokv.us`.
- **A/B config:** `libra-va/-sg` (ByteDance "Libra" experiments — matches the `clientABVersions` we saw on web).
- **Verification/risk:** `vcs-*`, `verification*`, `rc-verification*` (regionalised across `.com/.us/.eu`).
- **Realtime/live:** `webcast*`, `rtc*`, `im-va` (messaging), `sami*` (speech/audio AI).
- **Commerce:** `oec-api`, `tako*` (shop feed), `fp.pipopayment.us`, `choco`, `bytecast`.
- **Search:** `search.tiktokv.com`. **Push sync:** `bsync-va`. **JSBridge:** `jsb-va/-sg`.
- **CDN/storage:** `*.tiktokcdn-us.com`, `*.tiktokcdn-eu.com`, `ibytedtos.com` (ByteDance TOS object storage), `gecko`/`teko` (ByteDance Gecko hot-patch/resource delivery), `unpkg`/`draftcdn`.

**Data-residency split is explicit in the code:** `.tiktokv.us` + `useast5/useast8/useast1a` + **Oracle Cloud `us-ashburn-1.oci.oraclecloud.com`** (`config.mtp.sag…`, `proxy.telemetry…`) = US ("Project Texas" / USDS on Oracle); `.tiktokv.eu` + `useast…-ie`/`neutral` = EU/Ireland; `alisg` + `-sg` = Singapore; `tiktok-row.*` = Rest-of-World.

## 4. Anti-abuse / request-signing stack — confirmed in code
This is the layer behind the signed headers seen on the web (`X-Argus`/`X-Gnarly`/`X-Bogus`).
- **Signing headers present in dex:** `X-Argus`, `X-Ladon`, `X-Khronos`, `X-Gorgon`, `X-SS-REQ-TICKET`, `X-Tt-Token`, `req_sign`, `x-ss-stub`.
- **Native implementation:** `libmetasec_ov.so` / `liboecsec_ov.so` (ByteDance **Metasec/"Argus"**), plus `libpns_crypto`, `libttcrypto`, `libttboringssl`, `libEncryptor`, `libpsiencrypt`.
- **Risk control / CAPTCHA:** `com.bytedance.bdturing.*` — `RiskControlService`, `TwiceVerifyService`, `VerifyTaskHandler` (BDTuring is ByteDance's bot/risk verification SDK).
- **Anti-tamper / hooking:** `libgodzilla-*`, `libshadowhook`, `libbytehook`, `libfileprotect`, `libavocadoguard`.
- **Attestation:** Google **Play Integrity** (`integrity.properties`) + FIDO.
- **Device identity pipeline:** `com.bytedance.applog.*` (AppLog) + `com.bytedance.bdinstall.*`; identifiers `device_id`, `install_id` (iid), `openudid`, `cdid`, correlated via `x-tt-trace-id`. Native pieces: `applog/filter/NativeFilterJNI`, `applog/log/NativeLogWrapper`, `libapplog_rust.so`.

## 5. Networking
Own Chromium stack, not Android's: `com.bytedance.frameworks.baselib.network…cronet.impl.SsCronetHttpClient` + `libsscronet.so` + `libttquic.so` (QUIC), wrapped by **TTNet** with a Retrofit-style API layer and TTNet interceptors.

## 6. Native engines (591 libs) — by role
- **On-device AI:** `libbytennllm` (on-device **LLM**), `libAndroidPitayaCore`/`libdex_df_pitaya` (**Pitaya** client-side ML ranking/decisioning), Gemini Nano, `libwhisper` (speech-to-text), LiteRT/TFLite, and a bundled **Python VM + NumPy + OpenCV** (`libpythonA`, `libpy-numpy`, `libpy-cv-numpycv`, `libBD(Micro)PythonVM`).
- **Media:** `libeffect_plugin` (31 MB), `liblyrax` (render), `libttvesdk` (video edit), `libbmf_hydra` (BMF framework), `libttmplayer` (player), `libavmdl*` (adaptive/P2P downloader), proprietary **ByteVC1/VC2** codecs, `libttffmpeg`, `libvoip`.
- **Cross-platform UI:** **Lynx** (`liblynx*`) + **V8** (`libv8_libfull`, 20 MB) + WebAssembly (`libwasm`) — much of the UI is server-updatable JS.
- **Payments:** PIPO (`libdex_df_pipo_*` incl. BNPL + external payments), **Microblink** card/ID scan (`libBlinkCard` + `assets/microblink`), Braintree (manifest).
- **Storage/telemetry runtime:** `libwcdb` (SQLite), `libkeva` (KV), Slardar APM (`assets/slardar`), NPTH crash (`libnpth*`), `libbytemonitor`, `libjarvis-*`.
- **Dynamic delivery:** ~40 `libdex_df_*` modules + `assets/dynamic_feature_install` — on-demand feature code (camera, IM, search, live, wallpaper, kids_mode, location, RTC, ttwebview).

## 7. Bundled third-party SDKs (62 `.properties`)
Full **Firebase** (analytics, FCM, installations, measurement, datatransport), full **Google Play Services** (auth/FIDO/blockstore, location, maps, **Wallet/TapAndPay**, ads-identifier, **Play Integrity**, cloud-messaging), **AppsFlyer** (install attribution — its templated endpoints `%sattr.`, `%slaunches.`, `%sconversions.` appear in the URL set), **nanohttpd** (embedded HTTP server), Google API/HTTP client, Apache HTTP, RxKotlin, CommonMark. Social/login: Facebook, Google, Kakao. OneLink/`onelink.me` deep-link attribution.

## 8. Notable takeaways
1. **Hardened identity+signing** (Metasec/Argus + BDTuring risk + Play Integrity + custom crypto) is why the API is effectively unusable outside the real signed app.
2. **Heavy on-device compute** — an LLM, Pitaya ranking, Whisper, and a full embedded Python/NumPy/OpenCV runtime ship in the app.
3. **Deep telemetry** — AppLog pipeline (device_id/iid/openudid/cdid) + Slardar + NPTH + Firebase + AppsFlyer.
4. **Region-partitioned backend** baked into code: US on Oracle Cloud (Ashburn) + `.us`, EU on `.eu`/Ireland, SG on Alibaba Cloud, ROW separate.
5. **Large remotely-updatable surface** — Lynx/V8/WASM UI, Gecko hot-patch resource delivery, and ~40 dynamic-feature modules mean shipped ≠ running behavior.
6. **Commerce + payments** are first-class: TikTok Shop (`oec-api`, `tako`), PIPO/BNPL, Braintree, Microblink KYC, Tokopedia.

## Files produced
- `out/apktool/AndroidManifest.xml` (278 KB decoded) + decoded resources
- `out/dex_urls_raw.txt` (1,072 URLs), `out/dex_hosts.txt` (229 hosts), `out/dex_strings.txt`
- `evidence.json` (full REA inventory), `dex/classes*.dex` (56)
- `out/jadx-src/` — full Java decompile (was still generating)
