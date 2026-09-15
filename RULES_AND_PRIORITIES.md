# Rules & Priorities — the canonical specification

**How the rule system works: where rules come from, what priority each tier carries, why the numbers are what they are, and how the `script` flag changes what is served.**

This is the **baseline specification**. It describes the shared model — the rule tiers, the priority ladder, the `script` flag lifecycle and the guarantees that must hold — independently of any particular product or build. It states what the system *is*, not what any given implementation currently does.

> Per-product conformance and deviations live in this document's own **§8 per-extension table** (and the matching per-doc tables across the folder). The fraud/VM/behavioral detection layer is documented in `FRAUD_DETECTION_V5.md`.
>
> The **client's** obligations toward this rule system — how the extension applies each tier into its own rule-ID ranges — are in `EXTENSION_COMPOSITION.md` §8. The identity mechanism that transports the `script` flag is in `USER_CREATION_AND_UPDATE.md` §3. Start from `00_INDEX.md` for the whole folder.

---

## 1. In plain English

The whole system rests on four ideas:

1. **A stack of rules decides, for each network request, whether it loads, is blocked, is redirected to a stub, or has its headers rewritten.** These are Chrome Declarative Net Request (DNR) rules — the browser evaluates them natively; the extension never sees the traffic.

2. **When two rules disagree about the same request, priority breaks the tie. Higher number wins.** This is the whole game. A "whitelist" is nothing more than an *allow* rule with a priority higher than the *block* rule it must beat. "Pausing" a site is a high-priority `allowAllRequests` rule laid over the page. Understanding the priority ladder IS understanding the product.

3. **Rules come from three places, kept in strictly separated ID ranges so they never overwrite each other:** the static lists packaged in the extension, the small set of "inline" rules the backend sends on every sync, and the large compiled blocklist the extension lazily downloads. On top sit the user's own whitelist/blocklist and per-tab pauses.

4. **A server-side switch called `script` decides whether the product is a plain ad-blocker or also runs a search-monetization layer** (redirecting search traffic through partner URLs, stripping CSP so injected scripts run, and re-allowing the trackers/ads that pay). `script` is a database column, never a URL parameter. When ON, a bundle of high-priority rules is *added*; the ad-blocking floor is *always* present either way.

If you remember one sentence: **priority decides who wins; the `script` flag decides whether the monetization bundle is added on top of an always-present ad-blocking floor.**

---

## 2. The rules of the game — Chrome DNR precedence

Everything later in this document is an application of these five facts:

1. **Higher `priority` integer wins.** If `priority` is omitted, Chrome defaults it to **1**.

2. **At equal priority, the action type breaks the tie, in this fixed order:**
   `allow` > `allowAllRequests` > `block` > `upgradeScheme` > `redirect`.
   `modifyHeaders` is evaluated separately and is skipped entirely if a matching `allow`/`allowAllRequests` of equal-or-higher priority exists.

3. **`allowAllRequests` on a matched `main_frame` (or `sub_frame`) exempts the *entire* page** — every sub-resource is waved through unless a higher-priority block matches that specific sub-resource. This is the mechanism behind "pause on this site" and the system allowlist.

4. **Rules compete across all sources at once** — static packaged, dynamic, and session rules are pooled and compared by priority. When priority *and* action tie across sources, resolution order is **session > dynamic > static**.

5. **`block` cancels a request; `redirect` swaps its URL** (to a neutered stub or a "blocked" landing page). Per (2), at equal priority a `block` beats a `redirect`.

**Design rule that follows from (2):** never rely on equal-priority action ordering to express intent. If rule A must beat rule B, give A a higher number. Ties work, but the intent is invisible in the code and breaks silently the moment someone adds a third rule at the same tier.

**Design rule about redirects:** a `redirect` is only as strong as its target. If the surrogate it points at is not packaged and declared web-accessible, the redirect still *wins* at the DNR layer but resolves to a failed load — behaving as a hard block rather than the working shim the rule intended. When a redirect tier exists to prevent site breakage, the target must actually ship.

---

## 3. Where rules come from

### 3.1 The three tiers + the user overlay

Rules are partitioned into ID ranges so a per-user sync never clobbers the multi-megabyte compiled blocklist, and vice-versa:

| Tier | ID range | Lives where | Replaced by | Purpose |
| :-- | :-- | :-- | :-- | :-- |
| **Static packaged** | per-ruleset namespaces | JSON files in the extension, declared in `manifest.json` | never (ships with the build) | Baseline blocking that works before any network call and even if the backend is down — **including the vendor self-allow @`99999`** (`EXTENSION_COMPOSITION.md` §8.5), which must survive a dead backend and so is packaged, not sent per-sync |
| **Inline dynamic** | **< 11000** | authored in the backend endpoint, sent in the `rules` key every sync | client removes/re-adds only this range | Per-user rules: page exemptions, default block, allowlist chunks, and (when `script=1`) the injection bundle |
| **Compiled bulk** | **≥ 11000** | a generated cache file, served by a dedicated endpoint, lazily fetched | client removes/re-adds only this range, only when the version hash changes | The large ad/tracker blocklist compiled from upstream sources |
| **User + session overlay** | reserved IDs | created by the extension's background worker | on every list mutation / tab event | The user's own whitelist & blocklist, and per-tab "pause" |

