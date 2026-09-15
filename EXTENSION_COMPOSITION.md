# Extension Composition — the canonical specification

**What every extension's frontend must contain, and why each piece is shaped the
way it is — so a new build can be checked for completeness and an existing build
can be measured against the latest baseline.**

This is a **guideline and a checklist**, not a template. It defines the
*capabilities, invariants, defaults and reasons* an extension's client side must
embody. It intentionally says **nothing** about how to lay the code out — folder
structure, file names, module boundaries, naming, coding style and the concrete
implementation of each point are free, and should differ from build to build.

Two uses drive every entry:

- **Creating a new build** — read top to bottom; the §26 conformance checklist is
  the acceptance test.
- **Updating an existing build to the latest baseline** — use the §25 per-product
  matrix to find what a given extension is missing, then read the section it
  points to.

> **Companion documents — this doc never re-derives what the peers own.**
> The rule tiers, the priority ladder and the `script` flag lifecycle live in
> **`RULES_AND_PRIORITIES.md`**. Backend identity resolution, dedup and the
> V1 / V2 generations live in **`USER_CREATION_AND_UPDATE.md`**. The VM / risk /
> behavioral detection lives in **`FRAUD_DETECTION_V5.md`**. Where this doc
> touches those areas it states only the *client's* obligation and links out.
> Start from **`00_INDEX.md`** for the folder map and the end-to-end build
> procedure (and the frontend↔backend split that governs how these docs are used).

---

## 0. How to read this document

### 0.1 Status tags

Every numbered requirement carries exactly one tag.

| Tag | Meaning | Build rule |
| :-- | :-- | :-- |
| **`[MANDATORY]`** | Present in every extension, no exceptions. | Ship it. A build missing it is non-conformant. |
| **`[TOGGLEABLE]`** | Must be *implemented*, but exposed to the user as an on/off switch. The **default state** is stated with the item. | Ship it wired to a setting, defaulted as stated. |
| **`[OPTIONAL]`** | Included per-product at the team's discretion. Absence is conformant. | Decide per build. |
| **`[VARIANT: choose-one]`** | A capability that exists in two or more mutually-exclusive shapes. Exactly one must be chosen. | Pick one; record the choice in §26. |

### 0.2 This is a guideline, not a template

There is **no reference skeleton to copy**. This document describes *what* an
extension must do and *why* it matters — never *how* to organize or name the code
that does it. Two conformant builds should look nothing alike: different file
trees, different file and symbol names, different module boundaries, different
coding style, and different concrete implementations of the very same
requirement.

**Why this matters.** The point of the baseline is completeness and correctness,
not uniformity. Cloning a previous build's structure — same layout, same names,
same code — reproduces its bugs, makes the whole portfolio look like one project,
and turns "write a new extension" into "copy-paste the last one." Each build is
implemented independently against this list. If two of your builds share a file
tree, the guideline was used wrong.

So: when an item below names a concrete value, treat it as **a requirement on
behavior, not on code**. "Keep a visited-site map capped near 50" is mandatory;
whether you store it in a class, a closure, or a bare object, and what you call
it, is yours.

**The anti-template rule extends to comments and identifiers, not just structure.**
Shipped code must not **name another build** (its product name, codename, **build
number**, endpoint, or key prefix) or **describe the search-monetization mechanism** — "monetization",
"injection whitelist", "the `script` flag", "YHS", "the redirect to the partner
feed", a surrogate's real purpose. Both are leaks: a sibling name is a **correlation
signal** the store can use to relate the builds, and a description of the injection
scheme is a **disclosure risk** in a file anyone who unpacks the extension can read.
Reference mechanisms by neutral role ("the server allowlist", "server-allow-listed
hosts", "the server directive"), and describe *what* the code does, never *why the
business does it*. This applies to a fresh build **and** to a legacy port — porting
in a sibling's comments verbatim is the easiest way to reintroduce the leak. (Dev
notes that must record the real intent belong in `progress.txt` / `history/`, which
ship with the repo, not inside the `extension/` package.)

### 0.3 On names, structure and keys

- **Storage keys, rule identifiers, message types and function names are
  per-build.** This document refers to them by *role* ("the identity handle",
  "the user allow-list", "the system whitelist"), never by a fixed spelling.
  Choose your own conventions.
- **The server↔client DATA STRUCTURES are per-build too — not just the zero-flicker
  directive (§11.2), but EVERY wire format the client and its backend exchange:** the
  sync-response layout, each filter/cosmetic feed's shape, the compiled-rules and
  dynamic-styles/keyframe directive schemas, the update manifest. Two builds sharing a
  wire format is a **correlation signal** exactly like a shared file tree or a shared
  key prefix (§0.2) — a store or an observer can relate the builds by it. So a flat map
  in one build is a typed op-tree in another, a `{rules,keyframes}` object here is a
  `{v,blocks:[ops]}` tree there — **same behavior, different shape, every build.** (For a
  local-first build the live feed may be dormant; design its formats divergently anyway
  *before* wiring a real server, so the shape is already unique when it goes live.)
- **The sole fixed names are the attribution parameters `an` / `cid` / `sid`**
  (§7.1): they are copied verbatim from the store URL and matched byte-for-byte by
  the backend, so they are never renamed or prefixed.
- **A few values are load-bearing *contracts* between a build's own client and its
  own backend** (e.g. the inline↔compiled rule-ID boundary, an identity token's
  format). Those must be *internally consistent within a build*; they are not
  shared across builds. Each such case is flagged where it appears.

### 0.4 Example builds

Where a variant or deviation is discussed, existing builds are named
(North, Hunter, Wonder, Ninja, Ghost, and the newest, 27) purely as **examples**
of a choice already made in the wild — to illustrate the trade-off, never as a
structure to reproduce. §25 maps every shipped build (the family builds plus Gen 0 `12`) against this baseline.

---

## 1. In plain English

An extension is **four things stacked**:

1. **A local-first blocking floor** that works the instant it installs and even
   if the backend is permanently down — static blocking rules, bundled cosmetic
   filters, and a bundled scriptlet database.
2. **A thin sync client** that pulls *data* (never code) from the backend to
   augment that floor: per-user rules, a compiled blocklist, cosmetic feeds, a
   zero-flicker directive, a system whitelist.
3. **A durable identity** the backend issues and the client guards — mirrored to
   sync storage so it survives a wipe — because identity is what attribution,
   monetization state and churn accounting all hang on.
4. **A set of user surfaces** — a popup, an options page, an element picker,
   block notifications — plus the telemetry heartbeat that drives the sync loop.

If you remember one sentence: **everything the server sends is data the client
assembles locally; the client ships a working ad-blocker with no server at all,
and the server only ever adds.**

---

## 2. Platform baseline & manifest  `[MANDATORY]`

### 2.1 Manifest version and background

Manifest V3. The background is a **service worker**, never a persistent page.

**Why.** MV3 is the only accepted platform on the Chrome Web Store. Everything
downstream (no remote code, DNR-only blocking, the sleeping worker) follows from
it.

### 2.2 Capabilities to request

Request the **minimal** permission set that covers these capabilities, and no
more:

| Capability | Needed for |
| :-- | :-- |
| Declarative network blocking | the whole blocking floor + server rules |
| Script/style injection | cosmetics, scriptlets, the element picker |
| Local + sync storage | the store and the recovery mirror |
| Large local storage | the megabyte-scale compiled caches |
| Navigation events | zero-flicker pre-paint + the sync trigger |
| Active-tab access | the popup's per-site actions |

**Why minimal.** Every extra permission is review friction and attack surface. In
particular, **blocking `webRequest` is never requested** — network filtering is
declarative-only (§22.1) — and a broad `tabs` permission is avoided where
active-tab suffices. Whether you additionally use alarms is a choice, but note the
durable sync trigger is navigation-driven, not alarm-driven (§19).

### 2.3 Content-script injection points

The client must run page-context logic at three injection points:

- an **isolated-world** script at document-start in **all frames** — early
  cosmetic/scriptlet dispatch, the zero-flicker wake-up, visit recording;
- a **main-world** script at document-start in **all frames** — the uBO scriptlet
  library (§12), which must run in the page's own context;
- a **YouTube-scoped** script — the player ad-skip (§14).

**Why document-start + all-frames:** cosmetics and scriptlets must land before the
page paints and inside every ad iframe. **Why a separate main-world entry:**
page-context scriptlets cannot run from the isolated world; MV3's main-world
injection is the compliant way to ship them.

### 2.4 Localization  `[MANDATORY]`

Name, description and all UI strings are localized through the extension's
message catalog, with a sensible default locale and a real set of translations.

**Why.** Store listings and UI must localize; keeping strings out of code keeps
the surface translatable without code changes.

---

## 3. Storage model

### 3.1 Local storage is the single source of truth  `[MANDATORY]`

All reads resolve against local storage; no feature reads from sync on the hot
path. Sync is disaster-recovery only (§4). The architecture is **local-first,
then sync as a fallback for a tight recovery set** — never the reverse.

**Why.** Sync storage has a small quota (~100 KB) and a low write-rate limit
(~120/min); it cannot hold the cached rules/cosmetics, and treating it as a live
store would throttle the extension. Local is effectively unlimited and fast.

### 3.2 Service-worker listeners register synchronously  `[MANDATORY]`

Every event listener the worker relies on — messages, install/startup lifecycle,
navigation — must be registered **synchronously during the worker's initial
top-level evaluation**, not after an `await` or inside a promise.

**Why.** This is a hard MV3 fact, independent of how you structure modules: the
browser only wakes a sleeping worker for an event whose listener was registered at
top-level evaluation. Register late and the listener misses cold-start events. How
you organize the code that registers them is entirely yours.

### 3.3 The state an extension must persist  `[MANDATORY]`

Regardless of key names or layout, a build must keep the following state. The two
right-hand columns are the two invariants the purge (§5) and the mirror (§4)
enforce.

