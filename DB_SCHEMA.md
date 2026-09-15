# Database Schema — the canonical specification

**The user table: what each column is, who writes it, which indexes earn their
keep, and how the schema differs across the identity variants and grows safely.**

Like the rest of the backend, the schema is a **template to copy** (`00_INDEX.md`
§1): one `users`-style table per build with **identical column names and indexes in
every build — the only thing that changes inside the DB is the table name**. The
per-build client-facing key spellings are a server-code concern (`00_INDEX.md` §6
rule 2); they never appear in the schema. The reference tables are Ninja (`ninja25`)
and Ghost (`ghost26`).

> **Companion documents.** Who writes each column and when: `USER_CREATION_AND_UPDATE.md`
> §7. The fraud/behaviour columns' meaning: `FRAUD_DETECTION_V5.md`. The endpoint
> that reads/writes the row: `BACKEND_ARCHITECTURE.md`.

---

## 1. In plain English

One row per user. The row holds four kinds of thing: **who they are** (identity +
attribution), **where they came from** (network + install snapshot), **what we
decided** (the four state columns), and **what we've observed** (activity +
telemetry + behaviour). Almost every column is written on the sync path; a few are
written only once, and one (`conv`) is written only by the money path. Get the
*ownership* right and the schema is simple; get it wrong and users silently
fragment or payments silently double.

Two rules dominate the schema:

1. **The identity column is the anchor every lookup hits.** In **V2-B** it is
   write-once and **uniquely** indexed; in **V2-A** it is a recomputable HMAC handle
   that several paths rewrite (non-unique `idx_fingerprint`, §6); in **V1** there is
   no handle column at all — the row is keyed by `id`.
2. **The identity signature and the fraud hash are different columns from
   different component sets** — they want opposite volatility and must never be
   unified.

---

## 2. Column groups

| Group | Columns (the real canonical names — identical in every build, §9) | Written by |
| :-- | :-- | :-- |
| **Identity** | `fingerprint` (the handle — **V2 only**; V1 rows are keyed by `id` alone), `id` (PK) | `id` creation-only; `fingerprint` **write-once in V2-B**, but **recomputable in V2-A** (HMAC, rewritten by telemetry backfill and several paths — §1 rule 1) (`USER_CREATION` §7 row 1) |
| **Attribution** | `an`, `cid`, `sid`, `typetag` | creation; normalised identically everywhere |
| **Network** | `ip` (`varbinary(16)` via `INET6_ATON`), `provider`, `organisation`, `hostname`, `proxy`, `type`, `risk`, `asn` (V2) | creation (risk lookup); **only `ip` refreshed on sync** (roaming) — the rest are a frozen install snapshot, `USER_CREATION_AND_UPDATE.md` §2.1 |
| **Install snapshot** | `instdom` (serialized), `version` (client build), `extid` (short store id), `mkt` (market / ISO code) | creation; `version` on sync |
| **State (the four)** | `level`, `script`, `conv` (+ `conv_date`, its stamp — canonical 26+, directly after `conv`; §3), `decision` (V2) | see §3 |
| **Lifecycle stamps** | `enabled`, `disabled`, `flagged` (present in **V1 and V2** — the canonical V1 DDL (§9.1) and live V1 North both carry it; behaviour-written only where the behaviour layer runs), `created_date`, `updated`, `updates`, `duplicate`, `acceptable_ads_disabled` (V2 builds — a date stamp, not a flag), `updated_at` (canonical 26+ — always the **last** column; DB-maintained, not the heartbeat `updated` — §3) | enable gate / uninstall / sync; `updated_at` by the database itself (§3) |
| **Activity** | `visit` (serialized `{domain:count}`) | sync — **V2: non-destructive merge; V1: wholesale snapshot** |
| **User lists** | `whitelistedDom`, `blockedDom`, `cosmeticDom` (Ghost 26+) | sync |
| **Hardware / telemetry** (V2) | `cpu_cores`, `gpu_vendor`, `gpu_renderer`, `ram_gb`, `lang`, `color_depth`, `touch_points`, `bat_charging`, `bat_level`, `timezone`, `canvas_hash`, `audio_fp`, `webdriver_present`, `gl_ext_count`, `screen_res`, `user_agent`, `is_vm`, `vm_type`, `diagnostic_date` | creation + telemetry backfill |
| **Device signatures** (V2) | `device_hash` (fraud, volatile), `device_sig` (identity, stable — V2-B) | creation / backfill |
| **Behaviour / fraud** (V2) | `fraud_score`, `fraud_flags`, `fraud_checked`, `behavior_score`, `behavior_flags`, `behavior_hash`, `decided_date` | behaviour layer + sweeps |

(`isocode` is a field of the risk-lookup *response*, not a column — it lands in `mkt`.
`duplicate` exists in every variant, V1 included; it is not a behaviour-layer column.)

Two **separate tables** support the row: a **sync-log / journal** table (one row per
observed sync in the observation window) and an **errors** table (abnormal paths
only — V2-B moves routine reconstruction events off it to a `duplicate` counter so
it stays a signal).