**Compiled-bulk ID sub-ranges:** redirect `11000+`, block `21000+`, modifyHeaders `31000+`, allow `41000+`, allowAllRequests `51000+`, unknown-type fallback `91000+`.

**MV3 budget note.** The compiled bulk (and the inline tier) are applied as **dynamic** rules, so they draw on Chrome's dynamic-rule quotas: `block`/`allow` are "safe" (up to 30,000), but `redirect` and `modifyHeaders` are "unsafe" and share a single **5,000** cap across *all* dynamic rules (inline + compiled). The `main_frame`→interstitial redirects plus the monetization redirects and CSP-strip `modifyHeaders` must all fit in that 5,000 — size the redirect set accordingly. The static packaged rulesets draw on the separate static budget (30,000 enabled). Full table and implications: `DATA_SOURCES.md` §4.1.1.

**The ID-range split is a hard invariant.** Each apply-function removes only its own range — the inline applier touches `< 11000`, the compiled applier touches `≥ 11000`, the user applier touches only its reserved IDs. This is what lets a routine sync run without re-downloading or disturbing the bulk blocklist, and what lets a degraded response deliberately omit `rules` without wiping a user's already-applied inline rules.

**The compiled blocklist is never gated by `script`.** It is delivered through its own URL and is identical whether monetization is on or off. Only the *inline* tier changes with `script`.

### 3.2 How the compiled blocklist is produced

**Since 2026-09 the bulk tier is produced by ONE shared factory** — the
`blocklist-v4` repo ("Automated All In One List V2", fully documented in
`BLOCKLIST_WHITELIST_SYSTEM.md`) — and each backend's generator merely MIRRORS its
finished `dist/network/rules.json`, rebranding only the redirect target. The
artifact still contains the same three rule populations this section always
described (the tier contract is unchanged on purpose — same ID bands, same
priorities, same shapes):

- **Curated scam/navigation destinations** (the `popup` sheet + KADhosts) →
  `main_frame` **redirect** rules pointing at the extension's own blocked-landing page.
- **Auto-collected ad/tracker domains** (public hosts lists, DNS-verified) →
  **block** rules for sub-resources plus `main_frame` **redirect** twins. The old
  "most-recent-slice" cap is gone: retention is now policy-based (a domain ships
  while it still resolves and is still remembered — ledger, 12-month bound).
- **A surgical rule set** (the EasyList family, curated) kept as authored, with
  `block` at the lowest priority and domain-level `main_frame`-only blocks
  duplicated as redirect rules one tier up — plus allow lanes: the EasyList
  exceptions at priority 2, and the download-sites lane at priority **50**
  (2026-09-11, was 2 — above blocks AND redirects, still under the client
  user-block tier at 100).

Mirroring is **fail-safe**: if the artifact fetch fails, the shim serves its
last-good local mirror and never rewrites the cache or bumps the version, so a bad
refresh keeps the last-good rules rather than shipping an empty set.

---

## 4. The priority ladder

### 4.1 What the ladder guarantees

The numbers are not arbitrary. Every tier is chosen to clear the tier below it with room to spare, in order to guarantee four things:

1. **The backend can never be blocked by its own product.** The vendor domain is allow-listed at the maximum priority (`99999`) as both `requestDomains` and `initiatorDomains`, so sync and asset fetches can never be self-blocked — not even by a user who blocklists the vendor domain. Without this, one bad blocklist entry could permanently sever the update channel.

2. **When monetization is on, the money-making traffic always wins.** Injection redirects (`10000`/`10001`) and CSP strips (`9999`) sit *above* the system allowlist (`9990`); the tracker/ad allows (`5000`) sit *below* the `9990` allowlist but far above the blocklist — on the allow-listed search pages the `9990` `allowAllRequests` already waves that traffic through, so the search flow and its pixels fire regardless of what the blocklist says.

3. **An allow-listed or paused site is fully exempt.** `allowAllRequests` on the main frame out-ranks the blocklist, so the whole page loads untouched.

4. **Everything else is blocked.** The blocklist sits at the bottom (`1`/`3`), which is exactly right: it only wins when nothing above it has claimed the request.

### 4.2 The canonical ladder

Read top-down as "what beats what." **Presence** says whether the tier is part of the permanent floor or appears only when `script = 1`.