| Role | Purpose | Survives purge? | Mirrored to sync? |
| :-- | :-- | :-: | :-: |
| Identity handle | the server-issued user identity (§6) | ✅ | ✅ |
| Identity/fp version + migration flag | drives fingerprint re-collection (§5.5) | ✅ | — |
| Extension version | last-seen build version | ✅ | ✅ |
| Install-sites snapshot | domains open at install (attribution/dedup) | ✅ | ✅ |
| Attribution `an`/`cid`/`sid` | acquisition key (§7.1) | ✅ | ✅ |
| Monetization/risk state (`script`, `level`) | server-decided flags the client stores & echoes (§24; `RULES_AND_PRIORITIES.md` §5.1) — **V1**: ride inside the handle; **V2**: plain response keys re-delivered from the row each sync | ✅ | ✅ V1 (in handle) · V2 re-derived |
| User settings | master on/off, YouTube, cookies, acceptable-ads, user allow-list, user block-list, user cosmetics | ✅ | — |
| Visited-sites map | telemetry heartbeat data (§18) | ✅ | ✅ (deferred) |
| Sync bookkeeping | last-sync time, sync interval, data-version stamp, verbose flag | ✅ | — |
| Server content caches | **inline `rules`**, system whitelist, scriptlet **excludes**, dynamic/zero-flicker CSS, cosmetic feeds + their version stamps, compiled-rules version | ❌ (rebuilt) | — |
| Popup-block feed | blocked-popup list (per-tab, capped) — kept in **session** storage | — | — |

The **static scriptlet database** is deliberately **not** in the caches row: it ships
**in the package** (§12.1), is never server-delivered, and so is never fetched or
purged — it survives a data-version bump because a purge cannot touch packaged
assets (consistent with §5.2). Only the server-delivered scriptlet **excludes** list
is a rebuildable cache.

**Why one clear split** (settings vs. server caches): it is exactly the line the
purge draws — settings survive, caches are dropped and rebuilt.

---

## 4. Sync-storage mirror — identity survival  `[MANDATORY]`

### 4.1 The tight mirror set

A deliberately small subset is copied to sync storage as best-effort backup:
**identity handle, extension version, install-sites snapshot, visited-sites map,
and `an`/`cid`/`sid`.** Nothing else.

**Why so small.** Sync's tiny quota and write-rate limit forbid mirroring bulk
data. What *must* survive a local wipe is the user's distinction (identity) and
how they were acquired (attribution) — lose those and the user fragments into a
new row and, for monetized products, an opt-out silently resets.

### 4.2 Heal before build  `[MANDATORY]`

Before assembling a sync payload, restore any missing mirror value from sync into
local. This is how a wiped-local / reinstalled / new-machine user recovers
identity and attribution.

### 4.3 Deferred mirroring of the hot value  `[MANDATORY]`

The visited-sites map is written to local on every navigation but mirrored to sync
**only once per successful sync cycle**, not on every write.

**Why.** It is the highest-churn value; mirroring it on every navigation would
blow the sync write-rate limit. Identity/attribution change rarely and are
mirrored immediately.

### 4.4 The mirror ⊆ preserved invariant  `[MANDATORY]`

Everything in the mirror set must also be in the preserved set (§5). A purge must
never destroy the very mirror that recovery depends on.

---

## 5. Versioning & the storage-purge protocol  `[MANDATORY]`

### 5.1 The data-version purge

The server sends a **data-version constant** on **every** response, including the
degraded / DB-down path. If it differs from the stored value, the client reads all
local + sync keys, **removes every key not in the preserved set**, then the same
sync cycle repopulates the caches.

**Why it exists.** It is the only fleet-wide "reset your cached server data"
lever. When the server's data shape changes incompatibly, bumping the constant
forces every client to drop stale caches and rebuild cleanly, without shipping a
new extension version.

### 5.2 The preserved set  `[MANDATORY]`

**All local state survives except the server-delivered content caches** (the ❌
row of §3.3, which the same cycle rebuilds): identity + fingerprint-version, the
attribution keys, the **extension-version + install-sites snapshot + visited-sites
map** (the §4.1 sync-mirror set — recovery depends on these, §4.4), all user
settings, bookkeeping, the static scriptlet bundle, and the data-version stamp.

> **Load-bearing rule:** any NEW user-setting introduced in a later build **must**
> be added to the preserved set, or it is wiped on the next data-version bump.
> This is a real historical bug: a sibling omitted its cookie-blocker toggle from
> the preserved set, so a version bump silently reset the user's opt-in. Whenever
> you add a persisted setting, add it to the preserved set in the same change.

### 5.3 Stamp written last  `[MANDATORY]`

The new data-version value is written **after** every other write in the cycle
succeeds. A cycle that dies mid-way leaves the old stamp, so the next cycle
re-purges and re-tries rather than declaring a half-applied state final.

### 5.4 Version pin on every path  `[MANDATORY]`

The constant must be **echoed identically on every server path, including the
degraded HTTP-200 path** (§23.3). A server that returns a *different* or absent
value on an outage would trigger a fleet-wide purge during the outage. Cross-ref
`RULES_AND_PRIORITIES.md` §6 (Version pin).

### 5.5 Fingerprint-version migration  `[MANDATORY for V2; N/A for V1]`

A separate fingerprint-version constant drives a hardware re-collection migration
(distinct from the data purge). When the server returns a new fingerprint version
and the client has no raw signals to send, it flags a migration, re-attaches the
hardware signal set (§7.2) on the next sync so the backend can recompute/backfill,
then clears the flag on confirmation. Cross-ref `USER_CREATION_AND_UPDATE.md` §6.8.

---

## 6. Identity — the client side  `[MANDATORY]` · mechanism `[VARIANT: choose-one]`

Every extension has a **durable, server-issued identity** the client stores,
mirrors to sync, echoes on every sync, and appends to the uninstall URL. That much
is mandatory and identical in intent across products. *Which* identity mechanism
is a choose-one variant with three **selectable** shapes (V1 / V2-A / V2-B). The backend
semantics live in `USER_CREATION_AND_UPDATE.md` §3; below is the **client's** obligation for each.

> **`Gen 0` (Legacy, pre-family) is a fourth shape, but *not* a choose-one option.** A legacy
> build (e.g. 12) carries an AES `syncuid` handle on a frozen pre-family schema — its client
> obligation is exactly the §6.1 universal contract (store / mirror / heal / echo / append to the
> uninstall URL), the same shape as V1's opaque blob. It is a **frozen lineage, never selectable
> for a new build** (the identity-freeze — `00_GOAL.md` §4); it is the lane behind the §26 Record
> block's "`Gen 0` Legacy" checkbox. New builds choose only V1/V2-A/V2-B.

### 6.1 The client identity contract (all variants)  `[MANDATORY]`

1. The client **never invents** the handle — it stores what the server issues.
2. The handle **round-trips byte-for-byte** in the sync POST body
   (payload-embedded; no auth header).
3. The handle is **mirrored to sync** and **healed back** before every sync build
   (§4), so a local wipe recovers it.
4. The handle is appended to the uninstall URL as the user id (§20.3).
5. The client **never interprets** `script` or `level` — it only stores and echoes
   them; all behavior is server-decided and delivered as data (§24). *Where they
   live differs by variant:* under **V1** they are encoded **inside** the handle
   (§6.2), so echoing the handle echoes them; under **V2** the handle is an opaque
   pointer and they arrive as **plain response keys** alongside it (§6.3–6.4).

### 6.2 Variant V1 — encrypted state blob

*Example builds: North, Hunter, Wonder.*

The handle is an **AES-256-GCM ciphertext** encoding the user's state (row id,
`script`, `level`, `conv`, `updates`, `typetag`, creation date, market), wire form
`base64(iv ‖ tag ‖ ciphertext)`.

**Client obligations**
- Store and echo the opaque ciphertext. **Never decode it.**
- **Collect no hardware fingerprint** (§7.2 is skipped entirely).
- Mirror the blob to sync as the single most critical recovery value.

**Why choose it:** simplest client — no hardware collection, and the server serves
the whole request from one decrypt.
**Cost to respect:** the blob *is* the state, so a stale/lost blob rewinds or
loses state. The sync mirror (§4) is therefore non-negotiable for V1.
Backend: `USER_CREATION_AND_UPDATE.md` §3.1.

### 6.3 Variant V2-A — derived (HMAC) handle

*Example build: Ninja. The "different from the others" case.*

The handle is an **HMAC over hardware components + `an`/`cid`/`sid` + IP**,
computed server-side and returned as an opaque hash the client stores.

**Client obligations**
- **Collect the hardware signal set (§7.2) and send raw signals** whenever the
  handle is missing or a fingerprint-version migration is pending; store the hash
  the server returns.
- Store and echo the hash; **never recompute it client-side.**

**Why choose it:** the token doubles as a device descriptor, so a returning user
with a lost handle can be re-recognised from the same inputs.
**Cost to respect — and why V2-B exists:** because the hash folds in **IP and
attribution**, the *same* user on a new network or a new campaign produces a
*different* handle and cannot be recognised by it. The sync mirror is the primary
continuity mechanism. Backend: `USER_CREATION_AND_UPDATE.md` §3.2 (V2-A).

### 6.4 Variant V2-B — minted random token  *(recommended for new builds)*

*Example builds: Ghost, 27.*

The handle is a **write-once random token** (256 bits of CSPRNG entropy, rendered
as a fixed-length hex string), minted at row creation and **never recomputed**.
Identity and device-description are cleanly separated: the token means nothing, so
nothing is tempted to recompute it; a *separate* server-side device signature
(stable hardware only) is the reconstruction hint.

**Client obligations**
- **Collect the hardware signal set (§7.2) and send raw signals** while
  unidentified or migrating — but understand these feed the device signature and
  fraud analysis, **not** the identity token.
- Store and echo the token verbatim; mirror to sync.
- **The token format is a load-bearing contract with your own uninstall
  endpoint** (which routes on it). Keep it identical across the stored value, the
  sync mirror, the payload and the uninstall URL for *this* build.

**Why choose it (default for new builds):** a returning attributed user is
recognised by the token directly (the large majority of calls) with no collision
risk; a new campaign is a new acquisition; identity never desyncs because nothing
rewrites it. This is the endpoint of the identity evolution. Backend:
`USER_CREATION_AND_UPDATE.md` §3.2 (V2-B).

### 6.5 Variant comparison

| | **V1 blob** | **V2-A derived** | **V2-B token** |
| :-- | :-- | :-- | :-- |
| Client stores | ciphertext (is the state) | opaque hash | opaque random token |
| Hardware collection | **none** | **required** | **required** (device sig / fraud, not identity) |
| Client can decode | no | no | no |
| Recomputable server-side | n/a | yes (collision-prone) | **never** |
| Survives new network / campaign | via mirror only | **no** (handle changes) | **yes** |
| Example builds | North, Hunter, Wonder | Ninja | Ghost, 27 |
| Backend section | `USER_CREATION_AND_UPDATE.md` §3.1 | §3.2 (V2-A) | §3.2 (V2-B) |