---

## 3. The four state columns and their owners

This is the part builds get wrong. Each state column has exactly one owner path;
crossing them causes the classic bugs.

*Scope note:* this "four" is the **schema/decision** view — `level`, `script`,
`conv`, **`decision`** (V2). `USER_CREATION_AND_UPDATE.md` §2.2 names a different
"four" for the handle-carried working state (…, **`updates`** instead of `decision`).
Both are correct: `decision` is the behaviour verdict, `updates` the sync counter —
different columns, different lenses.

| Column | Question | Written by | Never written by |
| :-- | :-- | :-- | :-- |
| `level` | risk digits (network) — a `smallint` whose decimal digits encode the triggered signals | risk lookup **at creation** — **not re-scored on sync** (only `ip` refreshes for roaming; `USER_CREATION` §2.2) | — |
| `script` | monetization state (0/1/2) | enable gate, on sync | the client (echo only) |
| `conv` | money record (`0/1/2/3/4`) | **the money path** — `processConversion` / the finalization sweep (full writer list: `USER_CREATION` §7); every writer also maintains `conv_date` (below) | **the sync UPDATE** — `conv` must be absent from its SET list |
| `decision` | behavioural verdict | the behaviour layer | anything that reads `conv` (one-way dependency) |

`disabled` is load-bearing beyond churn accounting: it gates payment forfeiture and
every survival/chargeback metric. A path that fails to set it causes overpayment
(`USER_CREATION_AND_UPDATE.md` §7).

**The stamp pair (canonical from Ghost 26, 2026-09-02).** `conv_date` (`datetime
DEFAULT current_timestamp()`, directly **after `conv`**) records the moment `conv`
received its *current* value. At INSERT no code writes it: the DEFAULT fires in the
same statement as `created_date`'s, so `conv_date == created_date` exactly (`NOW()`
is per-statement) — which covers pay-at-install (`conv` already `'1'`/`'3'` in the
INSERT ⇒ `conv_date` = conversion time) and delayed conversion alike, with no config
gating. On UPDATE, **every statement that writes `conv` maintains `conv_date`**:
plain `conv_date = NOW()` where the WHERE guard (`conv = '0' OR conv IS NULL`)
guarantees a real transition; where a site can rewrite `conv` with its existing
value, the NULL-safe guard `SET conv_date = IF(conv <=> ?, conv_date, NOW()),
conv = ?` — `conv_date` assigned **before** `conv`, because UPDATE's SET list
evaluates left-to-right. A same-value rewrite therefore never moves `conv_date`,
keeping the column truthful for forensics (which code path set this `conv`, and
when). Any **new** `conv` write must follow the same pattern. (Full write-site
inventory: `USER_CREATION_AND_UPDATE.md` §7.)

`updated_at` (`datetime DEFAULT current_timestamp() ON UPDATE current_timestamp()`,
always the **last** column) is maintained entirely by the database: any UPDATE that
changes at least one value bumps it, in any file, present or future; a no-change
UPDATE does not; an explicit assignment overrides the clause (§5 uses this for the
migration backfill). It is **not** the extension-heartbeat `updated` — that column
has no auto-update clause and feeds the 36h-survival metrics; the two must never be
merged.

**`conv` is an ENUM — quote every SQL literal.** MariaDB/MySQL treat an *unquoted*
number assigned or compared to an ENUM column as the enum **index**, not a value:
with `enum('0','1','2','3','4')`, index 2 is value `'1'`, so `SET conv = 2` stores
`'1'` (paid) and `conv IN (1,3)` matches `'0'`/`'2'`. Every SQL literal touching
`conv` must be quoted — `'2'`, `IN ('1','3')` (fixed canonically in Ghost
2026-09-02; Ninja's local mirror is fully quoted since the 2026-09-03 sync —
upload pending, so the live host still carries the unquoted sites — §7).

**`conv` attribution convention (2026-09-03, builds 25+26 — synced same day; uploads pending).**
`'2'` is reserved for `evaluateRisk` (level) blocks alone; every fraud-detection,
behavior, forfeit or expiry closure writes `'4'`, with the `fraud_flags` tokens
carrying which mechanism closed the row (`FRAUD_DETECTION_V5.md` §14.1). Reading a
closed row is two steps: `conv` names the family, the tokens name the mechanism.

---

## 4. Indexes — keep only what a query uses

Index bytes rivalled data bytes before the reference tables were pruned from ~25–27
indexes to **~10**, EXPLAIN-verified. The canonical keep-set:

| Index | Serves | Notes |
| :-- | :-- | :-- |
| `PRIMARY (id)` | row identity | — |
| `uq_fingerprint` / `idx_fingerprint` | the hot per-sync identity lookup | **UNIQUE** in V2-B (`uq_`); non-unique HMAC lookup in V2-A |
| `idx_cid_sid` | attribution dedup / Tier-1 recovery | — |
| `idx_ip_an_instdom` **or** `idx_device_sig` | duplicate resolution | V1/V2-A use the IP triple; **V2-B replaces it with `device_sig`** (IP left identity) |
| `idx_device` | `device_hash` clone clustering | fraud sweep; live name is `idx_nj25_device` in **both** 25 and 26 (§9.5) |
| `idx_behavior_hash` | farm-collision detection | behaviour layer |
| `idx_decision_created` | terminalization sweep scan | — |
| `created_date`, `an`, `conv` | reporting / finalization scans | `conv` kept despite low cardinality — it is selective (`0`/NULL ≈ 8% of rows) |

**Hygiene rules (learned by EXPLAIN, not by intuition):** drop indexes that are only
*written* (e.g. `updated = NOW()` every sync), and indexes on non-sargable columns
used only inside `COALESCE`/`CASE` (`level`/`script`). A feared filesort that
EXPLAIN shows touching ≤ 2 rows does not justify an index (`cid` was dropped on that
basis). `[OWNER — manual]` Re-verify with `EXPLAIN` on the live row-count before adding or
removing — a live-DB action the owner runs; the agent reasons from the schema snapshot and
prepares the change.

The 2026-09-02 stamp pair (§3) ships **deliberately unindexed**: `conv_date` is
never filtered by a live query, and `updated_at` is rewritten on virtually every row
touch — the same write-only category the `updated` index was dropped for. Do not
"complete" the migration by indexing either.

---

## 5. Growing the schema safely — the capability probe

New columns are added **behind a one-time cached `SHOW COLUMNS` probe** so that code
and migration are deploy-order-independent: if the column is absent, the code
bypasses every new write and behaves exactly as before.

- **Never** put a new nullable column in the unconditional `INSERT` base list — every
  insert fails "Unknown column" until the migration runs.
- Keep **independent probes** for independent features — e.g. a `device_sig` probe
  must be separate from a `has_fraud_columns` probe, or one missing column disables
  the other's writes.

**When a probe cannot help — the strict-order migration.** A column whose
correctness rests on the *database itself* — a DEFAULT that must fire inside the
existing unconditional INSERT, an `ON UPDATE` clause — gains nothing from a probe.
The pattern there (used by the 2026-09-02 `conv_date`/`updated_at` pair, §3) is a
strict deploy order instead: **run the ALTER before uploading the PHP** — the edited
`conv` writers reference `conv_date`, and against an unmigrated table every such
UPDATE throws "Unknown column". Backfill existing rows by **explicit assignment** —
`SET conv_date = created_date, updated_at = created_date` — an explicit write
**overrides `updated_at`'s ON UPDATE clause**, so the backfill lands on
`created_date`, not the migration time. The backfill is run-once, not idempotent:
re-running it after real `conv` changes have been stamped would clobber them back to
`created_date`. Migration file:
`26 - Ad Block Ghost/backend/docs/migrate_conv_date_updated_at.sql`. (Like §4's
EXPLAIN checks, running the ALTER/backfill is an `[OWNER — manual]` live-DB action.)

---

## 6. Schema by identity variant

| Aspect | V1 | V2-A | V2-B |
| :-- | :-- | :-- | :-- |
| `fingerprint` content | *(client holds the encrypted blob; row keyed by `id`)* | HMAC hash (recomputable) | random 64-hex token (write-once) |
| `fingerprint` index | — / `idx_cid_sid` dedup | `idx_fingerprint` (non-unique) | **`uq_fingerprint`** (unique) |
| Duplicate index | `idx_ip_an_instdom` | `idx_ip_an_instdom` | **`idx_device_sig`** (IP triple removed) |
| Hardware / telemetry columns | absent | present | present |
| `device_hash` (fraud) | absent | present | present |
| `device_sig` (identity) | absent | absent | **present (separate column)** |
| Behaviour columns | absent | present | present |
| `conv` enum domain | `'0'..'3'` (no terminal `4`) | `'0'..'4'` | `'0'..'4'` |
| `conv_date` / `updated_at` stamp pair (2026-09-02, §3) | absent (frozen legacy lane) | present (canonical; live `ninja25` not yet migrated — §9.5) | present |
| Table charset | `utf8mb3_bin` (V1 default) | `utf8mb4_unicode_ci` | `utf8mb4_unicode_ci` |

V1 tables are lean (identity/attribution/network/state only); V2 adds the hardware,
device and behaviour groups; V2-B is the only one with a distinct `device_sig`
identity column alongside the fraud `device_hash`. One live deviation: Wonder (23)
is V1 but carries a partial 8-column diagnostic subset of the hardware group
(§9.5) — the column count follows the variant *plus* any such per-build backfill.

---

## 7. Per-extension implementation

**Legend:** ✅ present · ⚠️ partial · ❌ absent · ⬜ to copy.

