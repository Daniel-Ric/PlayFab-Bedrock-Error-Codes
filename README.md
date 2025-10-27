# PlayFab Economy v2 / Minecraft Bedrock Error Codes

Last updated: October 27, 2025 (Europe/Berlin)
Audience: Minecraft Bedrock client devs, LiveOps, QA, support engineers

## WARNING / Disclaimer
These error codes (for example `B-0010`, `C-0010`, `L-404`, `SD-002-0`) are not fully documented publicly by Mojang / Microsoft.
They surface when Minecraft Bedrock Edition talks to the commerce / Marketplace stack, which is built on top of PlayFab Economy v2
(catalog, entitlements, inventory, etc). Most meanings below are reverse engineered from Mojang bug tracker tickets,
player reports, and observed backend behavior in production-like builds as of October 27, 2025.

Treat this file as:
- a field guide / runbook,
- not an official spec.

---

## Table of Contents

1. High-Level Architecture Context
2. Error Code Families (Summary)
3. Detailed Reference
   - A-0010
   - A-200
   - B-0010
   - C-0010
   - C-2010 / C2010
   - D-0010
   - E-0010
   - F-0010
   - L-404 / L-XXXXXX
   - SD-002-0
   - U-000
4. Recommended Player-Facing Troubleshooting Flow
5. Notes for Developers / LiveOps

---

## 1. High-Level Architecture Context

Minecraft Bedrock Edition uses Microsoft PlayFab services for:
- Marketplace catalog listings (offers, pricing, thumbnails, descriptions),
- entitlements / ownership checks,
- cosmetic assets (character creator items, skins, capes),
- Realms / multiplayer access validation in some flows.

When something goes wrong, Bedrock does not always show raw HTTP status codes or PlayFab error enums like `ServiceUnavailable`.
Instead, the client surfaces short Marketplace-style codes such as `B-0010` or `L-404`.
Internally, these map to things like:
- network / auth / session problems,
- catalog lookup failures,
- downstream service outages.

This README groups those codes by observed behavior so you can:
- understand what actually broke,
- choose whether to retry,
- decide what to tell the player.

---

## 2. Error Code Families (Summary)

| Code / Family     | Player-facing symptom                                              | Root cause class                              | Retry hint                               |
|------------------:|--------------------------------------------------------------------|-----------------------------------------------|------------------------------------------|
| `A-0010`, `A-200` | You cannot connect / cannot use Marketplace in this build         | Version / environment / auth mismatch         | NO RETRY until build or account is fixed |
| `B-0010`          | Marketplace will not load, Still trying to connect                | Service unreachable / timeout / bad cache     | Safe to retry / relaunch                 |
| `C-0010`          | Owned skins / content will not load; stuck as Steve/Alex          | Entitlement / session validation failed       | Retry after cache clear and sign in      |
| `C-2010` / `C2010`| Same area as C-0010 (Marketplace and skins fail to load)          | Same class as above                           | Same as above                            |
| `D-0010`          | Marketplace + Realms + servers are all offline                    | Broad service outage / network block          | Retry later; nothing local helps         |
| `E-0010`          | Marketplace / dressing room broken (observed in bug reports)      | Similar to C-* or D-* cluster                 | Retry later / relaunch                   |
| `F-0010`          | Check your connection even though internet is fine                | Console Marketplace handshake failed          | Usually transient                        |
| `L-404`, `L-*`    | Specific skin / cape / item will not equip                        | Catalog item not found or not accessible      | Will not fix until item exists           |
| `SD-002-0`        | Marketplace is having some issues, please try later               | Commerce backend degraded / maintenance       | Retry later; nothing local               |
| `U-000`           | Marketplace browsing blocked                                      | Marketplace request rejected / throttled      | Retry after reconnect                    |

Legend:
- Safe to retry  = You can automatically backoff and retry the service call.
- NO RETRY       = Retry spam will not help; user or build is in the wrong environment.
- Retry later    = This is usually on Mojang / Microsoft side (outage or maintenance).

---

## 3. Detailed Reference

### A-0010
Category: Version / Auth / Environment mismatch

Typical player message / situation:
- Could not connect, cannot access Marketplace or Realms while on Preview / Insider build.

What it really means:
- The client's PlayFab environment or auth token does not match what the production backend expects
  (for example: you are on a Preview build talking to production).
- Backend rejects the session, so commerce / Realms calls are blocked.

Symptoms you will see:
- Multiplayer / Realms unavailable.
- Marketplace tab disabled entirely.

Developer notes:
- Treat this like InvalidServiceConfiguration or InvalidAuthToken.
- This is not a transient network error; you cannot just retry.

Player-facing guidance:
- Update or switch to the current public (non-preview) build.
- Fully sign out, restart the game, sign back in with the correct Microsoft account.

### A-200
Category: Account / license validation failed

Typical player message / situation:
- Trying to open Marketplace or access purchases on console, immediately blocked with A-200.

What it really means:
- The store session / entitlement handshake between the signed-in Microsoft / console account
  and the Marketplace service failed.

Developer notes:
- Similar class to A-0010: the backend does not accept the entitlement token at all.
- Usually not CDN / timeout.