---

## 7. Install attribution & fingerprint collection

### 7.1 Attribution capture  `[MANDATORY]`

On install, snapshot all open tabs; from a Chrome Web Store URL that carries the
attribution marker, harvest **all** its query params into `an` / `cid` / `sid`
(and any campaign `utm_*`). Also record all open-tab hostnames as the install
snapshot. Write to local and mirror to sync.

**Why the names are fixed.** `an` / `cid` / `sid` are copied verbatim from the
store URL and matched byte-for-byte by the backend as the acquisition key.
Rebranding them would break attribution and, for V2-A, identity. They are the
**only** keys that never carry a build-specific name.

**Why the install snapshot.** The set of domains open at the install millisecond
is a funnel-attribution and dedup signal (real tabs open = a real human;
`USER_CREATION_AND_UPDATE.md`). Captured once, never updated.

### 7.2 The hardware signal set  `[MANDATORY for V2; N/A for V1]`

Collected top-frame only, **in memory only (never persisted)**, and attached to
the payload **only** while the client is unidentified (no handle) or migrating.
Dropped once identity settles.

The signal set: CPU cores, device memory, GPU vendor + renderer (WebGL unmasked),
language, colour depth, timezone, a 2D canvas hash, touch points, **an audio
fingerprint** (OfflineAudioContext), plus metadata — screen resolution, user
agent, battery charging/level, a webdriver flag, and WebGL extension count — and
`an`/`cid`/`sid`.

**Why in-memory-only:** the raw signals are needed exactly once (to mint/derive
identity, then per migration). Persisting them would be a standing privacy
liability for no benefit. **Why the audio fingerprint and metadata:** the server
derives a stable device signature and a separate volatile fraud hash from them,
and runs VM/headless detection (`FRAUD_DETECTION_V5.md` §2, §4). The client only
ships raw signals; it makes no fraud decision.

---

## 8. The DNR rule system — client responsibilities  `[MANDATORY]`

The ladder, the tier model and the priority numbers are specified in
**`RULES_AND_PRIORITIES.md`** and are not restated here. This section is only the
**client's** obligations toward that system.

### 8.1 Strict rule-ID partitioning

- Reserve a **small, fixed, documented set of low IDs for the extension's own
  rules** — user allow, user block, the server-managed system whitelist, an
  optional system block, a YouTube-infra allow, and volatile session mirrors. The
  exact numbers are your choice; what is mandatory is that they are fixed within
  the build and **disjoint** from the server ranges.
- **Server-delivered rules occupy higher, non-overlapping ranges:** inline
  per-user rules below the inline↔compiled boundary, the compiled bulk blocklist
  at/above it. That boundary is a contract with your own backend (canonically the
  value fixed in `RULES_AND_PRIORITIES.md` §3.1); use the same value on both ends.

**Why partition at all:** so a per-user sync never clobbers the multi-megabyte
compiled blocklist, and vice-versa.

### 8.2 Range-scoped appliers are a hard invariant  `[MANDATORY]`

Each apply step removes and re-adds **only its own range** — the inline applier
touches only inline IDs, the compiled applier only compiled IDs, the user/session
appliers only their reserved IDs.

**Why.** This lets a routine sync re-apply per-user rules without re-downloading or
disturbing the compiled blocklist, and lets a degraded response omit rules
entirely without wiping already-applied inline rules (§23.3). Cross-ref
`RULES_AND_PRIORITIES.md` §3.1.

### 8.3 Instant feedback + restart persistence  `[MANDATORY]`

A user list change must (a) take effect **immediately** and (b) **persist across a
restart**. The proven technique is a double-write: apply a **session** rule at
once for instant feedback, delete any stale persistent twin so it cannot shadow
the fresh rule, then rebuild the **persistent (dynamic)** rule shortly after,
debounced.

**Why the double-write:** persistent (dynamic) rule updates re-index slowly
(seconds); session rules apply in milliseconds but are volatile. Using both buys
instant UI response and durability. The requirement is the two properties; the
double-write is how to get them.

### 8.4 Static rulesets shipped enabled  `[MANDATORY]`

Ship the all-in-one blocklist as static rulesets whose install-enabled state
**matches the manifest declaration exactly**. Heavy or optional lists (e.g.
cookie-consent, YouTube) ship **disabled** and toggle on demand.

**Why static-and-on:** the floor must block from the first millisecond and survive
a permanently-dead backend. **Why the manifest must agree:** the client toggles
rulesets by their declared identifiers; a mismatch silently no-ops a toggle.

### 8.5 Vendor self-allow at maximum priority  `[MANDATORY]`

The extension's own backend/CDN domain is allow-listed at the **maximum priority**
as **both** request-domain and initiator-domain, inside a static ruleset.

**Why.** Sync and asset fetches must never be self-blocked — not even by a user who
blocklists the vendor domain, nor by an over-broad blocklist entry. Without it,
one bad rule could permanently sever the update channel.
*Housekeeping:* when a build is derived from an earlier one, strip the parent's
self-allow domain — a build that still whitelists an ancestor's domain is a latent
leak.

### 8.6 Why static sits *below* server rules

The static floor is the lowest tier on purpose; it only wins when nothing above it
claims the request. Per-user server rules, the system allowlist, and (when live)
the monetization layer all sit above it. The full rationale and the numeric ladder
are in `RULES_AND_PRIORITIES.md` §4 — the client's job is only to apply each tier
into its own ID range and never to invent priorities that cross tiers.

---

## 9. Blocklist & compiled bulk  `[MANDATORY]`

The compiled blocklist (the high ID range) is lazy-fetched from a dedicated URL and
re-fetched **only when its version hash changes**. It is applied **directly to the
blocking engine and never held in local storage** — tens of thousands of rule
objects are too slow to round-trip through storage. The client passes its runtime
id so the server can substitute the per-client extension id into redirect targets;
conditional-request caching avoids re-downloading unchanged caches.

**Why version-gated + direct-to-engine:** the compiled list is the heaviest
payload; fetching it every sync or persisting it would dominate the extension's
cost. **Why never `script`-gated:** the blocklist is identical whether monetization
is on or off (`RULES_AND_PRIORITIES.md` §3.1); only the inline tier changes with
`script`.

---

## 10. Cosmetic filtering  `[MANDATORY]`

Four cooperating layers, all assembled locally from data:

1. **Static generic** — bundled generic hide rules injected on non-whitelisted
   pages.
2. **Static specific** — bundled per-domain selector maps (uBO-style indexed
   lookup by host + parent domains); exception selectors cancel hides.
3. **Dynamic generic + dynamic specific** — server feeds, each re-fetched only when
   its version changes; for the same selector an unhide beats a hide.
4. **Dynamic styles** — richer per-domain rules with an allowlisted property/value
   grammar plus optional delay/duration (timed inject, then auto-remove).

**Why four layers:** the static pair guarantees baseline hiding offline; the
dynamic pair lets the backend patch site breakage and new ad formats without an
extension release; the specific/unhide split matches uBO semantics so imported
filter lists behave correctly.
**Why data, never a stylesheet:** MV3 forbids remote code, and a remote raw
stylesheet is treated as code. The client builds all CSS through a fixed grammar
(§22.2).

### 10.1 Cosmetic implementation invariants (hard-won)

Each of these was a real regression that only surfaced under **live browser testing** — the
offline logic passes while the page shows nothing hidden. Assert them per build (first
observed on build 27, 2026-09-03):

- **Key the per-domain lookup by the EXACT host — do NOT strip `www.`.** uBO keys a rule by
  the host as written: `www.yahoo.com##…` is a *distinct* map key from `yahoo.com##…`. A lookup
  that normalizes `www.` away (correct for the *whitelist*, wrong here) silently drops every
  `www.*`-keyed rule. Look up the full host **and** its parent suffixes.
- **`:style()` is CSS, not procedural.** `selector:style(decls)` is a plain restyle rule
  (`selector{decls}`) — the reserved-space collapser uBO leans on (`height:0`, `min-height:0`,
  margin fixes). Parse it into the dynamic-styles layer (layer 4); do **not** bucket it with the
  procedural operators and drop it.
- **Separate truly-procedural selectors, and make the emitted hide list forgiving.**
  `:has-text()`, `:upward()`, `:xpath()`, `:matches-css()`, `:remove()` are **not** native CSS;
  they must never enter a comma-joined rule, because **one invalid selector voids the ENTIRE
  rule** and silently drops every other selector in that chunk. Route them to a procedural
  runtime (or set them aside), and wrap the hide list in **`:is(...)`** so an unexpected bad
  selector is ignored rather than nuking the rule. (`:has()`/`:not()`/`:is()` ARE native CSS —
  keep them; they need Chrome ≥ 105 for `:has()`.)
- **Target the sizing CONTAINER, not just the ad.** Hiding the ad element leaves the wrapper's
  reserved height (an empty box). The rule must hit the container — uBO's `:has()` wrapper
  selectors, a `:style(height:0)`, or a fixed-dimension `div[style="…"]` selector.
- **Offline validation cannot prove cosmetics WORK.** A passing sim proves the CSS is *built*;
  only a live load proves selectors *match* and *hide*. The `[OWNER — manual]` test must confirm
  visible hiding on real sites and watch for reserved-space/empty-container gaps — and packaged
  filters go **stale** as sites redesign (a site changing an ad-slot class/id makes a shipped
  rule miss). A fresh uBO pull, the dynamic feed, the element picker, or a curated patch closes it.
