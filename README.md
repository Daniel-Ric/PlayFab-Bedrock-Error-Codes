# PlayFab Economy v2 / Minecraft Bedrock Error Codes

Last updated: February 06, 2026 (Europe/Berlin)
Audience: Minecraft Bedrock client devs, LiveOps, QA, support engineers

> **Disclaimer**
> These codes are not fully documented publicly by Mojang / Microsoft. Meanings below are inferred from player reports,
> bug tickets, and observed backend behavior. Use this as a field guide, not an official specification.

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

Minecraft Bedrock Edition uses PlayFab services for Marketplace catalog, pricing, entitlements, inventory,
character cosmetics, and (in some flows) Realms access. When these calls fail, the client often surfaces short
Marketplace-style codes (for example `B-0010` or `L-404`) instead of raw HTTP status codes or PlayFab enums.

This README groups those codes by observed behavior so you can:
- identify the failure class quickly,
- decide whether to retry automatically,
- provide clear player guidance.

---

## 2. Quick Reference

### 2.1 Marketplace code families (letter-number codes)

| Code / Family     | Player-facing symptom                                              | Likely root cause class                        | Retry hint                               |
|------------------:|--------------------------------------------------------------------|-----------------------------------------------|------------------------------------------|
| `A-0010`, `A-200` | Cannot connect or use Marketplace in this build                    | Version / environment / auth mismatch         | **No retry** until build/account is fixed |
| `B-0010`          | Marketplace won’t load, “Still trying to connect”                  | Service unreachable / timeout / cache issues  | Safe to retry / relaunch                 |
| `C-0010`          | Owned skins won’t load; stuck as Steve/Alex                        | Entitlement / session validation failed       | Retry after cache clear + re-auth        |
| `C-2010` / `C2010`| Same area as `C-0010` (Marketplace + skins)                        | Same as `C-0010`                               | Same as `C-0010`                         |
| `D-0010`          | Marketplace + Realms + servers all offline                         | Broad service outage / network block          | Retry later; nothing local helps         |
| `E-0010`          | Marketplace / dressing room broken                                 | Same cluster as `C-*` / `D-*`                  | Retry later / relaunch                   |
| `F-0010`          | “Check your connection” on console Marketplace                     | Console Marketplace handshake failed          | Usually transient                        |
| `L-404`, `L-*`    | Specific skin / item won’t equip                                   | Catalog item missing / inaccessible            | Won’t fix until item exists             |
| `SD-002-0`        | Marketplace is having issues, try later                            | Commerce backend degraded / maintenance        | Retry later; nothing local              |
| `U-000`           | Marketplace browsing blocked                                       | Request rejected / throttled / mid-request fail| Retry after reconnect                    |

Legend:
- **Safe to retry**: Automatic backoff + retry is reasonable.
- **No retry**: Retrying won’t help until the environment/build/account is fixed.
- **Retry later**: Usually Mojang/Microsoft-side outages or maintenance.

### 2.2 Named error codes (Bedrock general)

These named codes are documented in the Minecraft Help Center and are separate from Marketplace letter codes.
They appear across sign-in, Realms, and gameplay flows.

| Named code | Typical context                               | Likely cause                                   | Quick guidance                               |
|-----------|-----------------------------------------------|------------------------------------------------|-----------------------------------------------|
| Armor     | Skin / character creator issues               | Invalid or unsupported skin                    | Re-apply a valid skin                         |
| Arrow     | Realms connection                             | Realm connection error                         | Retry; follow Realms troubleshooting           |
| Bamboo    | Microsoft account sign-in                     | Sign-in failure                                | Reset sign-in and retry                        |
| Bat       | Multiplayer / server disconnect               | Disconnected from server                       | Reconnect and check network                    |
| Blaze     | Sign-in (often on Realms)                     | Auth session conflict / sign-in issue          | Sign out/in; ensure single-device usage        |
| Breeze    | World join                                    | World version newer than client                | Update Minecraft                               |
| Cauldron  | Authentication server timeout                 | Auth service unreachable                       | Retry; check network                           |
| Chest     | Storage                                       | Insufficient device storage                    | Free space and retry                           |
| Creeper   | World join / session                          | Network issue while connecting                 | Retry; check network                           |
| Drowned   | Microsoft account sign-in                     | Sign-in error                                  | Reset Microsoft account sign-in                |
| Emerald   | World connection                              | Generic connection failure                      | Retry; general troubleshooting                 |

---

## 3. Marketplace Error Codes (Detailed)

### A-0010
Category: Version / auth / environment mismatch

Typical player message / situation:
- Cannot access Marketplace or Realms while on Preview / Insider builds.

What it usually means:
- The client’s PlayFab environment or auth token does not match production expectations
  (for example: Preview build talking to production).

Symptoms:
- Multiplayer / Realms unavailable.
- Marketplace tab disabled entirely.

Player guidance:
- Update or switch to the current public (non-preview) build.
- Fully sign out, restart the game, then sign back in with the correct Microsoft account.

### A-200
Category: Account / license validation failed

Typical player message / situation:
- Console Marketplace access blocked immediately with `A-200`.

What it usually means:
- The store session / entitlement handshake between the console account and Marketplace failed.

Player guidance:
- Sign out of the Microsoft / Xbox / PlayStation / Nintendo account,
  reboot the game (or console), then sign back in so a fresh entitlement token is generated.

### B-0010
Category: Marketplace service unreachable / timed out

Typical player message / situation:
- “We are still trying to connect. Please wait.”

What it usually means:
- Marketplace data (catalog, featured content, thumbnails) could not be downloaded.
- Common on Switch / consoles with flaky networks or short outages.
- Can also happen if the local Marketplace cache is corrupted.