Player-facing guidance:
- Sign out of the Microsoft / Xbox / PlayStation / Nintendo account,
  reboot the game (or console), then sign back in so a fresh entitlement token is generated.

---

### B-0010
Category: Marketplace service unreachable / timed out

Typical player message / situation:
- "We are still trying to connect. Please wait."
- Error pops when opening the Marketplace, or when the game tries to pull offer lists / thumbnails.

What it really means:
- The client could not download Marketplace data (catalog, featured content, etc.).
- Common on Switch / console when network is flaky or the service is briefly down.
- Can also happen if the local Marketplace cache is corrupted.

Symptoms you will see:
- Marketplace home never loads.
- Dressing Room cannot pull cosmetics.

Developer notes:
- Operationally this aligns with ServiceUnavailable / DownstreamServiceUnavailable in PlayFab terms:
  downstream commerce service did not answer in time.
- Safe to auto-retry with exponential backoff.

Player-facing guidance:
1. Close Minecraft completely and relaunch.
2. Check internet / try a different network.
3. Clear Marketplace (store) cache in Settings -> Storage.
4. In stubborn cases on Switch / console: reinstall Minecraft (worlds are usually preserved).

---

### C-0010
Category: Entitlement / session validation failed

Typical player message / situation:
- You cannot load purchased skins or character creator items.
- You get stuck with default Steve/Alex and cannot access Marketplace or multiplayer.

What it really means:
- The game could not confirm your ownership / profile customization data with PlayFab.
- Often tied to an invalid or expired account session or temporary sign-in problem.

Symptoms you will see:
- Character Creator / Dressing Room stuck on loading forever.
- Multiplayer joins may fail.

Developer notes:
- Functionally similar to B-0010, but narrower: it fails during entitlement / profile fetch
  instead of the global catalog fetch.
- Do not assume cheating; this is usually auth drift.

Player-facing guidance:
1. Check internet / reboot router / console / PC.
2. Settings -> Storage -> Clear Marketplace Cache, then restart the game.
3. Sign out of Microsoft / Xbox Live in Minecraft, then sign back in.
4. If still broken, reinstall Minecraft Bedrock (back up worlds first).

---

### C-2010 / C2010
Category: Same family as C-0010

Typical player message / situation:
- Marketplace and skins both will not load, reported specifically as C2010 / C-2010.

What it really means:
- Same entitlement / session handshake failed, just surfaced under a slightly different numeric suffix.

Developer notes:
- Handle exactly like C-0010.

Player-facing guidance:
- Same as C-0010 (clear cache, re-auth, restart).

---

### D-0010
Category: Broad online services outage / network block

Typical player message / situation:
- "Error code D-0010" and suddenly everything online is dead:
  Marketplace, Realms, joining friends, public servers.

What it really means:
- The client cannot establish a usable online session with Bedrock's online services at all.
- Common during Mojang / Microsoft service outages or regional routing incidents.

Symptoms you will see:
- Marketplace unavailable, Realms unreachable, public servers fail to connect,
  character creator stuck.

Developer notes:
- This usually is not specific to one user. You will see a spike in support tickets / social media
  at the same time.
- Retrying locally rarely helps until backend recovers.

Player-facing guidance:
- There is likely a temporary service outage. You can try rebooting your device and router,
  but most of the time you simply have to wait for the service to come back.
- Optional advanced step for PC: flush DNS / try different DNS / try different network.

---

### E-0010
Category: Marketplace / Dressing Room not loading (observed)

Typical player message / situation:
- Players report E-0010 while Marketplace and character customization fail to load.

What it really means:
- Functionally the same cluster as C-0010 / D-0010:
  the client cannot complete the PlayFab-backed handshake for Marketplace / character data.

Symptoms you will see:
- Dressing Room disabled, Marketplace locked, multiplayer sometimes blocked.

Developer notes:
- Seen in Mojang bug tracker as Marketplace failing across multiple users.
- Treat as transient unless it repeats for only one account or build.

Player-facing guidance:
- Same checklist as C-0010: clear Marketplace cache, restart, re-auth Microsoft account,
  then (if still broken) reinstall.

---

### F-0010
Category: Console Marketplace handshake failed

Typical player message / situation:
- On Xbox (and other consoles) Marketplace shows F-0010 plus "check your connection"
  even though the console reports good network.

What it really means:
- The console client could not complete its Marketplace / commerce handshake.
- Often appears after an update or after purchasing Marketplace content the previous day.

Symptoms you will see:
- Cannot open Marketplace on console.
- All network tests in console settings look fine.

Developer notes:
- Very similar to B-0010 but appears to happen post-update / post-purchase more often on Xbox.
- Safe to retry.

Player-facing guidance:
1. Fully quit Minecraft and relaunch.
2. Reboot the console.
3. Sign out / sign in with the Microsoft account.
4. If still happening, clear local cache or reinstall the game
   (worlds are usually kept on console cloud / local storage).

---

### L-404 / L-XXXXXX
Category: Specific item / cosmetic not found ("Not Found")