- **Build the dynamic-styles layer to its FULL capability, not the current-data subset.** The
  client is a *data-driven engine*: it must be able to compile ANY directive the feed can later
  send — the whole motion set (`transform`/`transition`/`animation` + timing) **and** `@keyframes`
  built locally from structured data — even if today's packaged data only uses `display:none` and a
  reserved-space collapse. This is the point of §11.2 ("the keyframes can all change server-side with
  no republish"): it only holds if the *shipped* grammar already handles it. Scoping the grammar to
  what the current data happens to use is a real regression (build 27 first shipped only the collapse
  properties and had to retrofit motion/keyframes — 2026-09-04). The general rule: **build every
  data-driven layer to the full contract capability, then let the data decide what to use.**

---

## 11. Zero-flicker  `[MANDATORY]`

### 11.1 Two-layer pre-paint hide

The two layers are **two independent injectors of the same pre-hide CSS**, ordered
so the fast one never depends on the slow one:

- **Layer 1 (the actual zero-flicker) — content script, direct from storage.** As
  its **very first action at `document_start`**, the top-frame content script reads
  the pre-hide directive **directly from `chrome.storage.local`** and injects a
  `<style>` into `document.documentElement`. This is what beats the flash: a
  content-script storage read resolves in a few ms — **before first paint of
  network-loaded content** — and involves **no service worker at all**.
- **Layer 2 (reinforcement) — the worker, USER-origin.** On the pre-paint
  navigation-commit event, the worker *also* injects the CSS via the scripting API
  from an **in-memory cache** kept fresh as storage changes. This copy is
  **USER-origin (CSP-immune, cannot be stripped by the page)** and covers exotic
  cases, but it is **not** the primary pre-paint path.

**Why Layer 1 must not go through the worker.** In MV3 the worker sleeps after
~30 s; a `sendMessage` to it on the hot path first has to **spin the worker up**
(import scripts + compile, 50–300 ms) — *slower than the network fetch of the page
itself*, so the results paint before the hide lands. Reading `chrome.storage.local`
from the content script does **not** wake the worker and is the correct fast path.
**A build whose content script only *messages* the worker for the pre-hide CSS (and
does not read storage itself) will flash on every cold start** — the most common
zero-flicker regression; the fix is always to move Layer 1 to a direct storage
read. (The worker keeps its in-memory cache for Layer 2, where a per-frame async
storage read on the hot path *would* be too slow.)

> **Engine reuse in the content world.** Layer 1 needs the same host-matching and
> the same CSS-grammar builder the worker uses (§22.2). Load those pure,
> `chrome`-free engine modules into the **content-script** context too (list them
> before the content script in the manifest) so both worlds build identical CSS from
> the same data — never duplicate the grammar, and never let the content script
> evaluate server strings (§22.1).

### 11.2 The server CSS directive — the highest-stakes payload  `[MANDATORY]`

Zero-flicker is driven by a **structured directive delivered by the server** — a
data description of which hosts to pre-hide, which selectors to hide, and how/when
to reveal them — **not** a raw CSS string and never an executable form. The client
compiles it to CSS locally through the fixed grammar of §22.2.

This directive is **the single most important server payload for the search /
monetization behavior** — it is the visual half of the injection layer (§24): the
thing that pre-hides a search page's ad slots and then reveals the organic results
after a short delay. It appears under a different name in every build (local CSS,
raw CSS, dynamic styles, and so on), and getting its handling wrong is a leading
source of both site breakage and MV3 policy warnings. Three rules are absolute:

- **It is DATA, compiled locally.** The server ships a structure; the client builds
  the CSS from it through the §22.2 grammar. It is never injected as a remote
  stylesheet and never evaluated (§22.1).
- **Its wire format and internal structure MUST differ from build to build**
  (per §0.2) — a flat string in one build, a typed block/op tree in another —
  while producing **identical behavior**. Same job, different shape, every time.
- **The backend fully controls it with no republish.** Because it is data, the set
  of hidden selectors, the reveal timing and the keyframes can all change
  server-side without shipping a new extension.

**Why structured rather than raw CSS:** it stays MV3-compliant (§22.1) and lets the
backend re-target the pre-hide/reveal as search layouts change without an extension
update — the endpoint of the zero-flicker evolution.

> **The allowlist trap — the flicker must survive the injection's own allowlist.**
> When monetization is on, the injection layer **expands the published allowlist**
> over the search surface so ad-block cosmetics/scriptlets stay off it
> (`RULES_AND_PRIORITIES.md` §5.4). Zero-flicker is **not** ad-block cosmetics — it
> is the injection's *own* pre-hide — so it must still run on that surface. Do **not**
> gate zero-flicker on the injection allowlist; gate it only on the **user's own**
> pause/whitelist (§13.4), which the **protected-domain carve-out (§13.5)** keeps
> empty for the search hosts, so the pre-hide always fires there. The recurring
> defect: a backend that **re-adds** the search hosts to the allowlist *after* the
> carve-out, so the client's cosmetic-bypass swallows the flicker — the results
> flash before they are masked. If the flicker is suppressed on a search page, check
> the *allowlist contents* first, not the injection timing.

---

## 12. Scriptlets & surrogates  `[MANDATORY]`

### 12.1 Static uBO scriptlet database

The scriptlet library is **shipped statically** — the per-scriptlet argument data
and the scriptlet implementations both bundled in the package — and loaded into
local storage at install/update. **The scriptlet *code* is never server-fetched**
(remote code = MV3 rejection). The per-domain argument *data* also ships static by
default; a **dormant** server-side dynamic layer for that data exists
(`DATA_SOURCES.md` §4.2, the scriptlet counterpart to the cosmetic feeds) but is not
active — the mandated behavior is fully static.

**Why static:** a server-fetched scriptlet (code *or* its argument config) is
remote code under MV3 and is a review rejection. Every sibling moved scriptlets
from dynamic to static for exactly this reason.

### 12.2 Main-world injection with CSP fallback  `[MANDATORY]`

Match this frame's host (exact + parent domains + entity buckets) against the DB;
inject each matched scriptlet into the page **main world** via an
extension-packaged script URL. If the page's CSP blocks that, fall back to a
scripting-API injection into the main world (not subject to page CSP), passing args
through a page-world handoff. Deliveries are **deduplicated** so multiple paths
cannot double-inject.

**Why a fallback:** strict-CSP sites block the injected script tag; the
scripting-API path is not subject to page CSP.

### 12.3 Skips and quiet aborts  `[MANDATORY]`

Scriptlets skip hosts in the user whitelist, the system whitelist, and the
scriptlet-exclude list. Silent aborts use a per-page **unbranded** marker that a
capturing error listener swallows (no console noise, no ad-block fingerprint).
Guard hot prototype built-ins so a hook cannot freeze the page.

### 12.4 Surrogates / redirect resources  `[MANDATORY]`

Bundle uBO surrogate/noop resources in the package (analytics stubs, silent
players/pixels, etc.) and declare them web-accessible, used as redirect targets by
blocking rules.

**Why they must actually ship:** a redirect whose target is not packaged and
web-accessible still *wins* at the rule layer but resolves to a failed load — it
degrades to a hard block-with-error instead of the working shim intended. Two
siblings shipped redirect rules whose surrogate folder was missing; do not repeat
it. Cross-ref `RULES_AND_PRIORITIES.md` §2 (redirect-strength rule).

---

## 13. Whitelist / blocklist / pause  `[MANDATORY]`

### 13.1 User lists

The user allow-list and block-list are the source of truth; normalize each entry
(strip protocol/path/`www.`/trailing dot). A user action writes storage, applies a
session rule instantly, and schedules the debounced persistent rebuild (§8.3).

### 13.2 Removal tracking  `[MANDATORY]`

Removing a whitelist entry records it in a **"recently removed" delta** sent to the
server so it can drop the domain from the user's server-side lists, then the delta
is cleared after the rule update.

**Why.** The server keeps a merged copy of the user's lists; without an explicit
removal signal a de-whitelisted domain would be re-pushed on the next sync and
never actually leave.

### 13.3 System whitelist  `[MANDATORY]`

The server sends a default/system whitelist applied as a high-priority allow,
**rewritten every sync**, and cached locally so the content scripts (which have no
rule-engine access) can also skip cosmetics/scriptlets on those hosts. Large lists
are chunked.

> **Both halves are load-bearing — the network allow is not optional (2026-09, build 12).**
> "Cached locally for the content scripts" is only the cosmetic half; the
> high-priority **network allow is the half that covers the user when server-side
> allow building has a gap**. 12's client long cached the list but built no rule
> from it, and the carve-out's host matcher (§13.5) ejected the canonical US
> search host from the server's own frame-allow pairs while missing that host's
> regional variants — which kept their cover — so bulk-feed blocks fired
> user-visibly on exactly that one host. The fix was this section's mechanism:
> a persistent allow rebuilt from the delivered list every sync, as the sibling
> builds already ran in production.

### 13.4 Whitelisted == paused  `[MANDATORY]`

A whitelisted domain and a paused domain must behave **identically**: no network
blocking, no scriptlets, no cosmetic CSS. (A phishing warning, if any, is the only
thing that still applies.)

**Why identical:** users expect "pause" and "always allow" to mean the same "leave
this site alone." The whitelist check must gate **every** injection path — network
rules, scriptlets, cosmetic CSS **and zero-flicker**.

**The one deliberate exception is a protected domain (§13.5).** Search surfaces are
force-removed from every allowlist, so a user *cannot* whitelist them; on those
domains the pre-hide/reveal always applies — not because zero-flicker ignores the
whitelist, but because there is no whitelist entry to honor. So the precise rule
is: *on a whitelistable site, a whitelist/pause suppresses zero-flicker too; on a
protected (non-whitelistable) site, zero-flicker always runs, by design.* A build
whose zero-flicker only ever targets protected search surfaces will look like it
"ignores the whitelist" — that is expected; what must not happen is zero-flicker
firing on a **whitelistable** site the user has paused.

### 13.5 Protected-domain carve-out  `[MANDATORY]`

Search TLDs and their asset hosts are force-removed from **every** allowlist, even
if a user explicitly whitelists them. This is enforced server-side and the client
must not attempt to re-add them. Cross-ref `RULES_AND_PRIORITIES.md` §6.

---

## 14. YouTube  `[TOGGLEABLE]` — default **OFF**

Two mechanisms behind one switch (default off):

1. **Network** — while OFF, YouTube infrastructure is whitelisted by a standalone
   allow rule and the YouTube blocking ruleset is **disabled**. While ON, the infra
   allow is dropped and the ruleset is **enabled**.
2. **Player** — the YouTube-scoped script skips pre-rolls (click the skip control,
   or fast-forward the ad video to its end), dismisses "still watching" dialogs,
   and injects fallback cosmetic CSS — driven by a short polling interval plus an
   observer on the player state.

**Why OFF by default:** YouTube is the highest-breakage, highest-anti-adblock
surface; a stable default install lets users opt into the risk. **Why
infra-whitelist when off:** so the general blocklist can't partially break playback
for users who never enabled it.

---

## 15. Cookie-consent blocker  `[TOGGLEABLE]` — default **OFF**

One switch (default off) enabling three cooperating layers: a cookie-consent
blocking ruleset (toggled on), a large banner-hide CSS injected pre-paint on the
top frame, and a main-world CMP auto-reject runtime injected into the top frame
**and** cross-origin CMP iframes, which calls the real reject APIs and unlocks
scroll, bounded by a short poll + observer.

**Why a first-class toggle:** cookie-consent handling breaks some sites and is a
distinct preference from ad-blocking; it must be independently switchable and
default off.

---

## 16. Element picker  `[OPTIONAL]`

An on-demand feature that may or may not be included in a given build. When
present: a control in the popup injects a picker into the active tab (refusing
non-http(s) pages), with hover-highlight, widen/tighten depth, and stable selector
generation (rejecting dynamic ids/classes with long digit runs or hashes). Picked
selectors persist per host, re-apply at document-start, and sync to the server
(double-sanitized client + server).

**Why you might include it:** it is the capability the two newest builds (Ghost, 27)
add over the older builds; user-authored hiding closes the gap where a filter list misses
a site-specific element. It is optional because it is a discretionary enhancement,
not part of the blocking floor — a build ships with or without it.

---

## 17. Popup-ad blocker & redirect landing  `[MANDATORY]` · style `[VARIANT: choose-one]`

A blocked popup/malicious top-frame navigation is redirected (by a compiled rule)
to a landing target. Two landing styles exist.

### 17.1 Variant A — self-closing interstitial  *(recommended)*

*Example builds: 12, Ninja, Ghost, 27.*

Redirect to a **web-accessible page** that logs the blocked popup to session
storage (deduped, per-tab cap), updates the badge, **auto-closes itself** after a
short countdown with a failsafe, and offers Allow / **Peek** (temporary whitelist
auto-revoked when the peeked tab closes).

**Why:** the page closes itself with **no service-worker dependency**, so it works
even if the worker is asleep, and the feed lives in session storage the popup reads
directly.

> **Two traps a Variant-A build must close (build 27, §17 phase):**
> 1. **The landing is web-accessible → it is a clickjack surface.** Because it's
>    reachable from any web origin, a hostile page can open it with crafted
>    `?domain=`/`?url=` params to trick the user into whitelisting an attacker domain
>    (which globally disables the blocker) or Peek-opening an attacker URL. The
>    interstitial must therefore treat its params as **untrusted** and enable the
>    whitelist actions only after validating a **worker-issued nonce** (stored in
>    `storage.session`, unknowable to web pages) echoed in the redirect. An un-trusted
>    open still self-closes but offers no whitelist power. Also render params with
>    `textContent`, never `innerHTML`.
> 2. **MV3 evicts the worker while a tab stays open.** Any popup-blocker state that
>    must outlive an eviction — the per-tab block feed, the badge source, and above all
>    the **peek temp-allow tracking** — must live in `storage.session`, not an in-RAM
>    `Map`. If peek tracking is in memory, the worker sleeps while the peek tab is open,
>    and on `tabs.onRemoved` (which re-wakes it) the tracking is gone, so the temporary
>    whitelist entry **leaks permanently into the user allowlist**. Likewise, handlers
>    that gate on async-loaded data (the popup-domain list, settings) must `await` a
>    readiness promise — the navigation event is exactly what cold-starts the worker, so
>    the first popup after every wake races the load and escapes otherwise.

### 17.2 Variant B — blocked page + real-tab notification

*Example builds: North, Hunter, Wonder.*

Redirect to a minimal blocked page that messages the worker, which re-activates the
last real tab and shows a content-script notification (Whitelist / Visit once /
Always hide) targeted via a recent-real-tabs buffer.

**Why it still exists:** it surfaces the block on the originating tab rather than a
throwaway page. New builds should prefer Variant A for its worker-independence.

---

## 18. Telemetry — visit recording  `[MANDATORY]`

On each top-frame navigation (after a few seconds), the content script reports the
hostname; the worker debounces it into a visited-sites map of **`{count,
last-time}`** per domain. Capped near **50** domains; on overflow, prune the
lowest by a decay score (roughly `count / (age-in-days + 1)`). The payload
**flattens to counts only** — the timestamp stays client-side. Root domain only;
no URLs, no reading of input fields or cookies.

**Why the timestamp is kept locally but not sent:** it powers the decay-based
eviction so the cap keeps the *relevant* domains, but the server only needs counts
— sending timestamps would be needless data exposure. **Why root-domain + no input
hooks:** data minimization — visited-domain counts are zero-party, unblockable
telemetry that reveals no browsing detail.

---

## 19. Sync cadence & triggers  `[MANDATORY]`

The **durable trigger is a due-check on every real user action** — navigation,
toggle, list edit — firing a sync only once the server-set interval has elapsed
since the last sync. A best-effort timer may arm the next cycle as a secondary
path. A single-flight guard prevents overlapping syncs. Default interval a few
hours; a much shorter interval while identity is still pending; a jittered longer
backoff in degraded mode. Also sync on install, first-fingerprint arrival, the
acceptable-ads toggle, and fingerprint migration.

**Why action-driven and not a timer:** an MV3 worker's timers die when the worker
sleeps; navigation is the only reliably-recurring wake signal, so gating a sync on
"is it due yet?" at every navigation is what actually keeps the fleet current.

---

## 20. Lifecycle  `[MANDATORY]`

### 20.1 Install

Open the onboarding page, capture attribution + the install snapshot (§7.1), seed
setting defaults (**acceptable-ads on, master switch on**), enable the default
rulesets, register the uninstall URL, and post the first sync.

### 20.2 Update must not reopen onboarding  `[MANDATORY]`

Treat update / browser-update / shared-module-update as plain startups — **never
reopen the onboarding page on update.**

**Why.** This is a recurring wart across every sibling: the browser's own
background update reopened the install tab. Onboarding is an install-only surface.

### 20.3 Uninstall URL  `[MANDATORY]`

Register an uninstall URL, **refreshed after every sync**, carrying the extension
id, version and identity handle. It must be **fail-safe even on backend failure**
(the endpoint redirects to a feedback destination regardless). The server marks the
record disabled (a churn ping); a returning sync resurrects it.

### 20.4 Master kill-switch  `[MANDATORY]`

The global off switch disables **all** static rulesets and clears session rules; on
re-enables the defaults and rewrites the user session + persistent rules. Toggling
reloads **only the active web page — never the extension's own options/popup tab.**

> **The master gate must hold on EVERY injection path, not just the network layer and
> the zero-flicker content script.** The kill-switch is easy to wire into the DNR
> rulesets and the `document_start` Layer-1 read, then forget on the **worker-driven**
> paths — the cosmetic Layer-2 `insertCSS`, the scriptlet `executeScript`, the
> cookie/YouTube injectors. Because Layer-2 cosmetics are injected **USER-origin**
> (CSP-immune, the page cannot override them), a worker path that ignores the master
> switch keeps hiding elements / running scriptlets with the extension globally OFF, and
> the page cannot undo it. Rule: any code that injects CSS or script into a page must
> read the same `master` flag the content script's Layer 1 gates on (mirror it in the
> subsystem's RAM cache + `storage.onChanged`, exactly as the whitelist/pause gate is
> mirrored). Build 27 hit this gap on cosmetics + scriptlets (Layer 1 honored master,
> Layer 2 did not) and closed it — before its owner ultimately removed the global
> kill-switch entirely (per-site pause as the only "off"). So the lesson stands for any
> build that KEEPS §20.4; a build may also legitimately decline the global switch.

