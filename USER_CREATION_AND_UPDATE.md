# User Creation & Update — the canonical specification

**How a user record comes into existence, what identity it is keyed on, what is written at creation, and every path that mutates it afterwards — for both generations of the identity layer (V1 and V2).**

This is the **baseline specification**. It describes the shared model — the record, the identity contract, the resolution cascade, the create/update sequences and the invariants that must hold — independently of any particular product or build. It states what the system *is*, not what any given implementation currently does.

> **Both generations are supported.** V1 and V2 are not a migration path; they are two live designs with different trade-offs. §11 is the selection guide.
>
> Companion documents: the rule tiers, priority ladder and `script` flag lifecycle are specified in **Rules & Priorities**. The fraud tiering, VM evaluation, behavioural classifier and the money merge are specified in the **Detection spec**. This document owns *the record and its writers*; where creation consults the detection layer, it names the gate and the column and defers the internals.

---

## 0. Naming convention used in this document

Every build renames its transport keys with its own product slug (`<slug>_fp`, `<slug>_sync`, …) and its own table name. Those names carry no meaning. This document uses **role names** throughout:

| Role name | Direction | What it carries |
| :-- | :-- | :-- |
| `identity_handle` | in **and** out | The client's identity. Its *type* is the defining difference between V1 and V2 (§3). |
| `format_version` | out | Version of the identity format itself. |
| `data_version` | out | Storage-contract pin. A mismatch makes the client purge its local storage — it MUST be echoed identically on every path, including degraded ones. |
| `sync_interval` | out | Milliseconds until the next sync. Server-controlled. |
| `client_version` | in | Build version of the client (the inbound wire role). Stored in the `version` column (`DB_SCHEMA.md` §2). |
| `visited_map` | in | `{ domain => count }` of browsing activity since the last sync. |
| `install_sites` | in | Domains open in tabs at install time. Empty for a headless/automated install. |
| `an` / `cid` / `sid` | in | The attribution triplet: ad network, click id, sub-id. **Never** prefixed. |
| `hardware_data` | in | Hardware/telemetry object (cores, RAM, GPU vendor + renderer, language, colour depth, timezone, canvas, screen, touch points, audio fingerprint, WebGL extension count, webdriver flag, battery). **V2 only.** |
| `user_allow` / `user_block` / `user_cosmetics` | in | The user's own lists. `user_cosmetics` exists only in later builds. |
| `acceptable_ads` | in | Tri-state: `true`, `false`, or absent/`null`. |
| `migration_hint` | in | Client-set boolean asking the server to backfill newly-collected telemetry. **V2 only.** |
| `users` | — | The single user table. One row = one install. |

Wherever this document says "the endpoint", it means the single sync endpoint that serves every request: install, every periodic sync, and every rule/asset refresh. **There is no separate "register" call.** Creation is a side effect of the first sync that carries enough information.

---

## 1. In plain English

The whole thing rests on five ideas:

1. **One endpoint, one row.** The client posts the same payload on install and on every sync. The server decides "is this somebody I already know?" — and if not, it creates the row. Creation and update are two branches of one request handler, which is why they share almost all of their code.

2. **The answer to "do I know you?" is the entire design.** Everything else — risk scoring, conversion, monetization — is downstream of identity resolution. Get resolution wrong in the merging direction and two people become one row; get it wrong in the splitting direction and one person becomes many rows, gets counted many times, and possibly gets paid for many times.

3. **The client holds a handle, not a session.** There is no login, no cookie of record, no server session. The client keeps one opaque string and sends it back forever. **What that string contains is the V1/V2 fault line:** in V1 it is *encrypted state* the server reads back and trusts; in V2 it is a *stateless key* into the row, and all state lives server-side.

4. **A handful of columns decide everything the user experiences.** `level` (risk), `script` (monetization state), `conv` (payment state) and `updates` (sync counter). Each has exactly one owner and one moment at which it is decided. Most bugs in this layer are two writers disagreeing about one of these four.

5. **The row outlives the client's storage, and the client's storage outlives the row's decisions.** A storage wipe must be recoverable (the same person must resolve back to the same row); a decision once taken at install must not be silently re-taken on a later sync. Those two requirements pull in opposite directions and every design choice below is a position on that tension.

If you remember one sentence: **V1 trusts the client to carry its own state; V2 gives the client an opaque key and keeps the state on the server — and everything else follows from that.**

---

## 2. What a user record is

### 2.1 Field groups

One row per install. The columns fall into seven groups; the group tells you who writes it and when.

| Group | Columns (role) | Written at | Mutable after? |
| :-- | :-- | :-- | :-- |
| **Identity** | `id` (surrogate key), identity token/handle, stable hardware signature | creation | Token: **never** (V2-B) / recomputed on some paths (V2-A) / not stored at all (V1). Signature: never. |
| **Attribution** | `an`, `cid`, `sid`, `typetag`, `mkt` | creation | Never. `typetag` = ISO-week + last digit of year + `-` + `an` (or `xx` if organic) + `-` + product number. |
| **Network / risk** | `ip`, `provider`, `organisation`, `hostname`, `proxy`, `type`, `risk`, `asn` | creation | `ip` refreshed on every sync (roaming). The rest are a snapshot of the install moment and are **not** refreshed. |
| **Hardware / telemetry** | cores, RAM, GPU vendor + renderer, language, colour depth, timezone, canvas hash, screen resolution, user agent, touch points, battery, webdriver flag, WebGL extension count, audio fingerprint | creation (**V2 only**) | Backfilled once by the telemetry-migration path (§7). Otherwise never. |
| **Decision state** | `level`, `conv`, `conv_date`, `script` | creation | `level`/`script`: rewritten on every sync. `conv`: **frozen at creation**; only the post-install sweep and the behavioural layer may change it. `conv_date`: the moment `conv` received its **current** value — moves only when `conv` genuinely transitions; every `conv` writer maintains it (§7). |
| **Fraud / behaviour** | `is_vm`, `vm_type`, `diagnostic_date`, `device_hash`, `fraud_score`, `fraud_flags`, `fraud_checked`, `decision`, `behavior_score`, `behavior_flags`, `behavior_hash`, `decided_date`, `flagged` | creation (**V2 only**) | Owned by the detection layer. See the Detection spec. |
| **Activity / lifecycle** | `created_date`, `updated`, `updates`, `version`, `visit`, `whitelistedDom`, `blockedDom`, `cosmeticDom`, `duplicate`, `enabled`, `acceptable_ads_disabled`, `disabled`, `updated_at` | **mixed — per column** (`updated`/`updates`/`version`/`visit`/user-lists on sync; `created_date` creation-only; `enabled` **enable-gate only**; `disabled` **uninstall only**; `acceptable_ads_disabled` opt-out only; `updated_at` **DB-maintained** — no code writes it) | See §7 for the full writer map. |

*The two canonical timestamps.* `conv_date` (sits directly after `conv`) and `updated_at` (the last column) are part of the canonical record. At creation **neither is written by code**: both fire by column DEFAULT in the same statement as `created_date`'s, so `conv_date == created_date` exactly — which is correct both for pay-at-install (`conv` already decided in the INSERT) and for delayed conversion (stamped later), with no config gating. `updated_at` is maintained entirely by the database (`ON UPDATE current_timestamp()`) and is **distinct from the heartbeat `updated`**, which is code-written on sync and feeds the survival metrics. Writer rules for both: §7.

