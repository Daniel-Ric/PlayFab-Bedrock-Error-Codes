# PlayFab Economy v2 / Minecraft Bedrock Error Codes

Last updated: March 05, 2026 (Europe/Berlin)
Audience: Minecraft Bedrock client devs, LiveOps, QA, support engineers

> **Disclaimer**
> - **Named error codes** (e.g., `Drowned`, `Creeper`, `Breeze`) are documented in the Minecraft Help Center.
> - **Marketplace letter-number codes** (e.g., `B-0010`, `C-0010`, `SD-002-0`) have only partial public documentation.
>   Most meanings below are derived from *observed in-game UI text*, Mojira bug reports, and player impact patterns.
> - Treat letter-number codes as **operational signals** (what to tell the player + whether retries help), not as a stable backend API contract.

---

## Table of Contents

1. Overview
2. Quick Reference
   - 2.1 Marketplace code families (letter-number codes)
   - 2.2 Named error codes (Bedrock general)
3. Marketplace Error Codes (Detailed)
4. Named Error Codes (Detailed)

---

## 1. Overview

Minecraft Bedrock Edition uses PlayFab-backed online services for Marketplace catalog/content delivery,
entitlements, inventory, and Character Creator/Dressing Room. When these calls fail, the client often surfaces
short Marketplace-style codes (for example `B-0010` or `C-0010`) instead of raw HTTP status codes or PlayFab enums.

This README groups those codes by **player-visible behavior** so you can:
- identify the failure class quickly,
- decide whether to retry automatically,
- provide clear player guidance.

Notes:
- The **same code** may show slightly different text per locale/platform.
- Many failures collapse into a small set of “can’t connect / restart / try later” screens.
- For engineering triage, always capture: platform, locale, build, sign-in state, and whether Realms/Servers also fail.

---

## 2. Quick Reference

### 2.1 Marketplace code families (letter-number codes)

| Code / Family        | Player-facing symptom / typical in-game text                                             | Likely root cause class                                            | Retry hint                                  |
|---------------------:|-------------------------------------------------------------------------------------------|--------------------------------------------------------------------|---------------------------------------------|
| `A-0010`             | Preview build: “Store is unavailable in Preview.”                                         | Preview/Beta environment restrictions, build gating, env mismatch  | **No retry** (switch build / flow)          |
| `A-200`              | Marketplace blocked early (often reported on consoles)                                    | Account/storefront entitlement handshake failed *(low-confidence)* | **Retry after re-auth**                     |
| `B-0010`             | “We are still trying to connect. Please wait.”                                            | Marketplace endpoint timeout / transient network / service reach   | Safe to retry with backoff                  |
| `C-0010`             | “We ran into an error. Please restart the game and try again.”                            | Session validation / account state / entitlement fetch failed      | Retry after restart + re-auth               |
| `C-2010` / `C2010`   | Marketplace + skins won’t load (reported as `C2010` variants)                             | Same bucket as `C-0010`                                             | Same as `C-0010`                            |
| `D-0010`             | “We had an issue connecting. Please go back and try again.” Often + Dressing Room breaks | Broad connectivity failure to online services / routing / outage   | Retry later; local steps sometimes help     |
| `E-0010` / `e-0010`  | Reported as “E-0010” in some tickets; often overlaps with `D-0010` screens               | Likely same connectivity family *(ambiguous in field reports)*     | Treat like `D-0010`                         |
| `F-0010`             | “We can’t detect an internet connection.” (commonly reported on Xbox/console)            | Console network detection / captive portal / network layer issue   | Retry after network reset + relaunch        |
| `L-404` / `L-4xx`    | Marketplace page/deep link fails; often “trouble connecting… try again later”            | Marketplace content fetch failed: service issue OR item not usable | Retry later; may be server-side/content-gate|
| `SD-002-0`           | “Hmm, Marketplace is having some issues, please try connecting later.”                    | Commerce/Marketplace backend degraded / maintenance                | Retry later; nothing local                  |
| `U-000`              | “We’re having trouble connecting to the Marketplace right now. Please try again later!”  | Marketplace request failed / connectivity / transient backend      | Retry later; backoff helps                  |