> **Turning master OFF must also win every ASYNC race that re-adds a DNR rule afterward.**
> "Disable rulesets + clear session/dynamic" is not enough if some other path re-adds a
> dynamic rule a moment later. Three real ones (build 27, §20 phase): **(a)** a *debounced*
> user-overlay rebuild armed just before the toggle — cancel the pending timer on OFF **and**
> re-check `master` inside the timer callback; **(b)** a feature coordinator that watches
> `masterEnabled` and, on the OFF change, *adds* a dynamic rule (the YouTube infra-allow) —
> its "off" branch must apply that allow only while master is ON, never leave a stray dynamic
> rule behind; **(c)** the **compiled bulk** (§9) applied direct-to-engine and never stored —
> OFF wipes it but ON cannot rebuild it (no local copy, and the version gate makes the next
> feed fetch skip as "unchanged"), so ON must reset the version gate to force a re-apply. The
> general rule: after OFF, assert that *nothing* — no timer, no coordinator, no feed gate —
> leaves or re-creates a live rule; and ON must be able to restore everything OFF removed.

---

## 21. UI surfaces  `[MANDATORY]`

A build must present these entry points; their design, layout and framework are
free:

- **Popup** — per-site power toggle (whitelist + reload matching tabs), YouTube
  toggle, cookies toggle, a blocked-count / popup feed, a link to options, and — if
  the element picker (§16) is included — its control.
- **Options** — whitelist manager, blocklist manager, user-cosmetics manager,
  general toggles (YouTube / cookies / acceptable-ads), version.
- **Onboarding** — an install-only page with an auto-close.
- **Block landing** — the interstitial or notification of §17.
- **Localized strings** throughout.

**Why these:** they are the complete set of user-facing entry points — a per-site
control, a full settings page, a first-run explainer, and the block surface.

---

## 22. MV3 compliance & privacy  `[MANDATORY]`

### 22.1 Everything ships in the package — no remote code, no remote resources

Two separate rules, both mandatory. The second is the one that most often trips MV3
policy review.

**(a) No remote code.** No `eval`, no `new Function`, no remotely-hosted script
executed anywhere. Network filtering is **declarative-only** — blocking
`webRequest` is never requested or used.

**(b) No remotely-referenced resources.** Nothing the browser auto-loads may point
at a remote origin — not a `<script src="http…">`, not a
`<link rel="stylesheet" href="http…">`, not `@import url(http…)`, not `url(http…)`
inside any CSS (background images, `@font-face` sources), not a remote `<img>` or
`<iframe>`. **Every script, stylesheet, font and image ships inside the extension
package** — this covers the UI pages, the injected cosmetic CSS, and the scriptlet
resources alike.

**The one allowed exception is DATA, fetched and applied locally.** The extension
may `fetch()` the sync endpoint and the versioned data assets (rule JSON, cosmetic
selector/CSS payloads) — but that content is **read as data and applied by the
extension's own code**, built into declarative rules or injected through the
scripting API. It is never handed to the browser as a URL to auto-load. A remote
CSS cache is fetched as text and injected via the scripting API; it is *never*
referenced as `<link href="http…">`.

