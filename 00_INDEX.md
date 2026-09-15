# Architecture — index & build guide

**The map for this folder: what each document owns, the frontend↔backend split
that governs how to use them, the order to read them, the end-to-end build
procedure, and the canonical key-mapping that makes copying a backend safe.**

Point a build agent at this folder to either (a) create a new extension from A to
Z, or (b) measure and upgrade an existing one. Start here, then follow the reading
order in §3.

---

## 1. Read this first: the frontend↔backend asymmetry

The family is built on a deliberate split. Getting it wrong is the most expensive
mistake, so it comes before everything else.

| | **Frontend (the extension)** | **Backend (the server)** |
| :-- | :-- | :-- |
| How a new build is produced | **Reimplemented from scratch, differently** | **Copied wholesale from a previous build** |
| What changes | file tree, file names, module boundaries, coding style, symbol names, **and all key names** | **the two endpoint files' names** (sync + uninstall) and **a short variable set** (table name, a freshly-generated identity key, product number, market list, host/asset URLs) + the **key names** — nothing else (§2.1–§2.3 of `BACKEND_ARCHITECTURE.md`) |
| What stays | only the *behavior and invariants* in the spec | the structure, the logic, **the function names** — everything except keys |
| Why | the Chrome Web Store flags near-identical extensions as related / plagiarized; structural sameness is a policy and correlation risk | the server is never seen by store review, so sameness is safe and desirable; reimplementing it would only add bugs |
| Its spec is… | a **what/why guideline with no skeleton to copy** (`EXTENSION_COMPOSITION.md`) | a **description of the canonical backend to copy**, citing the reference build by `file:line` on purpose |

**What legitimately differs between two builds' backends:** the identity variant
(V1 / V2-A / V2-B), the monetization posture (none / dormant / live), the two
renamed endpoint files (sync + uninstall) plus the per-build variable set
(§2.1–§2.3 of `BACKEND_ARCHITECTURE.md`), and — of course — every key name.
Everything else is the same server. (The popup-landing style, the element picker
and the cookie blocker are **frontend** choices — §5 — and do not change the
backend.)

> **So:** when a backend document cites `ninja-adb21.php:871` or `config.php:39`,
> that is not an example to paraphrase — it is the canonical code to copy. When the
> frontend document refuses to name a file, that is not vagueness — it is the
> anti-template rule. The two halves are documented in opposite spirits on purpose.

---

## 2. Document map

| Document | Owns | Side | For a new build |
| :-- | :-- | :-- | :-- |
| **`00_GOAL.md`** | the charter: the two co-equal goals, the convergence rule, the reachable-set filter (decision B), the per-generation capability levels | Keystone | **read first** — the target every checklist measures against (no per-extension table / checklist of its own) |
| **`EXTENSION_COMPOSITION.md`** | the extension's composition, features, invariants, and the client half of the server contract | Frontend | **reimplement** (guideline; run its §26 checklist) |
| **`RULES_AND_PRIORITIES.md`** | DNR rule tiers, the priority ladder, the `script` flag lifecycle, the enable gate | Shared | server authors the rules (copy); client applies them (its own way) |
| **`USER_CREATION_AND_UPDATE.md`** | the user record, identity generations V1/V2-A/V2-B, resolution & dedup, the write surface | Backend | **copy + remap keys**; pick the identity variant |
| **`FRAUD_DETECTION_V5.md`** | hardware/VM, network risk, the behaviour layer, tiers, finalization | Backend | **copy + remap keys** |
| **`BACKEND_ARCHITECTURE.md`** | endpoint request/response assembly, cache-generation pipelines, outage hardening, the scheduler | Backend | **copy + remap keys** |
| **`DB_SCHEMA.md`** | the user-table columns, identity/dedup indexes, index hygiene, the migration probe | Backend | **copy + remap keys** |
| **`DATA_SOURCES.md`** | the blocklist / cosmetic / scriptlet / surrogate sources and how the caches are generated | Backend / data | **copy** the pipeline; ship the data |
| **`BLOCKLIST_WHITELIST_SYSTEM.md`** | the **shared list factory** (repo `blocklist-v4`, "Automated All In One List V2") that produces every network/cosmetic artifact the backends mirror: the 15 curation sheets, the curate→verify→compile→catalog pipeline, the `dist/` contract, the precedence ladder, and the per-brand shim rules | Backend / data — **upstream of every build** | **read, never copy** — ONE factory serves all builds; a new backend gets a thin mirror shim, not its own pipeline. Its §17.5 playbooks answer "which sheet does a domain go in" and "how to add a source". Visual companion: `la-fabrique-des-listes.html` (same folder) |
| **`DETECTION_CHANGELOG.md`** | the chronological history of the detection rewrite — post-mortems and why the current design is what it is | Backend / history | reference for context (not a build step) |

