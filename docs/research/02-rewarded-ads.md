# 02 Rewarded ads: "watch an ad to get a try"

| Field | Value |
|---|---|
| Track | Rewarded ads (research and plan) |
| Date | 2026-09-24 |
| Status | Research complete. Input for the build plan. No project code written. |
| Scope | Ad-for-a-try flow in the PlayToEarn Playground: providers, policies, consent, verification, anti-fraud, adapter interface, demo mock |
| Legend | **[V]** verified in a primary source (fetched this session). **[3P]** third-party or vendor claim. **[EST]** estimate. **UNVERIFIED** = could not confirm. |
| Code status | All TypeScript in section 3 was type-checked with `tsc 5.9 --strict` (DOM lib for client, Node types for server). The server snippets (SSV verifier, mock signer, rules) passed a throwaway self-test under Bun (tamper, unknown key, duplicate, grace, caps). The DOM mock was type-checked only, not run in a browser. Re-checked in verification: all seven TS blocks type-check with tsc 5.9.3 `--strict`, and the SSV verifier plus mock signer pass an independent Bun test (valid, tampered, unknown key, percent-encoded `custom_data`). The rules self-test cannot be reproduced from the excerpt in 3.8. |
| Verification | Adversarial review on 2026-09-24. Wrong statements were fixed in place; unconfirmed ones are marked UNVERIFIED. See "Verification log" and "Omissions found in verification" at the end. |

---

## TL;DR