Legend:
- **Safe to retry**: Automatic backoff + retry is reasonable.
- **No retry**: Retrying won’t help until the environment/build/account is fixed.
- **Retry later**: Usually Mojang/Microsoft-side issues or maintenance windows.

### 2.2 Named error codes (Bedrock general)

These named codes are documented in the Minecraft Help Center and are separate from Marketplace letter codes.
They appear across sign-in, Realms, and gameplay flows.

| Named code | Typical context                               | Likely cause (Help Center)                      | Quick guidance                                   |
|-----------|-----------------------------------------------|--------------------------------------------------|--------------------------------------------------|
| Armor     | Skin / character creator issues               | Issue with the active skin                       | Switch to a valid skin; retry                     |
| Arrow     | Realms connection                             | Error connecting to a Realm                       | Retry; follow Realms troubleshooting              |
| Bamboo    | Microsoft account sign-in                     | Sign-in issue                                    | Reset sign-in; restart device; retry              |
| Bat       | Multiplayer / server disconnect               | Disconnected from server                         | Reconnect; check network                          |
| Blaze     | Sign-in (often on Realms)                     | Sign-in/session issue                             | Sign out/in; ensure account is stable             |
| Breeze    | World join                                    | World is newer than the client                   | Update Minecraft                                  |
| Cauldron  | Authentication server timeout                 | Auth servers not reachable / timed out           | Retry; check network; restart                     |
| Chest     | Storage                                       | Not enough storage space                         | Free space; retry                                 |
| Creeper   | World join / session                          | Network issue while connecting                   | Retry; general connection troubleshooting          |
| Drowned   | Microsoft account sign-in                     | Sign-in issue associated with Microsoft account  | Reset Microsoft sign-in; retry                     |
| Emerald   | World connection                              | Generic connection failure                        | Retry; general troubleshooting                     |

---

## 3. Marketplace Error Codes (Detailed)

### A-0010
Category: Preview/Beta build gating / environment restriction

Typical player message / situation:
- Preview builds show: “Store is unavailable in Preview.”

What it usually means:
- This build/channel does not have access to the standard Marketplace flow (or requires a Preview-specific flow).

Symptoms:
- Marketplace disabled or unavailable.
- Realms may also behave differently depending on build/channel.

Player guidance:
- Switch to the current public (non-preview) build.
- Fully sign out, restart the game/device, then sign back in.

### A-200
Category: Storefront / entitlement handshake failed *(low-confidence)*

Typical player message / situation:
- Marketplace fails early with `A-200` (often reported on consoles).

What it usually means (field inference):
- The platform storefront token / entitlement handshake did not validate correctly.

Player guidance:
1. Fully quit Minecraft and relaunch.
2. Sign out of the platform account and Microsoft account in-game, then sign back in.
3. Reboot the console/device.
4. If it persists across many users, treat as service-side.

### B-0010
Category: Marketplace service unreachable / timed out

Typical player message / situation:
- “We are still trying to connect. Please wait.”

What it usually means:
- Marketplace data could not be downloaded (timeouts / reachability issues).
- Often transient, but can correlate with ISP routing problems or regional incidents.

Symptoms:
- Marketplace home never loads.
- Dressing Room cannot pull cosmetics.

Player guidance:
1. Fully quit Minecraft and relaunch.
2. Check internet / try a different network.
3. Settings → Storage → Clear Marketplace Cache, then fully restart Minecraft.
4. If persistent on console: reboot device; consider reinstall as a last resort.

### C-0010
Category: Session validation / account state / entitlement fetch failed

Typical player message / situation:
- “We ran into an error. Please restart the game and try again.”

What it usually means:
- The client couldn’t validate session/account state strongly enough to load Marketplace and/or owned cosmetics.