### 2.2 The four state columns — and who owns each

*Scope note:* this doc's "four" are the **handle-carried working-state** columns — `level`, `script`, `conv`, `updates`. `DB_SCHEMA.md` §3 groups a different "four" for the **schema/decision** view (`level`, `script`, `conv`, **`decision`**), and `FRAUD_DETECTION_V5.md` §14 a **detection-state** set. These are different columns for different lenses — `updates` is the sync counter, `decision` is the behaviour verdict — not a disagreement.


| Column | Meaning | Decided | Owner | Re-derived on sync? |
| :-- | :-- | :-- | :-- | :-- |
| `level` | Risk digits. `"0"` = clean; otherwise a concatenation of digits, one per triggered signal. Its **only** consumer contract is `level == 0` as the gate for monetization and install-time payment. | creation | risk evaluator | **No** — it is carried forward and rewritten with the same value. A row's risk is never re-scored from a later IP. |
| `script` | Monetization state: `0` not enabled (default), `1` enabled, `2` explicitly opted out. | creation (`0`) then transitions on sync | the enable gate + the acceptable-ads opt-out | **Yes** — this is the one state column intentionally mutable on sync. |
| `conv` | Payment state: `0` pending/deferred, `1` converted, `2` blocked, `3` special, `4` expired/forfeited. | **creation, once** | creation gate; afterwards only the post-install sweep / behavioural layer | **No** — never in the sync UPDATE's SET list; the post-install sweep and (under enforcement) the behavioural layer are its only other writers. |
| `updates` | Sync counter. Feeds the enable gate and the behavioural collection cap. | creation (`0`) | the sync path | Incremented by exactly 1 per resolved request. |

**The `conv`-is-frozen rule is load-bearing.** It is what makes a reconstructed returning user safe: whichever tier of the cascade matched, the user inherits a payment decision already taken, and no second payment can fire from the sync path. In V1 this holds because the sync path simply never touches `conv`. In V2 it holds by explicit design, and the legacy sync-time re-check that used to violate it has been retired. `conv_date` is the freeze's audit trail: because a same-value rewrite never advances it (§7), it always answers *which writer set the current `conv`, and when*.

### 2.3 The identity contract

Six invariants. A build that violates any of them will silently fragment or silently merge users.

1. **The handle is issued by the server and stored by the client.** The client never invents one.
2. **The handle round-trips byte-for-byte.** Whatever the client sends back must resolve to the row it was issued for, forever.
3. **The handle survives a storage wipe** — either because the client mirrors it to sync storage and heals it back, or because the server can re-derive/re-recognise the row from other evidence (the resolution cascade, §6.3/§6.4).
4. **A handle the server does not recognise is not an error.** It is either a new install or a lost row; the correct response is to fall through the cascade, and at the bottom, to create.
5. **Resolution must be deterministic and total.** Every request ends in exactly one of four outcomes: `EXISTING` (matched a row), `CREATE` (inserted a row), `WAIT` (V2 only — insufficient evidence, respond with rules and create nothing), or `ABORT` (a genuine inconsistency).
6. **The identity mechanism is immutable for the life of the build.** The handle format and the uniqueness / dedup keys — and the columns and indexes that back them — are chosen once, at the build's creation, and **never** changed by an update. Every existing row is keyed by them; changing them orphans the entire user base in the database. A build is therefore **never migrated between generations**: a higher-generation product is a *new* build with a fresh user base, not an upgraded one (`BACKEND_ARCHITECTURE.md` §8.3).

---

## 3. The two generations

### 3.1 V1 — the handle *is* the state

The handle is an **encrypted state blob**. The server serialises a small key/value string and encrypts it with a symmetric authenticated cipher (AES-256-GCM; the wire form is `base64(iv ‖ tag ‖ ciphertext)`), and the client stores the ciphertext.

The payload carries: row id, `script`, `level`, `conv`, `updates`, `typetag`, creation timestamp, market.

On every sync the server decrypts it and **reads its working state out of the blob**, not out of the database row. It then writes that state back to the row.

Consequences, all of them direct:

- **The database row is downstream of the client's copy** for `level`, `script`, `conv` and `updates`. A stale blob rewinds the row. A lost blob loses the state.
- **A single read serves the whole request.** No row fetch is needed on the hot path — the sync update is a blind `UPDATE … WHERE id = ?`.
- **Tampering is prevented by cryptography, not by architecture.** Confidentiality and integrity rest entirely on the cipher and on the key never leaking. There is no server-side cross-check that the decrypted `script` matches the stored one.
- **Loss of the blob means loss of identity** unless the cascade re-recognises the user (§6.3), because there is no other client-held key.

### 3.2 V2 — the handle is an opaque key

The handle is an **opaque token**. It carries no state. Every value the request needs is read from the `users` row, which is fetched by the token.

This inverts every consequence above: the row is authoritative, a stale client cannot rewind server state, there is nothing to decrypt, and one extra `SELECT` per request buys the whole property.

V2 exists in **two variants**, differing only in how the token is produced:

| | **V2-A — derived handle** | **V2-B — minted handle** |
| :-- | :-- | :-- |
| Token value | `HMAC-SHA256` over hardware components **+ `an`/`cid`/`sid` + IP**, keyed by a hash of the app key | `bin2hex(random_bytes(32))` — 64 lowercase hex chars, CSPRNG |
| Recomputable from a row? | Yes — and several code paths do recompute it | **No, by construction** |
| Written | at creation, **and re-written** by the telemetry-migration and reconstruction paths | **once, at creation, never again** |
| Uniqueness comes from | the input components (collision-prone by design: two identical machines with no attribution and the same IP collide) | 256 bits of entropy + a UNIQUE index on the column |
| Reconstruction key | the token itself (recomputed and looked up) | a **separate** stable hardware signature column |
| Fail mode | Recompute + overwrite desynchronises the client's copy from the row → the next lookup misses → fall-through or a duplicate row | Mint failure is **fail-closed**: no row is created |

**Why V2-B exists.** V2-A conflates two jobs in one value: *identifying* a returning user and *describing* the device. Because the token is a function of its inputs, any path that has fresher inputs is tempted to recompute it — and V2-A has several such paths (telemetry backfill, reconstruction fallbacks). Each recompute writes a new value into the row while the client still holds the old one. Because the token also folds in the IP and the attribution triplet, a genuinely returning user on a new network or a new campaign produces a *different* token and cannot be recognised by it at all.

V2-B separates the two jobs:

- **`fingerprint`** — a minted, meaningless, write-once token. Its only job is to be echoed. Because it means nothing, nothing can be tempted to recompute it.
- **`device_sig`** — a hash of *stable* hardware only (GPU vendor + renderer, cores, RAM, colour depth, touch points, canvas, audio). Deliberately **collidable**: the same machine yields the same value. It is a *reconstruction hint*, never an identity. Uniqueness never comes from it.

Note the deliberate contrast with the **fraud** device hash, which is a *different* column computed from a *different* component set — it folds in the volatile signals (screen resolution, normalised user agent, timezone, WebGL extension count) precisely because it is looking for burst-cloning, not for continuity. **Identity signature and fraud hash must never be unified.** They want opposite volatility.

### 3.3 The difference in one line

> **V1:** the client's copy is the state, and the row is a mirror of it.
> **V2:** the row is the state, and the client's copy is a pointer to it.

---

## 4. Request → identity: what arrives