| Priority | Action | Rule ID(s) | What it does | Why this number | Presence |
| ---: | :-- | :-- | :-- | :-- | :-- |
| **99999** | allow | vendor self-allow (2 rules: by request + by initiator) | Exempts the backend domain from every rule | Must be unbeatable — above every other tier including the user's own blocklist | **always** |
| **10001** | redirect | `8014`, `8015`, `8017` | Bot-worker redirects for the search flow | Highest operational tier — must outrank even the system allowlist | `script=1` |
| **10000** | redirect | `8019`, `8020` | Swap analytics scripts | Just below the bot workers, still above the allowlist | `script=1` |
| **9999** | modifyHeaders | `33`, `31`, `8018` | Strip CSP / X-Frame-Options; set CORS | Must beat the allowlist so injected scripts can run on allow-listed search pages | `script=1` |
| **9998** | redirect | `8011` | Search query → partner URL. **The actual monetization redirect** | Above the allowlist, below the workers that must intercept it first | `script=1` |
| **9990** | allowAllRequests + allow | `8400 + i*2 (+1)` | System allowlist, chunked. **Two rules per chunk**: `allowAllRequests` by request domain (exempts the document and its whole frame tree) + `allow` by initiator domain (catches what the first misses — worker fetches). Together they mean "do not break the site the user is visiting" | High enough to beat the whole blocklist, low enough that the injection layer overrides it | **always** |
| **9990** | allow | `8500 + i` | Third allowlist band, chunked: `allow` by **request** domain over all resource types — "allow this host as a third-party resource on *every* page, whoever the visitor is". A strictly wider grant than the pair above, which is why it is scoped to the monetized surface and built from the injection-time allowlist only | Same tier as the pair — it is the same allowlist, asked a different question | `script=1`⁶ |
| **5000** | allow | `5005`, `5006` | Re-allow trackers (`5005`) and ad domains (`5006`, gated by initiator) | Must beat the blocklist so pixels/ads survive when monetizing | `script=1` |
| *(user tier)* | allowAllRequests / block | *implementation-defined* | User pause / whitelist / custom block | Must sit **above the blocklist** and **below the injection layer** — the exact number is an implementation choice | user action |
| **1000** | allow / block | `45` (allow), `8013` (block) | Search-result allow and consent-script block | Equal-priority pair; the two are kept apart by disjoint resource types rather than by priority | `script=1` |
| **6** | allowAllRequests | `8002` | Whole-page exemption for sites that must never break | Only needs to clear the blocklist (1–3), deliberately below every allowlist tier | **always** |
| **3** | block | `2002` | Default block over the canonical ad/analytics host list | One notch above the bulk block so it applies even where bulk rules don't | **always** |
| **3** | redirect | compiled bulk redirect | `main_frame` navigation → blocked landing page | Ties `2002`@3; at equal priority block beats redirect | **always** |
| **2** | allow | compiled allow exceptions | Un-blocks first-party resources inside the bulk list | Above bulk block, below everything else | **always** |
| **1** | block | compiled bulk block (`21000+`), static packaged blocks | The main blocklist body | Weakest on purpose — it only wins when nothing above claims the request | **always** |

**Two structural notes:**

- **The user tier has no canonical number, but it has a safe range and a common value.** Its placement is an implementation choice, but the constraint is invariant: it must beat the blocklist so a pause actually works, **and stay *under* the injection layer** (`5000`+) so a user cannot allow-list away the monetization surface. A concrete safe value the siblings use is **`200` for the user allow and `100` for the user block** — comfortably above the block/redirect tiers of the canonical ladder and far below the injection band (`5000`+). **Do not pick a very large number** (e.g. `2000000`): anything above the injection layer inverts the invariant — the user could then whitelist away the monetized search surface, and a monetization redirect could be overridden by a stray user rule. Higher-is-safer is the wrong instinct here; the ceiling is `4999`, not infinity.
- **`2002` always carries the system allowlist** in its `excludedRequestDomains` / `excludedInitiatorDomains`. It is a blunt block over ad hosts, made safe by exclusions rather than by priority.

---

## 5. The `script` flag

### 5.1 States

`script` is a per-user integer stored in a database column and re-derived on every sync. It is **never** an HTTP query parameter. **How it reaches the client depends on the identity variant** (`USER_CREATION_AND_UPDATE.md` §3): under **V1** it rides **inside** the encrypted per-user handle (`…&s={script}&…`): the server re-issues the handle every sync and the client echoes that opaque blob back — there is **no** separate plain `script` key on the wire. Under **V2** the handle is an opaque pointer and `script` is read from the row and returned as a **plain response key**. Either way the extension stores and echoes it but **never interprets it**; every behavioral difference is decided server-side and delivered as data.

