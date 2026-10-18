# TikTok 47.1.4 — UX, recommendation & engagement-engineering teardown

Every mechanic below is grounded in **real artifacts** from the APK: endpoint path literals, decompiled model fields (`com.ss.android.ugc.aweme.*`), string tokens, and native libs — cross-checked against my live web session. Signal **weightings** and the ranking math itself are server-side/native (not extracted); the *inputs, plumbing and UX* are what's recovered here.

---

## 1. The video algorithm — delivery & feedback plumbing

### Feed endpoints
`/aweme/v1/feed/` & `/aweme/v2/feed/` (For You pull) · `/aweme/v1/follow/feed/`, `/friend/feed/`, `/nearby/feed/` (location), `/following/interest/feed/` · `/api/v1/recommend/video_in_stream/get`. Negative feedback: `/aweme/v1/commit/dislike/item/` ("not interested"), `/aweme/v1/recommend/user/dislike/`.

### Watch-event telemetry → `/aweme/v1/aweme/stats/`
This is the signal that trains the ranker (web equivalent: the `mcs…/v1/list` collector, which fired ~12× in 20s live). Signal tokens found in code, by frequency: `duration`(581), `enter_from`(310), `replay`(163), `boost`(146), `enter_method`(98), `stay_time`(38), `impr_id`(33), `tag_id`(29), `predict`(27), `cold_start`(27), `video_duration`(25), `is_fullscreen`(22), `category_id`(21), `first_frame`(18), `dedup`(9), `loop_count`(7), `rank_score`(3), `is_end`, `play_time`, `finish_rate`, `skip_cnt`, `interest_tag`, `total_play`. Plus from the web request: `focus_state`, `is_page_visible`, `vv_count`, `history_len`, `time_of_day`, `day_of_week`.
→ It measures **how long you watch, whether you finish, whether you loop, how fast you skip, how you arrived (`enter_from`/`enter_method`), and whether the screen even has your attention** — all implicit, none of it asked.

### Response model `FeedItemList` (decompiled fields) — the delivery tricks
- `hasMore` **and** `hasMoreUp` → **bidirectionally infinite** feed (no top, no bottom).
- `cursor`, `chunkIndex`, `fetchType`, `appendCache` → streamed/chunked, appended seamlessly.
- `enableReRank` + `commercePitayaInfo` → **on-device re-ranking** via Pitaya before display.
- Client-cache flags: `isFromColdCache`, `isFromLocalCache`, `isFromBackupVideo`, `isFromUnusedFeed`, `isFromSplitData` → videos are pre-cached and even **reused from a backup pool** so the feed is never empty.
- `cardInsertResults`, `hasAd` → ads/cards injected inline into the organic stream.
- `hotSearchGuideWord` → a **trending-search prompt injected into the feed** to pull you into search.
- `interventionInfo`, `hidingInvalidItem`, `invalidItemIdList` → server can force-insert or strip items live.
- `FeedAppLogParams`: `pullType`, `unconsumedVideoCount`, `unconsumedVideoReuseCount` → the app tracks preloaded-but-unwatched videos and **reuses** them.

### On-device ML
`libAndroidPitayaCore` / `libdex_df_pitaya` (client ranking/decisioning), `libbytennllm` (on-device LLM), LiteRT/TFLite, bundled Python+NumPy+OpenCV. `enableReRank` means the final ordering can be decided locally → near-zero latency reactions.

---

## 2. Engineered addiction mechanics (UX)

| # | Mechanic | Evidence in build | Why it hooks |
|---|---|---|---|
| 1 | **Autoplay + silent loop** | 163 `replay` / 104 `autoplay` refs; `loop_count` | No decision to start/continue; looping removes the stop between items and inflates the completion signal. |
| 2 | **Single full-screen item, swipe = lever** | feed is one item; vertical pager | Each swipe is a slot-machine pull. |
| 3 | **Never a loading wait** | live: `item_list`(6)+`preload`(3)+`prefetch/explore`(24) fired together; `FeedItemList` cold/local/backup caches; `libttpreload`,`libavmdl` P2P | No spinner = no pause for the brain to quit. |
| 4 | **Bidirectional infinite feed** | `hasMore` + `hasMoreUp` | No natural endpoint ever. |
| 5 | **Variable-ratio reward** | hits interleaved with filler; `boost`, `cardInsert` | Unpredictable payoff = most compulsive reinforcement schedule. |
| 6 | **~2-hour niche lock-in** | watch-time-first signals (above) | Feed narrows fast, you feel "seen," you stay. |
| 7 | **Pull-to-refresh** | `pullType` | Manual gamble gesture (classic slot pull). |
| 8 | **Inline trending-search bait** | `hotSearchGuideWord`, `/trending/search/inflow/` | Routes you from passive watching into active searching → more surface. |

---

## 3. GIFs, stickers, emoji
- **Both Giphy and Tenor** embedded: `giphy_tab`, `giphy_bottom_panel`, `giphy_embedded_panel`, `giphy_stickers`, `giphy_show` (mob events), `tenor_popular_sticker_cache`. GIF/sticker search baked into DMs and comments.
- Sticker endpoints: `/aweme/v1/sticker/recommendation/`, `/discover/`, `/detail/`, `/listcollection/`, `/translation/sticker/`.
- **AI "Imagine" stickers**: `/tiktok/v1/imagine_sticker/generate` (+`/append`,`/status_update`,`/elements`) — generative stickers to boost expression/sharing.
- **Social avatars** (AI): `/tiktok/v1/social_avatar/generate` + like/notification/animation loops.
- Live **gifts**: `/tiktok/v1/gift/list|send|eligibility|setting` — monetized emoji in live.