Typical player message / situation:
- Trying to equip a purchased cape / skin or a special cosmetic results in L-404.
- Your custom appearance instantly reverts to Steve/Alex.

What it really means:
- The client asked PlayFab / Economy for a specific catalog item (skin, cape, vanity item)
  and got "not found" or "not accessible for this account".
- Sometimes happens right after a game update, when an item ID changed or an item was
  temporarily pulled.

Symptoms you will see:
- Only certain cosmetics break; Marketplace as a whole might still load.

Developer notes:
- This maps nicely to standard HTTP 404 Not Found from catalog lookup.
- Retrying will not help if the item truly does not exist or is not granted.

Player-facing guidance:
- Restart / sign in again once (in case it is just a stale entitlement).
- If it persists, it is probably on Mojang side (item misconfigured or removed).
  You just cannot equip that cosmetic until it is fixed / restored / regranted.

---

### SD-002-0
Category: Marketplace backend degraded / maintenance

Typical player message / situation:
- "Marketplace is having some issues, please try connecting later."
- Code: SD-002-0.

What it really means:
- The Marketplace / commerce backend itself is under load,
  returning errors instead of catalog data.
- Players often report this alongside B-0010.

Symptoms you will see:
- You cannot browse or buy Marketplace content at all.

Developer notes:
- This is almost always server-side. It is not caused by a player's save data.
- Expect volume spikes in support channels.

Player-facing guidance:
- There is nothing to fix locally. Reopen the game later once services stabilize.
- (Advanced: sign out / in and clear Marketplace cache is harmless but usually
  does not solve it.)

---

### U-000
Category: Marketplace request rejected / throttled / mis-communicated

Typical player message / situation:
- U-000 appears while scrolling Marketplace content on Bedrock (mobile / console).
- Player cannot access certain Marketplace pages or DLC listings.

What it really means:
- The client attempted to request Marketplace data and the service refused
  or failed mid-request.
- Reported alongside Marketplace not loading thumbnails, packs, or browsing sections.

Symptoms you will see:
- Marketplace panes fail to populate, but the rest of the game may still run.

Developer notes:
- Usually transient. Can be caused by network instability,
  firewall / parental controls, or temporary backend issues.
- Safe to retry.

Player-facing guidance:
1. Check the network connection / restart router.
2. Sign out and back in to refresh account auth.
3. Relaunch the game.
4. If persistent across many players at once, it is probably a backend issue.

---

## 4. Recommended Player-Facing Troubleshooting Flow

Use this decision tree when writing in-game help text or support macros.

1. Is the code A-* ?
   - Tell the player to update to the public / current build and sign in again.
   - Do not promise a quick retry fix; it is usually an auth / environment mismatch.

2. Is the code L-* (like L-404)?
   - Explain that a specific cosmetic item could not be loaded.
   - Ask them to relaunch once.
   - If still broken, it is on our side (item misconfigured or removed).

3. Is it one of B-0010, C-0010, C-2010, D-0010, E-0010, F-0010, SD-002-0, U-000?
   - These are almost always network / cache / service availability problems.
   - Standard macro:
     1. Fully quit and relaunch Minecraft.
     2. Make sure the device actually has internet (try another app / browser).
     3. In Minecraft -> Settings -> Storage -> Clear Marketplace Cache
        (wording may vary by platform).
     4. Sign out of the Microsoft / Xbox Live account in Minecraft,
        reboot device, sign back in.
     5. If the code is still there and lots of other players report it,
        call it a temporary service outage and ask them to try again later.

4. If Realms / Servers / Friends also fail (often D-0010)
   - Communicate clearly: "Online services are currently unavailable.
     This is not your console or world save."
   - No need to walk them through reinstall unless it is an isolated case.

---

## 5. Notes for Developers / LiveOps

### Mapping to PlayFab-style errors
Internally, these Marketplace codes often wrap lower-level failures such as:
- ServiceUnavailable, DownstreamServiceUnavailable:
  commerce endpoint did not respond in time (often surfaces as B-0010, SD-002-0, F-0010).
- InvalidAuthToken, InvalidServiceConfiguration:
  client token / build environment is wrong (A-0010, A-200).
- NotFound / 404:
  catalog item missing or not granted to this account (L-404).

### Retry policy suggestion
- Transient / backend outage (B-0010, C-0010, C-2010, D-0010, E-0010, F-0010, SD-002-0, U-000):
  - Auto-retry with exponential backoff, give player-friendly status text
    ("Connecting to Marketplace...").
  - After N failures, surface the code so Support can identify patterns.

- Auth / environment (A-0010, A-200):
  - Do not spam-retry. Prompt the player for corrective action (update build / re-auth).

- Item missing (L-404, L-*):
  - Retrying will not resurrect missing content. Log it so content ops / catalog owners
    can fix entitlements or restore the SKU.

### Telemetry / logging
When surfacing these codes to UI, always log:
- code string (for example "B-0010"),
- the PlayFab request path / operation name (for example GetCatalogItems, GetUserInventory),
- HTTP status if available,
- player platform plus game version.

This makes it easier to correlate outages in dashboards and bug trackers.