| Aspect | § | 12 Pro | 21 North | 22 Hunter | 23 Wonder | 25 Ninja | 26 Ghost | 27 |
| :-- | :-- | :-- | :-- | :-- | :-- | :-- | :-- | :-- |
| Table (example) | — | `adbprocup11` | `adbnorth21` | `adshunter22` | `c23wonderblock` | `ninja25` | `ghost26` | ⬜ |
| Identity index | 4 | — AES `syncuid`¹² | ⚠️ singles (`cid`, `sid`) | `idx_cid_sid` | `idx_cid_sid` | `idx_fingerprint` | **`uq_fingerprint`** | ⬜ |
| Duplicate index | 4 | ⚠️ `ip`/`cid`+script¹² | ⚠️ `ip` single | IP triple | IP triple | IP triple | **`device_sig`** | ⬜ |
| Hardware / telemetry columns | 6 | ❌ (network-risk only) | ❌ | *(no dump)* | ⚠️ 8 of 19 | ✅ 19 | ✅ 19 | ⬜ |
| `device_sig` ≠ `device_hash` | 6 | ❌ | ❌ | ❌ | ❌ | ⚠️ hash only | ✅ both | ⬜ |
| Capability probe | 5 | n/a | n/a | n/a | n/a | ✅ | ✅ | ⬜ |
| Index count pruned | 4 | ✅ (2 composites) | ❌ legacy (16 singles) | — | ✅ (4 composites) | ✅ (25→10) | ✅ (27→10) | ⬜ |
| `conv` excluded from sync UPDATE | 3 | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ⬜ |
| `conv_date` + `updated_at` stamp pair | 3 | ❌ (frozen Gen 0) | ❌ | ❌ | ❌ | ❌ | ✅ | ⬜ |
| `conv` SQL literals quoted (ENUM trap) | 3 | — | ✅ params only | ✅ params only | ✅ params only | ❌ fraud pipeline | ✅ | ⬜ |

> **Hunter (22) has no structure dump**, but its `idx_cid_sid` and IP-triple facts above are
> **confirmed from the endpoint code** (`22 - Ad Block Hunter/backend/adshunter_2.php` — the
> `cid`+`sid` and `ip`+`an`+`instdom` resolution queries). The `(no dump)` / `—` markers apply
> only to the hardware-column and index-count cells, which the code does not settle.

¹² **`12` is `Gen 0` (Legacy, pre-family)** — table `adbprocup11`, charset
`utf8mb4_unicode_ci`. Identity is an **AES-encrypted `syncuid` handle held by the
client** (no `fingerprint`/`device` column); the row is keyed by `id` and resolved
through `idx_adb_ip_script_updates` / `idx_adb_cid_script_updates`. It carries the
**pre-family ad-counter group** (`googleflag`, `googleinjflag`, `reset`, `blockedads`,
`bingads`, `yahoo`, `popads`, `totblock`) and a **network-risk group** (`level`,
`risk`, `provider`, `proxy`, `type`, `flag`, `duplicate`) — but **no** hardware/
fraud/behaviour columns, and **none** of the family user-data group (`visit`,
`whitelistedDom`, `blockedDom`, `instdom`, `cosmeticDom`). `conv` enum is `'0'..'2'`;
`script` enum is `'0'..'4'` but is functionally 0/1 (the AES handle is
client-rewindable, so no durable state-2 opt-out). This schema is **frozen** — never
migrated toward V1/V2. Canonical Gen 0 DDL: `12 - Ad Block Pro/backend/docs/db_schema/adbprocup11.sql`
(and the template for the other pre-family builds, 15/17/20/24).

---

## 8. Conformance checklist

Run the **Common** items for every build, then add the block for the build's
identity variant (§6), applying the reachable-set filter (`00_GOAL.md` §3, decision B): a
schema check is `n/a` where the build's frozen table lacks the column, never a failure. The
variant is fixed at birth and never migrated, so a table sits in exactly one lane for life:
**Gen 0 (Legacy) · V1 · V2**. (The schema is frozen at birth; §5's capability-probe growth is a
**birth-time / create-extension** act, never part of an update — `00_GOAL.md` §5.)

> **`Gen 0` (Legacy, pre-family — e.g. 12) runs the V1 block (§8.2) for the
> identity-absence checks** (no `fingerprint` column, no hardware/fraud/behaviour
> groups) **but not** the V1 user-data items (`visit` snapshot, canonical `conv
> '0'..'3'`): a Gen 0 table has the **ad-counter group** and `conv '0'..'2'` instead,
> and its own pre-family indexes (`idx_*_ip/cid_script_updates`). Do not "fix" a Gen
> 0 table toward the canonical V1 DDL — its schema is a frozen lineage, not a gap
> (§7 note ¹²).

### 8.1 Common — every build

- [ ] Every column name is byte-identical to the reference tables; the **table name** is the only build-specific identifier in the DB — intro, `00_INDEX.md` §6 rule 2.
- [ ] `conv` is **absent** from the sync UPDATE's SET list; only the money path writes it — §3.
- [ ] Every SQL literal assigned or compared to `conv` is **quoted** — an unquoted number is read as the ENUM *index* (`2` → `'1'`), not the value — §3.
- [ ] `disabled` exists and is set on every uninstall path — §3.
- [ ] Indexes are pruned to the EXPLAIN-verified keep-set (~10); no write-only or non-sargable index survives — §4.
- [ ] Attribution columns store `an`/`cid`/`sid` verbatim and normalised identically on every path — §2.
- [ ] The variant's schema shape (§6) matches the chosen identity variant (`USER_CREATION_AND_UPDATE.md` §11).