Symptoms:
- Marketplace home never loads.
- Dressing Room cannot pull cosmetics.

Player guidance:
1. Close Minecraft completely and relaunch.
2. Check internet / try a different network.
3. Clear Marketplace cache in Settings → Storage.
4. If persistent on Switch / console: reinstall Minecraft (worlds are usually preserved).

### C-0010
Category: Entitlement / session validation failed

Typical player message / situation:
- Owned skins or character creator items won’t load.
- Stuck as Steve/Alex; Marketplace and multiplayer may be blocked.

What it usually means:
- Ownership / profile data could not be confirmed with PlayFab.
- Often tied to an expired or invalid session.

Symptoms:
- Character Creator / Dressing Room stuck on loading.
- Multiplayer joins may fail.

Player guidance:
1. Check internet / reboot router / console / PC.
2. Settings → Storage → Clear Marketplace Cache, then restart.
3. Sign out of Microsoft / Xbox Live in Minecraft, then sign back in.
4. If still broken, reinstall Minecraft Bedrock (back up worlds first).

### C-2010 / C2010
Category: Same family as `C-0010`

Typical player message / situation:
- Marketplace and skins won’t load (reported as `C2010` / `C-2010`).

What it usually means:
- Same entitlement / session handshake failure as `C-0010`.

Player guidance:
- Same checklist as `C-0010` (clear cache, re-auth, restart).

### D-0010
Category: Broad online services outage / network block

Typical player message / situation:
- Everything online is down: Marketplace, Realms, public servers.

What it usually means:
- The client cannot establish a usable online session at all.
- Common during Mojang / Microsoft outages or regional routing incidents.

Symptoms:
- Marketplace unavailable, Realms unreachable, public servers fail to connect,
  character creator stuck.

Player guidance:
- Likely a temporary service outage. Rebooting can help in rare cases, but usually
  you must wait for services to recover.
- Optional advanced step for PC: flush DNS / try different DNS / try a different network.

### E-0010
Category: Marketplace / Dressing Room not loading

Typical player message / situation:
- Marketplace and character customization fail to load, reported as `E-0010`.

What it usually means:
- Same cluster as `C-0010` / `D-0010`: Marketplace / character data handshake failed.

Player guidance:
- Same checklist as `C-0010`: clear cache, restart, re-auth, then reinstall if needed.

### F-0010
Category: Console Marketplace handshake failed

Typical player message / situation:
- Console Marketplace shows `F-0010` plus “check your connection”.

What it usually means:
- The console client could not complete its Marketplace / commerce handshake.
- Often appears after an update or a recent purchase.

Player guidance:
1. Fully quit Minecraft and relaunch.
2. Reboot the console.
3. Sign out / sign in with the Microsoft account.
4. If still happening, clear local cache or reinstall the game.

### L-404 / L-XXXXXX
Category: Specific item / cosmetic not found (“Not Found”)

Typical player message / situation:
- Equipping a purchased cape / skin / cosmetic results in `L-404`.

What it usually means:
- The client requested a specific catalog item and got “not found” or “not accessible”.
- Can happen after updates if an item ID changed or was temporarily pulled.

Symptoms:
- Only certain cosmetics break; Marketplace may still load.

Player guidance:
- Restart and sign in again once (in case of a stale entitlement).
- If it persists, it is likely a backend catalog issue and must be fixed server-side.

### SD-002-0
Category: Marketplace backend degraded / maintenance

Typical player message / situation:
- “Marketplace is having some issues, please try later.”

What it usually means:
- Marketplace / commerce backend is under load or in maintenance.

Symptoms:
- Marketplace browsing and purchases fail broadly.

Player guidance:
- Nothing local to fix. Try again later when services stabilize.

### U-000
Category: Marketplace request rejected / throttled

Typical player message / situation:
- `U-000` appears while browsing Marketplace content.

What it usually means:
- Marketplace data request was rejected or failed mid-request.

Symptoms:
- Marketplace panes fail to populate, but other game features may still work.

Player guidance:
1. Check network connection / restart router.
2. Sign out and back in to refresh auth.
3. Relaunch the game.
4. If persistent across many players, it is likely a backend issue.

---

## 4. Named Error Codes (Detailed)

These are official, named Bedrock error codes and are not Marketplace-specific.

### Armor
- **Meaning**: Issue with the active skin.
- **Guidance**: Switch to a valid skin in the Dressing Room.

### Arrow
- **Meaning**: Error connecting to a Realm.
- **Guidance**: Retry and follow Realms connection troubleshooting.

### Bamboo
- **Meaning**: Sign-in issue.
- **Guidance**: Reset Microsoft account sign-in and retry.

### Bat
- **Meaning**: Disconnected from a server.
- **Guidance**: Reconnect and validate network stability.

### Blaze
- **Meaning**: Sign-in issue (often account used on multiple devices).
- **Guidance**: Sign out everywhere, then sign in on a single device and retry.

### Breeze
- **Meaning**: World version is newer than the client.
- **Guidance**: Update Minecraft and retry.

### Cauldron
- **Meaning**: Authentication server not reachable or timed out.
- **Guidance**: Retry and check network connectivity.

### Chest
- **Meaning**: Not enough storage space to proceed.
- **Guidance**: Free device storage and retry.

### Creeper
- **Meaning**: Network issue while connecting to a world.
- **Guidance**: Retry and check network stability.

### Drowned
- **Meaning**: Microsoft account sign-in failed.
- **Guidance**: Reset Microsoft account sign-in and retry.

### Emerald
- **Meaning**: Generic connection error to a world.
- **Guidance**: Retry and use general connection troubleshooting.
