# TikTok Algorithm & App Teardown (v47.1.4, Android)

Static reverse-engineering of the TikTok Android app (`com.zhiliaoapp.musically`, v47.1.4) plus live behavioral analysis of the web For You feed. No app installed/run; no proprietary algorithm or signing internals extracted — this is a structure, backend, endpoint/header, and UX-mechanics map built from public artifacts.

## Contents
- **[tiktok-47.1.4-teardown.md](tiktok-47.1.4-teardown.md)** — initial structural teardown (package composition, native engines, bundled SDKs).
- **[tiktok-47.1.4-FULL-teardown.md](tiktok-47.1.4-FULL-teardown.md)** — full teardown: manifest/permissions/components, 229-host backend map + data-residency split, anti-abuse/signing stack, native engines, SDKs.
- **[tiktok-47.1.4-decompile-addendum.md](tiktok-47.1.4-decompile-addendum.md)** — JADX decompile (308k Java files): obfuscation reality, 1,288-endpoint catalog, confirmed signing/telemetry classes.
- **[tiktok-UX-addiction-teardown.md](tiktok-UX-addiction-teardown.md)** — recommendation & engagement-engineering: video ranking plumbing, autoplay/preload/infinite feed, GIFs/stickers, messaging + streak-pet, suggestions/social graph, notifications, gamification, wellbeing countermeasures.

## Method
REA archive inventory · apktool manifest/resource decode · JADX decompile · dex string/URL extraction · live web session (network + client observation). Details and honest limits in each report.