1. **AdMob cannot run on playtoearn.com web pages.** It is for app inventory only; the web equivalents are AdSense and Ad Manager [V] ([S6](https://support.google.com/admob/answer/9234653?hl=en)). AdMob fits only if the Playground runs inside a PlayToEarn native app. An Android app already exists: `com.playtoearn.playtoearn`. Its store listing shows 100,000+ installs, a contains-ads label, a release date of 2021-01-10 and a last update on 2024-11-15 [V] ([S61](https://play.google.com/store/apps/details?id=com.playtoearn.playtoearn&hl=en_US)).
2. **On the open web, Google offers rewarded ads through two products: H5 Games Ads (AdSense, by application only) and Ad Manager GPT rewarded. Neither can verify rewards server-side.** Google says: "Server-side verification is an app only feature and it is unavailable for web use." ([S30](https://support.google.com/admanager/answer/9116812?hl=en)). Web rewards therefore rest on a browser event, and strict caps are mandatory.
3. **The biggest risk is Google's rewards policy, not the technology.** Google's rewarded-ads policies ban direct monetary rewards and name cryptocurrency explicitly. An indirect reward is allowed only if three things hold: it is usable only on the publisher's platform, it cannot be transferred, and it cannot be converted directly into money [V] ([S9](https://support.google.com/admob/answer/7313578?hl=en), [S10](https://support.google.com/adsense/answer/9121589?hl=en)). The H5 API guide is stricter: a reward must have no value outside the app and must not be easy to exchange for money [V] ([S16](https://developers.google.com/ad-placement/apis)). P2E Points can be redeemed for USDT and tokens ([S63](https://playtoearn.com/earn)). A try earned by watching an ad does not pay out directly, but it competes for prizes that can be cashed out, so it is a gray zone. **Get written clearance from Google before using Google demand**, and keep a config switch `adTriesPrizeEligible`.
4. **Third-party web networks rarely fit this case.** AppLixir rejects crypto-related sites [V] ([S32](https://support.applixir.com/frequently-asked-questions)). Playwire needs 500K monthly pageviews [V] ([S37](https://www.playwire.com/faq)). Venatus reportedly needs about 1.5M [3P] ([S41](https://blog.nitropay.com/nitro-vs-playwire-vs-venatus-which-ad-network-is-right-for-gaming-publishers/)). AdinPlay and Nitro offer rewarded formats but document no server callback. The GameDistribution and GameMonetize SDKs only cover games published in their catalogs. Adsgram, and Monetag's rewarded SDK, are documented for Telegram Mini Apps only. **Exception found in verification:** ayeT-Studios is already listed as a DIRECT seller in playtoearn.com's ads.txt. It documents an HTML5 rewarded video SDK for desktop and mobile browsers, with server-to-server callbacks and an optional HMAC-SHA256 header [V] ([S70](https://playtoearn.com/ads.txt), [S71](https://docs.ayetstudios.com/v/product-docs/rewarded-video/web-integrations/rewarded-video-sdk-for-html5), [S75](https://docs.ayetstudios.com/v/product-docs/callbacks-and-testing/callback-verification/hmac-security-hash-optional)). Ask ayeT first (see Omissions).
5. **Recommended architecture:** a provider-agnostic `AdsProvider` adapter on the client, plus a server-issued single-use **ad reward ticket** with four verification tiers (T1 to T4). The demo ships `MockAdProvider` with two modes:
   * a client-callback mode;
   * a simulated signed-callback mode that exercises the real AdMob SSV verifier with a local P-256 key.
6. **Recommended providers for production:**
   * **Web primary:** Google H5 Games Ads, if approved and policy-cleared.
   * **Web fallback 1:** a first-party "sponsored video" sold through PlayToEarn's existing ad business ([S64](https://business.playtoearn.com/)). It is crypto-friendly and server-timed.
   * **Web fallback 2:** always the 10-points option.
   * **Web candidate added in verification:** ayeT-Studios rewarded video with an HMAC-signed S2S callback (T2). playtoearn.com's ads.txt already lists an ayeT account as DIRECT. Use it only after ayeT confirms the try reward and crypto content in writing.
   * **Native apps:** AdMob rewarded plus SSV through a JavaScript bridge.
7. **Default daily caps per user:**
   * T1 (signed server callback): 10
   * T2 (shared-secret server callback): 6
   * T3 (first-party video): 5
   * T4 (client callback only): 3
   * All ad tries combined: at most 10
   * New accounts: 3 in total, of which at most 1 via T4
   * Other limits: 1 open ticket, 20 to 45 s cooldown by tier, 10 min ticket TTL.

   **Caps are checked before the ad is shown. Verified completions are always honored**, because Google requires publishers to deliver the promised reward [V] ([S10](https://support.google.com/adsense/answer/9121589?hl=en)).
8. **Consent:**
   * Personalized Google ads need a Google-certified TCF CMP: in the EEA and UK since 2024-01-16, in Switzerland since 2024-07-31 [V] ([S53](https://support.google.com/adsense/answer/13554116?hl=en)).
   * TC strings created after 2026-02-28 need the TCF v2.3 `disclosedVendors` segment [V] ([S54](https://iabeurope.eu/all-you-need-to-know-about-the-transition-to-tcf-v2-3/)).
   * In the US, a GPP opt-out triggers restricted data processing [V] ([S57](https://support.google.com/admanager/answer/14117049?hl=en)).
   * Without consent, expect limited or non-personalized ads and lower fill.
9. **Ad blockers:** about 29.5% of internet users worldwide use ad blockers (GWI, Q2 2025) [3P] ([S59](https://backlinko.com/ad-blockers-users)). The often-quoted 37% desktop figure is a 2020 US survey (corrected in verification). Brave blocks ads by default and passed 100M monthly users [V] ([S60](https://brave.com/blog/100m-mau/)). Every ad path must fall back to "Use 10 points" without nagging.
10. **Revenue will be modest [EST]:**
    * Web rewarded pays roughly $4 to $12 eCPM; native apps in Tier-1 markets pay $15 to $30 ([S36](https://www.applixir.com/blog/ecpm-optimization-for-web-games-benchmarks-and-fixes/), [S40](https://www.playwire.com/blog/admob-ecpm-benchmarks-what-publishers-should-expect)).
    * Example: 3,360 web impressions a day earn about $13 to $34 a day.
    * Treat the ad option as a way for players to keep playing, not as a revenue pillar.
    * An ad try and a points try are worth the same to the platform when the USD value of one point equals `eCPM_net / 5000`.

---

## 1. Scope, terms and key facts about PlayToEarn

| Term | Meaning in this report |
|---|---|
| Rewarded ad | Opt-in full-screen ad. The user chooses to watch it in exchange for an in-app reward. |
| Rewarded interstitial | Rewarded ad that appears at a natural break without an explicit opt-in. It needs an intro screen with a way to decline (AdMob and Ad Manager only) [V] ([S5](https://developers.google.com/admob/android/rewarded-interstitial)) |
| SSV | AdMob / Ad Manager server-side verification: Google calls our URL with a signed query string. Apps only. |
| S2S callback | Any provider-to-our-server reward notification (signed or shared-secret) |
| Client callback | Browser JS event such as `adViewed` or `rewardedSlotGranted`. It can be forged. |
| Ad reward ticket | Single-use server record created before the ad is shown. It binds user, provider, game and reward. |
| Ad try | One extra game attempt granted after a verified ad. It is the only reward type. It is never P2E Points and never USDT. |

| PlayToEarn fact relevant to ads | Evidence |
|---|---|
| Points cash out: the /earn page is titled "PlayToEarn Rewards & Tasks - Earn USDT, Tokens & Points Daily" (dash normalized) and says 2,000 points are needed for the first payout. The redemption catalog itself was not visible. | [V] /earn read with WebFetch during verification on 2026-09-24 ([S63](https://playtoearn.com/earn)). Plain curl still gets a Cloudflare challenge, which was not bypassed. |
| Plus membership: $9.99 per month or $99.90 per year. Benefits: double P2E Point earnings, ad-free browsing (no banners, pop-ups or interruptions), exclusive Reward Center prize pools, early access to in-house games on desktop, Android and iOS, and member-only challenges with special prize pools. | [V] page read during verification ([S62](https://playtoearn.com/plus)) |
| PlayToEarn already runs a self-serve Web3 ad business. business.playtoearn.com is the Business Center sign-in page ("Where Web3 Ads Drive Real Growth"). The /advertise page lists Top Trending, Top Game List, Banner, Notifications, Newsletter banners and Social Media features, with a three-step self-serve flow. No video or rewarded format is listed. The earlier "more than 14 ad products" could not be confirmed: UNVERIFIED. | [V] ([S64](https://business.playtoearn.com/), [S73](https://playtoearn.com/advertise)) (corrected in verification) |
| Audience, self-reported on /advertise: more than 180,000 registered users, "millions of monthly visitors", 60% of users on mobile, 40% of visitors from search. | [V] vendor self-report ([S73](https://playtoearn.com/advertise)) |
| playtoearn.com/ads.txt (189 lines, fetched 2026-09-24) has 173 seller records: 21 DIRECT and 152 RESELLER. The DIRECT records are three `google.com` publisher IDs, two `ayetstudios.com` entries (AYETSTUDIOS and PL-22398), and 16 others including Amazon APS, PubMatic, Sovrn (lijit), 33Across, Adform, SmileWanted, AdYouLike, vi.ai, AMX and Alkimi. Google demand and header bidding are therefore authorized on the site today. Which Google product (AdSense or Ad Manager / AdX) owns each ID is not visible: UNVERIFIED. | [V] ([S70](https://playtoearn.com/ads.txt)) |
| Android app `com.playtoearn.playtoearn` ("PlayToEarn - Crypto Games List"): flagged as containing ads, 100,000+ installs, updated 2024-11-15 | [V] ([S61](https://play.google.com/store/apps/details?id=com.playtoearn.playtoearn&hl=en_US)), re-checked in verification |
| Hints about the site stack from the saved /earn page head: Laravel-style `csrf-token` meta, jQuery, Backbone, WalletConnect, Google Identity Services and GA4. No CMP or ad tag in the saved head. | UNVERIFIED (partial page). The /earn page offers wallet (MetaMask, Phantom, WalletConnect), Discord, X and Google sign-in [V] ([S63](https://playtoearn.com/earn)). A missing ad tag in that head does not mean no programmatic ads: see the ads.txt row. |

---

## 2. Detailed findings

### 2.1 Google AdMob (native apps)

#### 2.1.1 Platform scope

| Question | Answer | Source |
|---|---|---|
| Web pages? | No. AdMob is app inventory only. AdSense covers web; Ad Manager covers web and apps. | [V] [S6](https://support.google.com/admob/answer/9234653?hl=en) |
| Platforms | Android and iOS through the Google Mobile Ads SDK (also Unity, Flutter and other wrappers). | [V] [S1](https://developers.google.com/admob/android/ssv), [S2](https://developers.google.com/admob/ios/ssv) |
| Formats relevant here | Rewarded (opt-in) and rewarded interstitial (auto-shown with an intro screen and skip option). Both support SSV options. | [V] [S5](https://developers.google.com/admob/android/rewarded-interstitial) |
| Ad expiry | A loaded rewarded interstitial expires after 1 hour, so reload it before showing. | [V] [S5](https://developers.google.com/admob/android/rewarded-interstitial) |
| Which format to use | **Rewarded (opt-in)**. The user asks for the ad on the "out of tries" screen, so the rewarded-interstitial intro-screen rules are unnecessary. | Recommendation |

#### 2.1.2 SSV callback specification [V] ([S1](https://developers.google.com/admob/android/ssv), [S2](https://developers.google.com/admob/ios/ssv), [S3](https://support.google.com/admob/answer/9603226?hl=en))

Parameters arrive in alphabetical order. `signature` and `key_id` are always the last two, in that order.

| Param | Meaning | Our use |
|---|---|---|
| `ad_network` | ID of the ad source that filled the ad | Log only (mediation analytics) |
| `ad_unit` | AdMob ad unit ID used to request the ad (numeric part) | Must equal the configured Playground ad unit |
| `custom_data` | Optional string set in the SDK before `show()`. Percent-escaped. Absent if not set. | **ticketId** (short opaque id, no PII) |
| `key_id` | ID of the key used to verify the signature | Look it up in the key cache |
| `reward_amount` | From ad unit settings | Must equal `1` |
| `reward_item` | From ad unit settings | Must equal `try` |
| `signature` | Google's signature over all preceding params | Verify |
| `timestamp` | Reward time. The docs say epoch ms, but the documented sample `1507770365237823` has 16 digits (microseconds). | Parse defensively. Treat a value above 1e14 as µs. |
| `transaction_id` | Unique hex id per reward grant | **Idempotency key** (unique index) |
| `user_id` | Optional string set in the SDK. Absent if not set. | Our internal user id. Must match the ticket. |

| Verification step | Detail |
|---|---|
| Signed content | The UTF-8 bytes of the raw query string **up to, but not including, `&signature=`**. Do not modify, re-order or re-encode it [V] ([S2](https://developers.google.com/admob/ios/ssv)). Caveat found in verification: Google's reference verifier (Tink `RewardedAdsVerifier`) reads `java.net.URI.getQuery()`, which returns the percent-decoded query, while raw-byte verifiers (this report, voonic) use the encoded form [V] ([S67](https://github.com/tink-crypto/tink-java-apps)). The two only agree when no value contains a percent-escape, so keep `custom_data` (ticketId) and `user_id` to URL-unreserved characters. ULIDs and numeric ids qualify. |
| Algorithm | ECDSA with SHA-256 [V]. The key served on 2026-09-24 is `prime256v1` (P-256), keyId `3335741209`. |
| Signature encoding | base64url (the sample contains `-` and `_`), DER-encoded ECDSA signature (sample decodes to 71 bytes starting `0x3045 0221`) [V, tested] |
| Key server | `https://www.gstatic.com/admob/reward/verifier-keys.json`. JSON `{keys:[{keyId, pem, base64}]}` [V] ([S4](https://www.gstatic.com/admob/reward/verifier-keys.json)). On 2026-09-24 it returned `Cache-Control: max-age=0`, one key, `Last-Modified: 2023-06-30`. |
| Key caching | Cache, but never longer than 24 h. Keys rotate on a variable schedule [V] ([S1](https://developers.google.com/admob/android/ssv)). Refetch on an unknown `key_id`, rate-limited. |
| Retries | Google expects HTTP 200. If the server is unreachable or doesn't answer 200, Google retries up to 5 times at 1-second intervals [V] ([S1](https://developers.google.com/admob/android/ssv)). **Implication:** answer 200 for every authentic callback, including duplicates and business-rule rejections. Use 5xx only for transient failures. |
| Setup | AdMob UI: Apps > ad unit > Advanced settings > Server-side verification. You enter the callback URL and **must press "Verify URL"** before saving. A mediation option applies SSV to all networks in the mediation groups, including third-party ones [V] ([S3](https://support.google.com/admob/answer/9603226?hl=en)). |
| Libraries | Google points to Tink `RewardedAdsVerifier` (Java, `tink-crypto/tink-java-apps`) [V] ([S2](https://developers.google.com/admob/ios/ssv)). Any ECDSA library works. Node `crypto.verify` works, as tested in section 3.8. |

| Platform | How to set `user_id` / `custom_data` (before `show()`) |
|---|---|
| Android (legacy GMA SDK) | `ServerSideVerificationOptions.Builder().setUserId(uid).setCustomData(ticketId).build()`, then `rewardedAd.setServerSideVerificationOptions(opts)` [V] ([S1](https://developers.google.com/admob/android/ssv), [S5](https://developers.google.com/admob/android/rewarded-interstitial)) |
| iOS | `GADServerSideVerificationOptions` with `userIdentifier` and `customRewardString`, assigned to `rewardedAd.serverSideVerificationOptions` [V] ([S2](https://developers.google.com/admob/ios/ssv)). The current Swift sample on that page uses the unprefixed `ServerSideVerificationOptions`. |
| Android next-gen SDK | `ServerSideVerificationOptions.Builder().setCustomData(...).build()`, then `rewardedAd.setServerSideVerificationOptions(options)` [V] ([S74](https://developers.google.com/admob/android/next-gen/rewarded)). A user id setter is not shown on that page (UNVERIFIED). (Corrected in verification: names were UNVERIFIED.) |

Framework gotcha, important for a PHP or Laravel backend:
* Symfony / Laravel `Request::getQueryString()` returns a **normalized** string: keys sorted with `ksort` and re-escaped. That breaks the signed content [V] ([S65](https://github.com/symfony/http-foundation/blob/7.3/Request.php)).
* Use the raw `$_SERVER['QUERY_STRING']` or `$request->server->get('QUERY_STRING')` instead.
* In Express, use `req.originalUrl`, not `req.query`.

#### 2.1.3 WebView scenarios (Playground inside a native app)

| Scenario | How it works | Rewarded? | Server verification | Verdict |
|---|---|---|---|---|
| A. H5 Games Ads inside an app WebView | The web game uses the Ad Placement API. The app registers the WebView with the Mobile Ads SDK, and the tag gets `data-admob-rewarded-slot` / `data-admob-interstitial-slot` [V] ([S24](https://support.google.com/adsense/answer/9955214?hl=en), [S22](https://developers.google.com/ad-placement/docs/example)) | Yes | No documented SSV for this path (UNVERIFIED) | **Android only** [V] ([S25](https://adsense.google.com/start/solutions/h5-games-ads/)). Client-callback trust (T4). |
| B. WebView API for Ads | Registers the WebView (`MobileAds.registerWebView`) so AdSense, GPT and IMA tags inside it get app signals. Needs GMA SDK 20.6.0+, API 21+ and the manifest meta-data `com.google.android.gms.ads.INTEGRATION_MANAGER=webview`. An iOS version exists [V] ([S7](https://developers.google.com/admob/android/browser/webview/api-for-ads)). | Only whatever the web tag offers | No | Needed if web ad tags run inside the app. Google encourages it for web content shown in WebViews [V] ([S8](https://support.google.com/admanager/answer/6310245?hl=en)). |
| C. **Native bridge (recommended)** | The web Playground calls `window.P2ENative.showRewarded(ticket)`. The native layer loads and shows an AdMob `RewardedAd` with SSV options (userId, customData = ticketId), then reports events back to JS. | Yes | **Yes, SSV (T1)** | Best trust and eCPM. Google allows AdMob in-app ads next to WebView content when the GMA SDK is used [V] ([S8](https://support.google.com/admanager/answer/6310245?hl=en)). |

Consent note [V] ([S7](https://developers.google.com/admob/android/browser/webview/api-for-ads)): consent collected by the app through mobile IAB frameworks (TCF v2.3, CCPA) does **not** carry over to web ad tags inside a WebView. With path A or B the web page needs its own CMP. With path C, the app's UMP consent governs.

#### 2.1.4 Google Play policy constraints on the native path (to confirm with the legal track)

| Policy | What it says (paraphrased) | Impact |
|---|---|---|
| Real-money games, contests and tournaments | Apps may not let users take part using real money, or in-app items bought with money, to win prizes of real-world monetary value. The exceptions are licensed gambling apps, daily fantasy sports apps, time-limited pilot programs and qualifying gamified loyalty programs; none obviously fits the Playground [V] ([S14](https://support.google.com/googleplay/android-developer/answer/9877032?hl=en)) (corrected in verification: the earlier text said pilots were the only exception). | Paid tries cost points, not money. Prizes are points that cash out to USDT. Plus is a paid membership that gives 9 free tries. **Needs legal review before shipping the Playground in the Play app.** |
| Gamified loyalty programs in game apps | Loyalty rewards tied to purchases must use a fixed ratio. Their earning or redemption value may not be awarded or multiplied by game performance or chance [V] ([S14](https://support.google.com/googleplay/android-developer/answer/9877032?hl=en)). | Weekly leaderboard point prizes are awarded by game performance. Needs legal review. |
| Blockchain-based content (2023) | Apps must not promote or glamorize potential earnings from playing or trading. Apps that sell or let users earn tokenized digital assets must declare this in the Play Console Financial features declaration [V] ([S15](https://support.google.com/googleplay/android-developer/answer/13607354?hl=en), fetched in verification) | Playground copy inside the app must not advertise "earn crypto by playing". If the app lets users earn tokens through the Reward Center, the declaration applies (legal track). |

### 2.2 Google rewarded-ads policies (AdMob, AdSense, Ad Manager)

Paraphrased from [S9](https://support.google.com/admob/answer/7313578?hl=en), [S10](https://support.google.com/adsense/answer/9121589?hl=en), [S11](https://support.google.com/admanager/answer/7496282?hl=en) and [S16](https://developers.google.com/ad-placement/apis) [V]:

| Rule | Requirement | How the Playground design complies |
|---|---|---|
| Opt-in | The user must opt in clearly before a rewarded ad is served. Exception: rewarded interstitial, which needs an intro screen with a "no" option. | Ads start only from our "Watch ad" button. No auto-play chains. |
| Disclosure | Before **each** rewarded ad, clearly state the action required and the reward. | The modal shows "Watch a short ad to get 1 extra try (for GAME). Closing early means no try." It also shows ad tries left today. |
| Dismissible | The ad must be skippable or dismissible. | Providers handle this. The mock shows a close button after 5 s and warns that closing forfeits the try. |
| Deliver the reward | The publisher must grant the promised reward after the required action. | Caps are checked **before** showing. Verified completions are always granted (section 3.5). |
| No implied Google endorsement | The publisher alone is responsible for granting rewards and must not imply Google verifies them. | Never write "verified by Google". |
| Direct monetary items | Forbidden in all cases. Defined to include cash, **cryptocurrency** and gift cards. | Never grant P2E Points, USDT or tokens for ads. **The reward is a try only.** |
| Indirect or non-monetary items | Allowed only if redeemable only on the publisher's platform and non-transferable. Non-transferable (same definition in the AdMob, AdSense and Ad Manager policies, re-checked in verification) also excludes rewards directly convertible into direct monetary items or transferable items. Discounts on physical items are capped at 25%. | A try is bound to the user, cannot be transferred and expires at the daily reset. **Gray zone:** the try competes for points that cash out (next table). |
| Random rewards | Allowed if the odds and all possible rewards are disclosed and the chance is above 0% (AdMob, Ad Manager). | Not used. A try is deterministic. The prize depends on skill and rank. |
| Clicks | Never encourage clicks. AdSense forbids compensating users for viewing ads except in rewarded inventory [V] ([S12](https://support.google.com/adsense/answer/48182?hl=en)). | No reward for clicks. No "click the ad" or "support us" wording. |
| H5 Ad Placement API wording (strictest) | Rewards must have no value outside the app, must not have (or be easily exchanged for) monetary value, and must not be saleable or exchangeable for goods or services. | See the gray-zone table. |

**Gray-zone assessment for the P2E model:**

| Element | Google risk | Mitigation built into the design |
|---|---|---|
| Ad grants a try, not points | Low on its own | The reward type is fixed to `try` in code and in the ad-unit config (`reward_item=try`, `amount=1`). |
| A try can win weekly prizes (points), and points cash out to USDT / tokens | **High uncertainty.** A reviewer may treat it as indirect access to a direct monetary item. A Google community thread asks exactly this ("chances" at monetary items), but the answer could not be read (UNVERIFIED) ([S68](https://support.google.com/admob/thread/321297311/are-chances-at-winning-direct-monetary-items-allowed-for-watching-reward-ads?hl=en)). | 1. Ask for written clearance: through the H5 Games Ads application, or a Google account manager if PlayToEarn has one. 2. Config flag `adTriesPrizeEligible` (default `true`). If Google objects, runs from ad tries become practice runs that count for personal best but not for prizes. 3. Keep ad tries fully separate in the ledger. |
| Ad revenue feeding prize pools | **High.** It would tie ad viewing directly to cash-out value. | **Do not** add ad revenue to the dynamic bonus or any prize pool while Google demand is used. |
| Paid tries (points) competing for prizes: "online gambling" publisher restriction | Medium. Google's restriction covers internet games where money or other items of value are paid or wagered to win real money or prizes. Result: fewer eligible advertisers, not a ban. There are country exclusions (US, UK, DE, FR, JP and others) [V] ([S13](https://support.google.com/publisherpolicies/answer/10437795?hl=en)). | Legal track to assess. Monitor AdSense policy center warnings for Playground URLs. |
| Invalid traffic from reward-motivated users | **High impact.** Enforcement is account-level: the AdSense program policies reserve the right to disable ad serving to the site and/or disable the AdSense account [V] ([S12](https://support.google.com/adsense/answer/48182?hl=en), confirmed in verification). playtoearn.com's ads.txt lists three Google publisher IDs as DIRECT ([S70](https://playtoearn.com/ads.txt)), so existing site revenue is exposed. | Strict caps, risk scoring before showing any Google ad, separate ad units and channels for the Playground, IVT monitoring. |

Advertiser-side note: Google Ads restricts crypto advertisers. Since 2023-09-15 it has also barred ads for NFT games that let players wager or stake NFTs for anything of real-world value [3P] ([S69](https://decrypt.co/155185/google-changes-policy-allow-nft-game-ads-some-limits)), confirmed in Google's own policy update [V] ([S78](https://support.google.com/adspolicy/answer/13985443?hl=en)). Google demand on PlayToEarn will therefore be mostly non-crypto brands. That is fine for fill, but it means crypto advertisers can only be reached through PlayToEarn's own sales (section 2.6).

### 2.3 Web option A: Google H5 Games Ads (Ad Placement API)

| Topic | Finding |
|---|---|
| Eligibility | A by-application product: you apply with a form. Approval is not guaranteed and depends on partner eligibility. An **approved AdSense account is required**. Once approved, the API grants a limited-scope permission to use full-screen ads [V] ([S19](https://developers.google.com/ad-placement/docs/signup)). Google also says any publisher owning an H5 games website may apply if it follows AdSense policies [V] ([S25](https://adsense.google.com/start/solutions/h5-games-ads/)). Whether a Playground section of a crypto directory qualifies is UNVERIFIED. **Apply early.** |
| Ad Manager | H5 Games Ads also exist in Ad Manager (interstitial and rewarded; fullscreen, iFrame/WebView and embedded structures) [V] ([S26](https://support.google.com/admanager/answer/14637831?hl=en)). Eligibility details were not on the page. |
| Formats | Interstitial and rewarded. Both can use display and video (TrueView, Bumper) [V] ([S23](https://support.google.com/adsense/answer/9959170?hl=en)) |
| Rewarded on the open web | Yes. The API targets HTML5 games both on the web and inside apps [V] ([S22](https://developers.google.com/ad-placement/docs/example)). Desktop rewarded fill is UNVERIFIED. The docs do not restrict devices. |
| Rewarded flow | 1. Call `adBreak({type:'reward', name, beforeAd, afterAd, beforeReward, adDismissed, adViewed, adBreakDone})`. 2. `beforeReward(showAdFn)` is called synchronously only if an ad is available. 3. The game shows its prompt and calls `showAdFn()` on click. 4. `adViewed`: grant the reward. `adDismissed`: no reward. 5. `afterAd`, then `adBreakDone(placementInfo)` always runs last. Calling a `showAdFn` from an earlier break has no effect. A new `adBreak` resets the previous one [V] ([S16](https://developers.google.com/ad-placement/apis), [S17](https://developers.google.com/ad-placement/apis/adbreak)) |
| `breakStatus` values | `notReady`, `timeout`, `invalid`, `error`, `noAdPreloaded`, `frequencyCapped`, `ignored`, `other`, `dismissed`, `viewed` [V] ([S17](https://developers.google.com/ad-placement/apis/adbreak)) |
| `adConfig` | `preloadAdBreaks: 'on'|'auto'` can be set only once, before the first `adBreak`. `sound: 'on'|'off'` (default on). `onReady` [V] ([S18](https://developers.google.com/ad-placement/apis/adconfig)) |
| Tag parameters | `data-ad-client` (required), `data-adbreak-test="on"` (test mode), `data-ad-frequency-hint`, `data-ad-channel`, `data-ad-host`, `data-admob-interstitial-slot`, `data-admob-rewarded-slot`, `data-tag-for-child-directed-treatment`, `data-tag-for-under-age-of-consent`. They are read at page load and cannot change during the pageview [V] ([S24](https://support.google.com/adsense/answer/9955214?hl=en)) |
| Frequency | Default hint `120s`. The fastest allowed rate is one ad every 30 s. The first ad is exempt [V] ([S21](https://developers.google.com/ad-placement/docs/ad-rate)). Expect `frequencyCapped` when players chain ads, so our cooldown should be at least the hint. |
| Placement rules | No full-screen ads that could be confused with normal operation, that interrupt continuous gameplay, or that interfere with navigation [V] ([S23](https://support.google.com/adsense/answer/9959170?hl=en)). Our placement is the "out of tries" screen, never mid-run. |
| Server verification | **None.** Client callbacks only, so tier T4. |
| Policy fit for P2E | See the gray-zone table in 2.2. The H5 wording is the strictest. |
| Effort | 1 to 2 dev-days for the adapter (section 3.9), plus approval wait time. |

### 2.4 Web option B: Ad Manager rewarded for web (GPT)

| Topic | Finding |
|---|---|
| API | `googletag.defineOutOfPageSlot(path, googletag.enums.OutOfPageFormat.REWARDED)` **may return null** on unsupported pages or devices. Events: `rewardedSlotReady` (call `event.makeRewardedVisible()` on user click), `rewardedSlotGranted` (payload `{amount, type}`), `rewardedSlotVideoCompleted`, `rewardedSlotClosed`. `slotRenderEnded.isEmpty` signals no fill. No `<div>` is needed [V] ([S27](https://developers.google.com/publisher-tag/samples/display-rewarded-ad), [S28](https://developers.google.com/publisher-tag/reference), [S29](https://github.com/googleads/google-publisher-tag-samples/tree/main/dist/display-rewarded-ad/ts)) |
| Page requirement | Only on mobile-optimized pages with neutral zoom, typically `<meta name="viewport" content="width=device-width, initial-scale=1">` [V] ([S27](https://developers.google.com/publisher-tag/samples/display-rewarded-ad)) |
| Devices | Desktop, mobile and tablet web inventory [V] ([S30](https://support.google.com/admanager/answer/9116812?hl=en)) |
| Reward rules | Reward type and amount are set on the ad unit (the default is 1 reward). Impressions count when the ad shows. A display creative must be in view **5 seconds** before the reward is granted. Completion means a video finished, a set display time passed, or the TrueView skip threshold was reached. The ad stays up until the user closes it [V] ([S30](https://support.google.com/admanager/answer/9116812?hl=en)) |
| Demand | Reservations, Preferred Deals, Open Auction, Private Auctions and Programmatic Guaranteed. Video and display both fill by default. The network setting "Block non-instream video ads" must be off [V] ([S30](https://support.google.com/admanager/answer/9116812?hl=en)) |
| Limitations | No simultaneous rewarded requests. The publisher must close or destroy slots. Do not show reward prompts for non-rewarded ads [V] ([S30](https://support.google.com/admanager/answer/9116812?hl=en)) |
| SSV | **Not available on web** [V] ([S30](https://support.google.com/admanager/answer/9116812?hl=en)), so T4 |
| Account | Signing up for Ad Manager requires a valid AdSense account, but only to set up the Ad Manager account; AdSense need not be used afterwards. Google may need to talk to the publisher to finish setup [V] ([S31](https://support.google.com/admanager/answer/7084151?hl=en), confirmed in verification). The Amazon APS line in playtoearn.com's ads.txt hints that GPT may already be in use (UNVERIFIED). |
| Bonus | PlayToEarn could sell Playground rewarded placements to its Web3 advertisers as Ad Manager reservations and backfill with AdX. Still T4 on web. |
| Effort | 1 to 2 dev-days if PlayToEarn already uses GPT. |

### 2.5 Web option C: third-party networks

| Provider | Web | Rewarded | Server callback / signed reward | Crypto-site eligibility | Onboarding | Effort [EST] | eCPM [EST] |
|---|---|---|---|---|---|---|---|
| **AppLixir** | Yes (HTML5, Phaser, PixiJS, Unity WebGL and others) [V] ([S35](https://support.applixir.com/ai-info/)) | Rewarded video only | **Yes.** HTTPS GET webhook with `gameApiKey, gameId, userId, customData, tid, signature`, where `signature = md5(gameApiKey+gameId+userId+tid+secret)`. `customData` is **not** signed. Dedupe on `tid`. Delivery is at-least-once. Found in verification: every callback also carries the raw `secretKey` in plaintext (deprecated for MD5 modes, to be removed), so redact it before logging [V] ([S33](https://support.applixir.com/applixir-integration/integration-for-html5-sites-apps/step-4-setting-up-local-callback-360053188774)) | **No.** Its policy excludes crypto-related content [V] ([S32](https://support.applixir.com/frequently-asked-questions)) | At least 5,000 daily ad impressions or active users. Approval in 1 to 2 business days. Net-30, $100 minimum payout [V] ([S32](https://support.applixir.com/frequently-asked-questions)) | 1 to 2 d | $4+ CPM (blog), $11.40 average CPM (live footer, vendor) [3P] ([S36](https://www.applixir.com/blog/ecpm-optimization-for-web-games-benchmarks-and-fixes/), [S34](https://support.applixir.com/applixir-integration/integration-for-html5-sites-apps/local-callback-error-codes)) |
| **ayeT-Studios** (added in verification) | Yes. HTML5 SDK `ayetvideosdk.min.js`, `AyetVideoSdk.init(placementId, externalIdentifier, ...)`, desktop and mobile browsers, detects external CMPs, ads.txt required [V] ([S71](https://docs.ayetstudios.com/v/product-docs/rewarded-video/web-integrations/rewarded-video-sdk-for-html5)) | Rewarded video (in-app SDKs "coming soon") | **Yes.** S2S postback with macros such as `{external_identifier}`, `{transaction_id}`, `{currency_amount}`, `{payout_usd}`, `{custom_1}` to `{custom_5}`. Optional `X-Ayetstudios-Security-Hash` header: HMAC-SHA256 keyed with the publisher API key over the alphabetically sorted, re-encoded query. Must answer 200; retries 12 times over one hour; callback IPs published. ayeT itself recommends client callbacks for instant UX, with S2S usually within 60 s [V] ([S72](https://docs.ayetstudios.com/v/product-docs/callbacks-and-testing/callbacks/rewarded-video-callbacks), [S75](https://docs.ayetstudios.com/v/product-docs/callbacks-and-testing/callback-verification/hmac-security-hash-optional), [S76](https://docs.ayetstudios.com/v/product-docs/callbacks-and-testing/callbacks/ip-whitelist)) | Likely. playtoearn.com's ads.txt lists ayeT (PL-22398) as DIRECT [V] ([S70](https://playtoearn.com/ads.txt)). Acceptance of this reward model: UNVERIFIED | Existing account; which ayeT product it covers is UNVERIFIED | 1 to 2 d | Not published |
| **Playwire** (RAMP) | Yes | Yes. Web rewarded is an overlay video triggered by custom events. House-ad backfill is possible [V] ([S38](https://www.playwire.com/blog/web-rewarded-video-ads), [S39](https://www.playwire.com/rewarded-video-ad-units)) | Not publicly documented (UNVERIFIED) | Not stated. Requires not being blacklisted by Google and IVT at or below 7% [V] ([S37](https://www.playwire.com/faq)) | 500,000 monthly pageviews (managed; 100K for self-serve per Nitro's blog [3P]), a strong US/UK/CA footprint, Google Analytics, HTTPS privacy page. Net 60, revenue share [V] ([S37](https://www.playwire.com/faq)) | 2 to 4 d plus onboarding | Claims roughly 4x the CPM of standard video [3P] ([S39](https://www.playwire.com/rewarded-video-ad-units)) |
| **Venatus / AdinPlay** (same group) | Yes | Yes, desktop and mobile rewarded for browser game publishers [V] ([S44](https://adinplay.com/publishers), [S45](https://www.venatus.com/publishers/browser-game-monetization)) | Not documented. Community code shows client callbacks (`aipPlayer`, `AIP_REWARDEDCOMPLETE`) (UNVERIFIED) | Not stated | Venatus: about 1.5M monthly pageviews and 20% Tier-1 traffic, Net-90 [3P] ([S41](https://blog.nitropay.com/nitro-vs-playwire-vs-venatus-which-ad-network-is-right-for-gaming-publishers/)). AdinPlay: contact form, no public minimum. | 2 to 3 d plus onboarding | Not published |
| **Nitro (NitroPay)** | Yes | `createAd` format enum includes `Rewarded` [V] ([S42](https://api-docs.nitropay.com/enums/_options_.formatoptions.html)). Rewarded setups are site-specific and arranged through their success team. | Not documented | Not stated | 100,000 monthly views, Net-7 [3P, own blog] ([S41](https://blog.nitropay.com/nitro-vs-playwire-vs-venatus-which-ad-network-is-right-for-gaming-publishers/), [S43](https://nitropay.com/faq/)) | 2 to 3 d | Not published |
| **GameDistribution** | Yes, but for games **published in the GD network** | `gdsdk.preloadAd('rewarded')`, `gdsdk.showAd('rewarded')`, event `SDK_REWARDED_WATCH_COMPLETE`. The rewarded flag must be enabled per game on the developer portal [V] ([S46](https://github.com/GameDistribution/GD-HTML5/wiki/Rewarded-Ads)) | None | n/a | Games must be listed in their catalog, which is incompatible with exclusive prize games | n/a | n/a |
| **GameMonetize** | Yes, for games **published in the GM network**. Self-hosting only by contacting them [V] ([S47](https://github.com/MonetizeGame/GameMonetize.com-SDK/blob/master/README.md)) | Via SDK | None documented | n/a | Upload the game ZIP and request activation | n/a | n/a |
| CrazyGames / Poki SDKs | Portal SDKs (described as connecting a web game to CrazyGames) ([S52](https://docs.crazygames.com/)) | Yes, on their portals | n/a | n/a | Usable only on those portals (UNVERIFIED for Poki) | n/a | n/a |
| **Adsgram** (Telegram) | **Telegram Mini Apps, bots and channels only** [V] ([S48](https://docs.adsgram.ai/publisher/)) | Reward, interstitial and task formats | Reward URL: HTTPS GET with a `[userId]` macro (the Telegram user id). No signature scheme documented, so put a secret in the URL. Debug mode sends no reward URL call [3P] ([S49](https://github.com/zekogg/MinerX-Realm/pull/33)) | Crypto-native ecosystem | Telegram only | 1 to 2 d | $0.50 to $1.00 per 1,000 (paid in USDT) [V] ([S48](https://docs.adsgram.ai/publisher/)) |
| **Monetag** (Telegram SDK) | TMA SDK. No web use documented [V] ([S51](https://docs.monetag.com/)) | Rewarded interstitial and rewarded popup | GET postback with macros `{ymid}`, `{reward_event_type}` (`valued` / `non_valued`), `{estimated_price}`, `{zone_id}`, `{sub_zone_id}`, `{request_var}`, `{telegram_id}`, `{event_type}`. Retries on non-200. Dedupe on `ymid`. No signature documented [V] ([S50](https://docs.monetag.com/docs/postbacks/configuration/)) | Not stated | Registration plus TMA moderation | 1 to 2 d | Per-event `estimated_price` |

Takeaways:
1. No web network offers an **asymmetric** (T1-grade) signed reward callback. Corrected in verification: ayeT-Studios offers an HMAC-SHA256-signed S2S callback (T2) and is already in playtoearn.com's ads.txt, which makes it the strongest web T2 candidate. AppLixir's MD5 webhook is weaker (it also sends the plaintext secret) and it excludes crypto sites.
2. Networks that resell Google AdX demand will likely bring the same Google rewarded policy question downstream (UNVERIFIED per network).
3. Before any contract, ask each network three written questions:
   * Do you accept crypto / P2E sites?
   * Is there a server-to-server reward callback, and is it signed?
   * What traffic minimum applies?

### 2.6 First-party option: sponsored rewarded video (PlayToEarn Business)

| Aspect | Design |
|---|---|
| Demand | Web3 game studios and projects that already buy PlayToEarn ads ([S64](https://business.playtoearn.com/)). New product "Playground Sponsored Try": a 15 to 30 s trailer, sold per CPM or per completed view. Backfill with PlayToEarn's own promos (Plus, new games). Backfill brings no revenue, but the user still gets the try. |
| Policy | Google rewarded policies do not apply to non-Google inventory, but local law does: ad disclosure ("Sponsored"), crypto marketing rules (for example, UK cryptoasset financial promotions, EU MiCA marketing rules), and geo-blocking where needed. Legal track. |
| Verification | T3: our server issues the creative with a ticket-bound signed media URL, records `shown_at` and grants only if `now - shown_at >= 0.9 x duration`. Optionally cross-check CDN logs. |
| Ad blockers | Opt-in media served first-party is less exposed to third-party blocking lists. Do not obfuscate. It is simply first-party content. |
| Effort | 8 to 15 dev-days: creative CRUD, selection with per-user frequency cap, events, advertiser report, IVT filter. Not in the demo, except as a `house-video` adapter stub. |

### 2.7 eCPM and revenue expectations [EST]

| Channel | eCPM (USD) | Basis |
|---|---|---|
| Native app rewarded, Tier-1 (US, UK, CA, AU, DE, FR, JP) | 15 to 30 | Playwire benchmark article, 2025-09-17 [3P] ([S40](https://www.playwire.com/blog/admob-ecpm-benchmarks-what-publishers-should-expect)). It also advises planning for 70 to 80% of calculator estimates. |
| Native app rewarded, global average | 8 to 18 | same [3P] |
| Web rewarded (HTML5 games) | about 4 to 12 | AppLixir claims ($4+ blog, $11.40 live average) [3P] ([S36](https://www.applixir.com/blog/ecpm-optimization-for-web-games-benchmarks-and-fixes/)) |
| Web display (baseline) | 0.50 to 2 | [3P] ([S36](https://www.applixir.com/blog/ecpm-optimization-for-web-games-benchmarks-and-fixes/)) |
| Telegram (Adsgram) | 0.50 to 1.00 | [V] ([S48](https://docs.adsgram.ai/publisher/)) |
| No consent (limited or non-personalized ads) | Lower, not quantified | Limited ads disable personalization. In AdMob they also disable frequency capping and some measurement [V] ([S56](https://support.google.com/admob/answer/10105530?hl=en)) |

Illustrative web scenario (assumptions, not data):
* 10,000 Playground DAU.
* 60% use up their free tries; 40% of those choose an ad.
* 2 ad requests each, 70% fill: **3,360 impressions per day**.
* At $4 eCPM that is about **$13 per day (about $400 per month)**. At $10 it is about $34 per day (about $1,000 per month). Native Tier-1 at $15 would be about $50 per day.

Value neutrality versus the points option. Let `P` be the USD liability of one P2E Point:
* A paid try burns 10 points and sends 5 of them back into the dynamic bonus. Net liability reduction: **5P**.
* An ad try earns `eCPM_net / 1000`.
* The two are equal when **`P = eCPM_net / 5000`**. At a $5 net eCPM that is $0.001 per point (1,000 points = $1).
* If `P` is higher, points tries are worth more to the platform; if lower, ads are. This is for the economy track.

### 2.8 Consent and privacy

| Region / topic | Requirement | Effect on ads | Source |
|---|---|---|---|
| EEA, UK | Personalized ads from AdSense, Ad Manager or AdMob need a **Google-certified CMP integrated with IAB TCF** (since 2024-01-16) | Traffic from non-certified CMPs may get non-personalized or limited ads, where supported | [V] [S53](https://support.google.com/adsense/answer/13554116?hl=en) |
| Switzerland | Same requirement since 2024-07-31 | same | [V] [S53](https://support.google.com/adsense/answer/13554116?hl=en) |
| TCF version | TCF v2.3 was released 2025-06-19. TC strings created **after 2026-02-28** without the `disclosedVendors` segment are invalid; older strings stay valid. | An outdated CMP produces invalid strings and falls back to limited ads | [V] [S54](https://iabeurope.eu/all-you-need-to-know-about-the-transition-to-tcf-v2-3/) |
| Google's own CMP | Google's CMP (Privacy & messaging, TCF CMP ID 300) is certified for web and app | Fastest path if PlayToEarn has none | [V] [S53](https://support.google.com/adsense/answer/13554116?hl=en) |
| Limited ads | Served when there is no consent for TCF Purpose 1 (or no TC string from a certified CMP). No personal-data personalization. Programmatic demand from Google and third parties is still eligible. | Lower eCPM and fill. Only IVT-detection cookies are allowed without consent. | [V] [S55](https://support.google.com/adsense/answer/14210870?hl=en), [S56](https://support.google.com/admob/answer/10105530?hl=en) |
| US states | Google accepts GPP strings (US National, CA, CO, CT, FL, VA) through `gpp` and `gpp_sid`. GPP National v2 supported from September 2025. An opt-out of sale, sharing or targeted ads triggers **restricted data processing** (non-personalized requests). RDP can also be set per request in GPT or AdSense tags. GPP is optional for Google. | Non-personalized ads for opted-out users | [V] [S57](https://support.google.com/admanager/answer/14117049?hl=en), [S58](https://support.google.com/adsense/answer/9560818?hl=en) |
| Age | H5 tags support `data-tag-for-under-age-of-consent` and `data-tag-for-child-directed-treatment`. AdMob has request configuration flags. | Set them if users under the age of consent can use the Playground | [V] [S24](https://support.google.com/adsense/answer/9955214?hl=en) |
| Our own anti-fraud data | Device hash and IP-prefix hash used for caps | Legal basis to be set by the legal track (TCF has a special purpose for security and fraud prevention). Hash with a server secret and delete after 30 days. | Recommendation |

Load order on the Playground page:
1. The CMP loads first.
2. Wait for TCF `tcloaded` / `useractioncomplete`, or GPP ready, with a 3 s timeout.
3. Inject the Google ad tag.
4. Call `prepareAdTry()`.

If the CMP is not resolved, `AdsProvider.isAvailable()` returns `consent_pending`. The UI then offers the house video or points only.

### 2.9 Ad blockers

| Fact | Source |
|---|---|
| 29.5% of internet users worldwide use ad blockers (GWI, Q2 2025). The 37% desktop vs 15% mobile split in the same article comes from a 2020 US AudienceProject survey, not from GWI; newer US figures there are 27% on desktop vs 22% on other devices. PlayToEarn reports 60% mobile users ([S73](https://playtoearn.com/advertise)), which lowers exposure. | [3P] [S59](https://backlinko.com/ad-blockers-users) (corrected in verification) |
| Brave passed 100M monthly active users (101M, post dated 2025-10-01) and blocks ads and trackers by default. Crypto audiences plausibly over-index (UNVERIFIED). | [V] [S60](https://brave.com/blog/100m-mau/) |

| Detection signal | Maps to |
|---|---|
| Ad script `onerror`, or the global (`adBreak`, `googletag`) missing after the timeout | `blocked` |
| H5 `adBreakDone` with `notReady` and no `beforeReward` | `blocked` (likely) |
| GPT slot is `null` | `not_supported` |
| H5 `noAdPreloaded` / `other`, or GPT `isEmpty` | `no_fill` |

The user experience is the same for all four: show the points option and the line "Ads are not available right now." Never lock the Playground behind an ad-block wall. Record the outcome in analytics to size the problem.

---

## 3. Recommendations

### 3.1 Provider strategy

| Phase | Web primary | Web fallbacks | Native | Verification | Gate |
|---|---|---|---|---|---|
| **Demo (now)** | `MockAdProvider` in `client` mode (T4) | `MockAdProvider` in `ssv` mode (simulated T1 using the real verifier), then points | n/a | T4 and T1 paths both exercised | None. No real ads, so no CMP needed. |
| **Web v1** | `h5-games-ads` (AdSense H5 Games Ads), if approved **and** Google confirms the try reward model in writing. Alternative primary: `gpt-rewarded`, if PlayToEarn is an Ad Manager publisher. | `house-video` (T3), then points | n/a | T4 plus T3 | Certified CMP (TCF v2.3) and GPP live |
| **Web v1.5** | Add AdinPlay / Venatus or Playwire only if they accept crypto content in writing and traffic qualifies. Prefer any network that offers a signed server callback. | same | n/a | T4 or T2 | Written answers to the three questions in 2.5 |
| **Native** | n/a | n/a | `admob-bridge` (AdMob rewarded plus SSV). Later: AdMob mediation with SSV applied to all networks. | T1 | Play and App Store policy review (legal) and UMP consent in the app |
| **Telegram (optional)** | n/a | n/a | Adsgram or Monetag in a Mini App | T2 | Only if PlayToEarn builds a Mini App |

Runtime selection (server-driven order per platform and geo):
1. Is the native bridge present? Use `admob-bridge`.
2. Otherwise, is consent resolved and a Google provider enabled? Call `isAvailable()` on it.
3. If it is unavailable (`no_fill`, `blocked`, `not_supported`, `frequency_capped`, `timeout`), try `house-video`.
4. If nothing is available, show points and the line "No ads right now". Never show an error dead end.

### 3.2 Verification tiers and daily caps

| Tier | Mechanism | Examples | Trust | Default cap per user per day |
|---|---|---|---|---|
| **T1_SIGNED_S2S** | Provider callback with an asymmetric signature bound to our ticket (`custom_data` = ticketId, `user_id` = uid) | AdMob SSV, mock SSV | High | **10** |
| **T2_SECRET_S2S** | Provider callback with a shared secret (MD5 / HMAC / secret in URL), bound to the ticket | ayeT-Studios (HMAC-SHA256 header; how to bind the ticket, for example through a custom parameter, is UNVERIFIED for rewarded video), AppLixir (`userId` = ticketId is signed; `customData` is not), Monetag (`ymid`), Adsgram | Medium | **6** |
| **T3_FIRST_PARTY** | Our own ad server with server-timed playback | House video | Medium | **5** |
| **T4_CLIENT_ONLY** | Browser event plus server timing between `shown` and `result` | H5 `adViewed`, GPT `rewardedSlotGranted`, mock client | Low | **3** |

| Rule | Default | Why |
|---|---|---|
| All ad tries combined per user per day | 10 | Bounds leaderboard advantage and inventory use |
| New account (under 7 days old or email unverified) | 3 in total; T4 at most 1 | Multi-account farming |
| Ad option offered only when free tries in scope = 0 | yes | Matches the product rule. Enforced by the server. |
| Caps are per user globally, not per game | yes | With 50 games, per-game caps would allow 150 or more ads a day |
| Open tickets per user | 1 (issuance is idempotent and returns the open ticket) | Blocks parallel farming |
| Tickets issued per user per day | 20 | Blocks probing |
| Cooldown after a grant | T1 20 s, T2 30 s, T3 and T4 45 s | Human pacing. Close to the H5 frequency floor. |
| Ticket TTL | 10 min | |
| Grace for verified S2S callbacks after TTL | 30 min | Policy requires honoring completed ads. Note from verification: AdMob retries only 5 times at 1 s intervals [V] ([S1](https://developers.google.com/admob/android/ssv)), so for AdMob this grace covers slow delivery, not retries. ayeT retries 12 times over one hour [V] ([S72](https://docs.ayetstudios.com/v/product-docs/callbacks-and-testing/callbacks/rewarded-video-callbacks)), so make the grace per provider and use at least 60 min for ayeT. |
| Minimum server-measured watch time (T4) | 5 s | Matches GPT's 5 s display threshold, so real short creatives are not denied |
| Tab hidden during the ad | Deny T4 if hidden more than 20% of the time | Weak client signal |
| Per IP prefix (/24 IPv4, /64 IPv6) per day | 30 grants | Farms |
| Per device hash per day | 10 grants, at most 3 distinct users | Farms |
| Risk score deny threshold (T4) | 60 or higher | Section 3.5 |
| Ad try validity | Until the next daily reset. Bound to user (and game, if tries are per game). Not bankable, not transferable. | Policy (non-transferable) and economy |

**Worst-case loss per account (T4):** 3 tries a day, or 30 points of value a day. That bounds what a forged client callback can win.

### 3.3 Ticket state machine

```
                      issue (all caps checked HERE)
   (none) ------------------------------------------------> ISSUED
                                                          |  |   |
                        user cancels / new modal         |  |   | TTL passes
          CANCELLED <-----------------------------------+  |   +--------------> EXPIRED
                                                             |                       |
                                     POST /shown             v                       |
                                                           SHOWN                     |
                                                         /   |   \                   |
         client-result: dismissed / no_fill /           /    |    \  T1/T2 verified  |
         blocked / error / timeout                       v     |     v  S2S callback   |
                                             NOT_REWARDED      |    GRANTED <---------+  (within TTL + 30 min grace)
                                                               |       ^
                   T3/T4 client-result 'completed' and checks  |       |
                   pass (timing, owner, not automated) --------+-------+
                                                               |
                   T3/T4 checks fail (too fast, hidden, bot) ->  DENIED
   T1/T2 client-result 'completed': stay SHOWN and wait for the callback
   (after 10 min without a callback: at most 1 fallback grant per day under T4 rules)
   A verified T1/T2 callback may also arrive while the ticket is still ISSUED: grant it the same way.
```

### 3.4 Sequence diagrams

**A. Native app: AdMob rewarded with SSV through the JS bridge (T1)**

```
User      Playground (WebView)      Native shell (GMA SDK)      PlayToEarn API                 Google AdMob
 |  out of tries  |                          |                          |                              |
 |--------------->| POST /api/playground/ads/tickets {provider:"admob-bridge", gameId} ------------->  |
 |                |<------------------------------------------- 201 {ticketId, providerParams:{userId,customData:ticketId}}
 | tap "Watch ad" |                          |                          |                              |
 |--------------->| POST /tickets/{id}/shown ------------------------->| shown_at = now               |
 |                | bridge.showRewarded(ticket) ->|                     |                              |
 |                |                          | RewardedAd.load(adUnit) ------------------------------->|
 |                |                          | setServerSideVerificationOptions(userId, customData)     |
 |                |                          | show() ------------------------------------------------->|
 |   watches ad   |                          |<------------------------------ onUserEarnedReward ------|
 |                |<- event "earned" --------|                          |                              |
 |                | POST /tickets/{id}/client-result {completed} ------>| T1: state stays SHOWN        |
 |                |                          |                          |<-- GET /api/ads/callbacks/admob?ad_network=..&..&signature=..&key_id=..
 |                |                          |                          | verify ECDSA P-256 / SHA-256 (cached keys, <= 24 h)
 |                |                          |                          | ad_unit, reward_item=try, amount=1 ok
 |                |                          |                          | custom_data -> ticket, user_id matches
 |                |                          |                          | INSERT transaction_id (unique) -> GRANTED, +1 ad try
 |                |                          |                          |-- 200 OK -------------------->|
 |                | GET /tickets/{id} (poll every 1 s, up to 20 s) ---->|                              |
 |                |<------------------------------------------ {state:"granted", triesAvailable}        |
```

**B. Web: H5 Games Ads, client callback only (T4)**

```
User      Playground page               Ad Placement API (adsbygoogle)        PlayToEarn API
 | out of tries |                                |                                 |
 |------------->| adBreak({type:'reward', ...}) ->|                                 |
 |              |<- beforeReward(showAdFn) -------| (only if an ad is available)    |
 |              | POST /ads/tickets {provider:"h5-games-ads", gameId} ------------->| caps OK -> 201 ticket
 |              | enable "Watch ad (1 try)"      |                                 |
 | click        |                                |                                 |
 |------------->| POST /tickets/{id}/shown (<= 800 ms, then continue) ------------->| shown_at = now
 |              | showAdFn() ------------------->| beforeAd -> pause + mute        |
 |   watches    |<- adViewed --------------------|                                 |
 |              |<- afterAd; adBreakDone({breakStatus:'viewed'})                   |
 |              | POST /tickets/{id}/client-result {completed, elapsedMs, hiddenMs} ->| T4 checks: owner, state, TTL,
 |              |                                |                                 | now - shown_at >= 5 s, no automation,
 |              |                                |                                 | hidden <= 20%, risk < 60 -> GRANTED
 |              |<------------------------------------------------------- {state:"granted"}
```

**C. Shared-secret server callback (T2), for example an AppLixir-style webhook**

```
Playground        Provider SDK            Provider server                 PlayToEarn API
 | POST /ads/tickets --------------------------------------------------------> 201 ticket
 | init(apiKey, userId = ticketId) ->|                                            |
 | play() ------------------------->| ad completes ->|                            |
 |<- status "complete" (UI only) ---|                | GET /api/ads/callbacks/applixir?gameApiKey&gameId&userId&tid&signature
 |                                  |                |                            | md5(gameApiKey+gameId+userId+tid+secret), constant-time compare
 |                                  |                |                            | userId -> ticket; dedupe tid -> GRANTED
 |                                  |                |<------------------ 200 ----|
 | poll GET /tickets/{id} ------------------------------------------------------> {state:"granted"}
```

**D. First-party sponsored video (T3)**

```
Playground                                   PlayToEarn API / ad server
 | POST /ads/tickets {provider:"house-video"} -> 201 {ticket, creative:{signedVideoUrl, durationMs, advertiser, clickUrl}}
 | click -> POST /tickets/{id}/shown -------> shown_at = now
 | <video> plays (muted autoplay not needed: user gesture)
 | ended -> POST /tickets/{id}/client-result {completed} -> now - shown_at >= 0.9 x durationMs, IVT checks
 |                                              -> GRANTED; completed view billed to the advertiser only if IVT-clean
```

**E. Demo mock: both paths**

```
client mode (T4):  click -> MockAdProvider overlay (countdown) -> 'completed' -> POST client-result -> T4 checks -> GRANTED
ssv mode (T1):     click -> overlay -> countdown ends -> POST /api/dev/mock-network/complete {ticketId}
                   -> server MockSsvNetwork signs an AdMob-format query with a local P-256 key
                   -> GET /api/ads/callbacks/mock-ssv?...&signature&key_id  (same verifyAdMobSsv code as production)
                   -> GRANTED; client polls GET /tickets/{id}
outcomes forced by dev panel or URL: ?mockAd=complete|skip|fail|no_fill|blocked|random
```

### 3.5 Server rules

**Issuance** (`POST /api/playground/ads/tickets`). Check in this order and refuse early:
1. Account not restricted.
2. If there is already an open ticket for the same user, scope and provider, return it (it passed these checks when issued).
3. Free tries in scope = 0.
4. Daily caps: per tier, all tiers, new account, tickets issued.
5. IP-prefix and device caps.
6. Cooldown.
7. The provider is enabled for this platform and geo.

**Completion:**

| Source | Rule |
|---|---|
| T1/T2 verified callback | Always grant if the ticket exists, belongs to `user_id`, and the reward config matches (`ad_unit`, `reward_item`, `reward_amount`). Grant from states `issued`, `shown` or `expired` (within TTL + 30 min grace), **even if caps changed since issuance**. |
| T1/T2 duplicate | `duplicate`, answer 200 |
| T1/T2 unknown ticket | `orphan`, answer 200, alert. Likely another environment's callback URL or a replay. |
| T1/T2 user mismatch | `rejected`, answer 200, security alert |
| T1/T2 bad signature or secret | Answer 403. Log only (not from the provider). |
| T3/T4 client result | Grant only if the state is `issued` or `shown`, `shown_at` is set, `now - shown_at >= minWatch`, the client `elapsedMs >= minWatch`, not automated and not hidden more than 20%. Otherwise `denied` with a reason. |
| T1/T2 client result `completed` | Keep `shown` and wait for the callback. If none arrives within 10 min, allow at most 1 fallback grant per day under T4 rules. This protects users from provider outages without reopening fraud. |

**Idempotency:**
* Unique index on `(provider, provider_txn_id)`.
* Unique grant per `ticket_id`.
* The grant and the try-credit insert happen in one DB transaction.

**Anomaly signals** (they feed `risk_score`, 0 to 100, stored per user with a 7-day decay):

| Signal | Threshold [EST, tune in production] | Score |
|---|---|---|
| Robotic timing: standard deviation of server-measured watch time over the last 20 T4 grants | < 300 ms | +40 |
| Perfect completion: T4 completion ratio over 30 or more shown tickets in 7 days (a healthy ratio is about 80 to 90% per AppLixir [3P] [S32](https://support.applixir.com/frequently-asked-questions)) | >= 98% | +20 |
| Velocity | > 4 ad grants in a rolling hour | +20 and 1 h cooldown |
| Shared device or IP | >= 4 users with grants on one device hash in 24 h | +30 for all of them |
| `navigator.webdriver` true | any | deny T4 |
| Top-100 run from an ad try | any | Route to the replay validation of the game-runtime track (no score change) |

**Reconciliation and kill switches:**
* Every day, compare T4 grants per provider with the impressions in the provider's reports (AdSense / Ad Manager reporting).
* If grants exceed 110% of impressions, the system automatically sets the T4 cap to 1 and alerts.
* Feature flags per provider, per geo and globally (`ads.enabled=false` means points only).

### 3.6 HTTP contract and data model (language-agnostic)

| Method | Path | Auth | Body / query | Responses |
|---|---|---|---|---|
| POST | `/api/playground/ads/tickets` | Session + CSRF | `{provider, gameId or null}` | `201 AdTicket`; `200 AdTicket` (open ticket reused); `409 {reason:"free_tries_remaining"}`; `429 {reason:"daily_cap"or"cooldown"or"ip_cap", retryAfterSec}`; `403 {reason:"account_restricted"}`; `503 {reason:"provider_disabled"}` |
| POST | `/api/playground/ads/tickets/{id}/shown` | Session + CSRF | none | `204`; `404`; `409` |
| POST | `/api/playground/ads/tickets/{id}/client-result` | Session + CSRF | `{outcome: AdOutcome, signals:{hiddenMs, webdriver}}` | `200 {state}` |
| GET | `/api/playground/ads/tickets/{id}` | Session | none | `200 {state, triesAvailable}` |
| GET | `/api/ads/callbacks/admob` | None (signature) | raw SSV query | `200` authentic (grant, duplicate, orphan, rejected); `400` malformed; `403` bad signature; `5xx` only transient |
| GET | `/api/ads/callbacks/{provider}` | None (secret or MD5) | provider query | same pattern |
| POST | `/api/dev/mock-network/complete` | Dev builds only (the route is absent in production) | `{ticketId}` | `204` |

Callback routes are excluded from CSRF and session middleware. Rate-limit them per IP. Store the raw query for 90 days for audits, after redacting secrets: AppLixir currently sends its plaintext `secretKey` in every callback [V] ([S33](https://support.applixir.com/applixir-integration/integration-for-html5-sites-apps/step-4-setting-up-local-callback-360053188774)). For ayeT, also store the `X-Ayetstudios-Security-Hash` header value, because the signature is not in the query.

```sql
CREATE TABLE ad_reward_tickets (
  id                CHAR(26)     PRIMARY KEY,          -- ULID (sortable, opaque)
  user_id           BIGINT       NOT NULL,
  provider          VARCHAR(32)  NOT NULL,
  tier              VARCHAR(16)  NOT NULL,            -- T1_SIGNED_S2S | T2_SECRET_S2S | T3_FIRST_PARTY | T4_CLIENT_ONLY
  game_id           VARCHAR(64)  NULL,                -- set when tries are per game
  reward_scope      VARCHAR(8)   NOT NULL,            -- game | global
  state             VARCHAR(16)  NOT NULL,            -- issued|shown|granted|denied|not_rewarded|expired|cancelled
  day_key           DATE         NOT NULL,            -- reset day the caps count against
  issued_at         TIMESTAMP    NOT NULL,
  shown_at          TIMESTAMP    NULL,
  resolved_at       TIMESTAMP    NULL,
  expires_at        TIMESTAMP    NOT NULL,
  min_watch_ms      INT          NOT NULL,
  ip_prefix_hash    CHAR(64)     NOT NULL,            -- HMAC(secret, ip /24 or /64)
  device_hash       CHAR(64)     NULL,
  client_outcome    VARCHAR(24)  NULL,
  client_elapsed_ms INT          NULL,
  provider_txn_id   VARCHAR(128) NULL,
  deny_reason       VARCHAR(48)  NULL,
  risk_score        SMALLINT     NOT NULL DEFAULT 0,
  CONSTRAINT uq_provider_txn UNIQUE (provider, provider_txn_id)
);
CREATE INDEX ix_ads_user_day ON ad_reward_tickets (user_id, day_key, state);

CREATE TABLE ad_provider_callbacks (
  id              BIGINT       PRIMARY KEY,           -- auto increment
  provider        VARCHAR(32)  NOT NULL,
  received_at     TIMESTAMP    NOT NULL,
  remote_ip       VARCHAR(45)  NOT NULL,
  raw_query       TEXT         NOT NULL,
  signature_ok    BOOLEAN      NOT NULL,
  ticket_id       CHAR(26)     NULL,
  provider_txn_id VARCHAR(128) NULL,
  result          VARCHAR(16)  NOT NULL               -- granted|duplicate|orphan|rejected|too_late|bad_signature
);
```

The granted try goes to the Playground tries ledger (economy / integration tracks) as `source='ad'`, `source_ref=ticket_id`, `provider`, `expires_at = next reset`, `prize_eligible = config.adTriesPrizeEligible`. It never counts as a "paid play" for the dynamic bonus.

### 3.7 TypeScript: shared types and the `AdsProvider` interface

```ts
// playground-ads/types.ts
// Framework-free contract for "watch an ad to get a try". No SDK imports here.

export type AdProviderId =
  | 'mock'
  | 'h5-games-ads'   // Google AdSense H5 Games Ads (Ad Placement API), web
  | 'gpt-rewarded'   // Google Ad Manager GPT rewarded slot, web
  | 'house-video'    // first-party sponsored video (PlayToEarn Business)
  | 'admob-bridge'   // native app shell: AdMob rewarded + SSV via JS bridge
  | 'applixir' | 'adinplay' | 'playwire' | 'adsgram' | 'monetag';

/** How the SERVER learns that the ad was completed. Drives daily caps. */
export type VerificationTier =
  | 'T1_SIGNED_S2S'   // provider callback with asymmetric signature (AdMob SSV)
  | 'T2_SECRET_S2S'   // provider callback with shared secret / MD5 / HMAC
  | 'T3_FIRST_PARTY'  // our own ad server times playback server-side
  | 'T4_CLIENT_ONLY'; // browser event only (H5 adViewed, GPT rewardedSlotGranted)

export type RewardScope = 'game' | 'global';

export interface AdTicket {
  ticketId: string;                 // ULID / UUIDv7, single use
  provider: AdProviderId;
  tier: VerificationTier;
  placement: 'playground_extra_try';
  gameId: string | null;            // set when tries are per game
  reward: { kind: 'try'; amount: 1; scope: RewardScope };
  issuedAt: string;                 // ISO 8601, server clock
  expiresAt: string;                // issuedAt + 10 min
  minWatchMs: number;               // enforced server-side for T3/T4
  /** Values the adapter must hand to the provider SDK, e.g. SSV userId/customData. */
  providerParams: Readonly<Record<string, string>>;
  /** Ad tries left today after this one is granted (for disclosure copy). */
  remainingToday: number;
}

export type AdOutcome =
  | { status: 'completed'; providerEvent: string; elapsedMs: number; providerTxnId?: string }
  | { status: 'dismissed'; elapsedMs: number }   // closed early: no reward
  | { status: 'no_fill' }
  | { status: 'blocked' }                        // ad blocker or SDK script failed
  | { status: 'not_supported' }                  // e.g. GPT slot null, no native bridge
  | { status: 'frequency_capped' }               // provider-side cap
  | { status: 'timeout' }
  | { status: 'error'; code: string; message?: string };

export type UnavailableReason =
  | 'no_fill' | 'blocked' | 'not_supported' | 'frequency_capped'
  | 'timeout' | 'error' | 'consent_pending';

export type AdAvailability =
  | { available: true }
  | { available: false; reason: UnavailableReason };

export interface AdShowHooks {
  onBeforeAd?: () => void;  // pause + mute page and game audio
  onAfterAd?: () => void;   // resume audio (any outcome)
}

export interface ConsentSnapshot {
  tcString: string | null;      // IAB TCF v2.3 (EEA/UK/CH), else null
  gppString: string | null;     // IAB GPP (US states), else null
  gppSectionIds: number[];
  pending: boolean;             // CMP not resolved yet: do not request ads
  underAgeOfConsent: boolean;
}

export interface AdsInitContext {
  consent: ConsentSnapshot;
  testMode: boolean;
  locale: string;
  mount?: HTMLElement;          // container for providers that draw their own overlay
}

export interface AdsProvider {
  readonly id: AdProviderId;
  readonly tier: VerificationTier;
  init(ctx: AdsInitContext): Promise<void>;
  /** Prepare an ad for the next show(). Must not render anything visible. */
  isAvailable(): Promise<AdAvailability>;
  /**
   * Render the ad. Call ONLY from the click handler of the "Watch ad" button.
   * Resolves when the ad UI has closed. 'completed' is a hint; the server decides.
   */
  show(ticket: AdTicket, hooks?: AdShowHooks, signal?: AbortSignal): Promise<AdOutcome>;
  dispose(): void;
}

export type TicketState =
  | 'issued' | 'shown' | 'granted' | 'denied' | 'not_rewarded' | 'expired' | 'cancelled';

export type IssueRefusal =
  | 'free_tries_remaining' | 'daily_cap' | 'cooldown' | 'ip_cap'
  | 'provider_disabled' | 'account_restricted';

export interface ClientSignals {
  hiddenMs: number;            // time the tab was hidden while the ad was up
  webdriver: boolean;          // navigator.webdriver (weak signal)
}

/** Thin HTTP client; paths are in the HTTP contract table. */
export interface PlaygroundAdsApi {
  issueTicket(req: { provider: AdProviderId; gameId: string | null }): Promise<
    | { ok: true; ticket: AdTicket }
    | { ok: false; reason: IssueRefusal; retryAfterSec?: number }
  >;
  markShown(ticketId: string): Promise<void>;
  reportClientResult(ticketId: string, outcome: AdOutcome, signals: ClientSignals): Promise<{ state: TicketState }>;
  getTicket(ticketId: string): Promise<{ state: TicketState; triesAvailable?: number }>;
}
```

Orchestration. Network work happens when the dialog opens, not on click, because browsers keep user activation only briefly and ad SDKs need it to start video with sound:

```ts
// playground-ads/flow.ts
import type {
  AdOutcome, AdShowHooks, AdTicket, AdsProvider, ClientSignals,
  IssueRefusal, PlaygroundAdsApi, UnavailableReason,
} from './types';

export type PrepareResult =
  | { kind: 'ready'; provider: AdsProvider; ticket: AdTicket }
  | { kind: 'refused'; reason: IssueRefusal; retryAfterSec?: number }
  | { kind: 'unavailable'; reason: UnavailableReason };

export type WatchResult =
  | { kind: 'granted'; triesAvailable?: number }
  | { kind: 'pending' }                         // S2S callback not in yet
  | { kind: 'not_rewarded'; outcome: AdOutcome }
  | { kind: 'denied' };

const sleep = (ms: number) => new Promise<void>((r) => setTimeout(r, ms));

/** Network + SDK work happens here, NOT in the click handler (keeps user activation). */
export async function prepareAdTry(
  api: PlaygroundAdsApi,
  providers: readonly AdsProvider[],   // priority order from server config
  gameId: string | null,
): Promise<PrepareResult> {
  let lastReason: UnavailableReason = 'no_fill';
  for (const provider of providers) {
    const availability = await provider.isAvailable();
    if (!availability.available) { lastReason = availability.reason; continue; }
    const issued = await api.issueTicket({ provider: provider.id, gameId });
    if (issued.ok) return { kind: 'ready', provider, ticket: issued.ticket };
    if (issued.reason === 'provider_disabled') continue;   // try next provider
    return { kind: 'refused', reason: issued.reason, retryAfterSec: issued.retryAfterSec };
  }
  return { kind: 'unavailable', reason: lastReason };
}

/** Call synchronously from the "Watch ad" click handler. */
export async function watchAdTry(
  api: PlaygroundAdsApi,
  prepared: { provider: AdsProvider; ticket: AdTicket },
  hooks: AdShowHooks,
  poll = { everyMs: 1_000, maxMs: 20_000 },
): Promise<WatchResult> {
  const { provider, ticket } = prepared;
  const visibility = trackVisibility();
  // Record shown_at on the server, but never block the SDK for long.
  await Promise.race([api.markShown(ticket.ticketId).catch(() => undefined), sleep(800)]);
  const outcome = await provider.show(ticket, hooks);
  const { state } = await api.reportClientResult(ticket.ticketId, outcome, visibility.stop());
  if (state === 'granted') return { kind: 'granted' };
  if (state === 'denied') return { kind: 'denied' };
  if (outcome.status !== 'completed') return { kind: 'not_rewarded', outcome };
  // T1/T2 providers: the reward arrives via the provider's server callback.
  const deadline = Date.now() + poll.maxMs;
  while (Date.now() < deadline) {
    await sleep(poll.everyMs);
    const t = await api.getTicket(ticket.ticketId);
    if (t.state === 'granted') return { kind: 'granted', triesAvailable: t.triesAvailable };
    if (t.state === 'denied') return { kind: 'denied' };
  }
  return { kind: 'pending' };   // UI: "Your try will appear shortly"
}

function trackVisibility(): { stop(): ClientSignals } {
  let hiddenMs = 0;
  let hiddenSince: number | null = document.hidden ? performance.now() : null;
  const onChange = () => {
    if (document.hidden) hiddenSince = performance.now();
    else if (hiddenSince !== null) { hiddenMs += performance.now() - hiddenSince; hiddenSince = null; }
  };
  document.addEventListener('visibilitychange', onChange);
  return {
    stop() {
      document.removeEventListener('visibilitychange', onChange);
      if (hiddenSince !== null) hiddenMs += performance.now() - hiddenSince;
      return { hiddenMs: Math.round(hiddenMs), webdriver: navigator.webdriver === true };
    },
  };
}
```

### 3.8 Demo: `MockAdProvider` and the mock signed network

What the mock does:
* It draws a full-screen overlay labeled "AD (simulated)", in PlayToEarn blue `#0019FF`, with a countdown and the reward line.
* The close button unlocks after 5 s. Closing asks for confirmation and forfeits the try.
* Clicking the "ad" never grants anything.
* Outcomes can be forced by a dev panel or URL: `complete`, `skip`, `fail`, `no_fill`, `blocked`, `random` (weighted) or `user`.
* There is no visual asset to generate: the overlay is pure DOM and CSS.

```ts
// playground-ads/providers/mock.ts
import type {
  AdAvailability, AdOutcome, AdShowHooks, AdTicket, AdsInitContext, AdsProvider, VerificationTier,
} from '../types';

export type MockOutcome = 'complete' | 'skip' | 'fail' | 'no_fill' | 'blocked';

export interface MockAdOptions {
  durationMs: number;                 // simulated ad length
  skippableAfterMs: number;           // close button unlocks (closing forfeits the try)
  outcome: MockOutcome | 'random' | 'user';   // 'user' = tester decides by clicking
  weights: Record<MockOutcome, number>;       // used when outcome = 'random'
  mode: 'client' | 'ssv';             // T4 path, or simulated signed server callback (T1)
  /** mode 'ssv': ask the dev-only mock network to fire a signed callback for this ticket. */
  requestServerCallback?: (ticket: AdTicket) => Promise<void>;
  random?: () => number;              // inject a seeded RNG in tests
}

const DEFAULTS: MockAdOptions = {
  durationMs: 15_000,
  skippableAfterMs: 5_000,
  outcome: 'user',
  weights: { complete: 70, skip: 10, fail: 5, no_fill: 10, blocked: 5 },
  mode: 'client',
};

const delay = (ms: number) => new Promise<void>((r) => setTimeout(r, ms));

export class MockAdProvider implements AdsProvider {
  readonly id = 'mock' as const;
  readonly tier: VerificationTier;
  private readonly opts: MockAdOptions;
  private mount: HTMLElement = document.body;
  private planned: MockOutcome | 'user' = 'user';

  constructor(options: Partial<MockAdOptions> = {}) {
    this.opts = { ...DEFAULTS, ...options, weights: { ...DEFAULTS.weights, ...options.weights } };
    this.tier = this.opts.mode === 'ssv' ? 'T1_SIGNED_S2S' : 'T4_CLIENT_ONLY';
  }

  async init(ctx: AdsInitContext): Promise<void> {
    if (ctx.mount) this.mount = ctx.mount;
  }

  async isAvailable(): Promise<AdAvailability> {
    await delay(300);                          // pretend to request an ad
    this.planned = this.pick();
    if (this.planned === 'no_fill') return { available: false, reason: 'no_fill' };
    if (this.planned === 'blocked') return { available: false, reason: 'blocked' };
    return { available: true };
  }

  show(ticket: AdTicket, hooks: AdShowHooks = {}, signal?: AbortSignal): Promise<AdOutcome> {
    return new Promise<AdOutcome>((resolve) => {
      const started = performance.now();
      const elapsed = () => Math.round(performance.now() - started);
      hooks.onBeforeAd?.();
      const ui = buildOverlay(this.mount, ticket);
      let done = false;
      const finish = (outcome: AdOutcome) => {
        if (done) return;
        done = true;
        window.clearInterval(tick);
        ui.root.remove();
        hooks.onAfterAd?.();
        resolve(outcome);
      };
      signal?.addEventListener('abort', () => finish({ status: 'error', code: 'aborted' }), { once: true });

      if (this.planned === 'fail') window.setTimeout(() => finish({ status: 'error', code: 'mock_playback_failed' }), 1_500);
      if (this.planned === 'skip') window.setTimeout(() => finish({ status: 'dismissed', elapsedMs: elapsed() }), this.opts.skippableAfterMs + 500);

      ui.close.addEventListener('click', () => {
        if (elapsed() < this.opts.skippableAfterMs) return;
        if (window.confirm('Close the ad? You will not get the extra try.')) finish({ status: 'dismissed', elapsedMs: elapsed() });
      });
      // Clicking the "ad" itself never grants anything (mirrors the no-reward-for-clicks rule).
      ui.card.addEventListener('click', (e) => { if (e.target === ui.card) ui.note('Clicks do not affect the reward.'); });

      const tick = window.setInterval(() => {
        const left = Math.max(0, this.opts.durationMs - elapsed());
        ui.setCountdown(Math.ceil(left / 1000), elapsed() >= this.opts.skippableAfterMs);
        if (left > 0) return;
        window.clearInterval(tick);
        const completed: AdOutcome = {
          status: 'completed', providerEvent: 'mock_complete', elapsedMs: elapsed(),
          providerTxnId: `mock_${ticket.ticketId}`,
        };
        if (this.opts.mode === 'ssv' && this.opts.requestServerCallback) {
          this.opts.requestServerCallback(ticket).then(() => finish(completed), () => finish(completed));
        } else {
          finish(completed);
        }
      }, 250);
    });
  }

  dispose(): void { /* no globals to clean up */ }

  private pick(): MockOutcome | 'user' {
    if (this.opts.outcome !== 'random') return this.opts.outcome;
    const entries = Object.entries(this.opts.weights) as [MockOutcome, number][];
    const total = entries.reduce((sum, [, w]) => sum + w, 0);
    const r = (this.opts.random ?? Math.random)();
    let acc = 0;
    for (const [k, w] of entries) { acc += w / total; if (r < acc) return k; }
    return 'complete';
  }
}

function buildOverlay(mount: HTMLElement, ticket: AdTicket) {
  const el = <K extends keyof HTMLElementTagNameMap>(tag: K, css: string, text = '') => {
    const n = document.createElement(tag);
    n.style.cssText = css;
    if (text) n.textContent = text;
    return n;
  };
  const root = el('div', 'position:fixed;inset:0;z-index:2147483000;display:grid;place-items:center;background:rgba(4,6,28,.92);color:#fff;font:600 16px/1.4 system-ui,sans-serif');
  root.setAttribute('role', 'dialog');
  root.setAttribute('aria-modal', 'true');
  root.setAttribute('aria-label', 'Simulated advertisement');
  root.tabIndex = -1;
  const card = el('div', 'position:relative;width:min(92vw,560px);aspect-ratio:16/9;border-radius:16px;background:#0019FF;display:grid;place-items:center');
  const badge = el('div', 'position:absolute;top:10px;left:12px;font-size:12px;opacity:.85', 'AD (simulated)');
  const countdown = el('div', 'font-size:20px');
  countdown.setAttribute('aria-live', 'polite');
  const reward = el('div', 'position:absolute;bottom:10px;left:12px;font-size:12px;opacity:.85',
    `Reward: 1 try${ticket.gameId ? ' for this game' : ''}`);
  const hint = el('div', 'position:absolute;bottom:10px;right:12px;font-size:12px;opacity:.85');
  const close = el('button', 'position:absolute;top:8px;right:8px;padding:6px 10px;border:0;border-radius:8px', 'Close');
  close.type = 'button';
  close.disabled = true;
  card.append(badge, countdown, reward, hint, close);
  root.append(card);
  mount.append(root);
  root.focus();
  return {
    root, card, close,
    setCountdown(seconds: number, closable: boolean) {
      countdown.textContent = `Ad ends in ${seconds}s`;
      close.disabled = !closable;
      close.textContent = closable ? 'Close (no try)' : 'Close';
    },
    note(text: string) { hint.textContent = text; },
  };
}
```

Server side: the production AdMob SSV verifier plus a dev-only mock network that signs callbacks the same way. The demo therefore tests the real verification code.

```ts
// server/ads/admob-ssv.ts  (Node 18+; same algorithm in any language)
import { createPublicKey, verify, type KeyObject } from 'node:crypto';

export const ADMOB_KEYS_URL = 'https://www.gstatic.com/admob/reward/verifier-keys.json';

export interface SsvKeySource { getKey(keyId: string): Promise<KeyObject | undefined> }

/** Caches keys for at most maxAgeMs (Google: do not cache longer than 24 h). */
export class HttpSsvKeySource implements SsvKeySource {
  private keys = new Map<string, KeyObject>();
  private fetchedAt = 0;
  private lastForcedRefresh = 0;
  constructor(private readonly url = ADMOB_KEYS_URL, private readonly maxAgeMs = 12 * 3_600_000) {}

  async getKey(keyId: string): Promise<KeyObject | undefined> {
    const stale = Date.now() - this.fetchedAt > this.maxAgeMs;
    const unknown = !this.keys.has(keyId) && Date.now() - this.lastForcedRefresh > 60_000;
    if (stale || unknown) {
      if (unknown) this.lastForcedRefresh = Date.now();   // rate-limit refetch on unknown key_id
      await this.refresh();
    }
    return this.keys.get(keyId);
  }

  private async refresh(): Promise<void> {
    const res = await fetch(this.url);
    if (!res.ok) throw new Error(`ssv key server HTTP ${res.status}`);
    const body = (await res.json()) as { keys: { keyId: number; pem: string }[] };
    this.keys = new Map(body.keys.map((k) => [String(k.keyId), createPublicKey(k.pem)]));
    this.fetchedAt = Date.now();
  }
}

export class SsvError extends Error {
  constructor(readonly code: 'missing_signature' | 'unknown_key' | 'bad_signature') { super(code); }
}

export interface SsvCallback {
  adNetwork: string; adUnit: string;
  customData: string | null;   // our ticketId
  userId: string | null;       // our user id
  rewardAmount: number; rewardItem: string;
  timestampMs: number;
  transactionId: string;       // dedupe key
  keyId: string;
}

/**
 * rawQuery MUST be the byte-exact query string as received (no re-encoding, no sorting).
 * Express: req.originalUrl.split('?')[1]. Laravel/Symfony: $_SERVER['QUERY_STRING'],
 * NOT $request->getQueryString() (that one is normalized).
 */
export async function verifyAdMobSsv(rawQuery: string, keys: SsvKeySource): Promise<SsvCallback> {
  const q = rawQuery.startsWith('?') ? rawQuery.slice(1) : rawQuery;
  const cut = q.indexOf('&signature=');
  if (cut < 0) throw new SsvError('missing_signature');
  const message = q.slice(0, cut);                        // signed content: everything before &signature=
  const tail = new URLSearchParams(q.slice(cut + 1));     // signature=...&key_id=...
  const signature = tail.get('signature');
  const keyId = tail.get('key_id');
  if (!signature || !keyId) throw new SsvError('missing_signature');
  const key = await keys.getKey(keyId);
  if (!key) throw new SsvError('unknown_key');
  const ok = verify('sha256', Buffer.from(message, 'utf8'), key, Buffer.from(signature, 'base64url'));
  if (!ok) throw new SsvError('bad_signature');

  const p = new URLSearchParams(message);                 // decodes percent-escaped custom_data
  const ts = Number(p.get('timestamp') ?? 0);
  return {
    adNetwork: p.get('ad_network') ?? '', adUnit: p.get('ad_unit') ?? '',
    customData: p.get('custom_data'), userId: p.get('user_id'),
    rewardAmount: Number(p.get('reward_amount') ?? 0), rewardItem: p.get('reward_item') ?? '',
    // Docs say epoch ms, but the documented sample has 16 digits (microseconds): accept both.
    timestampMs: ts > 1e14 ? Math.floor(ts / 1_000) : ts,
    transactionId: p.get('transaction_id') ?? '', keyId,
  };
}
```

```ts
// server/ads/mock-network.ts  (DEV ONLY: the route must not exist in production builds)
import { createPublicKey, generateKeyPairSync, sign, type KeyObject } from 'node:crypto';
import type { SsvKeySource } from './admob-ssv';

export class MockSsvNetwork implements SsvKeySource {
  readonly keyId = '1000000001';
  private readonly privateKey: KeyObject;
  private readonly publicKey: KeyObject;

  constructor() {
    const pair = generateKeyPairSync('ec', { namedCurve: 'prime256v1' });
    this.privateKey = pair.privateKey;
    this.publicKey = pair.publicKey;
  }

  /** Same JSON shape as the Google key server, for a /dev/mock-ssv-keys.json route. */
  keysJson() {
    const pem = this.publicKey.export({ type: 'spki', format: 'pem' }).toString();
    return { keys: [{ keyId: Number(this.keyId), pem }] };
  }

  async getKey(keyId: string): Promise<KeyObject | undefined> {
    return keyId === this.keyId ? createPublicKey(this.publicKey.export({ type: 'spki', format: 'pem' })) : undefined;
  }

  /** The query string the "network" would call our callback URL with. */
  buildCallbackQuery(a: { ticketId: string; userId: string; adUnit: string; rewardItem?: string; rewardAmount?: number }): string {
    const params: [string, string][] = [
      ['ad_network', '0000000000000000000'],
      ['ad_unit', a.adUnit],
      ['custom_data', a.ticketId],
      ['reward_amount', String(a.rewardAmount ?? 1)],
      ['reward_item', a.rewardItem ?? 'try'],
      ['timestamp', String(Date.now())],
      ['transaction_id', randomHex(16)],
      ['user_id', a.userId],
    ];                                                           // alphabetical, like AdMob
    const content = params.map(([k, v]) => `${k}=${encodeURIComponent(v)}`).join('&');
    const sig = sign('sha256', Buffer.from(content, 'utf8'), this.privateKey).toString('base64url');
    return `${content}&signature=${sig}&key_id=${this.keyId}`;
  }
}

function randomHex(bytes: number): string {
  return Array.from(globalThis.crypto.getRandomValues(new Uint8Array(bytes)), (b) => b.toString(16).padStart(2, '0')).join('');
}
```

PHP equivalent, for the case where the real backend is PHP (the saved page hints at Laravel, UNVERIFIED). **Untested here: no PHP runtime on this machine.**

```php
// $raw = $_SERVER['QUERY_STRING'];   // NOT $request->getQueryString() (normalized)
function verify_admob_ssv(string $raw, array $pemByKeyId): array {
    $cut = strpos($raw, '&signature=');
    if ($cut === false) throw new RuntimeException('missing_signature');
    $message = substr($raw, 0, $cut);
    parse_str(substr($raw, $cut + 1), $tail);                      // signature, key_id
    $pem = $pemByKeyId[$tail['key_id'] ?? ''] ?? null;
    if ($pem === null) throw new RuntimeException('unknown_key');
    $der = base64_decode(strtr($tail['signature'] ?? '', '-_', '+/'));   // base64url -> DER
    if (openssl_verify($message, $der, $pem, OPENSSL_ALGO_SHA256) !== 1) throw new RuntimeException('bad_signature');
    parse_str($message, $params);                                   // decodes custom_data
    return $params;
}
```

The pure rules are ported 1:1 to any language. The reference TS below was self-tested: cap order, cooldown, T4 too-fast denial, T1 waiting for the callback, grant within grace after TTL, duplicates, user mismatch. (Verification note: the excerpt contains only defaults and signatures, so this self-test cannot be reproduced from the report: UNVERIFIED. The build must ship the full implementation with these tests.)

```ts
// server/ads/ticket-rules.ts (excerpt: signatures and defaults)
export const DEFAULT_RULES = {
  perUserPerDay: { T1_SIGNED_S2S: 10, T2_SECRET_S2S: 6, T3_FIRST_PARTY: 5, T4_CLIENT_ONLY: 3 },
  perUserPerDayAll: 10,
  newAccount: { maxAgeDays: 7, perUserPerDayAll: 3, t4PerDay: 1 },
  perIpPrefixPerDay: 30, perDevicePerDay: 10, maxUsersPerDevicePerDay: 3,
  cooldownMs: { T1_SIGNED_S2S: 20_000, T2_SECRET_S2S: 30_000, T3_FIRST_PARTY: 45_000, T4_CLIENT_ONLY: 45_000 },
  ticketTtlMs: 600_000, s2sGraceMs: 1_800_000, t4MinWatchMs: 5_000, maxHiddenRatio: 0.2,
  maxTicketsIssuedPerUserPerDay: 20, riskDenyT4At: 60,
};
// decideIssue(tier, userDay, netDay, cfg, now)        -> ok | {reason, retryAfterMs}
// decideClientResult(ticket, clientResult, now, cfg)  -> granted | shown (T1/T2 wait) | not_rewarded | denied | expired
// decideVerifiedCallback(ticket, cb, expected, now, cfg) -> grant | duplicate | orphan | rejected | too_late
```

### 3.9 Reference adapters (disabled until approvals)

Event mapping to `AdOutcome`:

| Provider | Available | Completed | Dismissed | No fill / blocked / other |
|---|---|---|---|---|
| H5 Games Ads | `beforeReward(showAdFn)` fired (keep `showAdFn`; it is single use) | `adViewed` | `adDismissed` | `adBreakDone.breakStatus`: `notReady` = blocked, `noAdPreloaded` / `other` / `ignored` = no_fill, `frequencyCapped` = frequency_capped, `timeout` = timeout, `error` / `invalid` = error |
| GPT rewarded | `rewardedSlotReady` (call `makeRewardedVisible()` on click) | `rewardedSlotGranted` seen before `rewardedSlotClosed` | `rewardedSlotClosed` without granted | slot `null` = not_supported, `slotRenderEnded.isEmpty` = no_fill, `googletag` missing = blocked |
| AdMob bridge | native `loaded` | native `earned` (UI only; T1 waits for SSV) | native `dismissed` without earned | native `failed` with code (no fill, network, internal) |
| AppLixir-style | player loaded | `status.type === 'complete'` (UI only; webhook grants) | `skipped` / `manuallyEnded` | `getError().type` such as `vastEmptyResponse`, `vastNoAdsAfterWrapper`, `adsRequestNetworkError` ([S34](https://support.applixir.com/applixir-integration/integration-for-html5-sites-apps/local-callback-error-codes)) |
| House video | creative returned | `ended` plus server timing | user closed | no creative = no_fill |

```ts
// playground-ads/providers/h5-games-ads.ts  (T4). Host page loads adsbygoogle.js with data-ad-client
// (+ data-adbreak-test="on" in dev) AFTER the CMP resolved, then defines window.adBreak/adConfig.
import type { AdAvailability, AdOutcome, AdShowHooks, AdTicket, AdsProvider, UnavailableReason } from '../types';

type H5BreakStatus = 'notReady' | 'timeout' | 'invalid' | 'error' | 'noAdPreloaded'
  | 'frequencyCapped' | 'ignored' | 'other' | 'dismissed' | 'viewed';
interface H5PlacementInfo { breakStatus: H5BreakStatus; breakName?: string }
interface H5RewardPlacement {
  type: 'reward'; name: string;
  beforeAd?: () => void; afterAd?: () => void;
  beforeReward?: (showAdFn: () => void) => void;
  adDismissed?: () => void; adViewed?: () => void;
  adBreakDone?: (info: H5PlacementInfo) => void;
}
declare global {
  interface Window {
    adBreak?: (placement: H5RewardPlacement) => void;
    adConfig?: (cfg: { preloadAdBreaks?: 'on' | 'auto'; sound?: 'on' | 'off'; onReady?: () => void }) => void;
  }
}
const UNAVAILABLE: Partial<Record<H5BreakStatus, UnavailableReason>> = {
  notReady: 'blocked', timeout: 'timeout', invalid: 'error', error: 'error',
  noAdPreloaded: 'no_fill', frequencyCapped: 'frequency_capped', other: 'no_fill', ignored: 'no_fill',
};

export class H5GamesAdsProvider implements AdsProvider {
  readonly id = 'h5-games-ads' as const;
  readonly tier = 'T4_CLIENT_ONLY' as const;
  private showAdFn: (() => void) | null = null;
  private pending: ((o: AdOutcome) => void) | null = null;
  private hooks: AdShowHooks = {};
  private startedAt = 0;

  async init(): Promise<void> { window.adConfig?.({ preloadAdBreaks: 'on', sound: 'on' }); }

  isAvailable(): Promise<AdAvailability> {
    return new Promise((resolve) => {
      if (!window.adBreak) { resolve({ available: false, reason: 'blocked' }); return; }
      let settled = false;
      const settle = (a: AdAvailability) => { if (!settled) { settled = true; resolve(a); } };
      const timer = window.setTimeout(() => settle({ available: false, reason: 'timeout' }), 3_000);
      window.adBreak({
        type: 'reward', name: 'playground_extra_try',
        beforeAd: () => this.hooks.onBeforeAd?.(),
        afterAd: () => this.hooks.onAfterAd?.(),
        beforeReward: (showAdFn) => { window.clearTimeout(timer); this.showAdFn = showAdFn; settle({ available: true }); },
        adViewed: () => this.finish({ status: 'completed', providerEvent: 'adViewed', elapsedMs: this.elapsed() }),
        adDismissed: () => this.finish({ status: 'dismissed', elapsedMs: this.elapsed() }),
        adBreakDone: (info) => {
          window.clearTimeout(timer);
          settle({ available: false, reason: UNAVAILABLE[info.breakStatus] ?? 'no_fill' });
          if (info.breakStatus !== 'viewed' && info.breakStatus !== 'dismissed') this.finish({ status: 'error', code: `h5_${info.breakStatus}` });
        },
      });
    });
  }

  show(_ticket: AdTicket, hooks: AdShowHooks = {}): Promise<AdOutcome> {
    const fn = this.showAdFn;
    this.showAdFn = null;                         // a showAdFn is single use
    if (!fn) return Promise.resolve({ status: 'no_fill' });
    this.hooks = hooks;
    this.startedAt = performance.now();
    return new Promise((resolve) => { this.pending = resolve; fn(); });
  }

  dispose(): void { this.showAdFn = null; this.pending = null; }
  private elapsed() { return Math.round(performance.now() - this.startedAt); }
  private finish(o: AdOutcome) { const p = this.pending; this.pending = null; p?.(o); }
}
```

The GPT adapter follows the same pattern (type-checked in the scratchpad; condensed here):
1. `isAvailable()` defines the `REWARDED` out-of-page slot. A `null` slot means `not_supported`.
2. It then calls `display(slot)` and resolves on `rewardedSlotReady` (available) or on `slotRenderEnded` with `isEmpty` (no_fill).
3. `show()` calls `onBeforeAd`, then `event.makeRewardedVisible()`.
4. It resolves on `rewardedSlotClosed`: `completed` if `rewardedSlotGranted` fired, else `dismissed`.
5. It then calls `destroySlots([slot])`, because only one rewarded slot may be requested at a time.

Native bridge message contract (Android `@JavascriptInterface`; iOS `webkit.messageHandlers.p2eAds`):

| Direction | Message |
|---|---|
| Web to native | `{"type":"ads.rewarded.isReady"}` and `{"type":"ads.rewarded.show","ticketId":"...","userId":"...","customData":"<ticketId>"}` |
| Native to web | `window.dispatchEvent(new CustomEvent('p2e-native-ad', {detail:{ticketId, event:'loaded'or'failed'or'shown'or'earned'or'dismissed', code}}))` |
| Native rules | Load the ad unit configured with reward `try` x 1 and SSV on. Set `ServerSideVerificationOptions(userId, customData=ticketId)` before `show()`. Never grant locally. |

### 3.10 Disclosure copy and UI rules (policy-driven)

| Do | Don't |
|---|---|
| "Watch a short ad to get **1 extra try** for Sky Hopper." | "Earn points / crypto / USDT by watching ads" |
| "Closing the ad early means no try." | "Click the ad", "Support us", arrows pointing at ads |
| "Ad tries left today: 2 of 3" | "Verified by Google" or any claim that Google grants the reward |
| Three buttons: **Watch ad**, **Use 10 points**, **Not now** | Auto-playing a second ad, or ads during a run |
| After dismiss: "No try this time. You can watch another ad or use 10 points." | Ad-block walls in the Playground |
| Show the prompt again before **every** ad (disclosure per instance) | Reusing the reward prompt for a non-rewarded ad |

### 3.11 Config (server-owned, per environment)

```json
{
  "ads": {
    "enabled": true,
    "adTriesPrizeEligible": true,
    "providerOrder": { "web": ["h5-games-ads", "house-video"], "app-android": ["admob-bridge"], "demo": ["mock"] },
    "providers": {
      "mock": { "enabled": true, "mode": "client", "durationMs": 15000, "skippableAfterMs": 5000 },
      "h5-games-ads": { "enabled": false, "adClient": "ca-pub-XXXX", "testMode": true, "frequencyHint": "30s" },
      "gpt-rewarded": { "enabled": false, "adUnitPath": "/NETWORK/playground_rewarded" },
      "admob-bridge": { "enabled": false, "expectedAdUnit": "NUMERIC_AD_UNIT_ID", "rewardItem": "try", "rewardAmount": 1 },
      "house-video": { "enabled": false }
    },
    "rules": "see DEFAULT_RULES",
    "dailyResetTz": "UTC"
  }
}
```

### 3.12 Acceptance tests for the build

1. The mock in `client` mode with `complete`: ticket goes `issued`, then `shown`, then `granted`. The user gets +1 ad try (`source=ad`). Cap counters increase.
2. `skip`: `not_rewarded`, no credit. `no_fill` / `blocked`: the "Watch ad" button is disabled with a message, and **the points option stays visible**.
3. The 4th T4 ad in a day is refused with `daily_cap` **before** any ad renders.
4. A T4 client result 2 s after `shown` is `denied` with `too_fast`.
5. The mock in `ssv` mode is granted only after the signed callback. Replaying the same callback returns 200 `duplicate` with no second credit. A tampered callback returns 403. A callback 5 min after TTL is granted (grace).
6. With free tries remaining, the ticket is refused with `free_tries_remaining`.
7. An ad try expires at the daily reset, cannot be transferred, and is not counted in "paid plays" or the dynamic bonus base.
8. With `adTriesPrizeEligible=false`, runs from ad tries are excluded from prize leaderboards but kept for personal best.
9. The raw-query SSV verification passes on the target backend framework. This is a regression test for the normalization gotcha.

---

## 4. Risks

| # | Risk | Severity | Likelihood | Mitigation |
|---|---|---|---|---|
| 1 | Google treats the reward (try, then cash-out points) as a disallowed monetary reward. Result: ads disabled, or AdSense / AdMob account action. | High | Medium | Written pre-clearance; the `adTriesPrizeEligible` switch; never ads for points; no ad revenue in prize pools; the house provider as a policy-independent path |
| 2 | H5 Games Ads application rejected or slow | Medium | Medium | GPT if Ad Manager exists; house video; points |
| 3 | Invalid traffic from reward-motivated users hurts the whole site's Google account | High | Medium | T4 caps of 3 per day, risk scoring before showing, separate ad units and channels, reconciliation, kill switches |
| 4 | Forged client callbacks (web) | Medium | High | Tiered caps, server timing, one open ticket, anomaly scoring; worst case 30 points a day per account |
| 5 | Low fill or eCPM: crypto audience, ad blockers, no consent | Medium | High | Provider chain, house video, points always available; set expectations with the revenue model in 2.7 |
| 6 | Paid-entry prize games classed as "online gambling", so restricted demand | Medium | Medium | Legal review; monitor the policy center |
| 7 | Consent non-compliance in the EEA / UK / CH (no certified CMP, TCF below v2.3) | High | Unknown | Deploy Google's CMP or another certified TCF v2.3 CMP plus GPP before Google ads go live |
| 8 | The native Playground conflicts with Google Play real-money contest or loyalty rules | High (native path) | Medium | Legal review before shipping in the app; the web Playground is unaffected |
| 9 | House ads promote crypto products to regulated markets | Medium | Medium | Advertiser review, geo-blocking, "Sponsored" label, legal track |
| 10 | Missing SSV callbacks (provider outage), so users watch and get nothing | Low | Low | `pending` UI, 30 min grace, 1 fallback grant per day under T4 rules |
| 11 | The backend framework normalizes the query, so every SSV fails | Medium | Medium | Use the raw `QUERY_STRING`; regression test #9 |
| 12 | Plus members' "ad-free" promise is seen as broken | Low | Low | Opt-in only, never shown automatically; owner decides whether Plus sees the button |
| 13 | Leaderboard fairness: ad tries mean more attempts | Low | Medium | 10 per day overall; the owner can lower T1 |

## 5. Open questions for the owner

1. Exactly what can P2E Points be redeemed for (USDT, tokens, gift cards, NFTs)? What is the USD value of 1 point? This decides the Google policy risk and the value-neutral eCPM.
2. Which ad stack does playtoearn.com use today: AdSense, Ad Manager (GPT), header bidding, only its own ads? Is there a Google account manager? This is the fastest route to written clearance. (Partly answered in verification: ads.txt authorizes three Google publisher IDs, ayeT-Studios and 16 other direct sellers including Amazon APS, so Google demand and header bidding are live in some form. The Google account type, GPT use and any account manager remain open.)
3. Is a Google-certified CMP (TCF v2.3) and GPP live today? Nothing was visible in the saved /earn page head.
4. Should Plus members, who were promised "ad-free", see the optional "Watch ad" button?
5. If Google requires it, may ad tries become practice-only (no prizes)? Or should the ad option be dropped for Google demand?
6. Would PlayToEarn Business sell a "Playground Sponsored Try" video product to Web3 advertisers (house provider)?
7. Is an update of the Android app (last update Nov 2024) or an iOS app planned to host the Playground (AdMob plus SSV path)? (The Plus page promises early access to in-house games on desktop, Android and iOS, so some iOS presence may exist: UNVERIFIED.)
8. What are monthly pageviews, Playground DAU targets and the geo split? These decide eligibility for Playwire (500K pageviews) and Venatus (about 1.5M).
9. Which time zone does the daily reset use (for free tries and ad caps)? UTC is proposed.
10. What is the minimum user age for the Playground? It sets the under-age consent tags.
11. Does PlayToEarn already reward points for watching videos or ads anywhere (for example, in /earn tasks)? That could conflict with AdSense rules for non-rewarded inventory. (Partly answered in verification: /earn hosts a third-party offerwall with app-install and survey offers, one of them a casino app, plus daily-login, X and referral tasks. No ad-view task was visible. The offerwall provider is UNVERIFIED; ayeT is a likely candidate given ads.txt.)
12. Are tries per game or shared? The ticket supports both through `gameId` and `scope`. Caps are always per user globally.

## 6. Cross-track notes

| Track | Implication |
|---|---|
| Economy | Ad tries spend no points: exclude them from "paid plays" and from the 50% dynamic bonus base. Break-even: `P_usd_per_point = eCPM_net / 5000`. Do not fund prize pools with ad revenue (policy). Ad tries expire at the daily reset and cannot be banked. Proposed caps are in 3.2. Store `prize_eligible` per try. |
| Game runtime and anti-cheat | Each run records the source of its try (`free`, `points`, `ad:<provider>`). Top-100 runs from ad tries get the same replay validation. The run token binds to the consumed try credit. Ads are never shown mid-run, only on menus. Games need pause / mute hooks only for menu audio. |
| Integration architecture | Client `AdsProvider` adapters plus server `AdTicketService` plus a per-provider callback verifier. The HTTP contract is in 3.6. Callback routes skip CSRF and session. **Raw query string** for SSV (Symfony / Laravel `getQueryString()` normalizes). Config is server-owned (3.11). The mock network is dev-only. |
| Legal and compliance | Rewards policy exposure (points cash out). "Online gambling" publisher restriction. Google Play real-money contest and loyalty rules for native. Certified CMP, TCF v2.3, GPP. Legal basis for the fraud-prevention hashes. Crypto promotion rules for house ads (UK, EU MiCA). Age gating. |
| UX | Disclosure modal before every ad, remaining-ads counter, three-way choice (ad, points, not now), graceful `no_fill` / `blocked` states, "try arriving shortly" pending state, accessibility (focus, live countdown), no ad-block walls, the Plus member question. |
| Platform and market | Android app facts (100K+ installs, contains ads, updated 2024-11-15). The PlayToEarn Business ad product. Stack hints from the saved /earn head (Laravel-style CSRF meta, jQuery, Backbone, WalletConnect, Google Sign-In, GA4), UNVERIFIED. |
| Assets and audio | The mock ad needs no generated asset (CSS only). If a placeholder "sponsored" visual is wanted, it must follow the no-text-artifact rule. House video creatives come from advertisers. |
| Game roster | No ad hooks inside gameplay are needed. Keep a common `onBeforeAd` / `onAfterAd` (pause and mute) in the game shell for menu music. |

## 7. Sources

Accessed 2026-09-24. [V] = fetched this session. [SR] = seen in search results only. [3P] = third-party or vendor.

1. [S1] AdMob SSV, Android [V]: https://developers.google.com/admob/android/ssv
2. [S2] AdMob SSV, iOS [V]: https://developers.google.com/admob/ios/ssv
3. [S3] AdMob Help, validate rewarded ad views with SSV [V]: https://support.google.com/admob/answer/9603226?hl=en
4. [S4] AdMob SSV verifier keys [V]: https://www.gstatic.com/admob/reward/verifier-keys.json
5. [S5] AdMob rewarded interstitial, Android [V]: https://developers.google.com/admob/android/rewarded-interstitial
6. [S6] Google Ad Manager, AdSense and AdMob comparison [V]: https://support.google.com/admob/answer/9234653?hl=en
7. [S7] WebView API for Ads, Android [V]: https://developers.google.com/admob/android/browser/webview/api-for-ads
8. [S8] Technical requirements for web content viewing frames for apps [V]: https://support.google.com/admanager/answer/6310245?hl=en
9. [S9] AdMob policies for ad units that offer rewards [V]: https://support.google.com/admob/answer/7313578?hl=en
10. [S10] AdSense policies for ad units that offer rewards [V]: https://support.google.com/adsense/answer/9121589?hl=en
11. [S11] Ad Manager policies for ad units that offer rewards [V]: https://support.google.com/admanager/answer/7496282?hl=en
12. [S12] AdSense Program policies [V]: https://support.google.com/adsense/answer/48182?hl=en
13. [S13] Google Publisher Restrictions [V]: https://support.google.com/publisherpolicies/answer/10437795?hl=en
14. [S14] Google Play, Real-Money Gambling, Games, and Contests [V]: https://support.google.com/googleplay/android-developer/answer/9877032?hl=en
15. [S15] Google Play, Blockchain-based Content [V in verification]: https://support.google.com/googleplay/android-developer/answer/13607354?hl=en and https://android-developers.googleblog.com/2023/07/new-blockchain-based-content-opportunities-google-play.html
16. [S16] Ad Placement API overview [V]: https://developers.google.com/ad-placement/apis
17. [S17] adBreak() reference [V]: https://developers.google.com/ad-placement/apis/adbreak
18. [S18] adConfig() reference [V]: https://developers.google.com/ad-placement/apis/adconfig
19. [S19] H5 Games Ads, how to sign up [V]: https://developers.google.com/ad-placement/docs/signup
20. [S20] Placement types [V]: https://developers.google.com/ad-placement/docs/placement-types
21. [S21] Control the ad rate [V]: https://developers.google.com/ad-placement/docs/ad-rate
22. [S22] Example implementation [V]: https://developers.google.com/ad-placement/docs/example
23. [S23] AdSense, get started with H5 Games Ads [V]: https://support.google.com/adsense/answer/9959170?hl=en
24. [S24] AdSense, add the AdSense code to your game page [V]: https://support.google.com/adsense/answer/9955214?hl=en
25. [S25] AdSense H5 Games Ads product page [V]: https://adsense.google.com/start/solutions/h5-games-ads/
26. [S26] Ad Manager, manage H5 Games Ads [V]: https://support.google.com/admanager/answer/14637831?hl=en
27. [S27] GPT sample, display a rewarded ad [V]: https://developers.google.com/publisher-tag/samples/display-rewarded-ad
28. [S28] GPT reference [V]: https://developers.google.com/publisher-tag/reference
29. [S29] GPT rewarded sample source (TS) [V]: https://github.com/googleads/google-publisher-tag-samples/tree/main/dist/display-rewarded-ad/ts
30. [S30] Ad Manager, traffic rewarded ads for web [V]: https://support.google.com/admanager/answer/9116812?hl=en
31. [S31] Ad Manager, sign up [V in verification]: https://support.google.com/admanager/answer/7084151?hl=en
32. [S32] AppLixir publisher FAQ [V, vendor]: https://support.applixir.com/frequently-asked-questions
33. [S33] AppLixir, setting up callbacks [V, vendor]: https://support.applixir.com/applixir-integration/integration-for-html5-sites-apps/step-4-setting-up-local-callback-360053188774
34. [S34] AppLixir, local callback error codes [V, vendor]: https://support.applixir.com/applixir-integration/integration-for-html5-sites-apps/local-callback-error-codes
35. [S35] AppLixir, structured info [V, vendor]: https://support.applixir.com/ai-info/
36. [S36] AppLixir, eCPM optimization for web games [3P]: https://www.applixir.com/blog/ecpm-optimization-for-web-games-benchmarks-and-fixes/
37. [S37] Playwire FAQ [V, vendor]: https://www.playwire.com/faq
38. [S38] Playwire, rewarded video ads for websites [3P]: https://www.playwire.com/blog/web-rewarded-video-ads
39. [S39] Playwire, rewarded video ad unit [3P]: https://www.playwire.com/rewarded-video-ad-units
40. [S40] Playwire, AdMob eCPM benchmarks (2025-09-17) [3P]: https://www.playwire.com/blog/admob-ecpm-benchmarks-what-publishers-should-expect
41. [S41] Nitro blog, Nitro vs Playwire vs Venatus [3P, biased]: https://blog.nitropay.com/nitro-vs-playwire-vs-venatus-which-ad-network-is-right-for-gaming-publishers/
42. [S42] NitroAds FormatOptions enum [V]: https://api-docs.nitropay.com/enums/_options_.formatoptions.html
43. [S43] Nitro FAQ [V, vendor]: https://nitropay.com/faq/
44. [S44] AdinPlay publishers [V, vendor]: https://adinplay.com/publishers
45. [S45] Venatus browser game monetization [V, vendor]: https://www.venatus.com/publishers/browser-game-monetization
46. [S46] GameDistribution rewarded ads wiki [V]: https://github.com/GameDistribution/GD-HTML5/wiki/Rewarded-Ads
47. [S47] GameMonetize SDK README [V]: https://github.com/MonetizeGame/GameMonetize.com-SDK/blob/master/README.md
48. [S48] AdsGram publisher docs [V]: https://docs.adsgram.ai/publisher/
49. [S49] AdsGram reward URL, community implementation [3P, SR]: https://github.com/zekogg/MinerX-Realm/pull/33
50. [S50] Monetag postbacks and macros [V]: https://docs.monetag.com/docs/postbacks/configuration/ and https://docs.monetag.com/docs/postbacks/macroses/
51. [S51] Monetag SDK intro [V]: https://docs.monetag.com/
52. [S52] CrazyGames docs [V]: https://docs.crazygames.com/
53. [S53] Google consent management requirements for EEA, UK, Switzerland (publishers) [V]: https://support.google.com/adsense/answer/13554116?hl=en
54. [S54] IAB Europe, transition to TCF v2.3 [V]: https://iabeurope.eu/all-you-need-to-know-about-the-transition-to-tcf-v2-3/
55. [S55] AdSense, limited ads [V]: https://support.google.com/adsense/answer/14210870?hl=en
56. [S56] AdMob, limited ads [V]: https://support.google.com/admob/answer/10105530?hl=en
57. [S57] Ad Manager, supporting the IAB GPP [V]: https://support.google.com/admanager/answer/14117049?hl=en
58. [S58] AdSense, US states privacy laws [V]: https://support.google.com/adsense/answer/9560818?hl=en
59. [S59] Ad blocker statistics, GWI via Backlinko, updated 2026-03-19 [3P, read in verification]: https://backlinko.com/ad-blockers-users
60. [S60] Brave, 100M monthly active users, 2025-10-01 [V in verification]: https://brave.com/blog/100m-mau/
61. [S61] PlayToEarn Android app, Google Play listing [V]: https://play.google.com/store/apps/details?id=com.playtoearn.playtoearn&hl=en_US
62. [S62] PlayToEarn Plus [V in verification, via WebFetch]: https://playtoearn.com/plus
63. [S63] PlayToEarn rewards and tasks page [V in verification, via WebFetch]: https://playtoearn.com/earn
64. [S64] PlayToEarn Business [V]: https://business.playtoearn.com/
65. [S65] Symfony HttpFoundation Request (getQueryString normalization) [V]: https://github.com/symfony/http-foundation/blob/7.3/Request.php
66. [S66] Community Node SSV library (reference only) [V]: https://github.com/voonic/admob-rewarded-ads-ssv
67. [S67] Tink Java apps, RewardedAdsVerifier (referenced by S2) [V in verification, source read]: https://github.com/tink-crypto/tink-java-apps (file `rewardedads/src/main/java/com/google/crypto/tink/apps/rewardedads/RewardedAdsVerifier.java`)
68. [S68] AdMob community thread on "chances" at monetary rewards. The question (2025-01-29, iOS) asks about draw entries for a cash prize in exchange for ad views; the replies are not in the static page, so the answer stays UNVERIFIED [SR]: https://support.google.com/admob/thread/321297311/are-chances-at-winning-direct-monetary-items-allowed-for-watching-reward-ads?hl=en
69. [S69] Decrypt, Google policy change on NFT game ads (2023) [3P]: https://decrypt.co/155185/google-changes-policy-allow-nft-game-ads-some-limits
70. [S70] playtoearn.com ads.txt, 189 lines, fetched 2026-09-24 [V, added in verification]: https://playtoearn.com/ads.txt
71. [S71] ayeT-Studios, Rewarded Video SDK for HTML5 [V, vendor, added in verification]: https://docs.ayetstudios.com/v/product-docs/rewarded-video/web-integrations/rewarded-video-sdk-for-html5
72. [S72] ayeT-Studios, rewarded video callbacks [V, vendor, added in verification]: https://docs.ayetstudios.com/v/product-docs/callbacks-and-testing/callbacks/rewarded-video-callbacks
73. [S73] PlayToEarn advertise page [V, added in verification]: https://playtoearn.com/advertise
74. [S74] AdMob GMA Next-Gen SDK, rewarded ads, Android [V, added in verification]: https://developers.google.com/admob/android/next-gen/rewarded
75. [S75] ayeT-Studios, HMAC security hash [V, vendor, added in verification]: https://docs.ayetstudios.com/v/product-docs/callbacks-and-testing/callback-verification/hmac-security-hash-optional
76. [S76] ayeT-Studios, callback IP whitelist [V, vendor, added in verification]: https://docs.ayetstudios.com/v/product-docs/callbacks-and-testing/callbacks/ip-whitelist
77. [S77] Apple App Review Guidelines [V, added in verification]: https://developer.apple.com/app-store/review/guidelines/
78. [S78] Google Ads policy update on NFT games, effective 2023-09-15 [V, added in verification]: https://support.google.com/adspolicy/answer/13985443?hl=en

## Verification log

Adversarial review, 2026-09-24. Each claim was re-checked against the primary source listed, fetched in this review (Google pages, vendor docs and GitHub sources with WebFetch or curl; playtoearn.com pages with WebFetch only, because plain curl gets a Cloudflare challenge that was not bypassed). Rows 1 to 20 are the decision-critical set; the rest are supporting checks.

| # | Claim (as stated in the report) | Verdict | Source URL |
|---|---|---|---|
| 1 | AdMob serves app inventory only; AdSense covers web, Ad Manager covers web and app. | confirmed | https://support.google.com/admob/answer/9234653?hl=en |
| 2 | Ad Manager web rewarded: server-side verification is an app-only feature, unavailable for web (exact quote in TL;DR 2). | confirmed | https://support.google.com/admanager/answer/9116812?hl=en |
| 3 | Google rewarded-ads policies ban direct monetary items and name cryptocurrency; indirect rewards only if usable only on the publisher's platform and non-transferable (not directly convertible into direct monetary items); 25% cap on physical-item discounts; random rewards need disclosure and a chance above 0; the reward must be delivered; disclosure before each ad; no implied Google endorsement. The monetary, non-transferable, delivery and disclosure rules read the same in AdMob, AdSense and Ad Manager; the random-reward rule was checked in AdMob and Ad Manager, the endorsement rule in AdMob. | confirmed | https://support.google.com/admob/answer/7313578?hl=en , https://support.google.com/adsense/answer/9121589?hl=en , https://support.google.com/admanager/answer/7496282?hl=en |
| 4 | Ad Placement API: rewards must have no value outside the app and must not have, or be easily exchanged for, monetary value. | confirmed | https://developers.google.com/ad-placement/apis |
| 5 | H5 Games Ads: apply by form, approved AdSense account required, approval not guaranteed, limited-scope permission for full-screen ads (page updated 2026-06-18). | confirmed | https://developers.google.com/ad-placement/docs/signup |
| 6 | H5 Games Ads in app WebViews is Android only; any publisher owning an H5 games website that follows AdSense policies may apply. | confirmed | https://adsense.google.com/start/solutions/h5-games-ads/ |
| 7 | "No web network offers a signed reward callback" (2.5 takeaway 1). | corrected: ayeT-Studios offers an HMAC-SHA256-signed S2S callback for HTML5 rewarded video and is already in playtoearn.com's ads.txt | https://docs.ayetstudios.com/v/product-docs/callbacks-and-testing/callback-verification/hmac-security-hash-optional , https://playtoearn.com/ads.txt |
| 8 | AppLixir excludes crypto-related content; needs 5,000 daily ad impressions or active users; approval in 1 to 2 business days; Net-30; $100 minimum payout; healthy completion rate 80 to 90%. | confirmed | https://support.applixir.com/frequently-asked-questions |
| 9 | AppLixir webhook: GET, `md5(gameApiKey+gameId+userId+tid+secret)`, `customData` unsigned, at-least-once delivery, dedupe on `tid`. | confirmed; omission added: every callback also carries the plaintext `secretKey` | https://support.applixir.com/applixir-integration/integration-for-html5-sites-apps/step-4-setting-up-local-callback-360053188774 |
| 10 | Playwire: 500,000 monthly pageviews, English-speaking footprint, IVT at most 7%, not blacklisted with Google, Google Analytics and HTTPS privacy page, Net 60. | confirmed | https://www.playwire.com/faq |
| 11 | Venatus about 1.5M monthly pageviews plus 20% Tier-1 traffic, Net-90; Playwire self-serve from 100K; Nitro 100K, Net-7 (Nitro blog dated 2026-04-15). | confirmed as third-party claims | https://blog.nitropay.com/nitro-vs-playwire-vs-venatus-which-ad-network-is-right-for-gaming-publishers/ |
| 12 | AdMob SSV: parameter list, `signature` then `key_id` always last, key server URL, cache keys at most 24 h, keys rotate on a variable schedule, up to 5 retries at 1 s intervals. | confirmed | https://developers.google.com/admob/android/ssv |
| 13 | Key server on 2026-09-24: one key, keyId `3335741209`, P-256, `Cache-Control: max-age=0`, `Last-Modified` 2023-06-30. | confirmed (re-fetched) | https://www.gstatic.com/admob/reward/verifier-keys.json |
| 14 | Signature is base64url, DER-encoded ECDSA with SHA-256; the documented sample decodes to 71 bytes starting `0x30450221`. | confirmed (sample decoded again; Tink uses `urlSafeDecode` and DER). Caveat added: Tink verifies over the percent-decoded query | https://developers.google.com/admob/android/ssv , https://github.com/tink-crypto/tink-java-apps |
| 15 | Symfony / Laravel `getQueryString()` normalizes the query (ksort plus `http_build_query` RFC 3986), so SSV must use the raw `QUERY_STRING`. | confirmed (source read) | https://github.com/symfony/http-foundation/blob/7.3/Request.php |
| 16 | Consent: a Google-certified TCF CMP is required for personalized ads in the EEA and UK since 2024-01-16 and in Switzerland since 2024-07-31; Google's CMP (ID 300) is certified for web and app. | confirmed | https://support.google.com/adsense/answer/13554116?hl=en |
| 17 | TCF v2.3 released 2025-06-19; TC strings created after 2026-02-28 need the `disclosedVendors` segment; older strings stay valid. | confirmed | https://iabeurope.eu/all-you-need-to-know-about-the-transition-to-tcf-v2-3/ |
| 18 | Google Play real-money policy: "Pilot programs are the only exception". | corrected: exceptions are licensed gambling apps, daily fantasy sports apps, pilot programs and qualifying gamified loyalty programs | https://support.google.com/googleplay/android-developer/answer/9877032?hl=en |
| 19 | IVT enforcement acts at account level (was UNVERIFIED scope). | confirmed: Google may disable ad serving to the site and/or disable the AdSense account | https://support.google.com/adsense/answer/48182?hl=en |
| 20 | P2E Points cash out to USDT and tokens (/earn). | confirmed: page title "PlayToEarn Rewards & Tasks - Earn USDT, Tokens & Points Daily", 2,000 points needed for the first payout | https://playtoearn.com/earn |
| 21 | GPT rewarded runs on desktop, mobile and tablet web inventory; display creatives need 5 s in view; no simultaneous rewarded requests; "Block non-instream video ads" must be off. | confirmed | https://support.google.com/admanager/answer/9116812?hl=en |
| 22 | GPT `defineOutOfPageSlot(..., REWARDED)` may return null; rewarded ads need mobile-optimized pages with neutral zoom. | confirmed | https://developers.google.com/publisher-tag/samples/display-rewarded-ad |
| 23 | H5 ad rate: default hint 120 s, at most one ad every 30 s, first `adBreak()` exempt; the 10 `breakStatus` values; `beforeReward` fires only when an ad is available; an old `showAdFn` has no effect. | confirmed | https://developers.google.com/ad-placement/docs/ad-rate , https://developers.google.com/ad-placement/apis/adbreak |
| 24 | H5 formats serve display, TrueView and Bumper; placement rules; tag parameters are fixed for the pageview. | confirmed | https://support.google.com/adsense/answer/9959170?hl=en , https://support.google.com/adsense/answer/9955214?hl=en |
| 25 | SSV setup requires "Verify URL" before saving; a checkbox applies SSV to all networks in mediation groups. | confirmed | https://support.google.com/admob/answer/9603226?hl=en |
| 26 | WebView API for Ads: GMA SDK 20.6.0+, Android API 21+, manifest `INTEGRATION_MANAGER=webview`; app consent (TCF v2.3, CCPA) is not passed to web tags in the WebView. | confirmed | https://developers.google.com/admob/android/browser/webview/api-for-ads |
| 27 | A loaded rewarded interstitial expires after 1 hour and needs an intro screen with a skip option. | confirmed | https://developers.google.com/admob/android/rewarded-interstitial |
| 28 | Signing up for Ad Manager requires an AdSense account (was UNVERIFIED). | confirmed | https://support.google.com/admanager/answer/7084151?hl=en |
| 29 | Android next-gen SDK SSV method names (were UNVERIFIED). | corrected: `ServerSideVerificationOptions.Builder().setCustomData(...)` and `rewardedAd.setServerSideVerificationOptions(...)`; a user id setter is not shown | https://developers.google.com/admob/android/next-gen/rewarded |
| 30 | Adsgram and Monetag "work only inside Telegram". | corrected (narrowed): Adsgram documents Mini Apps, channels and bots; for Monetag, only its SDK docs were checked and they cover Telegram Mini Apps only. Adsgram CPM of 0.5 to 1 USDT confirmed | https://docs.adsgram.ai/publisher/ , https://docs.monetag.com/ |
| 31 | Nitro `FormatOptions` includes `Rewarded`; AdinPlay offers desktop and mobile rewarded video for browser games, powered by Venatus. | confirmed | https://api-docs.nitropay.com/enums/_options_.formatoptions.html , https://adinplay.com/publishers |
| 32 | Google Play blockchain-based content: do not promote or glamorize potential earnings (was a search result only). | confirmed; added the Financial features declaration requirement | https://support.google.com/googleplay/android-developer/answer/13607354?hl=en |
| 33 | Online gambling publisher restriction means fewer eligible ad sources, not a ban, with exclusions for users in the US, UK, DE, FR, JP and other countries. | confirmed | https://support.google.com/publisherpolicies/answer/10437795?hl=en |
| 34 | GPP: US National, CA, CO, CT, FL, VA accepted; National v2 from September 2025; opt-outs trigger RDP; GPP is optional for Google. | confirmed | https://support.google.com/admanager/answer/14117049?hl=en |
| 35 | Limited ads: only IVT-detection cookies without consent, programmatic demand still eligible, attempted for EEA / UK / CH requests without a certified-CMP TC string; in AdMob they also disable frequency capping. | confirmed | https://support.google.com/adsense/answer/14210870?hl=en , https://support.google.com/admob/answer/10105530?hl=en |
| 36 | Ad blockers: 29.5% of users (GWI Q2 2025), 37% on desktop and 15% on mobile. | corrected: 29.5% confirmed; the desktop / mobile split is a 2020 US AudienceProject survey; newer US data in the same article is 27% vs 22% | https://backlinko.com/ad-blockers-users |
| 37 | Brave passed 100M monthly active users and blocks ads by default. | confirmed (101M, 2025-10-01) | https://brave.com/blog/100m-mau/ |
| 38 | eCPM: AdMob rewarded Tier-1 $15 to $30, global $8 to $18, plan for 70 to 80% of estimates (2025-09-17); AppLixir web rewarded $4+, web display $0.50 to $2, $11.40 average. | confirmed as vendor claims | https://www.playwire.com/blog/admob-ecpm-benchmarks-what-publishers-should-expect , https://www.applixir.com/blog/ecpm-optimization-for-web-games-benchmarks-and-fixes/ |
| 39 | Android app: 100,000+ installs, contains ads, released 2021-01-10, updated 2024-11-15. | confirmed (re-checked) | https://play.google.com/store/apps/details?id=com.playtoearn.playtoearn&hl=en_US |
| 40 | Plus: $9.99 per month or $99.90 per year, ad-free browsing, double points, exclusive Reward Center prizes (was UNVERIFIED on page). | confirmed | https://playtoearn.com/plus |
| 41 | PlayToEarn Business "advertises more than 14 ad products" (was tagged [V]). | unverified: business.playtoearn.com is a sign-in page; /advertise lists about six formats and no video | https://business.playtoearn.com/ , https://playtoearn.com/advertise |
| 42 | Google Ads has barred ads for NFT games that let players wager or stake NFTs for real-world value since 2023. | confirmed with the primary source (effective 2023-09-15) | https://support.google.com/adspolicy/answer/13985443?hl=en |
| 43 | Code: the TS type-checks with tsc 5.9 `--strict`; the SSV verifier and mock signer pass tamper and unknown-key tests. | confirmed (re-run with tsc 5.9.3 and Bun 1.3.9 in the review scratchpad) | local re-run, no URL |
| 44 | The ticket-rules self-test (cap order, cooldown, grace, duplicates, user mismatch). | unverified: the excerpt holds only defaults and signatures | report section 3.8 |
| 45 | What Google answered in the community thread about "chances" at monetary prizes. | unverified: only the question is in the static page | https://support.google.com/admob/thread/321297311/are-chances-at-winning-direct-monetary-items-allowed-for-watching-reward-ads?hl=en |

Totals: 45 claims checked. 37 confirmed, 5 corrected, 3 unverified.

## Omissions found in verification

1. **PlayToEarn already runs Google and header-bidding demand.** ads.txt has 21 DIRECT records (three `google.com` publisher IDs, two ayeT entries, and 16 others including Amazon APS) and 152 RESELLER records [V] ([S70](https://playtoearn.com/ads.txt)). What this means:
   * Google's rewarded and IVT rules put the site's existing ad revenue at risk, not only the Playground. Whoever manages the Google account must be involved before any Google rewarded unit goes live.
   * An approved AdSense account, which H5 Games Ads requires, may already exist.
   * An Amazon APS line usually comes with GPT, which would make `gpt-rewarded` cheap to add (UNVERIFIED).
   * Ask the owner which Google publisher ID is PlayToEarn's own, and who operates the others.
2. **ayeT-Studios is a ready web T2 candidate.** ads.txt lists `ayetstudios.com, PL-22398, DIRECT`. ayeT documents:
   * an HTML5 rewarded video SDK for desktop and mobile browsers, with CMP auto-detection; ads.txt is required ([S71](https://docs.ayetstudios.com/v/product-docs/rewarded-video/web-integrations/rewarded-video-sdk-for-html5));
   * S2S postbacks that must get HTTP 200, retried 12 times over one hour, with a per-view `{payout_usd}` macro ([S72](https://docs.ayetstudios.com/v/product-docs/callbacks-and-testing/callbacks/rewarded-video-callbacks));
   * an optional HMAC-SHA256 header keyed with the publisher API key ([S75](https://docs.ayetstudios.com/v/product-docs/callbacks-and-testing/callback-verification/hmac-security-hash-optional));
   * published callback IPs, last updated 2025-01-30 ([S76](https://docs.ayetstudios.com/v/product-docs/callbacks-and-testing/callbacks/ip-whitelist)).

   Ask ayeT in writing: does the existing account cover rewarded video? Where does the demand come from? If it resells Google demand, Google's rewards policy applies downstream. Is a "try" that competes for point prizes an acceptable reward? How is a ticket id passed for rewarded video? For the build: add `'ayet'` to `AdProviderId` and an `ayet` adapter stub. `{payout_usd}` also gives the economy track real per-view revenue for the value-neutral formula in 2.7.
3. **Signature rules differ per provider.**
   * AdMob: verify the raw bytes before `&signature=`. Tink reads the percent-decoded query, so keep `custom_data` and `user_id` to URL-unreserved characters ([S1](https://developers.google.com/admob/android/ssv), [S67](https://github.com/tink-crypto/tink-java-apps)).
   * ayeT: sort all parameters alphabetically and re-encode them (spaces as `+`) before the HMAC. This is the opposite of the AdMob raw-bytes rule ([S75](https://docs.ayetstudios.com/v/product-docs/callbacks-and-testing/callback-verification/hmac-security-hash-optional)).
   * AppLixir: MD5 over the concatenated values ([S33](https://support.applixir.com/applixir-integration/integration-for-html5-sites-apps/step-4-setting-up-local-callback-360053188774)).

   Give each provider its own verifier and regression test, extending acceptance test 9.
4. **Secrets in callback logs.** AppLixir sends its plaintext `secretKey` in every callback ([S33](https://support.applixir.com/applixir-integration/integration-for-html5-sites-apps/step-4-setting-up-local-callback-360053188774)). The 90-day raw-query audit log must redact it. For AppLixir, T2 trust is lower than the MD5 scheme suggests.
5. **Grace window per provider.** AdMob retries only 5 times at 1 s intervals; ayeT retries for up to one hour. A single 30-minute grace would reject late but genuine ayeT callbacks, so make `s2sGraceMs` a per-provider setting.
6. **Apple rules for an iOS host app.** The report covered only Google Play. Apple's App Review Guidelines ([S77](https://developer.apple.com/app-store/review/guidelines/)):
   * 3.2.2(x) explicitly allows rewarding users for actions such as watching an ad.
   * 3.1.5(v) bars crypto apps from offering currency for completing tasks. This concerns /earn-style tasks, not ad tries.
   * 5.3.1 and 5.3.2 require contests to be sponsored by the developer, with official rules in the app that say Apple is not involved.
   * 5.3.3 bars in-app purchase of credit or currency for real-money gaming.

   The legal track should assess these before any iOS Playground.
7. **Google Play financial features declaration and listing copy.** Apps that sell or let users earn tokenized digital assets must declare it in Play Console ([S15](https://support.google.com/googleplay/android-developer/answer/13607354?hl=en)). The current listing says users can explore games "to earn Cryptocurrency" ([S61](https://play.google.com/store/apps/details?id=com.playtoearn.playtoearn&hl=en_US)). The legal track should compare that with the rule against glamorizing earnings before the Playground ships in the app.
8. **Audience facts that change eligibility and fill.** /advertise reports more than 180,000 registered users, millions of monthly visitors and 60% mobile users ([S73](https://playtoearn.com/advertise)). If accurate:
   * Playwire's 500K monthly pageview floor is likely met, though the geo split is still unknown.
   * Mobile-heavy traffic suits GPT rewarded, which needs mobile-optimized pages.
   * Mobile users block ads less often.
9. **Plus wording.** The Plus page promises ad-free browsing with no banners, pop-ups or interruptions ([S62](https://playtoearn.com/plus)). An opt-in "Watch ad" button is arguably not an interruption, but showing it to Plus members needs an explicit owner decision (open question 4).
10. **Incentivized offers already on /earn.** /earn runs a third-party offerwall with app-install offers, one of them a casino app, plus social and referral tasks ([S63](https://playtoearn.com/earn)). If Google ad units run on those pages, the legal track should check them against the AdSense rule on compensating users for viewing ads outside rewarded inventory ([S12](https://support.google.com/adsense/answer/48182?hl=en)).
11. **No back-to-back full-screen ads.** H5 Games Ads forbid full-screen ads triggered right after a user closes another one ([S23](https://support.google.com/adsense/answer/9959170?hl=en)). The design already needs a fresh click for every ad. Keep it that way: never auto-start a second ad from the "No try this time" state.
12. **User activation (engineering, UNVERIFIED).** `watchAdTry` waits up to 800 ms for `markShown` before it calls the SDK. Browsers keep click activation only for a limited time, and video with sound may need it. Test on iOS Safari. If the ad fails to start, send `markShown` without waiting for it.

Cross-track notes from verification:

| Track | Note |
|---|---|
| Platform and market | ads.txt lists 21 DIRECT records (three Google IDs, two ayeT entries, 16 others including Amazon APS) and 152 resellers. /advertise reports more than 180,000 registered users, millions of monthly visitors and 60% mobile users. The Plus page promises in-house games on iOS. /earn lists a "Teddy Jump Challenge" task, which suggests an existing jump-style game (UNVERIFIED). The game roster track should check it too. |
| Legal and compliance | Apple guidelines 3.1.5(v), 3.2.2(x) and 5.3.1 to 5.3.3. The Play financial features declaration and the "to earn Cryptocurrency" listing copy. Casino-app offers on /earn next to any Google ad units. |
| Integration architecture | Per-provider verifiers and canonicalization rules. Redact secrets before logging. Per-provider grace windows. An `ayet` provider id and adapter. For ayeT, store the HMAC header next to the raw query. |
| Economy | ayeT's `{payout_usd}` gives real per-view revenue for the value-neutral points price (2.7). |