| | V1 | V2 |
| :-- | :-- | :-- |
| Identity input | encrypted state blob | opaque token |
| Hardware input | none | `hardware_data` object |
| Attribution | `an`/`cid`/`sid` in the body, with a **cookie fallback** when a field is null | identical |
| Activity | `visited_map`, `install_sites` (both capped) | identical (`visited_map` merged rather than replaced, §6.7) |
| User lists | allow + block | allow + block (+ cosmetics in later builds) |
| Acceptable-ads | — | tri-state flag |
| Extra | a **client-echoed monetization flag** (§5.4 step 7) | a `migration_hint` boolean |

**Attribution is normalised before use in some builds and not in others.** Where it is normalised, `an` is stripped to alphanumerics and `cid`/`sid` to alphanumerics plus hyphen. This is worth stating because these three values are (a) written verbatim into the row, (b) part of V2-A's identity token, and (c) V2-B's Tier-1 reconstruction key. **Normalise them, and normalise them identically everywhere**, or the same click produces two different identities depending on which path read it.

**The cookie fallback is a correctness feature, not a nicety.** When the body carries no attribution, the server falls back to cookies before deciding the user is organic. An organic classification is sticky (it lands in `typetag` as `-xx-` and permanently disqualifies the row from the monetization enable gate), so a lost attribution is not recoverable later.

---

## 5. Version 1 — specification

### 5.1 Dispatch

```
if identity_blob present:
    decrypt it
    if decryption succeeded:
        → UPDATE path  (§5.5)   [when the payload contains a row id]
        → createUser   (§5.4)   [when it does not]
    else:
        → createUser   (§5.4)
else:
    → createUser       (§5.4)
```

A blob that fails to decrypt is treated exactly like no blob at all — the user falls through to creation, where the cascade gets a chance to re-recognise them. **This is the only self-healing in V1, and it is why the cascade matters so much.**

### 5.2 There is no create-guard

V1 creates a row on the very first contact, unconditionally. It needs nothing but a request. This is the single biggest behavioural difference from V2 (§6.2) and it is the reason V1 install counts include every scan, probe and warm-up hit that ever reached the endpoint.

### 5.3 The resolution cascade

Evaluated in order, inside `createUser`, before any thought of inserting:

| # | Tier | Predicate | On match |
| :-- | :-- | :-- | :-- |
| 1 | **Attribution pair** | `cid` **and** `sid` both non-empty and both match a row | Adopt the oldest matching row |
| 2 | **Network + install context** | `ip` **and** `an` (NULL-safe) **and** `install_sites` (serialised, exact byte match) all match a row | Adopt the oldest matching row, increment `duplicate` |
| 3 | **Create** | — | Insert |

Both adoption tiers do the same thing: read the row's state, build a temporary blob from it, run the shared update logic against it, increment `updates`, write the row, and return a freshly built blob. **Adoption therefore re-sources state from the database** — which is the one place V1 does *not* trust the client. The asymmetry is worth knowing: a user recognised by cascade gets DB-sourced state; a user recognised by their own blob gets blob-sourced state.

**Two structural notes on the cascade:**

- **Tier 1 requires both `cid` and `sid`.** A row with a `cid` but no `sid` is unreachable by this tier. It is also, deliberately, not keyed on `an` — so the same click arriving under a different network label still matches.
- **Tier 2's `install_sites` comparison is an exact match on the serialised array.** It is doing double duty as a same-session discriminator: identical install tabs from the same IP and network is read as "same install returning", any difference as "different install". This is brittle by nature — tab order and content both change the bytes — and it is the tier V2-B removed outright.

**Both tiers have an abort path with a sharp edge.** If the tier's cheap existence check matches but the subsequent full fetch returns nothing (a race, or a row deleted between the two queries), the handler logs and **terminates the request with no response body**. The client sees a successful HTTP status and an empty payload. Any V1 build should treat this as a defect to fix, not a behaviour to preserve — the correct action is to fall through to creation.

### 5.4 Creation sequence

1. Initialise `conv = 0`, `level = "0"`, `script = 0`, `duplicate = 0`, `updates = 0`.
2. Fill missing `an`/`cid`/`sid` from cookies.
3. Run the resolution cascade (§5.3). Any match returns and this sequence ends.
4. **Risk lookup** — one external call on the client IP returning provider, organisation, hostname, proxy flag, connection type, numeric risk score and ISO country code. Market is taken from that country code, falling back to an edge-provided market header.
5. **Risk evaluation** → `level` digits and possibly `conv = 2`. One digit is appended per triggered signal (build integrity, hostname/organisation checks, proxy-or-score, banned provider, desktop-Linux UA); if none trigger, `level = "0"`. *The signal set and its rationale belong to the Detection spec.*
6. **Conversion gate** — the only place a payment can fire at install:
   `an` present **and** `cid` present **and** `level === "0"` **and** `conv !== 2`.
   All four are required. Organic installs (no `an`) and any non-zero `level` never pay.
7. **Monetization seeding from a client-echoed flag — deliberate.** If the request's echoed monetization flag equals `1`, `script` is set to `1` at insert. This is V1's **only** live route to `script = 1`, and that is the design: the periodic enable gate is inert (§5.5), so the echoed flag *is* the mechanism, not a leak past it.
   The value originates server-side — the previous response echoed the row's `script` back, and the client's job is only to carry it. **Know what that costs:** the flag is unauthenticated, so a client that fabricates or replays it can seed a new row at `script = 1`, and there is no server-side cross-check. V1 accepts that trade in exchange for having no enable gate to run on the hot path. A build that cannot accept it needs V2, where the flag does not exist and the decision is entirely server-side.
8. `created_date` = now (DB-side). `typetag` composed. `install_sites` serialised.
9. **INSERT** — 19 columns: attribution, network/risk snapshot, build identity, `level`, `conv`, `script`, `duplicate`, `typetag`, serialised `install_sites`.
10. Build and return the blob with `updates = 0`.

Note what is **not** written at creation: no `visit`, no user lists, no `updated` — all of those first appear on the second request. A row that never syncs again is therefore distinguishable from one that did.

### 5.5 Update (sync) path

1. Decrypt the blob; read `level`, `script`, `updates`, `conv`, `id`, `typetag`, creation timestamp, market **from it**. Absent fields fall back to defaults — notably a missing `typetag` is synthesised as organic (`-xx-`).
2. **Enable gate — present but disabled, by design.** The periodic gate that would promote `script` from `0` to `1` is commented out in the live path. Consequently `script` never transitions on sync, the `enabled` stamp is never written, and the only route to `script = 1` is the creation-time echo (§5.4 step 7). This is the intended V1 arrangement, not an oversight: **monetization state is decided once, at creation, and thereafter only carried.** The gate's *conditions* are still the canonical ones and are documented at §8.
3. **Injection decision:** if the build is flagged approved **and** `level == 0` **and** `script == 1` **and** the market is on the approved list → assemble the monetization payload. *Rule IDs and priorities belong to Rules & Priorities.*
4. `++updates`.
5. **UPDATE:** `ip`, `level`, `script`, `updated = NOW()`, `updates`, `version`, `visit`, `whitelistedDom`, `blockedDom` — by `id`.
   - `visit` is **overwritten wholesale** with this sync's `visited_map`. History is not accumulated server-side. Anything that needs cumulative browsing is looking at the wrong column.
   - `conv` is not in the SET list. This is what freezes it.