Symptoms:
- Owned skins/Character Creator items won’t load; default skin fallback.
- Marketplace and/or multiplayer may be impacted depending on the underlying failure.

Player guidance:
1. Fully quit Minecraft and relaunch (this is the most consistent first step).
2. Clear Marketplace cache (Settings → Storage → Clear Marketplace Cache), then restart.
3. Sign out of Microsoft account in Minecraft, restart, then sign back in.
4. If persistent, check for broader outages; treat as service-side until proven otherwise.

### C-2010 / C2010
Category: `C-*` family variant

Typical player message / situation:
- Marketplace and skins won’t load; code shown as `C2010` / `C-2010`.

What it usually means:
- Same bucket as `C-0010` in field reports: session/account validation and cosmetics entitlement fetch failures.

Player guidance:
- Same checklist as `C-0010`.

### D-0010
Category: Connectivity failure across online services

Typical player message / situation:
- “We had an issue connecting. Please go back and try again.”
- Often correlates with Dressing Room not loading and other online features failing.

What it usually means:
- The client cannot establish a working connection path for Marketplace/online services (may be outage, routing, NAT, firewall, DNS, or regional filtering).

Symptoms:
- Marketplace fails.
- Dressing Room / Character Creator may fail to load.
- Servers/Realms/invites may also fail for the same user/session.

Player guidance:
1. Try again later (this is often transient).
2. Reboot router + device; try another network/hotspot to isolate ISP/router issues.
3. Clear Marketplace cache; restart Minecraft.
4. Advanced (PC): flush DNS; try different DNS; verify firewall rules for Minecraft.

### E-0010 / e-0010
Category: Ambiguous; likely `D-*` family

Notes:
- `e-0010` is referenced in some reports, but in practice it often overlaps with `D-0010`-style “issue connecting” screens.

Working assumption:
- Treat as a connectivity family code until stronger evidence differentiates it.

Player guidance:
- Same checklist as `D-0010`.

### F-0010
Category: “No internet detected” screen

Typical player message / situation:
- “We can’t detect an internet connection.”
- Often reported on console Marketplace.

What it usually means:
- The game’s network check fails (not necessarily true “no internet”; can be captive portal, DNS issues, Xbox/console network stack edge cases, blocked endpoints).

Player guidance:
1. Verify device network status; resolve captive portal if present.
2. Power-cycle router/modem; reboot console.
3. Try switching network (hotspot) to isolate ISP/router.
4. Fully quit and relaunch Minecraft after network is stable.

### L-404 / L-4xx
Category: Marketplace content fetch failed (service-side OR content availability)

What we see in the wild:
- Can appear while the Marketplace is loading, often with “trouble connecting… try again later.”
- Also appears when a specific Marketplace page or content deep-link cannot be loaded.

What it usually means:
- A Marketplace content request failed. This can be:
  - transient backend/service issues, or
  - the content is not available/usable on this platform/account (platform restrictions, delisting, entitlement mismatch).

Player guidance:
1. Retry later; relaunch once.
2. Sign out/in to refresh account state.
3. If it is tied to a specific pack/skin across many devices, treat as server-side/content-gating.
4. If restrictions are suspected, verify “usable everywhere”/platform compatibility within Marketplace UI.

### SD-002-0
Category: Marketplace backend degraded / maintenance

Typical player message / situation:
- “Hmm, Marketplace is having some issues, please try connecting later.”

What it usually means:
- Marketplace/commerce backend is degraded, under maintenance, or blocking requests.

Player guidance:
- Nothing local to fix. Try again later.

### U-000
Category: Marketplace request failed (generic)

Typical player message / situation:
- “We’re having trouble connecting to the Marketplace right now. Please try again later!”
- Can appear in Marketplace sub-views (e.g., Wishlist) while other areas may partially load.

What it usually means:
- A Marketplace request failed mid-flow (connectivity, backend transient, throttling, or session instability).

Player guidance:
1. Try again later; relaunch.
2. If it persists: reboot network, re-auth Microsoft account, clear Marketplace cache.
3. Treat widespread reports as service-side.