| Value | Meaning | What the extension must look like |
| :-- | :-- | :-- |
| **0** | **Default.** Not enabled. Plain ad-blocker. | The default at record creation. The always-present floor (§5.3) and nothing else. |
| **1** | **Enabled.** Ad-blocker **plus** the search-monetization injection bundle. | The floor (§5.3) **plus** everything in §5.4. |
| **2** | **Opt-out — Acceptable Ads disabled** *(V2 only)*. The user explicitly opted out. | **Byte-for-byte the same as state 0.** Everything state 1 added or changed must be actively undone — see §5.2. |

*State `2` exists only in V2. V1's handle is client-rewindable and cannot hold a durable opt-out, so the three V1 builds run `script` 0/1 only (§8 note ¹, `USER_CREATION_AND_UPDATE.md` §11). The rest of §5.2 therefore describes a **V2-only** path.*

### 5.2 State 2 — reversion must be explicit, not implied

State 2 is not a third behavior. It is **state 0, restored by force**. Two properties define it, and the second is the one that is easy to get wrong.

**(a) The target state is exactly state 0.** Because the user explicitly opted out, their extension must end up indistinguishable from a fresh `script = 0` install — *regardless of whether they would otherwise pass the enable gate*. Passing the gate is irrelevant once the user has opted out; the gate must not be able to pull them back (see §5.5).

**(b) Reversion must be *sent*, never expressed by omission.** The client merges the sync response **key by key**: a key that is absent from the response leaves the client's previously stored value **untouched**. A user who was previously at `script = 1` already holds the injected rules, style payloads and expanded lists in local storage. If the server merely *stops emitting* those keys, nothing overwrites them — the client keeps applying the old injected data indefinitely and the opt-out silently fails.

Therefore **every field that state 1 can add or modify must appear in the state-2 response carrying its default value.** By category:

| What state 1 did | What state 2 must send |
| :-- | :-- |
| Added inline rule IDs to the `rules` array | The `rules` array **with those IDs filtered out** — the client replaces the whole inline range, so the omission propagates as a removal *within a key that is still sent* |
| Populated a style / CSS payload | The **same key**, explicitly reset to its default object — or emptied (`[]`, `{}`, or `" "` as that field's type requires) |
| Populated an exclusion or suppression list | The **same key**, explicitly set to an empty array |
| Expanded the published allowlist | The **same key**, explicitly rebuilt from the default set only, with the injection-support domains dropped |
| Set any scalar or string field | The **same key**, explicitly set to its default or empty value — **not dropped** |

**Rule of thumb: if state 1 can write it, state 2 must overwrite it.**

The distinction matters because the two failure modes look identical in a fresh-install test and only diverge for a user who was previously monetized:

- A **new** user at state 2 has nothing stored, so omission and explicit-reset both look correct.
- An **existing** user transitioning `1 → 2` has stale data stored, and only the explicit reset actually clears it.

Any change to the injection bundle must therefore be made in two places: the code that *adds* the field in state 1, and the code that *resets* it in state 2. A field added to one without the other becomes permanently un-revertible for already-monetized users.

### 5.3 The invariant floor — what is ALWAYS present, regardless of `script`

The injection bundle is **purely additive**. Whether `script` is 0, 1 or 2, these are **always present** (packaged static and/or delivered in the response):

- **The vendor self-allow rules at `99999`** — packaged **static** in every build (so present even with a dead backend); **North** is the one build that instead delivers them in the per-sync response (§3.1).
- **The page exemption `8002` @6** (`allowAllRequests`) for the sites that must never break.
- **The default block `2002` @3**, always carrying the system allowlist in its excluded-domain lists.
- **The system allowlist and its `8400 + i*2` chunk pairs @9990**, always filtered through the protected-domain carve-out. The third band at `8500 + i` is **not** part of this floor — see §5.4.
- **The full compiled bulk blocklist**, delivered via its own URL and never `script`-gated.
- **The static packaged rulesets** — no reachable code path disables them.
- **The user's own whitelist / blocklist / pause overlay.**
- **The uninstall URL registration.**

Turning `script` on never *removes* any of this. It only adds a layer at priorities `5000` and above that wins over the blocklist for the specific search and tracker traffic it targets.

### 5.4 What `script = 1` adds

When (and only when) monetization is enabled, the injection routine adds to the response:

- **Inline DNR rules** — `33`, `31`, `8018` (CSP/X-Frame strip and CORS, @9999); `8011` (search → partner URL, @9998); `8014`/`8015`/`8017` (bot-worker redirects, @10001); `8019`/`8020` (analytics script swap, @10000); `5005`/`5006` (tracker and ad allows, @5000); `45` (search-result allow) / `8013` (consent-script block), @1000.
- **Search-results styling** that hides ads on search result pages and briefly cloaks results.
- **An expanded allowlist** — the injection-support domains (search TLDs and partner feeds) merged into the published allowlist **and** into the scriptlet-exclude list, so no cosmetic filtering or scriptlets run on the monetized search properties, which must stay pristine for the injection to work.
- **The third allowlist band `8500 + i` @9990** (`allow` · `requestDomains` · all resource types), built from that expanded allowlist. It is what covers endpoints consumed as **third-party resources and never visited** — the measurement/verification XHRs the search flow depends on, which the `8400` pair structurally cannot reach (one is document-scoped, the other initiator-scoped). Because it is built at injection time, it never carries the community-voted whitelist.

Everything in this list is what §5.2 must be able to undo.

> **The trap that co-locates §5.4 with §13.4/§13.5 — read both together.** The two
> bullets above pull in opposite directions on the same host: the **search-results
> styling** must fire on the search page, while the **expanded allowlist** is the
> very thing the client uses to switch cosmetics *off*. If the client gates
> zero-flicker on the raw allowlist (as it gates ad-block cosmetics), the allowlist
> entry the injection just added will **suppress the pre-hide the injection needs** —
> the search results flash before they are masked. The resolution is the
> **protected-domain carve-out (§6, `EXTENSION_COMPOSITION.md` §13.5)**: the search
> surfaces are *force-removed* from every allowlist, so there is no allowlist entry
> to suppress the flicker, and it runs. Two rules follow: **(a)** never gate
> zero-flicker on the injection allowlist — gate it only on the *user's own*
> pause/whitelist, which the carve-out guarantees is empty for these hosts; **(b)** a
> backend that **re-adds** the search hosts to the published allowlist *after* the
> carve-out (e.g. an "injection-enabled whitelist" merged unfiltered) reintroduces
> exactly this bug — the carve-out and the re-add must not both run. This is the
> single most time-expensive defect observed on a legacy port.

### 5.5 The enable gate

The promotion path `0 → 1` is a graduation gate: a user must look like a real, aged, active human before monetization is switched on.

```
level == 0                            AND   # no fraud/risk digits
script == 0                           AND   # not already enabled, and never re-promotes a 2
(now - creationdate) > threshold      AND   # account older than roughly a week
updates > threshold                   AND   # has synced enough times
(search-engine visit count) > threshold AND # shows real search activity
typetag has no "-xx-"                 AND   # came from an approved acquisition source
count(onInstallTabDomains) > 0              # had real tabs open at install
```

Plus outer gates: the build must be store-approved, the extension-id prefix must match the approved value, and the market must be on the approved list. Only if all pass does the server set `script = 1` and record the enablement.

**In the canonical V1 design this gate is inert** — commented out of the live sync path, so `script` never transitions on sync and the `enabled` stamp is never written; the canonical V1 route to `script = 1` is the creation-time echoed flag (`USER_CREATION_AND_UPDATE.md` §5.4 step 7, §5.5 step 2). The conditions above are still the canonical gate — V2 runs them live on every sync; V1 keeps them dormant. A specific build may nonetheless **dev-force** `script = 1` (a hardcoded manual override the owner controls at will — not the organic gate); treat that as an owner business toggle, and verify the **prod** file before assuming enablement is organic (§8).

**The `script == 0` clause is what makes opt-out durable.** Because the gate requires the flag to be exactly `0`, a user at `2` can never satisfy it, no matter how well they score on every other criterion. Any variant that also accepts `2` breaks the opt-out guarantee and must not be used.

Thereafter the injection itself fires on `level == 0 && script == 1`. **Any non-zero risk `level` silently suppresses injection while leaving ad-blocking fully intact** — a user flagged as risky keeps a working ad-blocker and simply never sees the monetization layer.

---

## 6. Operational guarantees

Guarantees preserved on every path, including the degraded database-down path. These are *not* priority rules — the rule-level guarantees are the ladder (§4.2) and the floor (§5.3). What follows is only what those do not express:

1. **Protected-domain carve-out.** The search TLDs and their asset hosts are **force-removed from every allowlist**, even if a user explicitly whitelists them. This is what keeps the monetization layer controllable — a user cannot accidentally allow-list away the search surface the injection depends on.

2. **Version pin.** The response carries a version constant that MUST be echoed identically on every path *including the degraded path*. A mismatch triggers the client's storage purge and would wipe local state fleet-wide.

3. **Identity survival.** The identity handle — which under V1 *carries* `script`/`level`/attribution and under V2 *points at* the row that holds them — is mirrored to sync storage and healed back if local storage is wiped, so a purge or a fresh machine never loses the state. This is also what makes an opt-out (state 2) survive a storage wipe. (The handle's form is the identity variant of `USER_CREATION_AND_UPDATE.md` §3; the client's mirror/heal duty is `EXTENSION_COMPOSITION.md` §4, §6.)

4. **Degraded-mode safety.** A database failure or uncaught error returns HTTP 200 with the client's own identity echoed, file-served asset versions, the baseline rules for new installs, and a jittered backoff so the fleet backs off instead of retry-storming. For *existing* installs the degraded response deliberately omits `rules`, relying on the merge semantics of §5.2(b) to leave already-applied inline rules in place.

5. **Generation is fail-safe.** See §3.2 — a failed upstream fetch keeps the last-good compiled cache rather than shipping a degraded set.

6. **Uninstall URL always registered**, refreshed after every sync, and fail-safe — it redirects to the feedback destination even on database failure.

---

## 7. Sync lifecycle in brief

1. **Install.** Static rulesets are active immediately (they ship with the build). The extension seeds install telemetry and posts its first sync.
2. **Record creation.** With no identity handle present the backend creates the user record — dedup, risk evaluation, `script = 0` — and returns the identity handle (V1: an encrypted UID blob; V2: an opaque token — and V2 may instead **`WAIT`**, returning a null handle and creating nothing until a sync carries hardware). The handle's form is the identity variant of `USER_CREATION_AND_UPDATE.md` §3.
3. **Response assembly.** Base inline rules → optional injection bundle if `script = 1`, or the explicit reset payload if `script = 2` → unconditional merge of the default block and the allowlist chunks → version/URL pointers for the lazily-fetched assets.
4. **Client application.** Inline rules (`< 11000`) applied via `updateDynamicRules`; compiled rules (`≥ 11000`) re-fetched only when the version hash changed; cosmetic caches likewise; allowlist rebuilt; uninstall URL refreshed.
5. **Update cycle.** On a server-controlled interval (a few hours). Version hashes mean the client only re-downloads assets that actually changed.
6. **Uninstall.** The registered uninstall URL fires a churn ping and marks the record disabled.

---

## 8. Per-extension implementation

A summary of how each shipped build realizes the rule model — this §8 table is the
per-extension treatment; the cells below are the at-a-glance map.

**Legend:** ✅ matches the baseline · ⚠️ works but deviates · ❌ absent · `—` not applicable / not present for this build · `n/a` not applicable to this generation.

| Aspect | § | 12 Pro | 21 North | 22 Hunter | 23 Wonder | 25 Ninja | 26 Ghost | 27 |
| :-- | :-- | :-- | :-- | :-- | :-- | :-- | :-- | :-- |
| Static-block priority scale | 4 | dense-rank ≤199 ⚠️ (+ opt-in `youtube` @160⁵) | flat @1 | mostly @1 | mixed (1/2/3/10/12) | uBO scale (10/11–19/30/40/46) | uBO scale | uBO scale |
| User-intent rule model | 3.1 | reserved IDs, allow>block ✅ | folded into server rules ⚠️ | client rules engine ⚠️ | dedicated @4000 + per-tab pause ⚠️ | reserved IDs, allow>block ✅ | reserved IDs ✅ | reserved IDs ✅ |
| Inline↔compiled ID split | 3.1 | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| Vendor self-allow (max prio) | 4.2 | ✅ | ✅ | ✅ | ✅ | ✅ | ⚠️ fork leftover | ✅ |
| No equal-priority ties for intent | 2 | ✅ | ✅⁴ | ⚠️ | ❌ 770000 vs 970000 @4000 | ✅ | ✅ | ✅ |
| Redirect targets actually ship | 2 | ✅ | ⚠️ (server-hosted) | ✅ | ❌ | ❌ | ✅ | ✅ |
| `script` states | 5 | 0/1² | 0/1¹ | 0/1¹ | 0/1¹ | 0/1/2 | 0/1/2 | 0/1/2 |
| Organic enable-gate (`script==0`) | 5.5 | dormant | dormant | dormant | dormant | live | live | n/a |
| User-tier priority value | 4.2 | 200/100 ✅ | — | — | 4000 ⚠️ | 200/100 ✅ | 200/100 ✅ | 200/100 ✅ |
| Protected-domain carve-out | 6 | ⚠️ re-add bug³ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| System-allowlist band shape | 4.2 | pairs + `8500+i` in-gate ✅ | pairs + `8500+i` in-gate ✅ | trio `8400+i*3` ⚠️ | trio `8400+i*3` ⚠️ | pairs + `8500+i` in-gate ✅ | pairs + `8500+i` in-gate ✅ | — |
| Community whitelist as a source | 3.2 | ✅ | ✅⁷ | ❌ | ❌ | ✅ | ✅ | — |

> **Monetization on/off is an owner-controlled business toggle** — flipped at will, without
> notice — so a per-build "live / dormant" state is deliberately **not** tracked here. Every build
> is monetization-**capable**; its **organic enable-gate** is either **live** (V2 runs it every
> sync) or **dormant** (V1/Gen 0 — commented out, so enablement comes via a creation-time echo or a
> manual dev-force override). Which builds are *on right now* is business logic, not documented state.

**Notes.** ¹ **State `2` (the Acceptable-Ads opt-out) is V2-only** — it needs
server-authoritative state to hold a durable opt-out, which V1's client-rewindable
handle cannot. The three V1 builds (North/Hunter/Wonder) run `script` 0/1 only and
have no state-2 reset path (§5.1, `USER_CREATION_AND_UPDATE.md` §11, `BACKEND_ARCHITECTURE.md` §8.2). their organic enable-gate is commented out, so any live monetization is a **manual
dev-force override** — an owner-controlled toggle, flipped at will; verify the **prod**
file before assuming enablement is organic, and do not treat any build's on/off as fixed
documented state. Wonder's `770000` (custom block) and
`970000` (aggregated allow) both sit at priority `4000`: the one outright
equal-priority-tie defect (§2). The historical Ninja/Wonder 50k long-list cap is
retired since the 2026-09 v4 shims — retention is ledger-based now (§3.2).

² **12 is `Gen 0` (Legacy, pre-family)** — its AES-encrypted `syncuid` handle is
client-rewindable like a V1 blob, so it runs `script` 0/1 with **no durable state-2
opt-out**. Its *frontend* was re-platformed onto the current baseline (reserved-ID
band model, structured zero-flicker, action-driven sync, `200`/`100`
user tier) in the 2026-08 modernization pass. Since 2026-09-15 the reserved band is
`1/2/3` dynamic + `21/22` session: id `3` is a persistent client-side **system allow**
(`allowAllRequests` @`200` over the server-delivered `allowlist` hosts, main+sub
frame) — the twin of Ninja/Ghost's `applyDefaultWhitelist` (same id, same priority),
rebuilt after each sync and at install/update, non-fatal by contract, and still
outside the server band so the wholesale band purges never touch it; its *identity/DB/backend* stay Gen 0
by definition (`00_INDEX.md` §5; `USER_CREATION_AND_UPDATE.md` §11).
³ 12's backend carries the protected-domain carve-out (`isProtectedDomain`) **but a
separate "injection-enabled whitelist" re-adds the search hosts to `allowlist`
*after* the filter** (§5.4 trap). The client currently works around it by running
zero-flicker unconditionally on those protected surfaces (safe today because
`proFlicker` only targets `google.*`); the cleaner fix is to stop the backend
re-add so the carve-out holds. Track to close. A second `isProtectedDomain` defect,
diagnosed 2026-09: its Yahoo regex (`^(www\.|search\.|r\.search\.)?yahoo\.com$`)
matches `search.yahoo.com` but not the regional subdomains, so the server band's
allow-pair builder ejected exactly the US host — `fr.search.yahoo.com` etc. kept
frame cover @9990 while the US SERP had none, and bulk blocks on
`beacon.search.yahoo.com`/`udc.yahoo.com` fired there user-visibly. Ninja/Ghost
never showed the gap because their clients rebuild a system allow from the server
`allowlist` key (which their backends merge unfiltered); 12's client formerly built
nothing from that key. Closed client-side by 12's id-`3` system allow (note ²); the
backend regex — and its now-stale comments saying 12's client ignores `allowlist`
for network rules — are unchanged.
⁴ North's `45`/`8013` is the **prescribed** equal-priority pair @1000 (search-result allow vs
consent-script block), kept apart by **disjoint resource types** — not a tie defect (§2). The one
real equal-priority-tie defect in the fleet is Wonder's `770000`/`970000` @4000.
⁵ 12's fourth packaged ruleset — `youtube` (2026-09-14): 8 block rules @160 backing the
client's opt-in "YouTube mode" (`proYtSkip`). It is the fleet's one static ruleset declared
`"enabled": false` in the manifest and toggled at runtime (`sw.js` reconciles it against
`getEnabledRulesets` on every SW wake and on every flag change) — a deliberate, feature-gated
deviation from the ships-enabled static floor (§5.3, §9.1 item 1), which 12's three baseline
rulesets (`default`/`badware`/`privacy`) still honor. At 160 the blocks sit under the `200`
user allow, so a user pause/allow of youtube still overrides the mode — the user-tier
invariant of §4.2 holds.
⁶ The `8500 + i` band exists only where the fleet whitelist port landed (2026-09-14). Before it,
those builds emitted a **trio** per chunk — the third rule was the same allow-by-request-domain,
but emitted to **every** user and over the whole system allowlist. It was narrowed, not added:
granting "allow as a third-party resource everywhere" to a vote-derived community list is the
wrong blast radius, so the grant now follows the monetized surface instead of the whitelist.
Builds still on the trio (22, 23) carry the wider grant.
⁷ 21's community list reaches the response key and the `8400` pair, never the `8500` band or rule
`2002`'s excluded lists. 21 also keeps the **user's own** whitelist in that key on purpose — a
declared divergence from the other ported builds, because 21's client is the only one that does
not build its own user-allow rule and never applies the user whitelist to the compiled bulk
range, so the `8400` pair is the only thing that lifts the blocklist off a user-whitelisted site.