6. Rebuild the blob from the post-update values and return it, alongside the version pin and a **fixed** sync interval (3 h). V1 has no adaptive cadence.

### 5.6 What V1 structurally cannot do

Not defects — consequences of the design, and the reason V2 exists:

- **No hardware layer.** No VM detection, no device clustering, no hardware-based reconstruction. Identity rests on attribution and network context only.
- **No cumulative activity.** `visit` is a snapshot, so there is no per-install browsing history to score.
- **No sync journal.** There is no record that a sync happened, only that `updates` went up. Cadence, gaps and per-sync deltas are unrecoverable.
- **No durable opt-out.** V1 uses `script` **0/1 only** — the enum carries `2` (the durable opt-out, V2's Acceptable-Ads-off), but V1's client-rewindable handle can never hold it, so once enabled there is no representable "user turned this off". *(Enum values `'3'`/`'4'` are unused/vestigial across all builds — functional `script` states are 0/1/2.)*
- **No terminal `conv`.** V1's `conv` domain is `0..3`; the V2 terminal `conv '4'` (expired/forfeited) cannot be represented.
- **Client-rewindable state.** An old blob replayed after a later sync moves `level`, `script` and `updates` backwards in the row.
- **State is only as safe as the key.** Compromise of the symmetric key exposes and permits forgery of every user's `script`, `level` and `conv`.

---

## 6. Version 2 — specification

### 6.1 Dispatch

```
if identity_handle present and non-empty:            # in V2 the handle is the opaque token
    row = SELECT * FROM users WHERE fingerprint = token
    if row:
        → optional telemetry backfill  (§6.8)
        → optional retroactive VM pass (§6.8)
        → UPDATE path                  (§6.7)
    else:
        → createUser                   (§6.6)
else:
    if hardware_data present and non-empty:
        → createUser                   (§6.6)
    else:
        → WAIT: serve rules, return identity_token = null, create nothing
```

### 6.2 The create-guard

**V2 never creates a row without hardware evidence.** This is a hard rule, enforced twice: once in the dispatch above, and again inside creation immediately before the insert.

The reason is a real ordering problem: on a fresh install the client's first sync fires before its content script has finished collecting hardware signals. Creating on that first call would produce a hardware-less row that no hardware-based tier could ever reconstruct — an orphan.

So the first call returns `identity_token = null` plus a complete rule set. A null token tells the client "identity is not established yet, sync again". The user is fully protected (rules are served) and no row exists. The row appears on the next call, which carries hardware.

**Two consequences to design around:**
- Install counts are *lower* and *cleaner* than V1's. A probe that cannot run scripts never becomes a row.
- The client MUST re-sync promptly after a null token, and MUST keep re-sending hardware until it receives a token. A client that treats null as terminal never registers.

### 6.3 V2-A resolution cascade

| # | Tier | Predicate | On match |
| :-- | :-- | :-- | :-- |
| 0 | **Token echo** (dispatch) | token matches `fingerprint` | Adopt |
| 1 | **Derived token** | recompute the token from this request's hardware + attribution + IP; look it up | Adopt |
| 2 | **Attribution pair** | `cid` **and** `sid` both present and match | Adopt, `duplicate++` |
| 3 | **Network + install context** | `ip` + `an` (NULL-safe) + serialised `install_sites` match | Adopt, `duplicate++` |
| 4 | **Create** | — | Insert |

Tiers 2 and 3 additionally run the telemetry backfill and the retroactive VM pass, and — **the structural hazard** — may recompute and overwrite the row's `fingerprint`, plus a fallback that regenerates it from the row's stored hardware columns when the column is empty. Each of those writes moves the row's token away from the copy the client holds.

Tier 1 also cannot recognise the two cases it most needs to: a returning user on a **new IP** or under a **new campaign** produces a different derived token, because both are token inputs.

### 6.4 V2-B resolution cascade

| # | Tier | Predicate | On match |
| :-- | :-- | :-- | :-- |
| 0 | **Token echo** (dispatch) | token matches `fingerprint` | Adopt. ~The common path. |
| 1 | **Attribution triplet**, guarded on `cid` present | `an <=> ? AND cid <=> ? AND sid <=> ?` | Adopt |
| 2 | **Stable hardware signature**, organic only (`cid` absent) | `device_sig` matches **exactly one** row, and `install_sites` says this is a storage wipe rather than a reinstall | Adopt |
| 3 | **Create** | — | Mint a token and insert |

Four design decisions, each closing a specific failure:

1. **The `cid`-present guard on Tier 1 is mandatory.** The predicate uses NULL-safe equality, so on a fully organic request (`an`, `cid`, `sid` all null) it is **true against every other organic row**. Without the guard, the first organic install becomes every organic install. Guarding on `cid` routes organic traffic to Tier 2, which has its own ambiguity check.
2. **Tier 1 does not fall back on a miss.** `cid` present but unmatched means a *new* click — a fresh acquisition — so it falls straight through to CREATE rather than trying weaker tiers. This is intentional, and it is the one path in the whole design that can produce two rows for one human. §6.5 works through exactly when that happens, what it costs, and where the mitigation is supposed to live.
3. **Tier 2 merges only when unambiguous.** The lookup takes two rows; if two exist, the signature is a common hardware configuration and the tier declines to merge. *A recoverable duplicate is safer than a wrong merge*, because a duplicate can be reconciled later while a merge silently hands one user another's `level`, `script` and `conv`.
4. **Tier 2 discriminates reinstall from storage wipe** using `install_sites`: non-empty **and** different from the row's stored value → a genuinely new install (create); empty or identical → the same person returning after losing local storage (reconstruct).

**The IP tier is gone.** Removing it removes the false merges it caused (shared NAT, carrier CGNAT, offices) at the cost of the true merges it caught (a token-less returning user on a new campaign — see decision 2).

### 6.5 Reconstruction, duplicates and re-payment

The cascade's job is to answer "have I seen this person before?" with only the evidence the request carries. It gets that right in every case but one, and the exception is worth understanding precisely, because it is the difference between an *acquisition* model and a *user* model.

**The decisive fact: Tier 1 only ever runs when the client has no working token.** Tier 0 is checked first, so the ordinary case — the extension is still installed and the same person clicks another ad months later — never reaches the attribution tier at all.

| Scenario | Token | `cid` | Resolves at | Outcome |
| :-- | :-- | :-- | :-- | :-- |
| Re-click, client still holds its token | held | new | **Tier 0** | Same row. The new `cid` is **ignored** — the sync UPDATE never rewrites `an`/`cid`/`sid`. No duplicate, no re-payment. |
| Storage wipe, attribution cookie still present | lost | same | **Tier 1** | Reconstructed onto the original row. No duplicate, no re-payment. |
| Storage wipe, organic install (no `cid`) | lost | none | **Tier 2** | Reconstructed via the stable hardware signature. No duplicate. |
| Storage wipe **plus a new click** | lost | new | Tier 1 misses → no fallback → **CREATE** | **Second row, second token, and the conversion gate fires again.** |

Only the last line duplicates, and it needs *both* halves: a lost token **and** a genuinely different `cid`. Tier 2 cannot rescue it — the hardware tier is deliberately mutually exclusive with "`cid` is present", so that a present-but-unmatched attribution key can never over-merge onto an unrelated organic row (§6.4, decision 1). Closing one hole would open the other.

**So the design is correct as an acquisition model, and imprecise as a user model.**

- **On the money side it is defensible.** A new `cid` is a real click that a real campaign really generated. Treating it as a fresh acquisition and paying it is the intended product decision, recorded as such.
- **On the counting side it overcounts.** Two rows, one human. Anything that counts *people* rather than *acquisitions* — unique installs, survival cohorts, retention curves, reconstruction rate — drifts on this path. The mitigation is measurement, not prevention: `[OWNER — manual]` re-run the classifier and fraud report before and after any change to the cascade and diff the funnel/cohort counts against a known baseline (a live server-side action the owner runs; the agent prepares and interprets it), so the drift is quantified rather than discovered.

**Where the mitigation is meant to live, and why it currently does not bite.** The identity layer deliberately hands repeat abuse — wipe storage, click again, get paid again — to the fraud layer: the fraud device hash counts same-hardware installs in a bounded window and labels a burst as a borderline tier. Three things blunt that today, and a build inheriting this design should know all three:

1. The tier label only affects payment **when enforcement is switched on**. In shadow mode it is an observational string with no effect on `conv`.
2. A storage wipe is **not** an uninstall, so nothing stamps the churn column — meaning a held-then-paid payout is never forfeited on this path.
3. The window and threshold are tuned to catch *bursts* of clone installs, not one deliberate wipe-and-reclick weeks apart.

Net: **deliberate wipe-and-reclick is unmitigated at both layers while the system runs in shadow.** That is a hand-off whose receiving end is not switched on — not a flaw in the resolution cascade.

> **Optional lever, not current behaviour anywhere.** If a build needs to close the re-payment path without weakening the over-merge guard, the shape that does it is: when Tier 1 **misses** and a `cid` **is** present, consult the stable hardware signature **not to merge the row but to withhold the conversion** — create the user normally, serve rules normally, skip the payout. That keeps identity separate (a new acquisition is still a new row) while making the money decision hardware-aware. It is recorded here as a design option; no implementation currently does this.

### 6.6 Creation sequence

1. Apply per-network scoping to the behavioural/delay config flags.
2. Initialise `conv = 0`, `level = "0"`, `script = 0`, `duplicate = 0`, `updates = 0`.
3. Fill missing attribution from cookies.
4. Run the resolution cascade (§6.3 / §6.4). Any match returns.
5. **Create-guard:** no hardware → return `WAIT`, create nothing.
6. **Mint the identity token** (V2-B). On CSPRNG failure, **abort — do not create**. Fail-closed: a row with no usable token is worse than no row.
7. **Risk lookup** (IP) → provider, organisation, hostname, proxy, type, risk score, ASN, country, network timezone. Market from the country code, falling back to the edge-provided market.
8. **Risk evaluation** → `level` digits, possibly `conv = 2`. A failed lookup returns `level = "0"` — **fail-open on risk**, so an outage at the risk provider does not mass-flag legitimate installs.
9. **VM evaluation** over the hardware plus the network timezone → `is_vm`, `vm_type`. *Owned by the Detection spec.*
10. **Fraud hashes:** canvas hash; fraud device hash; a bounded-window collision count over that hash.
11. **Tier classification** → `fraud_flags`, a label string (`tier1:…` / `tier2:…` / `tier3:…`). *Membership rules belong to the Detection spec.* At creation these are **observational labels** unless the enforcement flags are on.
12. **Conversion decision** — the single gate:
    - *Pure shadow* (all behavioural/delay flags off): apply the same four-condition gate as V1 (`an` **and** `cid` **and** `level === "0"` **and** `conv !== 2`). Optionally set a *hold marker* in `fraud_flags` instead of firing, so a post-install sweep fires it later.
    - *Enforcing:* Tier 3 → `conv = 4` (was `2`; 2026-09-03 conv-attribution convention — `conv 2` is reserved for risk-level blocks); Tier 2 → `conv = 0` deferred to the behavioural layer; Tier 1 → the same gate.
13. **Sync cadence:** a fresh install is inside the observation window, so it gets the fast collection interval (10 min) rather than the default 3 h — unless it is a **trusted hard block**: an install whose decision is already final (a `level != "0"` risk block, or a Tier-3 verdict under enforcement), so there is nothing left to observe. It skips the fast cadence and the sync journal (§6.7 step 8) and gets the default 3 h straight away.
14. `typetag`, serialised `install_sites`.
15. **INSERT** — ~44 columns: everything from §5.4's 19, plus the whole hardware/telemetry group, the identity token, the stable hardware signature (V2-B), `is_vm`/`vm_type`/`diagnostic_date`, the fraud hash/score/flags, the ASN, and the acceptable-ads stamp. Columns added after the schema baseline are appended under a **capability probe** so an insert cannot fail with "unknown column" on a not-yet-migrated database. `created_date`, `conv_date` and `updated_at` are deliberately **not** in the column list — all three fire by DEFAULT in this one statement, so `conv_date == created_date` exactly whether the row pays at install or defers (§2.1); creation needs no code to stamp them.
16. Return the token.

### 6.7 Update (sync) path

1. Read **every** value from the row: `id`, `script`, `level`, `conv`, `updates`, `typetag`, `created_date`, market, `visit`, `install_sites`, attribution. Nothing comes from the client's handle.
2. Default `sync_interval` to 3 h; later stages may lower it.
3. **Acceptable-ads opt-out** (`acceptable_ads === false`):
   - `script == 1` → `script = 2`.
   - If `script == 2` (now or previously), emit an **explicit reset payload**: default cosmetic rules, empty local CSS, empty scriptlet excludes, the default filtered system whitelist, and **the injected rule IDs filtered out of the rules array**. Reverting monetization is explicit, never implied by omission.
4. **Otherwise, the enable gate is live** (unlike V1): if the build is approved, and the gate's conditions hold, and the market is approved, and the build is the approved store build → `script = 1` and stamp `enabled` (idempotently: only when it is still null).
5. **Injection decision:** `level == 0` **and** `script == 1` **and** market approved.
6. `++updates`.
7. **UPDATE:** `ip`, `level`, `script`, `updated = NOW()`, `updates`, `version`, merged `visit`, user allow/block (+ cosmetics), and a CASE-toggled acceptable-ads stamp — by `id`. `conv` is absent from the SET list, and so is `conv_date`; `updated_at` needs no entry — the database bumps it itself whenever the UPDATE changed anything (§7).
   - **`visit` is merged, not replaced:** read the stored map, take `max(stored, incoming)` per domain, write the union back, and return it to the client. This makes `visit` a cumulative browsing history — which is what the behavioural layer scores. It also means the column grows monotonically and needs a cap policy.
   - Deserialisation of the stored map goes through a **tolerant parser** with a regex fallback. A strict parse that fails would return an empty map and the merge would then *overwrite the whole history* with just this sync's domains. Silent total history loss; guard against it.
8. **Sync journal:** insert one row into the sync log — capturing timestamp, source IP, cumulative domain count, new-domain delta, and a stable hash of the domain set — but only while the install is un-decided, inside the observation window, under the per-install sync cap (20), and not a trusted hard block. The cap × the fast interval is sized to cover the whole window.
9. **Behavioural layer**, wrapped so that any exception is logged and the request still succeeds.
10. On a *reconstruction* (a cascade match rather than a token echo), `duplicate++` as the reconstruction tally.

### 6.8 The two backfill paths

Both exist because rows predate signals, and both are **write-once**, not per-sync work:

- **Telemetry backfill** — gated on a client hint *and* a check that the row is actually missing a signal the client now has. Writes the newly-collected telemetry columns, re-runs VM evaluation, stamps the diagnostic date.
  ⚠️ **In V2-A this path also rewrites the identity token** (see §6.3). In V2-B it must not: backfill telemetry, leave `fingerprint` alone. This is the single most important line separating the two variants.
- **Retroactive VM pass** — for rows whose diagnostic date is null (or older than a re-evaluation cutoff): re-run VM evaluation from the row's stored hardware plus the current network timezone, write `is_vm`/`vm_type`/`diagnostic_date`. Hygiene for reporting and for the tier fallback; it does **not** feed the install-time tier gate, which always uses a freshly computed result.

---

## 7. The full write surface — who writes what, when

Every writer of a user row. If a column changes and it is not on this list, something is wrong.

| # | Writer | Trigger | Columns written | Notes |
| :-- | :-- | :-- | :-- | :-- |
| 1 | **Creation** | resolution ends in CREATE | the full insert set (§5.4 / §6.6) | The only INSERT. |
| 2 | **Sync update** | every resolved request | `ip`, `level`, `script`, `updated`, `updates`, `version`, `visit`, user lists, acceptable-ads stamp | Never `conv` — and therefore never `conv_date`. |
| 3 | **Enable stamp** | enable gate passes | `enabled` | V2 only in practice (V1's gate is inert). Guard with `IS NULL` so the first-enable date is preserved. |
| 4 | **Reconstruction tally** | a cascade tier below the token echo matched | `duplicate` | V2-B replaces the per-event error rows with this counter. |
| 5 | **Telemetry backfill** | client hint + row missing a signal | telemetry columns, `is_vm`, `vm_type`, `diagnostic_date`, ASN — **and, in V2-A only, `fingerprint`** | Write-once. |
| 6 | **Retroactive VM pass** | `diagnostic_date` null / stale | `is_vm`, `vm_type`, `diagnostic_date` | Hygiene only. |
| 7 | **Identity repair** | row's token column empty (V2-A) | `fingerprint` | **Must not exist in V2-B.** |
| 8 | **Sync journal insert** | §6.7 step 8 conditions | *(sync-log table, not the user row)* | V2 only. |
| 9 | **Behavioural layer, on sync** | every V2 sync | `behavior_score`, `behavior_flags`, `behavior_hash`, `decision`, `decided_date`, `flagged`, `device_hash`, `fraud_flags`, and `conv` + `conv_date` **when enforcing** | *Detection spec.* The only sync-path writer of `conv`, and only under enforcement. |
| 10 | **Post-install sweep** | scheduled, or triggered by traffic | `conv`, `conv_date`, `decided_date`, `behavior_flags`, `fraud_flags` (hold → paid / forfeit) | *Detection spec.* Owns held and expired payments. |
| 11 | **Uninstall endpoint** | the browser fires the registered uninstall URL | `disabled` | See §9. |
| 12 | **Error log** | abnormal paths | *(error table)* | V2-B moves routine reconstruction events off this table so it stays a signal. |

**Every `conv` write carries a `conv_date` write — one pattern, no exceptions.** At creation this is free: neither column appears in the INSERT, both DEFAULTs fire in the same statement, and `conv_date == created_date` (§2.1). After creation the pattern is: a writer whose WHERE guard guarantees a real transition (e.g. `conv = '0' OR conv IS NULL`) sets `conv_date = NOW()` plainly; a writer that can rewrite `conv` with its existing value uses the NULL-safe guard `conv_date = IF(conv <=> ?, conv_date, NOW())`, assigned **before** `conv` in the SET list (SET evaluates left-to-right). A same-value rewrite therefore never advances `conv_date`, which keeps the column truthful for forensics: it always names the writer that set the current `conv`, and when. **Any new `conv` writer must follow the same pattern.** The concrete write sites and their guards are enumerated in the Detection spec; the shipping history is `DETECTION_CHANGELOG.md`.

**`updated_at` belongs to the database, not to any writer on this list.** `DEFAULT current_timestamp() ON UPDATE current_timestamp()`, last column: any UPDATE that changes at least one value bumps it — from any writer, present or future — a no-change UPDATE does not, and an explicit assignment overrides the auto-clause (reserved for migration backfills). It is **not** the heartbeat `updated` (writer 2's explicit `NOW()`, which feeds the survival metrics); the two answer different questions and neither replaces the other.

**The `disabled` column is load-bearing far beyond uninstall accounting.** It gates payment forfeiture, the behavioural payout decision, and every survival/chargeback metric. A path that fails to set it causes **overpayment** and inflates every survival figure at once.

---

## 8. The monetization enable gate

The gate that promotes `script` from not-enabled to enabled. Documented here because it is a **sync-path write to the user row**; the rules it produces belong to Rules & Priorities.

All of the following must hold:

| Condition | Rationale |
| :-- | :-- |
| `level == "0"` | No risk digit. Any flag suppresses monetization while leaving ad-blocking intact. |
| `script == "0"` | **Not yet enabled — and never opted out.** |
| age > ~7.5 days | The install has survived long enough to be real. |
| `updates` > a threshold (8–10) | It has actually been used, not just installed. |
| > 15 visits to a search host | The user actually uses search — the surface monetization depends on. |
| `typetag` does not contain `-xx-` | The install is attributed. Organic installs never enable. |
| `install_sites` non-empty | A real browser session existed at install. |
| `acceptable_ads !== false` | An explicit opt-out short-circuits the gate. (V2 only.) |

Plus outer gates: the build must be the approved store build, and the market must be on the approved list.

> ⚠️ **The `script == "0"` clause is what makes opt-out durable.** Because the gate requires exactly `0`, a user at `2` can never satisfy it, no matter how well they score on every other criterion. A variant that accepts `0 || 2` **re-enables users who have explicitly opted out**, which defeats the entire purpose of having a state `2`.
>
> **This deviation exists in one live build. It is not a pattern to copy.** A new build MUST gate on `script == "0"` exactly. Whether to correct the existing build is a separate operational call; this document records the requirement, not the remediation.

---

## 9. Failure and degraded paths

Behaviour that must hold when something is broken. These are the paths that decide whether an outage is invisible or fleet-wide.

1. **Database unreachable → HTTP 200, never an error.** Echo the client's own identity handle back unchanged, echo the `data_version` pin **identically** (a mismatch triggers the client's storage purge — a wrong value here wipes local state fleet-wide), serve file-based asset versions and URLs, and return a **jittered** back-off (3–4 h) so the recovering database does not take a synchronised retry storm.
2. **Degraded responses are identity-conditional.** A request *with* a handle is a user the server already knows and whose client already holds inline rules, a whitelist and cosmetics — so the degraded response omits every per-user key and lets the client's merge semantics keep what it has. A request *without* a handle has nothing locally, so it gets the full fresh-install baseline. Sending the baseline unconditionally **wipes an existing user's applied rules**.
3. **Risk lookup failure → fail-open.** `level = "0"`. A provider outage must not flag the whole fleet as risky.
4. **Token mint failure → fail-closed.** Do not create. A row whose identity cannot be issued is unrecoverable.
5. **Uninstall write failure → still redirect.** The user reaches the feedback destination even when the `disabled` write is lost. The redirect is the user-visible contract; the write is bookkeeping.
6. **Behavioural layer exception → swallow and continue.** Log it; the sync must still return rules. Ad-blocking is never allowed to depend on the analytics layer.
7. **A cascade tier that half-matches must fall through, not abort.** Terminating the request with an empty body (V1's behaviour, §5.3) leaves the client with no rules and no diagnosis.
8. **Every response must carry the version pin and a sync interval**, on every path without exception.

---

## 10. Lifecycle timelines

**V1**

1. Install. Static rules are already active. First sync fires with no handle.
2. **Row created immediately** — risk lookup, `level`, conversion gate, `script = 0` (or `1` from the echoed flag). Handle issued.
3. Periodic sync (fixed 3 h). Handle decrypted → state read from it → injection decided → `updates++` → row updated → new handle issued.
4. `script` never changes on this path. `conv` never changes at all.
5. Uninstall → `disabled` stamped.

**V2**

1. Install. Static rules already active. First sync fires with no handle **and no hardware yet**.
2. **`WAIT`** — rules served, `identity_handle = null`, nothing created.
3. Second sync carries hardware → **row created**: risk, VM, fraud hashes, tier label, conversion decision, token minted. Fast collection cadence returned (10 min).
4. Observation window (~3 h, capped at 20 logged syncs): each sync merges `visit`, appends a journal row, and runs the behavioural layer.
5. Decision point: the behavioural verdict merges with the install-time tier; `conv` may resolve; cadence relaxes to 3 h.
6. Steady state: sync every 3 h. State read from the row, `updates++`, `visit` merged, `script` may transition through the enable gate or the opt-out.
7. Uninstall → `disabled` stamped → held payments forfeit.

---

## 11. Choosing a version for a new build

| Requirement | V1 | V2-A | V2-B |
| :-- | :--: | :--: | :--: |
| Minimal moving parts, no hardware collection | ✅ | ❌ | ❌ |
| No client-side hardware/telemetry collection at all | ✅ | ❌ | ❌ |
| Server-authoritative state (client cannot rewind it) | ❌ | ✅ | ✅ |
| Survives a client storage wipe without hardware | ⚠️ cascade only | ⚠️ | ⚠️ |
| Recognises a returning user on a **new IP** | ⚠️ tier 1 only | ❌ | ✅ |
| Recognises a returning user under a **new campaign** | ⚠️ tier 2 only | ❌ | ⚠️ deliberately treated as a new acquisition (§6.5) |
| Never false-merges users behind shared NAT | ❌ | ❌ | ✅ |
| Cumulative browsing history | ❌ | ✅ | ✅ |
| Sync journal / cadence analysis | ❌ | ✅ | ✅ |
| VM & fraud tiering at install | ❌ | ✅ | ✅ |
| Representable monetization opt-out | ❌ | ✅ | ✅ |
| Install counts exclude script-less probes | ❌ | ✅ | ✅ |
| Identity token cannot desynchronise from the row | ✅ n/a | ❌ | ✅ |

**Recommendation for a new build: V2-B.** It is the only variant where the identity contract cannot break by construction, and it is a strict superset of V2-A's capability apart from the deliberate new-campaign-is-a-new-acquisition decision (§6.5) — which is a product choice, not a limitation.

**Choose V1** only when the build genuinely must not collect hardware signals, and accept in exchange: no fraud layer, no behavioural layer, no opt-out state, and client-rewindable state.

**Do not start a new build on V2-A.** Its two structural problems — a recomputable token that several paths overwrite, and an identity that folds in the IP and the campaign — are exactly what V2-B was written to remove. It remains documented because it is deployed and must be maintained.

> **The identity choice also fixes the backend generation — permanently.** The
> fraud / VM / behaviour / finalization stack and the *live* enable gate require
> hardware, hence V2 — so a **V1 build is network-risk-only for its whole life**.
> The generation is chosen once, at creation, and **never migrated**: an update must
> never change an existing build's identity or uniqueness keys, or it orphans every
> user already in the DB (§2.3 invariant 6). A build that needs fraud is a *new* V2
> build, not an upgraded V1. The capability dependency map is in
> `BACKEND_ARCHITECTURE.md` §8.

---

## 12. Conformance checklist

Run the **Common** items for every build, then add the block for the build's
generation (§3, §13), applying the **reachable-set filter** (`00_GOAL.md` §3, decision B):
run a generation/substrate item only where the build's frozen schema carries the column —
otherwise it is permanent `n/a`, not a failure. Each item is a real failure observed in one
design or another. The identity-generation choice is fixed at birth and never migrated
(§2.3 invariant 6), so a build sits in exactly one lane for life: **Gen 0 (Legacy) · V1 · V2**.

> **Gen 0 (Legacy, pre-family — e.g. 12).** Runs the Common items its schema supports; **skips
> the V2 block entirely** (no fingerprint/fraud); and runs the V1 block **only** where an item
> matches its substrate and mechanism. It shares V1's client-rewindable handle and network-risk
> model, but (a) its **pre-family ad-counter schema has no family user-data group**
> (`visit`/`whitelistedDom`/`blockedDom`/`instdom`/`cosmeticDom`), so any item touching those
> columns is `n/a`, and (b) items describing V1's *enable mechanism* (e.g. the inert/commented
> enable gate) are `n/a` — Gen 0 enables monetization its own way. Its monetization **itself** is
> a **Common** item like every build's, never treated as absent (`00_GOAL.md` §3). Gen 0 detail:
> §13, `DB_SCHEMA.md` §7 note ¹².

### 12.1 Common — every build

**Identity & resolution**
- [ ] The handle round-trips byte-for-byte and always resolves to the row it was issued for — invariant §2.3.2.
- [ ] The handle format matches every downstream format check, notably the uninstall endpoint's length/charset test. Changing the format means auditing those checks.
- [ ] Resolution is deterministic and total: every request ends in exactly one outcome (V1: `EXISTING` / `CREATE` / `ABORT`; V2 adds `WAIT`).
- [ ] No resolution tier can match on an **all-NULL attribution** predicate (V1 Tier 1 requires `cid` **and** `sid`; V2-B guards Tier 1 on `cid` present).
- [ ] A partial tier match falls **through** to creation; it never terminates the request without a body (V1's abort-on-half-match, §5.3, is the defect this item catches).
- [ ] Attribution is normalised identically on every path that reads it, and falls back to cookies before an install is classified organic.

**Creation & money**
- [ ] Risk-lookup failure fails **open** (`level = "0"`).
- [ ] The conversion gate requires all four conditions (`an` + `cid` + `level == "0"` + `conv != 2`); organic never pays.
- [ ] `conv_date` is **not written by creation code**: it and `created_date` both fire by DEFAULT in the single INSERT statement, so `conv_date == created_date` at creation — pay-at-install and delayed conversion covered with no config gating (§7).

**Update**
- [ ] `conv` is **absent** from the sync UPDATE's SET list.
- [ ] Every statement that writes `conv` maintains `conv_date` in the same statement — plain `NOW()` under a transition-guaranteeing WHERE guard, otherwise the NULL-safe `IF(conv <=> ?, conv_date, NOW())` guard assigned **before** `conv` (§7).
- [ ] `updated_at` is written by **no code path** (DB `ON UPDATE` clause only; explicit assignment reserved for migration backfills) and is never conflated with the heartbeat `updated` (§7).
- [ ] `updates` increments exactly once per resolved request.

**Degraded**
- [ ] Database failure returns HTTP 200 with the handle echoed and the version pin **identical**.
- [ ] The degraded response is identity-conditional: full baseline for new installs, per-user keys omitted for known ones.
- [ ] Back-off is jittered; the uninstall redirect fires even when its `disabled` write fails; every response carries the version pin and a sync interval.

### 12.2 V1 (Gen 1) only

- [ ] The handle is an **encrypted state blob**; the row has **no minted identity column and no UNIQUE identity index** — a V1 row is keyed by `id`, and its working state is read back out of the blob (§3.1).
- [ ] `script = 1` is reachable **only** via the creation-time echoed flag; the periodic enable gate is **inert** (commented out) and never transitions `script` or stamps `enabled` on sync (§5.4 step 7, §5.5 step 2).
- [ ] Monetization seeded from the client-echoed flag is a **recorded, accepted** trust boundary (the flag is unauthenticated; no server-side cross-check) — deliberate in V1; V2 has no such input (§5.4 step 7).
- [ ] `visit` is written **wholesale** each sync (snapshot; no merge, no server-side history) (§5.5 step 5).
- [ ] There is **no `script = 2` opt-out** and no hardware / fraud / behaviour columns — their absence is correct, not a gap (§5.6).

### 12.3 V2 (Gen 2/3) only

**Identity**
- [ ] No row is created without hardware evidence — the create-guard is enforced in dispatch **and** again immediately before insert (§6.2).
- [ ] All state is read from the **row**, never from the client handle (§6.7 step 1).
- [ ] *(V2-B only)* The identity signature (`device_sig`) and the fraud device hash (`device_hash`) are **separate columns from separate component sets** — never unified (§3.2). *(V2-A has `device_hash` only; there is no `device_sig` to separate — do not run this item against a V2-A build, `DB_SCHEMA.md` §8.3.)*
- [ ] Any column added after the schema baseline is written **behind a capability probe** (V2-only — V1 ships a fixed column set and adds none; `DB_SCHEMA.md` §7/§8.3), never in the unconditional INSERT list.
- [ ] *(V2-B)* The identity token is **minted, write-once, and UNIQUE-indexed**; nothing recomputes or overwrites it after issue, and mint failure **aborts creation** (fail-closed) (§3.2, §6.6 step 6, §9.4).
- [ ] *(V2-A)* The recompute-and-overwrite paths (telemetry backfill, identity repair, cascade tiers) are the known desync hazard; the token folds in IP + campaign. A **new** build MUST NOT start on V2-A (§6.3, §11).

**Resolution**
- [ ] Any hardware-signature tier declines to merge when **more than one** candidate matches — a recoverable duplicate beats a wrong merge (§6.4 decision 3).
- [ ] Reinstall and storage-wipe are distinguished (via `install_sites`) before merging (§6.4 decision 4).
- [ ] The re-payment posture when an attribution key is **present but unmatched** is an explicit, recorded decision (§6.5) — not an accident of tier ordering.

**Update**
- [ ] `visit` merging is **non-destructive** (cumulative; a deserialisation failure cannot wipe history) (§6.7 step 7).
- [ ] The enable gate requires `script == 0` **exactly** — never `0 || 2` — so an opt-out at `2` is durable (§8).
- [ ] The `enabled` stamp is written **idempotently** (only when still NULL) (§7 writer 3).
- [ ] Reverting monetization sends an **explicit reset payload**, never a silent omission (§6.7 step 3).
- [ ] An exception in any analytics/behavioural layer cannot fail the sync (§9.6).

---

## 13. Per-extension identity implementation

Which generation each shipped build runs, and the distinctive details. The
variant model is §3; this maps it onto the fleet.

| Build | Variant | Handle (example key) | Hardware collected | Resolution cascade | Notes |
| :-- | :-- | :-- | :-- | :-- | :-- |
| 12 Pro | **Gen 0 Legacy** | `syncuid` (AES-encrypted UID) | none (network-risk only) | decrypt `syncuid` → `id`; else IP/CID + `script`+`updates` | pre-family lineage: no `fingerprint`/user-data columns, ad-counter schema, `conv 0–2`; client-rewindable AES blob like V1, so sync-mirror is the only survival path (`EXTENSION_COMPOSITION.md` §4). **Frozen — never migrated.** |
| 21 North | **V1** | `backdetail` (encrypted UID) | none | CID+SID → IP+`an`+instdom | client-rewindable state; sync-mirror is the only survival path |
| 22 Hunter | **V1** | `hunterinfo` (encrypted UID) | none | CID+SID → IP+`an`+instdom | same as North |
| 23 Wonder | **V1** | `wonderinformation` (encrypted UID) | none | CID+SID → IP+`an`+instdom | legacy `devVM` diagnostics flow now stale — treat as historical |
| 25 Ninja | **V2-A** | `ninja_fp` (HMAC of hardware+`an`/`cid`/`sid`+IP) | full set | token echo → fingerprint recompute → CID+SID → IP+`an`+instdom | recomputable token that several paths overwrite; folds in IP+campaign — the two problems V2-B removes; schema predates `conv_date`/`updated_at` |
| 26 Ghost | **V2-B** | `ghost_fp` (minted 64-hex token) | full set | token echo → `an`/`cid`/`sid` (cid-guarded) → `device_sig` (single-candidate) | IP removed from identity; separate stable `device_sig`; new-cid = new acquisition (§6.5); first build on the canonical `conv_date`/`updated_at` record (§7) |
| 27 | **V2-B** | `ab_fp` (minted token) | full set | as Ghost | reference frontend; backend to be copied from Ghost (canonical since 2026-09 — copy Ghost, not Ninja's pre-fix conv pipeline; `FRAUD_DETECTION_V5.md` §18) |

**Reading the fleet:** `12` is the sole documented **`Gen 0` (Legacy, pre-family)**
build — an AES `syncuid` handle on a pre-family schema, closest to V1 in identity
*style* but not a V1 table (§13 note; `DB_SCHEMA.md` §7 note ¹²); the other
pre-family builds (15/17/20/24) join it as they are picked up. The three V1 builds
share one identity codebase; Ninja is the sole V2-A; Ghost and 27 are V2-B. A **new**
build starts at V2-B (§11) and copies the Ghost backend (the canonical copy source
since 2026-09-02 — not Ninja's pre-fix conv pipeline, `FRAUD_DETECTION_V5.md` §18),
remapping the handle key (`00_INDEX.md` §6). A **legacy** build keeps its Gen 0 identity untouched and
only modernizes its frontend (`00_INDEX.md` §3, third reading order). The client-side
half of whichever variant is in play — collection, mirror, echo, uninstall-URL — is
`EXTENSION_COMPOSITION.md` §6–§7.

**Record-column adoption:** the canonical `conv_date`/`updated_at` pair (§2.1, §7)
ships first in Ghost (2026-09); every other build's schema currently predates it —
Ninja included — so their `conv_date`/`updated_at` checklist items run `n/a` under
the reachable-set filter (§12) until each backend adopts the migration. 27 inherits
the pair when its backend is copied from Ghost.

---

*Baseline specification — implementation-independent. Rule tiers and priorities are specified in the Rules & Priorities document; fraud tiering, VM evaluation and the behavioural merge in the Detection spec; the **client-side** identity obligations (collection, sync-mirror, echo, uninstall-URL) in `EXTENSION_COMPOSITION.md` §6–§7. Start from `00_INDEX.md`.*