**Why this is called out on its own:** the most common MV3 policy warning in this
line of extensions came from exactly this — an external stylesheet, web font, or
CDN URL left in a UI page, or a `url(http…)` inside injected CSS. The rule is
absolute: **if the browser would load it over the network, it must instead be
packaged.** The line that keeps you compliant is *data your code fetches and
applies* (fine) versus *a resource a page references by remote URL* (banned).

### 22.2 CSS/selectors as a fixed grammar

CSS is assembled by the extension's own code from an **allowlisted** set of
selectors/properties; values containing `url(...)`, `@`-rules, or statement-breakers
(`{ } < >`) are rejected. A remote feed can only parameterize a fixed grammar, never
deliver a raw stylesheet.

> **Animations & `@keyframes` — built locally from structured data, never a raw feed
> string** (clarified on build 27, 2026-09-04). The dynamic-styles layer (§10.4) may carry
> motion — `transform`, `transition`, `animation` and the timing properties — and full
> reveals. The client is **data-driven-ready**: a feed can push a new reveal/animation with
> **no extension update** (§11.2's "the keyframes can all change server-side"). The safety
> line is *how* the CSS is sourced, not what it does: `@keyframes` are **compiled by the
> client from a STRUCTURED directive** (`{ name, steps: { '0%': {prop:val}, … } }`) — each
> step's declarations pass the same property allowlist, the name is a validated identifier —
> so the emitted `@keyframes` is local CSS built from data, **not** a raw remote stylesheet a
> reviewer would flag. A raw `@keyframes`/CSS string from the feed is still rejected. (This
> supersedes the older "animations use a fixed *packaged* keyframe name" phrasing: a fixed
> packaged keyframe is one valid case; a keyframe compiled locally from structured feed data
> is the general, still-MV3-clean form.)

### 22.3 CSP and self-hosting enforce §22.1(b)

Lock the extension-pages CSP to self-only sources (no remote script/style origins,
no `unsafe-eval`) so that any stray external reference **fails closed** instead of
silently loading. Self-host all fonts and icons. Trim web-accessible resources to
only what pages actually need. The CSP is the backstop that turns the "remove all
external links / sheet styles" rule of §22.1(b) from a convention into an enforced
guarantee — if a remote `<link>`, font or `url(http…)` slips in, it is blocked
rather than shipped.

### 22.4 No phishing module

Phishing/safe-browsing protection is **removed and must not be present.** *(If a
build ever wants a phishing warning, it must be a purely **local, data-only** list
rendered client-side — never a server-code path and never a network lookup.)*

**Why removed:** it was dropped across the line as out-of-scope surface area and a
maintenance/liability cost; the default is its absence.

### 22.5 Verbose-log gate

All debug logging is gated behind a verbose flag (default off).

**Why.** Production consoles stay clean, but a support toggle can turn on tracing
without a rebuild.

---

## 23. The server contract the frontend conforms to  `[MANDATORY]`

The endpoint shapes are the server's spec; the client's obligation is to send and
consume exactly these and to survive the degraded path.

### 23.1 Request

The identity handle, the extension version, the install snapshot, the visited-sites
map (as counts), the last-sync time and interval, `an`/`cid`/`sid`, the echoed
`script`/`level` (under V1 these ride **inside** the handle rather than as separate
fields), the runtime id, the user allow/block lists, the user cosmetics, the
acceptable-ads flag, the migration flag, and — first-run / migration only — the raw
hardware signals.

### 23.2 Response — and the key-by-key merge rule  `[MANDATORY]`

The response carries: the identity handle, the `script`/`level` monetization/risk
state (under V1 carried **inside** the handle rather than as separate keys), the
fingerprint-version, the data-version, the sync interval, the merged visited-sites
map, the inline rules array, the system whitelist, the dynamic and zero-flicker CSS
directives, the scriptlet-exclude list, and the version+URL pairs for the cosmetic
feeds and the compiled rules.

**Local storage changes on sync in exactly two ways, and no other:**

1. **Key-by-key merge (every normal sync).** Only the keys actually present in the
   response are written; **a key the server does not send leaves its stored value
   untouched.** The client never wholesale-replaces local storage from a response.
2. **Data-version purge (only on a data-version change).** The non-preserved keys
   are dropped and the same cycle repopulates them (§5).

**Why this is critical:** it is what makes a server error harmless. A malformed,
truncated, partial, or empty response — a 500 degraded into a 200, a half-built
payload, a dropped key — can never wipe the user's rules, whitelist, identity or
settings, because absent keys are simply not touched; the extension keeps running
on its last-good local state. The **only** thing that may clear local data is an
explicit, deliberate data-version bump — never an accident. This is precisely why
the degraded path (§23.3) can safely omit the rules array, and why the data-version
constant must be pinned on every path (§5.4). It is also what §24.2 (opt-out
reversion) depends on.

### 23.3 Degraded HTTP-200 the client must survive

On a backend failure the server returns **HTTP 200** (never 500) with: the client's
own identity echoed, the **unchanged data-version** (so no purge fires),
file-served asset versions/URLs, baseline rules **for new installs only**, and a
jittered backoff. For an **existing** install the degraded response deliberately
**omits the rules array**, relying on §8.2 + §23.2 merge semantics to leave
already-applied inline rules in place.

**Why the client must be built for this:** if the client treated a 200-with-no-rules
as "wipe my rules," an outage would disable blocking fleet-wide; the range-scoped
appliers (§8.2) are what make the omission safe. Cross-ref `RULES_AND_PRIORITIES.md`
§6.

---

## 24. The monetization layer, client view  `[VARIANT: choose-one]` — none / dormant / live

Monetization is a **server-decided, server-delivered** layer. The client's total
involvement is: store and echo `script` and `level`, and apply whatever rules / CSS
/ allowlist the server sends. It **never interprets** the flag. Full ladder,
states, enable gate and reversion discipline: `RULES_AND_PRIORITIES.md` §5.

### 24.1 The three build postures

| Posture | Meaning | Client difference |
| :-- | :-- | :-- |
| **none** | No monetization machinery in the backend at all. | None — the client is identical; it simply never receives an injection bundle. |
| **dormant** | The machinery exists server-side but the enable gate/call-site is disabled. | None — the client is identical; `script` stays 0. |
| **live** | The enable gate is active; qualifying users are promoted. | Still none in code — the client just applies the extra rules / CSS / expanded allowlist the server now sends. |

**Why the client is identical across all three:** every behavioral difference is
carried as data. This is what lets the same extension be shipped with monetization
off, dormant, or on, decided entirely server-side.

### 24.2 What the client must get right for the opt-out  *(V2 only)*

> The durable Acceptable-Ads opt-out is **state 2**, a **V2-only** capability — it
> needs server-authoritative state that V1's client-rewindable handle cannot hold.
> **V1 builds have no opt-out state** (`script` 0/1 only); this whole subsection is
> N/A for them (`RULES_AND_PRIORITIES.md` §5.1, `USER_CREATION_AND_UPDATE.md` §11).

Acceptable-Ads **OFF** must make the extension **byte-for-byte identical to the
plain state**. Because the client merges key-by-key (§23.2), the *server* must send
every field the monetized state could have written, reset to its default — but the
**client's obligation** is to (a) surface the Acceptable-Ads toggle (default on),
(b) send its value every sync, and (c) faithfully apply the reset payload (replacing
the whole inline range, emptying the reset values) rather than merging it partially.
The specific server-delivered monetization rules the client must tolerate when
present (search-redirect, header-strip, tracker/ad re-allow, analytics-swap, and the
consent-handling rules — including the well-known analytics whitelist entry) are
enumerated in `RULES_AND_PRIORITIES.md` §4.2 / §5.4.

---

## 25. Per-product matrices

Two per-build tables sit here: **§25.1** the *deviation* matrix (the **conformance** axis —
how each build stands against the baseline) and **§25.2** the *structural-variant* map (the
**structure/format** axis — the different *shapes* a feature has been given, all behaving the
same). §25.1 measures *whether* a build has a feature; §25.2 records *how* it built it.

### 25.1 Deviation matrix — conformance axis

Where each shipped build stands against this baseline. Use this to find what an
existing build must add to reach the latest baseline. **27** is simply the newest
build, not a template.

**Legend:** ✅ conformant · ⚠️ present but deviates/partial · ❌ absent · — n/a ·
❓ not yet audited on this build (needs a feature check, treat as unknown not as ✅).
**`12`** is a **`Gen 0` (Legacy) build** whose *frontend* was re-platformed onto this
baseline in the 2026-08 modernization pass (`00_INDEX.md` §3 legacy track); the ✅ cells below
are that pass's result, the ❓ cells are frontend features it did not cover, and the
identity row stays Gen 0 by definition.

| Feature | § | 12 Pro | 21 North | 22 Hunter | 23 Wonder | 25 Ninja | 26 Ghost | 27 |
| :-- | :-- | :-: | :-: | :-: | :-: | :-: | :-: | :-: |
| Identity variant | 6 | Gen 0¹² | V1 | V1 | V1 | V2-A | V2-B | V2-B |
| Hardware fingerprint (incl. audio) | 7.2 | — | — | — | — | ✅ | ✅ | ✅ |
| Local-first + sync mirror | 3–4 | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| Data-version purge + preserved set | 5 | ✅ | ✅ | ✅ | ✅ | ✅ | ⚠️¹ | ✅ |
| Reserved own-rule ID contract | 8.1 | ✅ | ⚠️² | ⚠️² | ⚠️² | ✅ | ✅ | ✅ |
| Inline/compiled range split | 8.1 | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| Session/dynamic double-write | 8.3 | ✅ | ⚠️² | ⚠️² | ⚠️² | ✅ | ✅ | ✅ |
| Static rulesets enabled by default | 8.4 | ✅ | ✅ | ⚠️³ | ✅ | ✅ | ✅ | ✅ |
| Vendor self-allow at max priority | 8.5 | ✅ | ✅ | ✅ | ✅ | ✅ | ⚠️⁴ | ✅ |
| Compiled bulk, version-gated | 9 | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| Cosmetic four-layer | 10 | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| Zero-flicker two-layer pre-paint | 11.1 | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| Zero-flicker directive: data, compiled locally | 11.2 | ✅ | ✅ | ✅ | ⚠️¹¹ | ✅ | ✅ | ✅ |
| Scriptlets static + CSP fallback | 12 | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| Surrogates actually shipped | 12.4 | ✅ | ⚠️⁵ | ✅ | ❌ | ❌ | ✅ | ✅ |
| Whitelist removal tracking | 13.2 | ❓ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| Whitelisted == paused (all whitelistable paths) | 13.4 | ⚠️¹³ | ✅ | ✅ | ✅ | ⚠️⁶ | ✅ | ✅ |
| YouTube skipper (toggleable, off) | 14 | ⚠️¹⁵ | ⚠️⁷ | ✅ | ❌ | ✅ | ✅ | ✅ |
| Cookie-consent (toggleable, off) | 15 | ❌ | ❌ | ❌ | ⚠️⁸ | ❌ | ✅ | ✅ |
| Element picker *(optional)* | 16 | ❌ | ❌ | ❌ | ❌ | ❌ | ✅ | ✅ |
| Popup landing style | 17 | A | B | B | B | A | A | A |
| Visit recording {count, last-time} | 18 | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| Action-driven sync cadence | 19 | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| Onboarding not reopened on update | 20.2 | ❓ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| MV3 clean (no remote code) | 22.1 | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| Phishing removed | 22.4 | ✅ | ✅ | ✅ | ⚠️⁹ | ✅ | ✅ | ✅ |
| Monetization posture (capable — **on/off is an owner toggle**, not tracked) | 24 | ✅¹⁴ | ✅ | ✅ | ✅ | ✅ | ✅ | ⬜ |

**Notes.**
1. Ghost omitted its cookie-blocker toggle from the preserved set — a version bump would reset the opt-in (§5.2). The newest build addresses this.
2. North/Hunter/Wonder predate the reserved-ID contract: North folds user lists into server rules; Wonder uses dedicated high-number user rules + a per-tab pause; Hunter uses a client rules engine. Migrate to a fixed disjoint own-ID set + the double-write to reach baseline.
3. Hunter ships two static rulesets present on disk but not declared in the manifest — dead weight; either register or remove.
4. Ghost still self-allows a parent build's domain (fork leftover) alongside its own — strip it (§8.5).
5. North ships no surrogate resources (its monetization redirect targets are server-hosted); acceptable for its rule set but not the general pattern.
6. Ninja's zero-flicker applied regardless of the user whitelist. Correct on protected search surfaces (which cannot be whitelisted, §13.5); it is only a §13.4 concern if it also fires on a **whitelistable** site the user paused — confirm and gate if so.
7. North handles YouTube via scriptlets + an allow rule only (no player skipper / no toggle).
8. Wonder has a cookie-notice realm toggle but not the full three-layer cookie feature (§15).
9. Wonder kept a local phishing-warning list; backend phishing was removed. Baseline is full absence, or local-data-only if retained (§22.4).
   *(Former note 10 — a per-build "Hunter dev-forces monetization" state — removed: monetization on/off is an owner toggle, not documented state; see the §25 posture row.)*
11. Wonder delivers the pre-hide as a server-built **raw CSS string** injected through the scripting API, rather than a structured directive compiled locally from an allowlisted grammar. Functional, but the structured form (§11.2) is preferred — it is safer for MV3 review and lets the format differ per build while behaving identically.
12. **12 is `Gen 0` (Legacy, pre-family)**: an AES-encrypted `syncuid` handle on a pre-family schema (ad-counter columns, `conv 0–2`), no fraud stack — its own permanent lineage, **never** migrated (`00_INDEX.md` §3 legacy track, `00_GOAL.md` §4, `USER_CREATION_AND_UPDATE.md` §11). The ✅ frontend cells are the 2026-08 modernization; the identity/DB/backend cells are Gen 0 by definition and are covered in the backend docs.
13. 12's zero-flicker was made **unconditional** to fix the allowlist-suppression bug (§11.2 trap). Same status as Ninja's note 6: correct *because* 12's pre-hide only targets protected search surfaces (§13.5), which cannot be whitelisted — but it is gated on nothing, so if a future directive ever targets a **whitelistable** host, a user's pause would not suppress it. Preferred end-state: gate on the user's own pause/whitelist and let the carve-out keep the search hosts out of it.
14. 12 is a **legacy build still in use**; its backend can **dev-force** `script=1` (hardcoded, bypassing the organic enable gate) — a manual owner toggle, like any build. Verify the prod file to read the current mechanism, and treat monetization on/off as business logic, not documented state (`RULES_AND_PRIORITIES.md` §8 — the Organic enable-gate row + the owner-toggle note).
15. 12 gained its YouTube mode on 2026-09-14 (toggleable, default OFF): a `youtube` static ruleset shipped disabled and toggled at runtime, plus a YouTube-scoped player skipper (skip-click / fast-forward, dialog dismissal). The flag joined the preserved set in the same change (the §5.2 load-bearing rule) and is additionally sync-mirrored — a deliberate one-boolean widening of the §4.1 tight set so a reinstall keeps the opt-in. ⚠️ because the OFF half deviates from §14 mechanism 1: there is no standalone YouTube-infra allow rule while the mode is off — add it to reach baseline.

### 25.2 Structural-variant map — structure/format axis

> **This is a divergence *record*, not a template.** It documents how existing builds have each
> **structured** a feature — the file layout, the data format, the module tree — to show that a
> feature's *shape* is genuinely free: the same behavior has already been built several different
> ways, all working. Its purpose is the **opposite** of a copy-menu — it exists so a new build
> can see the possibility space, understand the feature and its **constraint** (the behavioral
> contract every shape must honor), and **invent its own** shape. Reading a row and cloning that
> build's structure defeats the anti-template rule (§0.2). *"Converge the behavior, diverge the
> shape"* (`00_GOAL.md` §2) — this table is the evidence the shape is diverge-able.
>
> Read it as: **feature → its constraint (converge on this) → the shapes already proven (so invent
> the next).** A row that reads "uniform across builds" is an honest finding — that feature has no
> proven variety yet, so it is exactly where the "write your own" work still needs doing.

*Grounded in a code survey of the seven documented frontends (12 / 21 / 22 / 23 / 25 / 26 / 27) on
2026-09-03 — including **12 (Gen 0 legacy)**, whose re-platformed frontend contributes some of the
most distinct shapes (a single consolidated scriptlet library + DB, an *unconditional* zero-flicker,
a two-message (`GET_PLAN`/`YT_STATUS`) storage-driven message surface) — evidence that frontend *shape* is independent
of identity lineage. Each row: the **constraint** any shape must honor, then the **distinct shapes**
already proven, with the builds using each. "Uniform" is an honest finding — those builds share
the shape (often because a newer build was seeded from an older one), so that is where the "write
your own" work still remains. **27 currently mirrors 26 (Ghost) on several axes** — noted inline.*

| Feature (§) | Constraint — converge on this | Proven shapes — diverge; invent the next |
| :-- | :-- | :-- |
| **File tree / modules** (§2) | MV3 worker (ES-module *or* `importScripts`); content script self-injects once (re-inject guard); every feature reachable from the worker | **ES-module `import` of per-concern helpers** (21, 22, 23) · **`importScripts()` of per-concern modules** into the worker (25) — or a **shared `engine/` set loaded into *both* the worker and content-script worlds** (12) · **named kernel global** wiring contracts→features→core (`self.Ghost`, 26) · **composition-root worker** where each feature is a self-registering side-effect `import` (27) |
| **Storage-key scheme** (§3) | local-first (`storage.local` = truth); sync-mirror subset ⊆ preserved set; `an`/`cid`/`sid` verbatim | **Multi-prefix by feature-family** (`abn*`/`north*`/`back*`/`user*`+`*Rem` twins, 21; two-namespace `hunter*`+plain, 22; four-style hybrid, 23; two-family `ninja_`+plain, 25; `pro`-camelCase caches + bare-canonical identity keys, 12) · **single prefix + central frozen registry** (`ghost_`/`GHOST_KEYS`, 26; `ab_`/`STORAGE_KEYS`, 27) |
| **Static DNR rulesets** (§8) | reserved own-ID ranges + inline/compiled boundary; static sets enabled by default; vendor self-allow at top priority | Partition axis + file count both vary: **by shard-number** (`rules-1/2/3`, 21; `net-1..5`, 26) · **by category** (`snare-<cat>`, 22; `defaultlist/privacy/…`, 25; `ruleset-<cat>` default/badware/privacy + a disabled `youtube` opt-in set (2026-09-14), 12) · **by realm** (basic/social/trackers/locations-sharded, 23) · **by theme** (`net-ads/privacy/filters/…`, 27). ID schemes differ (contiguous-global-sharded in 21 & 12 vs per-file; 12's later `youtube` add-on set restarts ids per-file); 12 puts the inline/compiled boundary in the **dynamic** id space (server-inline `[30,11000)` verbatim, compiled bulk `[11000,61000)`; since 2026-09-15 the reserved low ids below both bands also carry a **persistent system-whitelist allow** rebuilt each sync from the server allowlist — the §13.3 network half, which 12's client previously did not build, its allowlist cache feeding only the content-side bypass) |
| **Scriptlets** (§12) *(widest space)* | scriptlet **code is always static/packaged** — never fetched/eval'd; only the domain→`[name,args]` map is data; params via `dataset`; CSP fallback to `executeScript(world:MAIN)` | **One code file per scriptlet** + one consolidated domain-map JSON (21: 28 files; 22: 36) · **family-bundled** min.js (8 bundles by family, 23) · **one consolidated code library** + a consolidated arg source — per-scriptlet SW-global modules (25), or one big `scriptlet-db.json` (12: a `library.js` of `f0..fN` + a ~31k-key DB) · **one JSON file per scriptlet** `presets-<name>.json` + generated index (26, 27: 40 files) |
| **Zero-flicker / local-css** (§11) | directive is **data compiled to CSS locally through a fixed packaged grammar** — never a remote/executable stylesheet; Layer 1 reads `storage.local` directly at `document_start`; gate on user pause (bar protected surfaces) | **Typed directive compiled locally**: selector-array-or-object→`@keyframes abnInternalFade` (21); array + hide/unhide map (22); `local_css` domain→`{hide,style}` compiled-once (25); `{v:1,blocks:[{on,ops:[rule\|raw]}]}` (26, 27); `{hostPatterns:[{match,hide,cloak,fade}]}` with an **unconditional** host-pattern gate — 12 (pre-hide fires on match regardless of the ad-block bypass; §13.4/§13.5) · **raw CSS string** (`google_css_raw`, Layer-2 inject) — 23, the §11.2 less-preferred form. **26 & 27 uniform** (27 inherited the typed schema) |
| **Cosmetics** (§10) | four layers (generic/specific/procedural/user); works backend-down; content scripts skip on whitelist | **All-JSON-in-storage** (21) · **JSON + one ~1 MB packaged `cosmetic.css` `<link>`** (22) · **per-realm CSS files** `cosmetic_css/{realm}.css` insertCSS-as-files (23) · **raw CSS files + static specific** (25) · **raw generic CSS (+ a server raw-CSS string) + a JSON hide/unhide specific map** (12) · **raw CSS files + compiled/indexed `specific-map-*.json`** (26, 27 — uniform) |
| **Surrogates / WAR** (§12.4) | if a redirect rule targets a surrogate it **must ship + be web-accessible** (else degrades to block-with-error) | **`surrogates/` folder** named after the replaced resource (22: 27 files) · **flat `web_accessible_resources/` dir** (26, 27: 37 files, `vendor_product.js`; 12: 24 files, uBO/AdGuard canonical names) · **no surrogate library**, all rules block, WAR exposes only internal assets (23) · **⚠️ 21 ships redirect DNR rules with NO bundled targets** — a real gap, not a variant (audit-flag) |
| **Message-types / sync** (§19) | message types are per-build internal names; sync is action-driven | **Flat camelCase `type`/`action` verbs** (21, 25) · **camelCase + `Msg`-suffix router set** (22) · **dot-namespaced `{eventName}`** `blocker.updateStatus` (23) · **namespaced kebab-case in a frozen central registry** (`ghost:`/`GHOST_MSG`, 26; `ab:`/`MSG`, 27) · **`SCREAMING_SNAKE` with a two-message runtime surface** (`GET_PLAN`, plus the `YT_STATUS` content-script diagnostic added 2026-09-14), UI driven entirely by `storage.onChanged` — no UI messages (12, still the most minimal surface) |

> **Two findings worth acting on** (beyond the map itself): **(1)** build **27 currently mirrors 26**
> on the typed local-css schema, the compiled specific-maps, the per-scriptlet presets DB, the single-
> prefix registry, and the flat WAR — so 27 has the *least* structural distance from a sibling and is
> where fresh divergence is most owed. **(2)** build **21 ships redirect rules with no surrogate targets**
> (§12.4 constraint violated) — a genuine defect the survey surfaced.

---

## 26. Conformance checklist

A new build is conformant when every `[MANDATORY]` item is present, every
`[TOGGLEABLE]` item ships wired to its switch at the stated default, each
`[VARIANT: choose-one]` has a recorded choice, and `[OPTIONAL]` items are
consciously decided. This doubles as the end-of-build test plan. Fill it in per
build.

> **Who runs what (`00_INDEX.md` §7).** Every item from *Platform & storage* down to
> *Compliance & contract* is **`[AGENT — offline]`** — provable by reading the code, a
> `grep`/static audit, or a logic simulation. The **Runtime smoke test** block is
> **`[OWNER — manual]`** — it needs a real browser, so the agent prepares the steps and
> expected results and hands them over; it never loads the extension or navigates itself.

> **Common vs. variant.** The frontend is deliberately **identity-agnostic** — the
> extension code is byte-identical across identity variants and monetization
> postures (§24.1), so nearly every item below is **Common (all builds)**. Only two
> are variant-specific, tagged **[V2 only]** inline: **hardware collection** (§7.2)
> and the **opt-out reset** (§24.2). A **V1** build skips those two and runs
> everything else unchanged. The identity-variant choice itself is the `[VARIANT]`
> item under *Identity*.

**Platform & storage**
- [ ] MV3, service-worker background, minimal capabilities, declarative-only blocking, three content-script injection points, static rulesets (default-on + the disabled optional ones) — §2
- [ ] Local storage is the source of truth; SW listeners registered synchronously; one clear settings/cache split; popup feed in session storage — §3
- [ ] Sync mirror = tight set; heal-before-build; hot value deferred; mirror ⊆ preserved — §4
- [ ] Data-version purge echoed on every path; preserved set includes **every** setting; stamp written last — §5

**Identity `[VARIANT]`**
- [ ] Chosen: ☐ V1 blob ☐ V2-A derived ☐ V2-B token — client obligations for that variant met — §6
- [ ] Attribution harvested from the store URL; `an/cid/sid` kept verbatim — §7.1
- [ ] Hardware set collected in-memory-only *(V2 only; skip for V1)* — §7.2

**Rules & engines**
- [ ] Own rules in a fixed, documented, disjoint low-ID set; server rules in higher non-overlapping ranges; inline/compiled boundary honored on both ends; range-scoped appliers — §8.1–8.2
- [ ] Instant-feedback + restart-persistence for user list changes — §8.3
- [ ] Vendor self-allow at max priority (request + initiator); any parent-fork self-allow stripped — §8.5
- [ ] Compiled bulk version-gated, applied direct-to-engine, never stored — §9
- [ ] Cosmetic four-layer; dynamic feeds version-gated; unhide beats hide — §10
- [ ] User tier at a **low priority above the blocklist and below the injection layer** (safe value `200`/`100`, canonical in `RULES_AND_PRIORITIES.md` §4.2); never above the injection band — §8.1, `RULES_AND_PRIORITIES.md` §4.2
- [ ] Zero-flicker: **Layer 1 = content script reads the directive straight from `chrome.storage.local` at `document_start` and injects (no worker round-trip)**; Layer 2 = worker USER-origin reinforcement from an in-memory cache; **server directive delivered as data and compiled locally** (format is yours, behavior fixed) — §11
- [ ] **Zero-flicker is NOT gated on the injection allowlist** — it runs on the protected search surfaces (kept out of every allowlist by the carve-out) and is suppressed only by the user's own pause/whitelist on whitelistable sites — §11.2 trap, §13.4
- [ ] Scriptlets static, main-world + CSP fallback, deduplicated; surrogates actually ship — §12
- [ ] **Static rulesets carry no block/redirect on the injection surfaces** (search TLDs, partner-feed hosts, injected analytics/quality hosts) — a stale packaged rule fights the injection in the pre-first-sync window — `RULES_AND_PRIORITIES.md` §9.1
- [ ] System whitelist applied as a high-priority allow, **rewritten every sync**, and cached locally so content scripts (no rule-engine access) also skip cosmetics/scriptlets on those hosts; large lists chunked — §13.3
- [ ] Whitelist == paused on **every** whitelistable path (zero-flicker included, per the §13.4 nuance); removal tracking; protected-domain carve-out respected — §13

**Features**
- [ ] YouTube `[TOGGLEABLE]` default OFF (infra-whitelist when off) — §14
- [ ] Cookie-consent `[TOGGLEABLE]` default OFF — §15
- [ ] Element picker *(optional)* — included? ☐ yes ☐ no — §16
- [ ] Popup landing `[VARIANT]`: ☐ A self-closing (recommended) ☐ B notification — §17

**Telemetry, lifecycle, surfaces**
- [ ] Visit recording `{count, last-time}`, cap ~50 / prune lowest by decay, counts-only on the wire — §18
- [ ] Action-driven sync cadence (interval a few hours; shorter while pending) — §19
- [ ] Install seeds defaults (acceptable-ads on, master on); onboarding NOT reopened on update; uninstall URL fail-safe; kill-switch reloads only the web page — §20
- [ ] All user surfaces present and localized — §21

**Compliance & contract**
- [ ] No remote code **and no remotely-referenced resources** — every script, stylesheet, font and image packaged; no `<link href=http…>` / `<script src=http…>` / `@import` / `url(http…)`; server content is fetched as data and applied by code, never referenced as a URL; CSS a fixed grammar; CSP locked (fails closed); no phishing module; verbose gated — §22
- [ ] Degraded HTTP-200 survivable (unchanged data-version; merge preserves inline rules) — §23
- [ ] Monetization `[VARIANT]`: ☐ none ☐ dormant ☐ live — client echoes `script`/`level` only; Acceptable-Ads toggle sent every sync; **[V2 only — N/A for V1]** opt-out reset (state 2) applied faithfully — §24, §24.2
- [ ] **No shipped comment or symbol names another build or describes the monetization/injection scheme** (correlation + disclosure risk); a legacy port has had any ported-in sibling/scheme comments scrubbed — §0.2

**Runtime smoke test** — `[OWNER — manual]` *(a real load-and-navigate pass — the static checklist above does not prove the build actually runs; the **owner** loads unpacked and runs these, the agent only prepares the steps and reads back what the owner reports)*
- [ ] Loads unpacked with no manifest/console errors; the worker boots and logs its version
- [ ] Navigate to a live search page → the pre-hide fires **before first paint** (no flash) and reveals within the cap; the content-script log shows the direct-storage Layer-1 injection
- [ ] The worker console shows a sync firing (due-gated on navigation) and the dynamic + session DNR rules applied, with no exceptions
- [ ] On a few ad-heavy sites, **cosmetics VISIBLY hide** the ad slots — not just "the CSS was injected" but the boxes are gone with **no reserved empty space**; spot-check a `www.`-prefixed host and a site with `:has()`/`:style()` rules (the §10.1 traps). A leftover empty box = a stale/missing container selector, not (usually) an engine fault
- [ ] **Every server feed URL returns real data** (compiled rules, cosmetic feeds) — not empty, HTML, or 404; the compiled cache self-generates on first request — `DATA_SOURCES.md` §6
- [ ] A **paused whitelistable** site truly shows ads (no network blocking, no cosmetics, no scriptlets, no flicker); a **protected search** surface still gets the pre-hide — §13.4
- [ ] Manifest requests only permissions actually used (drop unused, e.g. `alarms` when sync is navigation-driven — §19)

**Record**
```
Build: ____________     Backend endpoint: ____________________
Identity: ☐ Gen 0 Legacy  ☐ V1  ☐ V2-A  ☐ V2-B
Monetization: ☐ none  ☐ dormant  ☐ live
Popup landing: ☐ A  ☐ B
Legacy modernization pass? ☐ n/a (new build)  ☐ yes — frontend re-platformed, identity frozen
```

---

*Baseline specification — a completeness guideline and test plan, implementation
independent by design: it fixes what an extension must do and why, and leaves how
to build it entirely open. Companion docs: `RULES_AND_PRIORITIES.md` (rule ladder &
`script` flag), `USER_CREATION_AND_UPDATE.md` (backend identity & dedup),
`FRAUD_DETECTION_V5.md` (VM / risk / behavior).*