---

## 9. Conformance checklist

Run the **Common** items for every build, then add the block for the build's
identity generation (§8, `USER_CREATION_AND_UPDATE.md` §11), applying the reachable-set filter
(`00_GOAL.md` §3, decision B). The rule ladder and floor are **identical across generations —
monetization included**; only the `script` **enable mechanism** differs by generation.

> **Gen 0 (Legacy, pre-family — e.g. 12).** Runs all Common items (the ladder and floor are
> universal). For the `script` lifecycle it is closest to **V1** (`script` 0/1, **no durable
> state `2`**), but its enable **mechanism** differs — monetization is currently **dev-forced**,
> not the §9.2 "inert/commented" gate, so that specific item is `n/a`. It never runs the §9.3 V2
> block. Monetization itself is a Common goal, never treated as absent for being Gen 0.

### 9.1 Common — every build

- [ ] Static rulesets ship **enabled by default**; the blocking floor works with no network and survives a dead backend — §5.3.
- [ ] Rules are partitioned into strictly separated ID ranges (inline below the boundary, compiled at/above it, own rules in reserved IDs); **each applier touches only its own range** — §3.1.
- [ ] The priority ladder holds: vendor self-allow unbeatable at the top; the injection layer above the system allowlist; the allowlist above the blocklist; the user tier **above the blocklist and below the injection layer** — §4.2.
- [ ] No intent is expressed by an **equal-priority tie**; if A must beat B, A has a higher number — §2.
- [ ] Every `redirect` target is packaged and web-accessible (else it degrades to a hard block) — §2.
- [ ] Injection fires only on `level == 0 && script == 1`; any non-zero risk `level` suppresses it while ad-blocking stays intact — §5.5.
- [ ] The protected-domain carve-out force-removes search TLDs from every allowlist — §6.
- [ ] The compiled blocklist is delivered via its own URL and is **never** `script`-gated — §3.1.
- [ ] The version pin is echoed identically on every path, including degraded; the degraded path omits `rules` for existing installs — §6.
- [ ] **The user tier sits at a low priority, above the blocklist and below the injection layer** (a safe value is `200`/`100`); it is **never** placed above the injection band — §4.2.
- [ ] **The static packaged rulesets carry no block/redirect on the injection surfaces** (the search TLDs, the partner-feed hosts, the injected analytics/quality hosts). The server allow band overrides them at runtime, but a stale packaged redirect (e.g. a search-param stripper or an `adsense…caf.js`→noop) can fight the injection **in the pre-first-sync window** — strip them from the static bundle — §4.2, §6.
  - **Surfaces to strip** (examples): `google.<tld>/search|webhp|websearch`, bare `||google.<tld>`, `google.com/adsense/(domains/caf|search/ads).js`, `search.yahoo.*`, `syndicatedsearch.goog`, `afs.googlesyndication.com`, `partner.googleadservices.com`, `adtrafficquality.google`, `clarity.ms`, `gstatic`.
  - **Do NOT over-strip — distinguish the injection *surface* from normal ad-domain blocking.** A bare `||googlesyndication.com^` / `adsbygoogle.js` / IMA-SDK / `sites.google.com/...` rule **scoped to a specific publisher `initiatorDomains`** is legitimate ad-blocking on that site and must stay — it never touches the search page. Only rules that fire **on the search/partner surface itself** (unscoped, or scoped to the search hosts) are the ones to remove. A naive substring sweep flags mostly false positives; match the *host being loaded on the search surface*, not any string containing "google".

### 9.2 V1 (Gen 1) only

- [ ] `script` runs **0/1 only** — there is **no state `2`** and no reset payload (§5.1, §8 note ¹).
- [ ] The enable gate is **inert** (commented out): `script` never transitions on sync and `enabled` is never stamped. `script = 1` is reached **only** via the creation-time echoed flag, never auto-promotion — §5.5, `USER_CREATION_AND_UPDATE.md` §5.4 step 7.

### 9.3 V2 (Gen 2/3) only

- [ ] `script` has states 0 / 1 / 2; **state 2 is byte-identical to state 0**, achieved by an explicit reset payload, never by omission — §5.2.
- [ ] The enable gate runs **live** on every sync and requires `script == 0` **exactly** (never `0 || 2`), plus the age/updates/visits/attribution/market/store gates — so an opt-out at `2` is durable — §5.5.

---

*Baseline specification — implementation-independent. Per-product conformance and deviations are in §8 (this doc's per-extension table).*