### 8.2 V1 (Gen 1) only

- [ ] There is **no `fingerprint` handle column and no UNIQUE identity index** — the row is keyed by `id`, and dedup rests on `idx_cid_sid` + the IP triple — §2, §6.
- [ ] The hardware / telemetry, `device_hash`, `device_sig`, and behaviour column groups are **absent** — that absence is correct, not a gap. (Wonder's partial 8-of-19 diagnostic subset is a per-build backfill, not the fraud stack — §6, §9.5.)
- [ ] `visit` holds a **per-sync snapshot**, overwritten wholesale — §2, `USER_CREATION_AND_UPDATE.md` §5.5.
- [ ] `conv`'s enum domain is `'0'..'3'` — **no** terminal `4` — §6.

### 8.3 V2 (Gen 2/3) only

- [ ] The identity column `fingerprint` is **write-once and UNIQUE-indexed** (`uq_fingerprint`) in V2-B; V2-A's `idx_fingerprint` is **non-unique** and recomputed by several paths — §2, §4, §6.
- [ ] `device_sig` (stable, identity — V2-B only) and `device_hash` (volatile, fraud) are **separate columns from separate component sets** — never one column — §6.
- [ ] `visit` merging is **non-destructive**; a deserialisation failure cannot wipe history — §2, `USER_CREATION_AND_UPDATE.md` §6.7.
- [ ] Every column added after the baseline is written **behind a capability probe**, never in the unconditional INSERT base list; independent features have **independent** probes — §5. (The `conv_date`/`updated_at` pair is the deliberate exception: §5's strict deploy order replaces the probe.)
- [ ] `conv`'s enum domain extends to the terminal `'4'` — §6.
- [ ] `conv_date` sits directly **after `conv`** and `updated_at` is the **last** column — both `datetime DEFAULT current_timestamp()`, `updated_at` also `ON UPDATE current_timestamp()`; neither is indexed — §3, §4, §9.3.
- [ ] **Every statement that writes `conv` also maintains `conv_date`** — plain `NOW()` under a transition-guaranteeing WHERE guard; the NULL-safe `IF(conv <=> ?, conv_date, NOW())` (assigned **before** `conv`) at every site that can rewrite the same value — §3.

---

## 9. Appendix — canonical DDL by variant

Compiled from the live dumps of `adbnorth21`, `c23wonderblock` (V1), `ninja25`
(V2-A) and `ghost26` (V2-B), taken 2026-08-27. The cross-check confirmed the copy
rule exactly: every column shared by two builds is spelled identically, and the
column sets nest cleanly — 21 (29) ⊂ 23 (37) ⊂ 25 (60) ⊂ 26 (62). The 2026-09-02
`conv_date`/`updated_at` stamp pair (§3) has since brought Ghost to 64 columns and
is included in the V2 blocks below (§9.5). `NEWTABLE`
marks the **only** per-build identifier; copy the block for the chosen variant,
rename the table, touch nothing else. Live-vs-canonical deviations are logged in
§9.5.

### 9.1 V1

Columns are Ad Block North (21) verbatim. The index set below is the §4-conformant
one (composites + reporting singles); no dumped V1 build matches it exactly —
see §9.5.

```sql
CREATE TABLE `NEWTABLE` (
  `id` int(8) UNSIGNED NOT NULL AUTO_INCREMENT,
  `ip` varbinary(16) NOT NULL,
  `mkt` char(2) DEFAULT NULL,
  `level` smallint(5) UNSIGNED NOT NULL DEFAULT 0,
  `script` enum('0','1','2','3','4') DEFAULT NULL,
  `conv` enum('0','1','2','3') DEFAULT NULL,
  `an` char(2) DEFAULT NULL,
  `created_date` datetime DEFAULT current_timestamp(),
  `disabled` date DEFAULT NULL,
  `enabled` date DEFAULT NULL,
  `updated` datetime DEFAULT NULL,
  `updates` smallint(5) UNSIGNED DEFAULT NULL,
  `flagged` date DEFAULT NULL,
  `visit` varchar(1500) DEFAULT NULL,
  `whitelistedDom` varchar(1500) DEFAULT NULL,
  `blockedDom` varchar(1500) DEFAULT NULL,
  `extid` char(4) DEFAULT NULL,
  `version` varchar(4) DEFAULT NULL,
  `duplicate` smallint(3) UNSIGNED NOT NULL DEFAULT 0,
  `provider` tinytext CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci DEFAULT NULL,
  `organisation` varchar(32) DEFAULT NULL,
  `hostname` varchar(64) DEFAULT NULL,
  `proxy` enum('yes','no') DEFAULT NULL,
  `type` enum('Business','Compromised Serv','HTTP','HTTPS','Residential','SOCKS','SOCKS4','VPN','Wireless') DEFAULT NULL,
  `risk` tinyint(3) UNSIGNED DEFAULT NULL,
  `cid` varchar(127) DEFAULT NULL,
  `sid` varchar(63) DEFAULT NULL,
  `typetag` varchar(12) NOT NULL,
  `instdom` varchar(511) DEFAULT NULL,
  PRIMARY KEY (`id`),
  KEY `idx_cid_sid` (`cid`,`sid`),
  KEY `idx_ip_an_instdom` (`ip`,`an`,`instdom`(255)),
  KEY `an` (`an`),
  KEY `conv` (`conv`),
  KEY `created_date` (`created_date`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb3 COLLATE=utf8mb3_bin;
```

### 9.2 V2-A

Ninja (25) verbatim, with two departures: the `device_hash` index is renamed to its
neutral name, and the canonical 2026-09-02 stamp pair `conv_date`/`updated_at` (§3)
is included — live `ninja25` has not migrated yet (§9.5). Adds to V1: `acceptable_ads_disabled`, the 19 hardware/telemetry columns,
`device_hash`, the fraud/behaviour columns, `decision`, and the `'4'` value in
`conv`. `fingerprint` is the HMAC hash — non-unique index.

```sql
CREATE TABLE `NEWTABLE` (
  `id` int(8) UNSIGNED NOT NULL AUTO_INCREMENT,
  `ip` varbinary(16) NOT NULL,
  `mkt` char(2) DEFAULT NULL,
  `level` smallint(5) UNSIGNED NOT NULL DEFAULT 0,
  `script` enum('0','1','2','3','4') DEFAULT NULL,
  `conv` enum('0','1','2','3','4') DEFAULT NULL,
  `conv_date` datetime DEFAULT current_timestamp(),
  `an` char(2) DEFAULT NULL,
  `created_date` datetime DEFAULT current_timestamp(),
  `disabled` date DEFAULT NULL,
  `enabled` date DEFAULT NULL,
  `updated` datetime DEFAULT NULL,
  `updates` smallint(5) UNSIGNED DEFAULT NULL,
  `acceptable_ads_disabled` date DEFAULT NULL,
  `flagged` date DEFAULT NULL,
  `visit` varchar(1500) DEFAULT NULL,
  `whitelistedDom` varchar(1500) DEFAULT NULL,
  `blockedDom` varchar(1500) DEFAULT NULL,
  `extid` char(4) DEFAULT NULL,
  `version` varchar(4) DEFAULT NULL,
  `duplicate` smallint(3) UNSIGNED NOT NULL DEFAULT 0,
  `provider` tinytext DEFAULT NULL,
  `organisation` varchar(32) DEFAULT NULL,
  `hostname` varchar(64) DEFAULT NULL,
  `proxy` enum('yes','no') DEFAULT NULL,
  `type` enum('Business','Compromised Serv','HTTP','HTTPS','Residential','SOCKS','SOCKS4','VPN','Wireless') DEFAULT NULL,
  `risk` tinyint(3) UNSIGNED DEFAULT NULL,
  `cid` varchar(127) DEFAULT NULL,
  `sid` varchar(63) DEFAULT NULL,
  `typetag` varchar(12) NOT NULL,
  `instdom` varchar(511) DEFAULT NULL,
  `cpu_cores` tinyint(4) DEFAULT NULL,
  `gpu_vendor` varchar(255) DEFAULT NULL,
  `gpu_renderer` varchar(255) DEFAULT NULL,
  `ram_gb` float DEFAULT NULL,
  `lang` varchar(10) DEFAULT NULL,
  `color_depth` tinyint(3) UNSIGNED DEFAULT NULL,
  `touch_points` int(11) DEFAULT NULL,
  `bat_charging` varchar(10) DEFAULT NULL,
  `bat_level` varchar(10) DEFAULT NULL,
  `timezone` varchar(64) DEFAULT NULL,
  `canvas_hash` char(64) DEFAULT NULL,
  `audio_fp` varchar(64) DEFAULT NULL,
  `fingerprint` char(64) DEFAULT NULL,
  `webdriver_present` tinyint(4) DEFAULT 0,
  `gl_ext_count` tinyint(4) DEFAULT NULL,
  `asn` varchar(24) DEFAULT NULL,
  `screen_res` varchar(50) DEFAULT NULL,
  `user_agent` mediumtext DEFAULT NULL,
  `is_vm` tinyint(1) DEFAULT 0,
  `vm_type` varchar(100) DEFAULT NULL,
  `diagnostic_date` datetime DEFAULT NULL,
  `device_hash` char(64) DEFAULT NULL,
  `fraud_score` smallint(6) DEFAULT NULL,
  `fraud_flags` varchar(255) DEFAULT NULL,
  `fraud_checked` datetime DEFAULT NULL,
  `behavior_score` smallint(6) DEFAULT NULL,
  `behavior_flags` varchar(255) DEFAULT NULL,
  `behavior_hash` char(64) DEFAULT NULL,
  `decision` enum('pending','human','bot','deferred') DEFAULT 'pending',
  `decided_date` datetime DEFAULT NULL,
  `updated_at` datetime DEFAULT current_timestamp() ON UPDATE current_timestamp(),
  PRIMARY KEY (`id`),
  KEY `conv` (`conv`),
  KEY `an` (`an`),
  KEY `created_date` (`created_date`),
  KEY `idx_fingerprint` (`fingerprint`),
  KEY `idx_cid_sid` (`cid`,`sid`),
  KEY `idx_ip_an_instdom` (`ip`,`an`,`instdom`(255)),
  KEY `idx_device` (`device_hash`),
  KEY `idx_behavior_hash` (`behavior_hash`),
  KEY `idx_decision_created` (`decision`,`created_date`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```

### 9.3 V2-B (the default for new builds)

Ghost (26) verbatim, same index rename. Differs from V2-A by exactly two columns —
`cosmeticDom` (feature) and `device_sig` (identity) — and by the index swaps:
`fingerprint` becomes **UNIQUE** (`uq_fingerprint`, random write-once token), and
`idx_device_sig` **replaces** `idx_ip_an_instdom` (IP left identity).

```sql
CREATE TABLE `NEWTABLE` (
  `id` int(8) UNSIGNED NOT NULL AUTO_INCREMENT,
  `ip` varbinary(16) NOT NULL,
  `mkt` char(2) DEFAULT NULL,
  `level` smallint(5) UNSIGNED NOT NULL DEFAULT 0,
  `script` enum('0','1','2','3','4') DEFAULT NULL,
  `conv` enum('0','1','2','3','4') DEFAULT NULL,
  `conv_date` datetime DEFAULT current_timestamp(),
  `an` char(2) DEFAULT NULL,
  `created_date` datetime DEFAULT current_timestamp(),
  `disabled` date DEFAULT NULL,
  `enabled` date DEFAULT NULL,
  `updated` datetime DEFAULT NULL,
  `updates` smallint(5) UNSIGNED DEFAULT NULL,
  `acceptable_ads_disabled` date DEFAULT NULL,
  `flagged` date DEFAULT NULL,
  `visit` varchar(1500) DEFAULT NULL,
  `whitelistedDom` varchar(1500) DEFAULT NULL,
  `blockedDom` varchar(1500) DEFAULT NULL,
  `cosmeticDom` mediumtext DEFAULT NULL,
  `extid` char(4) DEFAULT NULL,
  `version` varchar(4) DEFAULT NULL,
  `duplicate` smallint(3) UNSIGNED NOT NULL DEFAULT 0,
  `provider` tinytext DEFAULT NULL,
  `organisation` varchar(32) DEFAULT NULL,
  `hostname` varchar(64) DEFAULT NULL,
  `proxy` enum('yes','no') DEFAULT NULL,
  `type` enum('Business','Compromised Serv','HTTP','HTTPS','Residential','SOCKS','SOCKS4','VPN','Wireless') DEFAULT NULL,
  `risk` tinyint(3) UNSIGNED DEFAULT NULL,
  `cid` varchar(127) DEFAULT NULL,
  `sid` varchar(63) DEFAULT NULL,
  `typetag` varchar(12) NOT NULL,
  `instdom` varchar(511) DEFAULT NULL,
  `cpu_cores` tinyint(4) DEFAULT NULL,
  `gpu_vendor` varchar(255) DEFAULT NULL,
  `gpu_renderer` varchar(255) DEFAULT NULL,
  `ram_gb` float DEFAULT NULL,
  `lang` varchar(10) DEFAULT NULL,
  `color_depth` tinyint(3) UNSIGNED DEFAULT NULL,
  `touch_points` int(11) DEFAULT NULL,
  `bat_charging` varchar(10) DEFAULT NULL,
  `bat_level` varchar(10) DEFAULT NULL,
  `timezone` varchar(64) DEFAULT NULL,
  `canvas_hash` char(64) DEFAULT NULL,
  `audio_fp` varchar(64) DEFAULT NULL,
  `device_sig` char(64) DEFAULT NULL,
  `fingerprint` char(64) DEFAULT NULL,
  `webdriver_present` tinyint(4) DEFAULT 0,
  `gl_ext_count` tinyint(4) DEFAULT NULL,
  `asn` varchar(24) DEFAULT NULL,
  `screen_res` varchar(50) DEFAULT NULL,
  `user_agent` mediumtext DEFAULT NULL,
  `is_vm` tinyint(1) DEFAULT 0,
  `vm_type` varchar(100) DEFAULT NULL,
  `diagnostic_date` datetime DEFAULT NULL,
  `device_hash` char(64) DEFAULT NULL,
  `fraud_score` smallint(6) DEFAULT NULL,
  `fraud_flags` varchar(255) DEFAULT NULL,
  `fraud_checked` datetime DEFAULT NULL,
  `behavior_score` smallint(6) DEFAULT NULL,
  `behavior_flags` varchar(255) DEFAULT NULL,
  `behavior_hash` char(64) DEFAULT NULL,
  `decision` enum('pending','human','bot','deferred') DEFAULT 'pending',
  `decided_date` datetime DEFAULT NULL,
  `updated_at` datetime DEFAULT current_timestamp() ON UPDATE current_timestamp(),
  PRIMARY KEY (`id`),
  UNIQUE KEY `uq_fingerprint` (`fingerprint`),
  KEY `conv` (`conv`),
  KEY `an` (`an`),
  KEY `created_date` (`created_date`),
  KEY `idx_cid_sid` (`cid`,`sid`),
  KEY `idx_device` (`device_hash`),
  KEY `idx_behavior_hash` (`behavior_hash`),
  KEY `idx_decision_created` (`decision`,`created_date`),
  KEY `idx_device_sig` (`device_sig`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```

### 9.4 Support tables (shared shape, from Ninja 25)

```sql
CREATE TABLE `sync_log` (
  `id` bigint(20) UNSIGNED NOT NULL AUTO_INCREMENT,
  `install_id` int(10) UNSIGNED NOT NULL,
  `server_ts` datetime NOT NULL,
  `src_ip` varbinary(16) NOT NULL,
  `visit_size` smallint(5) UNSIGNED DEFAULT NULL,
  `new_domains` smallint(5) UNSIGNED DEFAULT NULL,
  `visit_hash` char(16) DEFAULT NULL,
  PRIMARY KEY (`id`),
  KEY `idx_install_ts` (`install_id`,`server_ts`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;

CREATE TABLE `errors` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `ip` varchar(45) NOT NULL,
  `time` timestamp NOT NULL DEFAULT current_timestamp(),
  `type` varchar(32) NOT NULL,
  `source` varchar(127) NOT NULL,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```

### 9.5 Live deviations from canonical (observed 2026-08-27)

- **21 North — legacy pre-pruning index set.** 16 indexes, all single-column
  (`ip`, `mkt`, `level`, `script`, `conv`, `an`, `created_date`, `enabled`,
  `extid`, `version`, `sid`, `disabled`, `updated`, `cid`, `flagged` + PK); no
  composite `idx_cid_sid` or IP triple. It contains exactly the write-only
  (`updated`, `flagged`) and non-sargable (`level`, `script`) indexes §4 says to
  prune.
- **23 Wonder — V1 plus a partial diagnostic backfill.** Eight columns from the V2
  hardware group (`cpu_cores`, `gpu_vendor`, `gpu_renderer`, `screen_res`,
  `user_agent`, `is_vm`, `vm_type`, `diagnostic_date` — same spellings, confirming
  the naming rule even mid-deviation). This was the **first attempt at hardware
  collection**, historical and superseded by the full V2 telemetry set — canonical
  V1 stays the lean 29-column shape (§9.1). Its three list columns are `text`
  instead of `varchar(1500)`, and reporting is served by one composite
  `idx_created_disabled_enabled_updated` instead of the `an`/`conv`/`created_date`
  singles.
- **25 + 26 — leaked index name.** The `device_hash` index is named
  `idx_nj25_device` in *both* tables: the Ninja slug travelled into Ghost with the
  wholesale copy. Harmless (index names are table-local) unless code ever uses
  `FORCE INDEX`; new builds use the neutral `idx_device`.
- **`conv` terminal state.** V1 enums stop at `'3'`; the `'4'` state (§3) is only
  representable from V2 on.
- **Feature columns are not variant columns.** `acceptable_ads_disabled` (25, 26)
  and `cosmeticDom` (26) follow the *feature*, not the identity variant — each
  added behind its own capability probe (§5).
- **25 Ninja — 2026-09-02 stamp pair not yet migrated.** `conv_date`/`updated_at`
  shipped first in Ghost (`26 - Ad Block Ghost/backend/docs/migrate_conv_date_updated_at.sql`
  — ALTER **before** PHP, §5) and are canonical in the V2 blocks above; live
  `ninja25` carries neither column, and its live fraud pipeline still has the
  unquoted `conv` literals (§3, §7 — the local mirror was fully quoted in the
  2026-09-03 sync; upload pending). Ninja's migration file now exists:
  `25 - Ninja Block/backend/docs/migrate_conv_date_updated_at_ninja25.sql` (run
  **before** uploading the synced PHP), with a deliberate divergence from
  ghost26's backfill: pre-migration rows get `conv_date = NULL` ("set before the
  column existed" — ghost26 backfilled `created_date` instead) and `updated_at =
  GREATEST(…)` of the row's known date columns rather than a flat
  `created_date`. The V1 tables are a frozen lane and stay without the
  pair (§6, §8).

---

*Database schema reference. Copy the DDL from §9 for the chosen variant and rename
**only the table itself**; every column name and index stays canonical. Client-facing
key spellings are remapped in the server code, never in the schema (`00_INDEX.md`
§6). Column ownership → `USER_CREATION_AND_UPDATE.md` §7; fraud columns →
`FRAUD_DETECTION_V5.md`.*