---

## 4. Named Error Codes (Detailed)

These are official, named Bedrock error codes and are not Marketplace-specific.

### Armor
- **Meaning**: Issue with the active skin.
- **Guidance**: Switch to a valid skin in the Dressing Room and retry.

### Arrow
- **Meaning**: Error connecting to a Realm.
- **Guidance**: Retry; follow Realms connection troubleshooting.

### Bamboo
- **Meaning**: Microsoft account sign-in issue.
- **Guidance**: Reset Microsoft account sign-in in Minecraft; restart device; retry.

### Bat
- **Meaning**: Disconnected from a server.
- **Guidance**: Reconnect and validate network stability.

### Blaze
- **Meaning**: Sign-in/session issue.
- **Guidance**: Sign out/in; ensure account is stable; retry.

### Breeze
- **Meaning**: World version is newer than the client.
- **Guidance**: Update Minecraft and retry.

### Cauldron
- **Meaning**: Authentication servers not reachable or timed out.
- **Guidance**: Retry; check network connectivity; restart.

### Chest
- **Meaning**: Not enough storage space to proceed.
- **Guidance**: Free device storage and retry.

### Creeper
- **Meaning**: Network issue while connecting to a world.
- **Guidance**: Retry and follow general network troubleshooting.

### Drowned
- **Meaning**: Microsoft account sign-in issue.
- **Guidance**: Reset Microsoft sign-in in Minecraft and retry.

### Emerald
- **Meaning**: Generic connection error to a world.
- **Guidance**: Retry and use general connection troubleshooting.
---

## 5. PlayFab Backend and Redemption Errors

These official PlayFab Economy v2 / redemption errors are mainly useful for developers, proxy logs,
support correlation, and entitlement debugging. Keep them separate from the player-visible Bedrock
Marketplace and named error codes above.

PlayFab redemption APIs may also pass external store failures through `errorDetails` as `MarketplaceErrorCode`.
When that happens, correlate the API-layer error with the player-visible Marketplace code and platform logs.

| Code name | Numeric code | Typical layer | Meaning | Triage guidance |
|----------|-------------:|---------------|---------|-----------------|
| `AccountNotLinked` | `1014` | Account / entitlement | Platform/store account is not linked to the required entity or account context. | Check account linking, entitlement ownership, and platform account identity. |
| `InvalidXboxLiveToken` | `1188` | Xbox / Microsoft Store redemption | Xbox token is invalid during redemption. | Refresh Xbox/Microsoft auth and retry with a valid token. |
| `XboxXASSExchangeFailure` | `1306` | Xbox auth exchange | Xbox XASS token exchange failed. | Treat as Xbox auth/service integration failure; collect auth traces. |
| `XboxInaccessible` | `1339` | Xbox service reachability | Xbox service could not be reached. | Check Xbox service status, network reachability, and retry later. |
| `InvalidPSNAuthCode` | `1092` | PlayStation redemption | PlayStation auth code is invalid. | Refresh PSN auth and retry with a valid authorization code. |
| `NotAuthorized` | `1089` | PlayStation redemption | Authorization failed in the PlayStation redemption flow. | Verify account permissions, token scope, and entitlement context. |
| `InvalidCatalogItemConfiguration` | `4015` | Catalog / product config | Product or catalog configuration does not match the redemption flow. | Validate catalog item configuration, store mapping, and platform product setup. |

---

## Not Included Yet

The following candidates are intentionally not part of the main catalog yet:
- `L-400` and `L--2122640862`: field evidence is currently too thin or inconsistent.
- Creator / Script API HTTP runtime errors such as `InternalHttpRequestError`, `MalformedHttpRequestError`,
  `HttpRequestBodyTooLargeError`, `HttpRequestLimitExceededError`, `HttpRequestNotAllowedError`, and `HttpsOnlyError`:
  these belong in a separate developer appendix if the project scope expands beyond player-visible Bedrock and Marketplace errors.