Each **specification** document ends with two things: a **per-extension table**
(where each shipped build stands) and a **conformance checklist** (the acceptance
test for that document's subject). Two documents are exempt: `DETECTION_CHANGELOG.md`
(history, not a specification) and `BLOCKLIST_WHITELIST_SYSTEM.md` (a **system doc**
for the one shared factory — there is nothing per-extension about it; its own
"acceptance test" is the factory's invariants section, §19 there).

---

## 3. Reading order

**To create a new extension**

1. `00_GOAL.md` — the charter: the two co-equal goals and what "done" means, before anything else.
2. This index — the asymmetry (§1) and the build procedure (§4).
3. `EXTENSION_COMPOSITION.md` — the whole frontend.
4. `USER_CREATION_AND_UPDATE.md` §11 — pick the identity variant (default **V2-B**).
5. `RULES_AND_PRIORITIES.md` — the rule/priority/`script` contract both sides honor.
6. `BACKEND_ARCHITECTURE.md` → `DB_SCHEMA.md` → `USER_CREATION_AND_UPDATE.md` → `FRAUD_DETECTION_V5.md` — the backend to copy.
7. `DATA_SOURCES.md` — the data and cache pipeline — then `BLOCKLIST_WHITELIST_SYSTEM.md`
   for where that data actually comes from (the shared factory the new backend's
   generators will MIRROR, not rebuild — the shim pattern is described there, §17.2).
8. Run every document's end checklist — the agent runs the `[AGENT — offline]` items; the `[OWNER — manual]` runtime smoke test (`EXTENSION_COMPOSITION.md` §26) is handed to the owner to load and run (§7).

**To upgrade an existing extension**

Read `00_GOAL.md` first (the target and the reachable-set filter), then open each
document's **per-extension table** (its final section), find the target build's `⚠️` /
`❌` cells, read the section each points to, and apply. The tables are the gap-analysis
surface; the checklists are the acceptance test.

> **Hard limit on any upgrade:** never change an existing build's **identity or
> uniqueness mechanism** — it re-keys and orphans every user already in the database
> (`BACKEND_ARCHITECTURE.md` §8.3, `USER_CREATION_AND_UPDATE.md` §2.3). A `❌` in an
> **identity-coupled** row — the whole fraud stack on a V1 build — is therefore
> **permanent by design, not a gap**; leave it. Only close feature gaps that do not
> touch identity (rules, cosmetics, zero-flicker, scriptlets, the picker, YouTube /
> cookie handling, UI).

**To modernize a legacy (`Gen 0`, pre-family) build** — e.g. `12 - Ad Block Pro`,
and the other pre-family builds (`15`, `17`, `20`, `24`).

A legacy build predates the family: its identity is an **AES-encrypted handle on a
pre-family schema** (ad-counter columns, `conv 0–2`, no `fingerprint`/fraud
columns). It is a **third mode**, distinct from both "create new" and "upgrade a
family member":

- **The identity, DB schema, dedup and backend generation are frozen `Gen 0`
  forever** — never migrated toward V1/V2 (that orphans the user base, same hard
  limit as above). These rows read `Gen 0` / `—` in every table and are **not gaps**.
- **The whole frontend is progressively re-platformed onto this baseline** — the
  reserved-ID rule model, the priority ladder, local-first storage, structured
  zero-flicker, action-driven sync, MV3 hygiene. Each update closes a few more
  frontend `⚠️`/`❌`/`❓` cells; the per-build table is the progress tracker, exactly
  like a family upgrade.
- **Read order:** `00_GOAL.md` (the target + the reachable-set filter — for `Gen 0`,
  most DB-backed family behaviors are permanent `n/a`) → this index (§1, §4) →
  `EXTENSION_COMPOSITION.md` (reimplement the frontend gaps in the build's own structure)
  → `RULES_AND_PRIORITIES.md` (the rule ladder the frontend must honor) →
  `DATA_SOURCES.md` (feeds). The backend docs apply only as `Gen 0` context — you are
  **not** rebuilding the backend, only keeping the frontend's client-contract obligations
  correct against it.

> The goal of a legacy build is not to *become* a family member — it can't, its
> identity is fixed — but to **look more and more like the latest baseline on the
> frontend with every update**, so the portfolio converges in behavior while each
> build keeps its own lineage and its own distinct code.

---

## 4. The A-Z build procedure

1. **Name & keys.** Choose the product name and a unique key prefix. Every
   storage / request / response key is **remapped** from the canonical set (§6);
   only `an` / `cid` / `sid` stay verbatim. The DB is **not** remapped: across the **family**
   builds every table has the identical canonical column names — only the **table name** is new
   (§6 rule 2). *(This holds for the V1/V2 family; a **Gen 0** legacy build sits on a frozen
   pre-family schema with different columns — `00_GOAL.md` §4, `DB_SCHEMA.md` §7 note ¹².)*
2. **Frontend.** Reimplement the extension fresh against `EXTENSION_COMPOSITION.md`
   — new structure, naming and style. Run its `EXTENSION_COMPOSITION.md` §26 checklist.
3. **Identity.** Pick V1 / V2-A / V2-B (`USER_CREATION_AND_UPDATE.md` §11; default
   **V2-B**) and wire the client obligations (`EXTENSION_COMPOSITION.md` §6–§7).
4. **Backend.** Copy the canonical backend wholesale; keep functions and structure;
   remap the keys per §6. Apply `BACKEND_ARCHITECTURE.md`, `DB_SCHEMA.md`,
   `USER_CREATION_AND_UPDATE.md`, `FRAUD_DETECTION_V5.md`, `RULES_AND_PRIORITIES.md`.
5. **Data.** Point the pipeline at the shared sources (`DATA_SOURCES.md`) and
   generate the compiled/cosmetic caches; ship the static bundles.
6. **Distinctive choices.** Set the monetization posture, the popup-landing style,
   and which optional features ship.
7. **Verify.** Run every document's end checklist — the agent runs the `[AGENT — offline]` items; the `[OWNER — manual]` runtime smoke test (`EXTENSION_COMPOSITION.md` §26) is the owner's load-and-navigate pass (§7).

---

## 5. What differs per build — quick reference

| Choice | Options | Where specified |
| :-- | :-- | :-- |
| Key names / prefix | any, unique per build (`an`/`cid`/`sid` excepted) | §6; `EXTENSION_COMPOSITION.md` §0.3 |
| Identity variant | **Gen 0 Legacy (pre-family)** · V1 · V2-A · V2-B (default for new) | V1/V2: `USER_CREATION_AND_UPDATE.md` §3, §11 · Gen 0: `USER_CREATION_AND_UPDATE.md` §13, `DB_SCHEMA.md` §7 note ¹², `00_GOAL.md` §4 |
| Backend generation | **Gen 0 legacy (pre-family, AES handle, ad-counter schema)** · Gen 1 network-only · Gen 2/3 fraud stack | **fixed at creation, gated by identity, never migrated** → `BACKEND_ARCHITECTURE.md` §8 |
| Monetization posture | none · dormant · live | `EXTENSION_COMPOSITION.md` §24; `RULES_AND_PRIORITIES.md` §5 |
| Popup-landing style | A self-closing interstitial · B on-page notification | `EXTENSION_COMPOSITION.md` §17 |
| Element picker | present · absent | `EXTENSION_COMPOSITION.md` §16 |
| Cookie-consent blocker | present (default off) · absent | `EXTENSION_COMPOSITION.md` §15 |
| Frontend structure/style | entirely per build | `EXTENSION_COMPOSITION.md` §0.2 |

Everything not in this table is the **same** across builds.

---

## 6. The canonical key mapping

This is what makes "copy the backend, change the keys" safe. Every keyed value has
a fixed **role** that appears in up to five places: the frontend's local storage,
the sync-mirror subset, the sync **request** body, the sync **response** body, and
a DB column. **Copying a backend means keeping the roles and the flow identical and
changing only the spellings.** Only `an` / `cid` / `sid` never change.

Names below are shown as *roles*, with an example spelling from a recent build in
parentheses purely to illustrate — a new build invents its own spelling for every
non-fixed row.

| Role (example spelling) | Local | Sync mirror | Request | Response | DB column | Fixed? |
| :-- | :-: | :-: | :-: | :-: | :-: | :-: |
| Identity handle (`…_fp` / `…information` / `backdetail`) | ✅ | ✅ | ✅ | ✅ | `fingerprint` (neutral) | remap |
| Fingerprint version (`…_fp_version`) | ✅ | — | ✅ | ✅ | — | remap |
| Migration flag (`…_fp_needs_migration`) | ✅ | — | ✅ | — | — | remap |
| Extension version (`…_ext_v`) | ✅ | ✅ | ✅ | — | `version` | remap |
| Install-sites snapshot (`…_install_sites`) | ✅ | ✅ | ✅ | — | `instdom` (serialized) | remap |
| Attribution network (`an`) | ✅ | ✅ | ✅ | — | `an` | **FIXED** |
| Attribution click id (`cid`) | ✅ | ✅ | ✅ | — | `cid` | **FIXED** |
| Attribution sub id (`sid`) | ✅ | ✅ | ✅ | — | `sid` | **FIXED** |
| Visited-sites map (`…_visited_sites`) | ✅ | ✅ (deferred) | ✅ (counts) | ✅ (merged) | `visit` (serialized) | remap |
| Sync interval (`…_sync`) | ✅ | — | ✅ | ✅ | — | remap |
| Last-sync time (`…_last_sync`) | ✅ | — | ✅ | — | `updated` (derived) | remap |
| Data-version stamp (`…_data_version`) | ✅ | — | — | ✅ | *(constant)* | remap |
| Acceptable-ads flag (`…_acceptable_ads`) | ✅ | — | ✅ | — | `acceptable_ads_disabled` (date stamp) | remap |
| User allow-list (`…_user_allow`) | ✅ | — | ✅ | — | `whitelistedDom` | remap |
| User block-list (`…_user_block`) | ✅ | — | ✅ | — | `blockedDom` | remap |
| User cosmetics (`…_user_cosmetics`) | ✅ | — | ✅ | — | `cosmeticDom` (26+) | remap |
| Hardware signals (`…_fingerprint_data` — wrapper key only, see rule 4) | — (in-memory) | — | ✅ (V2, transient) | — | telemetry columns | remap |
| Inline rules (`rules`) | ✅ (cache) | — | — | ✅ | *(assembled)* | remap |
| System whitelist (`…_system_whitelist`) | ✅ (cache) | — | — | ✅ | *(assembled)* | remap |
| Zero-flicker directive (`…_local_css`) | ✅ (cache) | — | — | ✅ | *(assembled)* | remap |
| Cosmetic feeds + versions (`…_css_*` / `…_specific_*`) | ✅ (cache) | — | — | ✅ | — | remap |
| Compiled-rules version (`…_rules_version`) | ✅ (cache) | — | — | ✅ | — | remap |
| Scriptlet excludes (`…_scriptlet_excludes`) | ✅ (cache) | — | — | ✅ | — | remap |
| Server state (`script` / `level` / `conv` / `updates` / `typetag`) | ✅ | — | ✅ | ✅ (V1: in handle · V2: plain keys) | own columns | server-owned |

**The four rules the mapping enforces:**

1. **`an` / `cid` / `sid` are never renamed.** They are copied verbatim from the
   store URL and matched byte-for-byte by the backend. (`EXTENSION_COMPOSITION.md`
   §7.1, `USER_CREATION_AND_UPDATE.md` §4.)
2. **Remapping is a wire-layer concern only — it never reaches the DB.** The DB
   column names are canonical (`fingerprint`, `visit`, `instdom`, `whitelistedDom`)
   and **identical across the family builds** (V1/V2); the only per-build difference inside the
   DB is the **table name** (`DB_SCHEMA.md`). *(A **Gen 0** legacy build is the exception — its
   frozen pre-family ad-counter schema has different columns and none of `fingerprint`/`visit`
   — `00_GOAL.md` §4.)* The client-facing spellings live in the
   server code: keep the backend's read/write of each role pointed at the remapped
   wire key while the column underneath keeps its canonical name.
3. **The server-owned state** (`script`, `level`, `conv`, `updates`, `typetag`,
   `enabled`, `disabled`, tier/behaviour columns) is authoritative in the row and,
   for **V1 only**, additionally transported inside the encrypted handle. The
   client stores and echoes it but never interprets it (`RULES_AND_PRIORITIES.md`
   §5; `USER_CREATION_AND_UPDATE.md` §3).
4. **The hardware-signals array is remapped at the wrapper only.** The outer key
   (`…_fingerprint_data`) gets a per-build spelling, but the **inner keys**
   (`audio_fp`, `screen_res`, `webdriver_present`, …) are canonical and never
   remapped — they match the telemetry DB columns one-for-one, which is what lets
   the fraud layer consume a fresh posted array and a `SELECT *` row through the same
   code path (`FRAUD_DETECTION_V5.md` §2.1). *One deliberate exception:* the raw
   `canvas` signal is hashed on arrival and stored as `canvas_hash`, so that single
   pair is name-shifted by design — every other inner key is identical on the wire
   and in the column.

---

## 7. Conventions across the folder

- **Tags** `[MANDATORY]` / `[TOGGLEABLE]` / `[OPTIONAL]` / `[VARIANT: choose-one]`
  are defined in `EXTENSION_COMPOSITION.md` §0.1 and reused throughout.
- **Actor tag** `[AGENT — offline]` / `[OWNER — manual]` marks who performs each step of the
  build/update **procedure and its checklists**. `[AGENT — offline]` = things the agent can prove
  without a browser or a live server: `node`/logic simulations, static/`grep` audits, code review,
  reading console logs or screenshots the owner pastes in. `[OWNER — manual]` = anything needing a
  real browser or the live server/DB: loading the unpacked extension, navigating/visual testing,
  uploading files, DB migrations, store submission, any live-server request. **The agent never runs
  an `[OWNER — manual]` step** — it finishes its offline work, hands the owner an exact checklist of
  the manual steps, and waits. (This tag is applied throughout the procedure/spec docs; it is *not*
  applied to `DETECTION_CHANGELOG.md`, which is a historical log, not a procedure.)
- **Backend docs cite the canonical implementation by `file:line`** — that is the
  code to copy. **The frontend doc never cites a path** — that is the anti-template
  rule (§1). Do not "fix" either convention to match the other.
- **Every specification document ends with a per-extension table + a conformance checklist** (the history doc, this index, and the `00_GOAL.md` charter carry neither). The backend-spec checklists are **bucketed Common / V1-only / V2-only** — run the set for the build's generation, **filtered by the reachable-set rule** (`00_GOAL.md` §3: a Gen 0 build runs Common + only the items its frozen schema supports). One exception: `EXTENSION_COMPOSITION.md` §26 is bucketed **by topic** (with inline `[V2 only]` tags), not by generation — apply the filter per item there.
- Absolute values (priority numbers, the inline↔compiled rule-ID boundary, cache
  TTLs) are backend contracts shared across builds; per-build freedom is limited to
  the choices in §5.

---

*Index & build guide. It opens with the `00_GOAL.md` charter (the target) and maps the seven
specification documents plus two system/history docs — `BLOCKLIST_WHITELIST_SYSTEM.md` (the
shared list factory every build consumes) and `DETECTION_CHANGELOG.md`; read them in the
order of §3.*