---

## 4. Messaging (DMs / IM) — engineered stickiness
- Cloud IM: `/aweme/v1/im/cloud/token/`, conversation/message/read endpoints, conversation wallpaper, typing/read states.
- **Streaks (Snapchat-style daily compulsion):** `/tiktok/v1/im/streaks/get|info|compare|compensation` — *plus a virtual "streak pet"*: `/im/streaks/pet/info/update`, `/pet/share`, `/pet/invitation/accept`, `/pet/delete`. Also `story/get_streaks_info`, `effect/streak/stats`. A streak + a decaying pet = guilt-driven daily return and a two-person obligation loop.
- `streaks/compensation` = lets users **buy back a broken streak** (removes the natural quit-point).
- **AI chatbots in DMs:** `/tiktok/v1/im/bot/create|list|intent_recognition|recommend_prompt|task_list|message_action`, `im/ai_moji/create`, visual-poet — always-available conversational engagement.
- Business messaging (`im/b2c/*`): quick replies, welcome messages, DM frequency management, popups.
- Group mechanics: invites, invite popups, fan groups, related-friends suggestions.

---

## 5. Suggestions & the social graph (the "who/what to engage" engine)
- **Contact harvesting:** `/aweme/v1/upload/contacts/`, `/upload/hashcontacts/`, and `/tiktok/v2|v3/upload/psihashcontacts/` (**PSI = private-set-intersection hashed contacts**) → "People you may know" without sending raw numbers.
- People suggestions: `/aweme/v1/recommend/user/` (+`/dislike/`), `/social/friend/`, `/user/recent/contact/`, `/following/page/user/recommend/`.
- **Search suggestions everywhere:** `/search/sug/`, `/sug/stream/`, `clicksug`, `videosug`, `user/sug`, `challengesug`, `drama/sug`, `sug_cached_words`, `sug_pre_info`, `top_history_words`, `/hot/search/list/`, `/trending/search/inflow/` — autosuggest is streamed and pre-fetched (`/search/sug/prefetch/`).
- **Viral invite loops:** "Add Yours" sticker (`/addyours/invite`, `/topic/suggest`, `/topic/auto_suggest`), `mutual_feed/send_invite|accept_invite`, group invite shares.

---

## 6. Notifications & re-engagement (dragging you back)
- **Red dots / badges:** `/aweme/v2/follow/feed/red_dot`, `/effect/api/ttapp/badge/get|mark`, `/user/profile/self/badges/`, `im/bulletin/clear/unread_count/`.
- **Social-validation pings:** `/aweme/v1/commit/item/digg/` + `/digg/list/` (likes), `/upvote/digg`, `/notice/count|multi|multi_user/count`, `/inbox/agg_list/`.
- **Server-pushed content:** `/tiktok/v1/im/push/content/fetch`, `/reach/ttinbox/*` (inbox "move"/entry assets), FCM (Firebase) + Amazon ADM push. Notification settings exist but pings are opt-out.

---

## 7. Gamification & rewards
- User-growth ("UG") rewards: `/api/v1/benefit/claim_benefit`, `/ttec/ug_guide/claim_incentive`, `latam_ug/jumanji/get_activity_direct_message` (regional reward campaigns via DM).
- Ad-incentive loops: `/tiktok/v1/ad/incentive/`, `/incentive/action/report/`, `/qcpx_incentive/claim/`.
- Shop task/marketing loops: `/shop/marketing_interactive/task/register`, creator live-task cards, shipping-address claim incentives.
- Creation quests: AIGC tasks, music-generation tasks, photo tasks, effect streak stats.

---

## 8. The "countermeasures" (present, deliberately non-intrusive)
`/tiktok/v1/screen_time/upload/` + `/screen_time/action/`, `/digital_wellbeing/settings/`, teen/minor settings, `/teen/protector/vote/`, `/item/restrict/`, and a **full separate Kids mode** (its own feed, effects, music, `/kids/break/set`, `/kids/screen_time/upload`, `/kids/check/in`). Screen-time is **uploaded/tracked regardless**, but interventions are opt-in and dismissible — compliance tooling, not a brake on the core loop. (App also holds `DETECT_SCREEN_RECORDING` / `DETECT_SCREEN_CAPTURE`.)

---

## The whole machine in one picture
A **watch-time-driven online-learning loop** (`/feed/` out → `/aweme/stats/` in, re-ranked partly on-device by Pitaya) is wrapped in a **zero-friction, autoplay-looping, always-preloaded, bidirectionally-infinite** feed delivering **variable-ratio rewards** per swipe. Around it: **streaks + a decaying virtual pet**, **AI chatbots**, **GIF/sticker/AI-sticker** expression, **contact-graph + search suggestions** to widen engagement, and **red-dot/like/push** re-engagement — all monetized through inline ads and TikTok Shop. Time-management tools exist but are off by default.

### Honest limits
Endpoint paths, model field names, and signal tokens are **real** (string literals + decompiled `com.ss.android.ugc.aweme.*` model classes). The **ranking weights and the Argus-signed, Pitaya-scored model internals are not extracted** — they're native/server-side and obfuscated. The "why addictive" ties named, in-build mechanics to established behavioral-design principles + the WSJ/Monolith research, not to recovered proprietary math.
