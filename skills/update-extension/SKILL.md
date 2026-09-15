---
name: update-extension
description: >-
  Bring an EXISTING browser-extension build up to the latest architecture baseline
  the way it is actually done: against a WRITTEN target, in a with-you loop that
  validates every phase before moving on, across three stages — frontend, then
  backend, then the merge of the two. Works on a family member (21/22/23/25/26/27)
  OR a documented Gen 0 legacy build (e.g. 12 - Ad Block Pro) via its
  legacy-modernization track (frontend re-platformed, identity frozen). Detects the
  build's identity from its own code, drafts a Target State for you to confirm, finds
  the closable (non-identity-coupled) gaps, and applies them iteration by iteration
  with a test/validate gate between phases. It NEVER migrates a build's identity
  mechanism (that orphans every user), and treats "monetization must not break" as a
  standing invariant. Ends by writing new lessons back to the specs and emitting a
  deployment checklist — it does not touch the live server or DB.
  Slash-only; run it as /update-extension <project-folder> [scope].
argument-hint: <project-folder> [scope]
disable-model-invocation: true
---

# update-extension

Bring an already-shipped extension up to the current baseline **the way a person
who holds the finished picture in their head would do it** — not as a one-shot
plan-then-apply, but as a **collaborative, validate-every-phase loop against a
written target**. This is the "upgrade an existing extension" and
"modernize a legacy build" half of the Architecture folder's job (`00_INDEX.md` §3).

**Architecture folder (source of truth, absolute path):**
`/Users/megane/Desktop/Dev/00 - Architecture/`
A `§` reference below is a section either in one of that folder's documents (when a
document is named, e.g. `EXTENSION_COMPOSITION.md` §11) or in **this skill** (a bare `§N` —
e.g. §4, §8); the context makes clear which. If that folder ever moves, update this one path.

---

## 0. The rule that outranks everything

> **Never change an existing build's identity or uniqueness mechanism.** The handle
> format, the dedup/uniqueness keys, and the DB columns/indexes that back them are
> fixed at the build's birth. Changing any of them re-keys and **orphans every user
> already in the database** — a fleet-wide crash, not a bug you can patch
> (`00_INDEX.md` §3 hard limit; `BACKEND_ARCHITECTURE.md` §8.3;
> `USER_CREATION_AND_UPDATE.md` §2.3 invariant 6).

Consequences this skill honors at all times:

- A build's **identity variant / generation is permanent** — `Gen 0` (Legacy) / V1 /
  V2-A / V2-B never migrate. "This build needs fraud" is a reason to build a *new* V2
  product, never to upgrade a legacy or V1 one.
- On a **legacy (`Gen 0`) or V1 build**, the fraud stack (hardware/VM, device hashing,
  tiers, behaviour layer, finalization, `script=2` opt-out, the live enable gate) and
  the pre-family schema are **permanent properties, not gaps** — never "close" them.
- `an` / `cid` / `sid` are **never renamed or prefixed** (`00_INDEX.md` §6 rule 1).
- `device_sig` (identity) and `device_hash` (fraud) are **never unified**.
- Detection stays **shadow-only** — no upgrade flips `BEHAVIOR_ENFORCE` /
  `DELAYED_CONVERSION` / `DEFER_CONVERT`.

If any requested change would touch identity, **stop and say so** — out of scope by
definition.

---

## 1. How this skill runs — the operating loop

This is the heart. The skill does **not** dump a plan and apply it. It runs the loop
below, and the loop is the reason a legacy port lands correctly instead of shipping
half-modernized with silent regressions.

### 1.1 The target is a written artifact, not the user's memory

A modernization only goes right when someone holds **"what the finished build looks
like."** That picture must be **written down** (PHASE T) so the skill carries it — the
closable gaps to close, the sibling behaviors to match, and the **must-not-break
invariants**. Everything downstream is measured against that artifact.

### 1.2 The three-stage macro-arc: frontend → backend → merge

Real modernization is not one flat backlog. It runs in three stages, in order:

1. **Frontend stage** — bring the extension to target.
2. **Backend stage** — audit/align the server to the frontend's contract.
3. **The merge stage** — verify the two halves **agree end-to-end**. The agent does the
   offline seam analysis; the **owner** runs the live end-to-end verification (§9). This is
   where front↔back contract mismatches surface (the whitelist-collapse that suppressed the
   search-page pre-hide lived *here* — it was invisible in either half alone). §9.

Narrow scope collapses the arc to just the relevant stage(s); a full modernization
runs all three.

### 1.3 Inside each stage: the micro-loop, gated per phase

Each stage runs: **assess (PHASE A) → plan & sequence (PHASE P) → execute + validate
(PHASE E), one feature-batch at a time.** Work is **batched by feature** ("some at
once") — a single feature that spans several files (e.g. a zero-flicker fix touching
the content script, the worker, and the manifest) is one phase; unrelated concerns
are separate phases.

> **The validation gate is mandatory and per phase — and the agent validates only what it
> can prove offline.** After each feature-batch: *implement → `[AGENT — offline]` validate
> (a `node`/logic simulation, a static/`grep` audit, a rule/priority matrix, a code review,
> or a console log / screenshot **the owner pastes in**) → run the offline monetization
> wiring check (§1.4) → show the owner the proof and the diff → hand over an **optional
> `[OWNER — manual]` targeted checklist** ("if you want to load-and-verify this one now:
> load X, do Y, expect Z") → **WAIT for the owner's go** → only then start the next phase.*
> A phase is never "done" because the code was written; it is done when it is **proven
> offline** and the owner has said continue.
>
> **The agent never loads the extension, never opens a browser, never hits the live server**
> — loading unpacked, navigating, visual testing, uploading, DB/store steps are all
> `[OWNER — manual]` (`00_INDEX.md` §7). The owner decides whether to unpack-and-test a phase
> now, defer it, or batch all in-browser testing to the end (§11) — that call is theirs.
>
> **Phases are revisable.** A validated phase is not frozen: if a later phase reveals that an
> earlier one was off, going back to revise it is a normal move, not a failure — say so, and
> re-validate the touched phase.

### 1.4 Two invariants that run through every phase

- **Copy the siblings' *behavior and logic* exactly; diverge only *structure, names,
  and file tree*.** The proven builds are the reference for *what the code does* — do
  not reinvent the algorithm and reintroduce its solved bugs. The anti-template rule
  (`00_INDEX.md` §1) forbids copying their *file tree, names, and structure*, not
  their logic. Same algorithm, different spelling. (This resolves the tension that
  otherwise recurs all engagement: "match the siblings" vs "don't clone" — both are
  true, on different axes.)
- **Monetization must not break (the agent's job here is protective and offline-only).** The
  `$script` injection, the search→partner redirect, the pre-hide mask, and the allow band are
  a standing invariant. Every phase, the agent runs a **static wiring assertion** — confirming
  this batch did not remove or disturb that path (the injection rule IDs still present in the
  static rulesets, the zero-flicker directive still targeting the search selectors, no
  allowlist edit re-adding the search hosts, the redirect/inject code path untouched). That is
  a `grep`/read check, not a browser test. **The agent never live-tests monetization — the
  owner does, and monetization work is sequenced as the *last* phase** (§7). The agent stays on
  the ad-block engine; the injection/monetization layer is the owner's.

### 1.5 Keep the user oriented

From time to time — and at every phase gate — emit a compact **"what was asked / what
was done"** summary, and append a one-line entry to the build's `progress.txt`. The
user should never have to reconstruct where things stand.

### 1.6 Close the loop back into the docs

When a fix is required that the specs did **not** already describe (an emergent defect
or a new gotcha), the run's final step **writes that lesson back into the
Architecture docs** (PHASE H) — so the next legacy port (15/17/20/24) inherits it
instead of re-learning it by bleeding.

**Phase map:** §4 GATE-0 identity safety · §5 PHASE T target · §6 PHASE A assess ·
§7 PHASE P plan (approval stop) · §8 PHASE E execute+validate · §9 the merge stage ·
§10 PHASE H harden + write-back · §11 acceptance/deploy · §12 stop conditions.

---

## 2. Inputs

1. **`<project-folder>`** — the project directory, e.g. `12 - Ad Block Pro` or just
   `12`. Convention: `<project>/extension/` + `<project>/backend/`. If a build's
   layout deviates, stop and confirm the two folders.
2. **`[scope]`** *(optional)* — default: **"the full modernization toward the Target
   State (§5)."** A narrower scope is a plain-language list (`zero-flicker,
   priorities`) and collapses the macro-arc (§1.2) to the relevant stage(s).

**You never pass — the skill DETECTS and VERIFIES (§4):** the identity variant/
generation, the backend generation, the handle/key prefix, the DB table name. Typing
them and being wrong is how you orphan a user base; if you assert a variant it must
agree with detection or the skill hard-stops.

---

## 3. Build map (for locating the row and sanity-checking detection)

Every specification ends in a **per-extension table** whose columns are these builds.
Map the folder to a row by its **leading number or codename**.

| Project folder | Codename | Variant | Generation | Sync endpoint | Handle key | DB table |
| :-- | :-- | :-- | :-- | :-- | :-- | :-- |
| `12 - Ad Block Pro` | Pro | **Gen 0 Legacy** | Gen 0 (pre-family) | `adbpro-202609.php` | `syncuid` (AES) | `adbprocup11` |
| `21 - Ad Block North` | North | **V1** | Gen 1 | `adbnorth_18.php` | `backdetail` | `adbnorth21` |
| `22 - Ad Block Hunter` | Hunter | **V1** | Gen 1 | `adshunter_2.php` | `hunterinfo` | `adshunter22` |
| `23 - Wonder Blocker` | Wonder | **V1** | Gen 1 | `wonder58.php` | `wonderinformation` | `c23wonderblock` |
| `25 - Ninja Block` | Ninja | **V2-A** | Gen 2 | `ninja-adb21.php` | `ninja_fp` | `ninja25` |
| `26 - Ad Block Ghost` | Ghost | **V2-B** | Gen 3 | `ext-server.php` | `ghost_fp` | `ghost26` |
| `27 - New Extension` | 27 | **V2-B** (new build — not an update; see note) | Gen 3 | *(empty)* | `ab_fp` | *(to create)* |

This table is the **expectation**, not the authority — detection from the actual code
(§4) is. If they disagree, the code wins and you **stop**. `12` is a documented
**Gen 0 (Legacy)** build; the remaining pre-family builds (`15`, `17`, `20`, `24`,
older named folders) are **not yet documented** — when one is picked up, run the
**intake procedure (§4.5)** first, then modernize.

> **This skill is for EXISTING builds only.** A build whose row shows a table `*(to create)*`
> and an `*(empty)*` backend — `27` today, `28`/`29`/… tomorrow — has no backend or table yet;
> it is **create-extension territory, not an update**. Stop and point to the new-build procedure
> (`00_INDEX.md` §3–§4); never provision a table or copy a backend from within this skill.
> **On a new build, always ask which backend to copy from — never assume Ninja/Ghost or any
> specific build** (§0). *(Creating a new build's whole backend is a wholesale copy; an update
> only **augments** an existing build's own backend — §8, `BACKEND_ARCHITECTURE.md` §2.)*

---

## 4. GATE 0 — Detect & verify identity (the safety gate)

Do this **before PHASE T and before touching anything.** Read the signals from the
build's own code; require **multiple independent signals to agree**; cross-check
against §3. Any disagreement is a **hard stop**.

**Schema signal sources**, strongest first (never rely on schema alone):
1. A local snapshot at `<project>/backend/docs/db_schema/<table>.sql` (§4.4) — check
   its `-- Generation Time:` header for staleness; identity-defining structure is
   immutable and trustworthy at any age, feature columns drift.
2. `DB_SCHEMA.md` §9 canonical DDL for the detected variant + the §9.5 deviations.
3. Code inference — the identity-key type in `config.php`, presence/absence of the
   V2-only backend files, the `INSERT`/`UPDATE` column lists, `SHOW COLUMNS` probes,
   the resolution-cascade shape.

**Step 0 — family member, documented `Gen 0`, or undocumented pre-family?**
- **Family (`21/22/23/25/26/27`)** → normal path.
- **`12 - Ad Block Pro`** → the **`Gen 0` legacy-modernization track**: it has a table
  row, so run the same loop, reading its cells with the Gen 0 rule — *identity/DB/
  backend cells are frozen and never gaps; frontend/rules/cosmetics/zero-flicker/sync
  cells are closable*. **Do not hard-stop.**
- **An undocumented pre-family build (`15/17/20/24`, older folders)** → no table row
  yet. **Stop**, say which build it is, and offer to add its `Gen 0` row to every
  table first (12 is the template). Do not silently treat it as V1.

> **Recognizing a `Gen 0` build (the trap).** A pre-family build passes *every* V1
> test in Step A — no hardware telemetry, no V2 backend files, no `fingerprint`
> column, an AES-keyed handle — and would be misclassified as V1. It is **`Gen 0`, a
> distinct legacy lineage**. Verify against the canonical V1 DDL (`DB_SCHEMA.md` §9.1):
> a true V1 table carries `visit`, `whitelistedDom`, `blockedDom`, `instdom`,
> `organisation`, `hostname`, `idx_cid_sid` + `idx_ip_an_instdom`, `conv '0'..'3'`,
> charset `utf8mb3_bin`. A table that instead carries ad-counter columns
> (`blockedads`/`bingads`/`yahoo`/`popads`/`totblock`/`googleflag`), lacks the
> user-data group, and uses `conv '0'..'2'` with `idx_*_ip/cid_script_updates` is
> **`Gen 0`** (this is exactly `12`, identity `syncuid`/AES) — not V1. Its identity/DB
> rows are permanent; its **frontend** is modernized to baseline.

**Step A — V1/Gen 0 or V2?** V1/Gen 0 if *all* hold: no hardware telemetry collected;
none of the V2-only backend files (`finalize_pending_installs.php`, `behavior_lib.php`,
`classifier_v4.php`/`classifier_lib.php`, `generate_fraud_report.php`,
`generate_funnel_cache.php`); no `fingerprint` column / hardware columns; handle
produced by an AES key with no plain `script` key on the wire.

**Step B — if V2, V2-A or V2-B?** `idx_fingerprint` (A) vs `uq_fingerprint` (B); HMAC
handle (A) vs minted `bin2hex(random_bytes(32))` (B); `device_sig` absent (A) vs
present (B); telemetry backfill rewrites `fingerprint` (A) vs leaves it alone (B).

**Step C — cross-check & gate.** Compare detected `(variant, generation)` + sync
endpoint + handle key against §3. All consistent → proceed to PHASE T. Any mismatch or
self-contradiction → **STOP**, report which signals disagree.

> Watch-outs: `-dev.php` twins (`adbpro-202609-dev.php`, `ninja-adb21-dev.php`,
> `wonder58-dev.php`) carry a "DEV PURPOSE ONLY" block that force-sets
> `script=1`/`level=0` — verify against the **prod** file. **12's *prod* file also
> currently dev-forces `script`** — note it, don't trust the enablement as organic.

### 4.4 Schema snapshot convention

One structure-only `.sql` per table at `<project>/backend/docs/db_schema/<table>.sql`
(phpMyAdmin → Export → Structure only). Read the directory, not one file; the **user
table** is the one carrying `an`/`cid`/`sid`, `script`, `conv`, `level`. Identity
structure is authoritative at any age; feature columns are a dated observation. A
build with no snapshot is not blocked — fall back to sources 2–3 and say so; producing
a snapshot is a cheap, safe follow-up for the deployment checklist.

**Never assume the DB's contents.** If any backend/schema work is in scope, **ask the owner
whether the live DB is ready** and work **only** from the actual `docs/db_schema` snapshot (or
a DDL the owner provides) — never guess that a column, index, or table exists. A wrong schema
assumption is how an update writes against a column that isn't there.

### 4.5 Intake — onboarding an undocumented build

When Step 0 lands on an **undocumented pre-family build** (`15/17/20/24`, older folders), do
**not** modernize yet — run the intake first, then **stop for approval**:

1. **Discover its identity from its own code** (the §4 signals): the handle/key type, the
   schema (from the `docs/db_schema` snapshot or a DDL the owner provides — never assumed), the
   resolution/dedup shape, and whether any V2 backend files exist. Apply the Gen 0 trap check
   (the §4 recognizer).
2. **Draft its `Gen 0` rows** for every per-extension table — `EXTENSION_COMPOSITION.md` §25,
   `RULES_AND_PRIORITIES.md` §8, `USER_CREATION_AND_UPDATE.md` §13, `FRAUD_DETECTION_V5.md` §18,
   `BACKEND_ARCHITECTURE.md` §9, `DB_SCHEMA.md` §7, `DATA_SOURCES.md` §5 — using 12's rows as the
   template, grounded in the code (never guessed).
3. **Draft its `TARGET_STATE.md`** from the baseline + those rows (PHASE T), reading the frozen
   identity/DB/backend cells with the Gen 0 rule and the reachable-set filter (`00_GOAL.md` §3).
4. **← GATE (mandatory): present the drafted rows + Target State and WAIT.** The agent performs
   the discovery and drafting only; it **never commits the rows or starts PHASE A until the
   owner confirms or edits them.**

---

## 5. PHASE T — Establish the Target State (the written picture)

**Before assessing gaps, write down what "done" is.** This is the artifact that
replaces the picture the driver would otherwise carry in their head.

1. **Draft it from what already exists** (do not require the user to write it from
   scratch): the baseline (`EXTENSION_COMPOSITION.md`), the build's per-extension row
   in every doc, the sibling behaviors it must match, plus any project memory / audit
   notes / `progress.txt` history the folder already holds.
2. **Write it to `<project>/TARGET_STATE.md`** (template in this skill's folder). It
   captures, for THIS build: the identity/generation (frozen), the **closable target**
   per area (what "modernized" means here), the **sibling behaviors to match**, and
   the **must-not-break invariants** (the exact monetization path — inject, redirect,
   mask, allow band — and anything else that is load-bearing). **Seed the definition-of-
   done from the FULL conformance surface** (`EXTENSION_COMPOSITION.md` §26 + every
   per-doc checklist), one row per baseline area — never a hand-picked subset. An area
   that didn't matter for the last build may be required for this one (e.g. the
   `recordVisit` timestamp, the acceptable-ads opt-out, a `[TOGGLEABLE]` feature); mark
   it `n/a` with a reason rather than omitting it, so nothing is silently dropped.
3. **← GATE (mandatory):** show the drafted Target State; the user **confirms or
   edits** it. No assessment or code until the target is agreed. A wrong target makes
   every downstream phase wrong.

The Target State is the yardstick for PHASE A and the acceptance test at the end.

---

## 6. PHASE A — Assess current vs. the Target (from the real code)

For the build's row, walk **each specification's per-extension table** and, for every
non-`✅`/`❓` cell, apply the **reachable-set filter** (`00_GOAL.md` §3, decision B) against
the actual code. Classify the area:
- **Universal** (touches no DB column — the ad-block engine, zero-flicker, monetization path,
  local-first/sync, hygiene): always closable → converge to baseline.
- **Substrate-shaped** (reads/writes a DB column, or leans on the identity generation):
  closable **only if this build's frozen schema already carries the column(s)**. The concrete
  test is *does the column exist?* — **No ⇒ permanent `n/a` (stop); Yes ⇒ closable**, reshaped
  to this build's own columns.
- **Already permanent** by lineage (the identity mechanism itself).

Ground every finding in the files — never assert a gap from the table alone, and **never
propose adding a column to reach a behavior** (growing the schema is birth-time only,
`00_GOAL.md` §5).

**Gap surfaces (read the row for this build in each):** `EXTENSION_COMPOSITION.md` §25
(→ §26 checklist), `RULES_AND_PRIORITIES.md` §8 (→ §9), `USER_CREATION_AND_UPDATE.md`
§13/§11 (→ §12), `FRAUD_DETECTION_V5.md` §18 (→ §19), `BACKEND_ARCHITECTURE.md` §9
(→ §10), `DB_SCHEMA.md` §7 (→ §8), `DATA_SOURCES.md` §5 (→ §6).

**Markers:** `✅` conformant (skip) · `⚠️` closable (a footnote names the fix) · `❌`
closable **only if the reachable-set filter allows it** — universal, or substrate-shaped with
the column already present · `—`/`n/a` permanent by design (never close) · `❓` not yet audited
(verify before trusting) · `⬜` to-copy (27 only).

**Permanent — never close:** any **substrate-shaped area whose column this build's schema
lacks** (decision B); on `Gen 0`/V1 builds, the whole fraud stack, the
`fingerprint`/`device_sig` columns and pre-family schema; on Gen 0 also the entire family
user-data group (`visit`/`whitelistedDom`/`blockedDom`/`instdom`/`cosmeticDom`). On V2-A (Ninja), `device_sig`.
Recorded operational states that look like gaps (dormant monetization, a Variant-B
popup, an absent `[OPTIONAL]` picker) — confirm intent first.

Produce the **delta**: for each closable gap, the doc `§` that specifies the fix, the
side it touches (frontend / backend / merge-seam / data / schema-for-deploy), and the
concrete change. Filter against `[scope]`.

> The per-doc audit MAY be fanned out — one reader per spec against the real code —
> then merged. Keep GATE 0 and PHASE E in the main flow.

**← GATE:** present the delta; the user confirms it is right (and that nothing real is
missing) before planning.

---

## 7. PHASE P — Plan & sequence, then STOP for approval

Order the delta into **feature-batched phases**, along the macro-arc **frontend →
backend → merge** (§1.2): frontend phases first, then backend phases, then the merge
stage (§9). Within a stage, sequence by dependency and batch related files into one
phase. **Sequence any monetization-touching work as the *last* phase** (§1.4) — the
ad-block-engine phases come first, monetization last, and its live check is the owner's.
Mark each phase's `[AGENT — offline]` validation method (how the agent will prove it) and
any `[OWNER — manual]` check the owner may run at that gate.

**Report and WAIT for explicit go-ahead** before any code. Template:

```
# update-extension — <project-folder>

## Detected identity (VERIFIED)   Variant/Gen · evidence · cross-check vs map
## Target State   (link to TARGET_STATE.md — confirmed in PHASE T)
## Delta (closable gaps)   table: # · gap · stage(FE/BE/merge) · doc § · change · how-validated
## Permanent — will NOT touch (and why)
## Proposed phase sequence   ordered, feature-batched, FE → BE → merge
## Deployment steps this will produce (for you to run afterward)
```

This is the plan gate. The **per-phase** validation gates (§8) come after.

---

## 8. PHASE E — Execute + validate, one feature-batch at a time

For **each** phase in the approved sequence, in order:

1. **Implement** the feature-batch (all its files together), respecting the front↔back
   asymmetry:
   - **Frontend — reimplement, don't clone; copy behavior, not code (§1.4).** Close the
     gap against the **behavior** the spec + siblings describe, in this build's own
     structure and naming. Any new user setting joins the **purge preserved set** in
     the same change (mirror ⊆ preserved). Keep write ordering: mirror-heal before
     payload assembly; data-version stamp **last**.
   - **Backend — copy, don't reinvent.** Where a doc cites the canonical file by
     `file:line`, that is the code to copy; remap keys at the **wire layer only**
     (never DB columns); preserve the rule-ID partitioning, the priority ladder, the
     degraded HTTP-200 path, single-flight generation, `conv` frozen, the idempotent
     `enabled` / `disabled` stamps. **No schema change in an update** — the schema is
     frozen at birth (`00_GOAL.md` §3, decision B); adding a column to reach a behavior is
     birth-time / create-extension only (§5). A behavior needing an absent column is `n/a`.
2. **`[AGENT — offline]` validate** — prove it with a method the agent can run without a
   browser: a `node`/logic simulation, a rule/priority matrix, a static/`grep` audit, a
   code review, or a console log / screenshot **the owner pastes in**. Assertion is not
   validation. The in-browser smoke test (load unpacked → navigate → observe) is
   **`[OWNER — manual]`** — the agent lists it for the owner, never runs it.
3. **Monetization wiring check (§1.4):** every phase, statically assert this batch did not
   remove or disturb the inject/redirect/mask/allow-band path (a `grep`/read check). The
   *live* money-path test is the owner's, at the final monetization phase — never the agent's.
4. **Show the owner the proof and the diff, plus an optional `[OWNER — manual]` targeted
   checklist for this phase; ← GATE: WAIT for their go.** The owner may review-only, unpack
   and test this phase now, or defer all in-browser testing to §11 — their call. If a later
   phase shows this one was off, revising it is a normal backtrack (§1.3). Only after the
   owner says continue does the next phase start.
5. **Summarize** — append "what was asked / what was done" and a `progress.txt` line.

### 8.1 Legacy-port failure catalog — run each as a driven check

These are the defects that recur on a legacy port; each is *check → fix → verify*, not
"remember to":

- **Zero-flicker Layer 1 reads `chrome.storage.local` directly at `document_start`**,
  not `sendMessage` to the worker (the round-trip flashes on cold start) — load the
  pure engine modules into the content-script context so it builds CSS locally
  (`EXTENSION_COMPOSITION.md` §11.1).
- **Zero-flicker is NOT gated on the injection allowlist** — it runs on the protected
  search surfaces (kept out of every allowlist by the carve-out) and is suppressed
  only by the user's own pause/whitelist. If it's suppressed on a search page, inspect
  the allowlist contents — and check the backend isn't **re-adding** the search hosts
  after the carve-out (`EXTENSION_COMPOSITION.md` §11.2; `RULES_AND_PRIORITIES.md` §5.4).
- **User-tier priority is `~200/100`, above the blocks and BELOW the injection band** —
  never a huge number like `2000000`, which inverts the invariant
  (`RULES_AND_PRIORITIES.md` §4.2).
- **Static rulesets carry no block/redirect on the injection surfaces** (search TLDs,
  partner-feed hosts, injected analytics/quality hosts) — a stale packaged rule fights
  the injection in the pre-first-sync window (`RULES_AND_PRIORITIES.md` §9.1).
- **Comments/symbols name no sibling build and describe no monetization/`script`
  scheme** — a legacy port carries these in verbatim; scrub them (`EXTENSION_COMPOSITION.md`
  §0.2).
- **Drop unused permissions** the modernization exposes (e.g. `alarms` once sync is
  navigation-driven — §19).

---

## 9. The MERGE stage — verify the two halves agree end-to-end

After the frontend and backend stages are each at target, run the merge stage — the seams
where the two halves meet. Front↔back mismatches are invisible in either half alone (the
whitelist-collapse is the canonical example). The agent does the `[AGENT — offline]` static
seam analysis; the `[OWNER — manual]` live confirmation is handed over as a checklist. Cover,
at minimum:

- **Zero-flicker directive match** — `[AGENT — offline]` read both sides and confirm the
  directive the backend *sends* matches what the frontend *builds* and *targets* (host
  patterns, selectors, timing). `[OWNER — manual]` confirm the mask actually fires on the
  live search page.
- **Allowlist does not collapse the flicker** — `[AGENT — offline]` the protected-domain
  carve-out holds and nothing re-adds the search hosts (§8.1).
- **Feed health** — `[AGENT — offline]` confirm the client's feed URLs and version-gating are
  wired correctly. `[OWNER — manual]` fetch each feed URL and confirm it returns real data,
  not empty/HTML/404, and the cache self-generates on first request (`DATA_SOURCES.md` §6) —
  the client fails safe, so a broken feed is invisible from the extension.
- **`extid` / attribution params round-trip** — `[AGENT — offline]` trace the params through
  both sides' code.
- **The priority ladder aligns across tiers** — `[AGENT — offline]` static blocks < server
  band < the injection band; user tier above blocks and below injection (§4.2).
- **The full live flow** — `[OWNER — manual]` search → inject/redirect → mask → sync → DNR
  applied — works as one system, monetization intact. This is the owner's final money-path
  test (§1.4).

**← GATE:** the agent presents its offline seam analysis and the owner's live checklist; the
**owner** validates the end-to-end flow before the run closes.

---

## 10. PHASE H — Harden + write the lessons back

1. **Cross-cutting hardening pass** (if not already covered phase-by-phase): the §8.1
   catalog swept once more across the whole build.
2. **Write-back:** any fix that was needed but **not** already in the specs — an
   emergent defect, a new gotcha, a per-build cell that changed — is **written into the
   Architecture docs** (the relevant spec section + the build's table cell + a
   `DETECTION_CHANGELOG.md` entry). The Target State is updated to reflect the new
   reality. This is what makes the next legacy port cheaper.

---

## 11. Acceptance & deployment

After applying, run each affected document's **conformance checklist** — the **Common
(universal) items for every build**, then the **capability items the build's generation
actually carries** (`00_GOAL.md` §3: run a `[V2 only]` / substrate-shaped item only where the
build's frozen schema holds the column — never against a Gen 0 or V1 build that lacks it).
**Monetization is a Common item on every generation, Gen 0 included** — never skipped or read
as "off" for being Gen 0/V1; only the *enable mechanism* differs by generation (`00_GOAL.md`
§3 note). The agent runs the `[AGENT — offline]` checklist items
(static/`grep`/simulation) and produces the `[OWNER — manual]` list for the rest. The
**runtime smoke-test block** (`EXTENSION_COMPOSITION.md` §26) — a real load-and-navigate
pass — is `[OWNER — manual]`: the agent hands the owner the exact steps and expected
results; the owner runs it and pastes back what they saw. Record the `[VARIANT]`/legacy
Record block from the results.

**Outputs:**
1. **Full report** → `<project>/history/update-<YYYY-MM-DD>.md` (the §7 report + the
   applied changes, per-phase validation results, and the merge-stage results).
2. **One line** appended to `<project>/progress.txt`.
3. **Updated `<project>/TARGET_STATE.md`** (§10) and any doc write-backs.
4. **Deployment checklist** — the ordered `[OWNER — manual]` live steps you run yourself
   (files to upload and their pairs, backups, post-deploy verifications incl. `php -l`,
   feed-URL health, and a live request returning 200). **No schema migration** — an update
   never changes the frozen schema (`00_GOAL.md` §3); table/column creation is birth-time /
   create-extension only.
   This skill stops at local folder edits; it does **not** touch the live server or DB.
   If the build had no schema snapshot, the checklist ends with: produce one.
5. **Store hand-off `[OWNER — manual]`** — if the extension package changed, the final step by
   which it reaches users is yours: repackage the `extension/` folder and submit/update it on
   the Chrome Web Store. The agent prepares the package and lists what changed; it never
   uploads or submits.

---

## 12. Stop conditions (hard stops — report and wait)

- An **undocumented pre-family build** with no table row yet (§3, §4 Step 0) — stop,
  name it, offer to add its `Gen 0` row first. (`12` is documented and does **not**
  stop here.)
- Identity detection is ambiguous or disagrees with the build map (§4 Step C).
- The requested change would touch the identity/uniqueness mechanism (§0) — including
  any attempt to migrate a `Gen 0`/V1 build toward a higher generation.
- The scope implies a generation migration (Gen 0→V1, V1→V2) — that's a *new build*;
  point to the **new-build skills**: `create-extension-frontend` (reimplement the frontend fresh)
  + `create-extension-backend` (copy a de-injected template + remap keys), verified by
  `extension-audit` (`00_INDEX.md` §3–§4 for the underlying procedure).
- **The Target State cannot be agreed** (PHASE T) — do not assess or code against an
  unconfirmed target.
- A **phase fails validation and the fix is unclear** — stop at that gate, report what
  failed and the hypotheses; do not advance past a broken phase.
- The project layout doesn't match `<project>/extension/` + `<project>/backend/`.
- A `-dev`/legacy twin can't be confirmed traffic-dead when it writes the same table.
