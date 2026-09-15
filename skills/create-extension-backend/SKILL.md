---
name: create-extension-backend
description: >-
  Stand up a brand-new build's BACKEND the way it is actually done: by copying the
  canonical backend set wholesale from a designated source/template and remapping only
  the wire keys and the per-build variable set — never reimplementing it. Copy the
  whole PHP set, rename exactly the two endpoint files, remap the frontend-facing keys,
  set the per-build variables (DB, table name, freshly-generated identity key, endpoint
  filenames, host/asset URLs, market list, tokens), and change nothing else. Keeps every
  function name, the structure, and the logic byte-identical — sameness is safe because
  the store never sees the server. The agent copies and validates OFFLINE only (php -l,
  grep audits, code review); DB provisioning, hosting, upload, and every live request are
  yours, handed over as exact checklists it waits on. Sibling of create-extension-frontend.
  Slash-only; run it as /create-extension-backend <project-folder> [scope].
argument-hint: <project-folder> [scope]
disable-model-invocation: true
---

# create-extension-backend

Stand up a **new build's backend from A to Z** — **the way it is actually done**: the
server is **copied wholesale from a previous build and only its keys + a short variable
set are remapped** (`00_INDEX.md` §1). This is the opposite discipline from the frontend
(which is reimplemented fresh): **the backend is never seen by store review, so sameness
is safe and desirable — reimplementing it would only add bugs.** This skill is the
backend counterpart to `create-extension-frontend` (its sibling); together they build a
whole new build.

**Architecture folder (source of truth, absolute path):**
`/Users/megane/Desktop/Dev/00 - Architecture/`
A `§` reference below is a section either in one of that folder's documents (when a
document is named, e.g. `BACKEND_ARCHITECTURE.md` §2.3) or in **this skill** (a bare
`§N`); the context makes clear which. If that folder ever moves, update this one path.

The backend-copy spec this skill executes is **`BACKEND_ARCHITECTURE.md` §2** (the file
set, the two renames, the per-build variables) with the **canonical key mapping of
`00_INDEX.md` §6**. The DDL to provision the table is **`DB_SCHEMA.md` §9**.

---

## 0. Scope & the rules that outrank everything

### 0.1 What this skill does — copy wholesale, remap keys, change nothing else

> **Copy the whole set. Rename exactly two files (§2.1). Remap the frontend-facing keys.
> Set the per-build variables (§2.3). Everything else is byte-identical.** Keep every
> function name, the module structure, and the logic exactly as the source has them.
> **Do not refactor, remove dead code, "clean up," or restructure** — the source is a
> proven, working server; sameness is the point (`00_INDEX.md` §1). The only edits are
> (a) the wire-key spellings and (b) the §2.3 per-build variable set.

This is the **create-a-new-build** operation — a fresh wholesale copy of the entire set
(`BACKEND_ARCHITECTURE.md` §2, intro). It is not an update: an update *augments* an
existing backend; this *creates* one from the copy source.

### 0.2 Always ask which source to copy — never assume

**Always ask which build's backend (or which template) is the copy source — never assume
Ninja / Ghost or any specific one** (`BACKEND_ARCHITECTURE.md` §2 intro). The source
fixes the build's **identity variant and generation**, which are **birth-time and never
migrated** (`00_GOAL.md` §5, `BACKEND_ARCHITECTURE.md` §8.3): copy a V2-B source → a V2-B
build. Confirm the source before touching anything.

*(The current designated source is the de-injected **sync-endpoint** template
`26 - Ad Block Ghost/backend/docs/templates/backend-template.php`; the rest of the copy set
— `config.php`, `ext-uninstall.php`, the cache generators, and the V2 fraud files (§3) — is
taken from `26 - Ad Block Ghost/backend/` directly. A V2-B set. Confirm this is the intended
source at the start of every run; it can change.)*

### 0.3 The template is copied as-is — no editing its logic

The designated template is the working reference and ships **as-is**. This skill copies it
verbatim and changes only the keys + variables (§0.1). It does **not** add, remove, or
alter any behavior in it — including any inert/legacy code the owner has chosen to keep.
Editing the template's logic is **out of scope for this skill** (that is a separate,
owner-directed change, not part of standing up a copy).

### 0.4 The agent copies offline; the owner provisions the DB and deploys

The **agent validates only offline** — `php -l` lint, `grep`/static audits that the remap
is complete and consistent, and code review. **Provisioning the database, hosting/upload,
TLS/permissions, and every live request are `[OWNER — manual]`** (`00_INDEX.md` §7). The
agent **never touches the DB or the live server** — it finishes the offline copy+remap,
hands the owner an exact provisioning + deploy checklist, and **waits**. (This mirrors the
owner's own standing rule that the DB is not the agent's to touch.)

### 0.5 Compliance the owner owns (noted, not gated by the agent)

The backend collects and stores user data (identity/attribution, and on a V2 source the
hardware-signal/anti-fraud set). Ensuring that collection is **disclosed in the build's
privacy policy at store submission** is an `[OWNER — manual]` compliance step. The agent
notes it on the acceptance checklist; it does not author the policy or gate on it.

---

## 1. How this skill runs — the operating loop

The skill does **not** dump a plan and apply it. It runs the loop below; the loop is why
a wholesale copy lands as a *working* backend instead of one with a half-finished remap
and silent runtime failures (a missing dependency here is a silent failure —
`BACKEND_ARCHITECTURE.md` §2).

### 1.1 The target is a written artifact: the key map + the variable set

Before copying, write down the two things that change: **the wire-key remap table**
(every frontend-facing key → its role → the source's spelling) and **the per-build
variable set** (§2.3). That written map (PHASE M, §4) is what the copy is validated
against — a key present in the source but missing from the map is how a remap goes
half-done.

### 1.2 Assess → map (STOP) → execute + validate, per file-batch

The build runs: **inventory the copy set (PHASE A, §3) → build & confirm the key map +
variable set (PHASE M, §4, approval stop) → execute the copy + remap (PHASE E, §6), one
file-batch at a time.**

> **The validation gate is mandatory and per batch — offline only.** After each file-batch:
> *copy verbatim → remap the wire keys → set the §2.3 variables → `[AGENT — offline]`
> validate (`php -l`; a `grep` audit that every mapped key was remapped consistently, that
> `an`/`cid`/`sid` are untouched, that DB **column** names are unchanged, that no frontend
> key spelling leaked into a DB column, and that the two endpoint filenames + host/asset
> URLs are updated) → show the owner the diff → hand over the `[OWNER — manual]` steps for
> this batch → **WAIT for the owner's go** → next batch.* A batch is done when it lints,
> the remap audit is clean, and the owner has said continue.
>
> **The agent never provisions the DB, never uploads, never hits the live server** — those
> are `[OWNER — manual]` (§0.4).

### 1.3 Keep the user oriented

At every gate, emit a compact **"what was asked / what was done"** summary and append a
one-line entry to the build's `progress.txt`.

**Phase map:** §3 PHASE A inventory · §4 PHASE M key map + variables (approval stop) ·
§6 PHASE E execute + validate · §7 acceptance + owner deploy · §8 stop conditions.

---

## 2. Inputs

1. **`<project-folder>`** — the new build's directory, e.g. `28 - <Name>` or just `28`.
   Convention: the backend is written under `<project>/backend/`. If the folder doesn't
   exist, confirm the name and create `<project>/backend/`.
2. **Copy source `[ask]`** — which build/template to copy from (§0.2). Never assumed.
3. **`[scope]`** *(optional)* — default: **"the full backend copy toward a working,
   remapped set."** A narrower scope copies only the named files against the same key map.

**Owner facts to gather at the start (ask — never assume):**
- The **copy source** (§0.2) and the new build's **product name + key prefix**.
- The **per-build variable values** (§2.3): DB name/user/password, table name, a **freshly
  generated identity key** (never shared across builds), proxycheck/payout-provider tokens,
  product number, approved-market list, store ext-id prefix, backend host (`BG_HOST`), asset
  base URL, and the two endpoint filenames.
- Whether the target **DB is provisioned** (`[OWNER — manual]`) — the agent works only from
  the DDL/snapshot; it never assumes a table/column exists and never provisions it.

---

## 3. PHASE A — Inventory the copy set (`BACKEND_ARCHITECTURE.md` §2)

Copy the whole set; know exactly what carries a new name, what stays verbatim, and what is
**not** copied.

**Renamed per build — exactly two (§2.1):**
- **Sync endpoint** (the heartbeat: identity, risk/anti-fraud, response assembly) — the
  template's `ext-server*.php` → `<build-sync>.php`.
- **Uninstall endpoint** (stamps churn `disabled`, redirects to the feedback form, never
  fatal) — `ext-uninstall.php` → `<build-uninstall>.php`, its redirect **re-pointed to THIS
  build's** feedback destination.

The client hard-codes both names (the sync URL it POSTs to; the uninstall URL it registers),
so these two filenames **must agree with the frontend** (§2.1).

**Copied verbatim — same filename every build (§2.2):** `config.php`; `compiled_rules.php`;
`generate_compiled_rules.php`; `generate_cosmetic_rules.php`; `generate_scriptlet_details.php`;
`short.php`; `long.php`; and, on a **V2** source, the fraud/behaviour set
(`finalize_pending_installs.php`, `behavior_lib.php`, `classifier_lib.php`, `classifier_v4.php`,
`generate_fraud_report.php`, `generate_funnel_cache.php`, `dashboard_detection.php`, and the
diagnostics `dry_run_classifier.php` / `validate_behavior.php` / `validate_outcomes.php` /
`_index_diagnostics.php` / `_test_identity.php`). *(A V1 source omits the fraud files entirely.)*

**Do NOT copy (§2.5):** the `-dev`/legacy twins (they force `script=1`/`level=0`) and the
one-off rescue/migration/host-specific scripts.

**Do NOT copy — regenerated at runtime (§2.4):** `compiled_rules_cache.json`, the cosmetic
caches, `allowlist_cache_<market>.json`, the `*_version.txt` stamps, `fraud_cache.json`,
`fraud_report.json`, and the finalize/run stamps. A fresh build starts empty and the
generators rebuild them.

---

## 4. PHASE M — Build the key map + the per-build variables, then STOP for approval

**Write the two change-sets, and only these change.**

**(a) The wire-key remap table** — every frontend-facing key the backend reads/writes, its
**role**, the source's spelling, and this build's new spelling. Seed it from
`00_INDEX.md` §6 (the canonical role table) against the actual template. The rules that
govern it (§5) are absolute: `an`/`cid`/`sid` never change; DB **column** names never
change; only the wire spellings do.

**(b) The per-build variable set (`BACKEND_ARCHITECTURE.md` §2.3 — "change these, nothing
else"):**
- **`config.php`:** DB name/user/password; **table name**; the **identity key** (V1 AES /
  V2-A HMAC — **freshly generated per build, never shared**; V2-B needs none, its handle is a
  random token); proxycheck key; payout-provider token; product number; approved-market list;
  store ext-id prefix; backend host (`BG_HOST`); asset base URL. *(A de-injected template's
  search-monetization type token / injection hosts are inert; copy them as-is per §0.3 — this
  skill does not activate or remove them.)*
- **The sync endpoint:** its own filename; the uninstall filename it registers; the
  compiled-rules URL + cosmetic-cache filenames it points the client at; the product number
  baked into `typetag`.
- **The uninstall endpoint:** its own filename; for a **V1** source, the identity key **must
  match** the sync endpoint's; the feedback-form URL (this build's).

**← GATE (mandatory):** present the key-map table + the variable set; the owner **confirms
or edits**. No file is copied until both are agreed — a wrong map remaps half the wire and
fails silently.

---

## 5. The canonical key-mapping rules (`00_INDEX.md` §6 — absolute)

1. **`an` / `cid` / `sid` are never renamed** — copied verbatim from the store URL and
   matched byte-for-byte by the backend.
2. **Remapping is a wire-layer concern only — it never reaches the DB.** The DB **column**
   names are canonical (`fingerprint`, `visit`, `instdom`, `whitelistedDom`, …) and identical
   across the family; the only per-build difference inside the DB is the **table name**. Keep
   the backend's read/write of each role pointed at the remapped **wire** key while the column
   underneath keeps its canonical name.
3. **Server-owned state** (`script`, `level`, `conv`, `updates`, `typetag`, `enabled`,
   `disabled`, tier/behaviour columns) stays authoritative in the row; the client stores and
   echoes it but never interprets it.
4. **The hardware-signals array is remapped at the wrapper only** — the outer key gets a
   per-build spelling, but the **inner keys** (`audio_fp`, `screen_res`, …) are canonical and
   match the telemetry columns one-for-one. *One deliberate exception:* raw `canvas` is hashed
   on arrival and stored as `canvas_hash`.

---

## 6. PHASE E — Execute the copy + remap, one file-batch at a time

For **each** file (or a small related batch), in order — start with `config.php` (every file
includes it), then the sync endpoint, then the uninstall endpoint, then the shared
generators/sources, then (V2) the fraud set:

1. **Copy the file verbatim** into `<project>/backend/`. Then apply **only** the two edits:
   (a) remap the wire keys per the confirmed map (§4a), (b) set the §2.3 variables that live
   in this file. **Rename the two endpoint files** as they are copied (§2.1). **Remove
   nothing, rename nothing else, restructure nothing** (§0.1, §0.3).
2. **`[AGENT — offline]` validate:**
   - `php -l` the file (syntax clean).
   - **Remap audit (`grep`):** every key in the map is remapped consistently; **no source
     (old-build) key spelling remains**; `an`/`cid`/`sid` untouched; **no DB column name
     changed**; no frontend key spelling leaked into a column read/write.
   - **Cross-file agreement:** the sync + uninstall filenames, the compiled-rules/cosmetic
     URLs, and `config.php`'s table name/host/asset URL all match the confirmed variable set,
     and the two endpoint filenames match what the frontend hard-codes.
3. **Show the owner the diff + the `[OWNER — manual]` steps for this batch; ← GATE: WAIT.**
4. **Summarize** — append "what was asked / what was done" and a `progress.txt` line.

> The agent's whole job here is the offline copy+remap and its static proof. It never runs
> the file against a live DB, never uploads it, never issues a live request.

---

## 7. Acceptance, DB provisioning & deploy (owner-manual)

Run `BACKEND_ARCHITECTURE.md` §10 — the backend conformance checklist (§10.1 Common +, for a
V2 source, §10.3 V2). The agent runs the `[AGENT — offline]` items (`php -l` across the set;
the remap-completeness audit; the copy-set completeness check vs §2.2 so no dependency is
missing; confirmation the `-dev`/rescue files were **not** copied and the runtime caches were
**not** copied). It produces the `[OWNER — manual]` list for the rest:

1. **Provision the DB `[OWNER — manual]`** — create this build's table from the canonical DDL
   for the source's variant (`DB_SCHEMA.md` §9 — §9.3 for V2-B, the default; + the §9.4 support
   tables) **before first sync**. A **missing table fails silently as a degraded HTTP-200, not
   a visible error** — so this must be done before the endpoint is hit. The **agent never
   provisions or migrates the DB**; it hands over the DDL and the step.
2. **Host & deploy `[OWNER — manual]`** — upload the set, TLS, file permissions, and the first
   live request returning 200. The runtime caches (§2.4) self-generate on first request.
3. **Disclosure `[OWNER — manual]`** — the data collection is disclosed in the privacy policy
   at submission (§0.5).

**Outputs:**
1. **Full report** → `<project>/history/create-backend-<YYYY-MM-DD>.md` (the §4 map + the
   copied files, per-file remap-audit results).
2. **One line** appended to `<project>/progress.txt`.
3. **Provision + deploy checklist** — the ordered `[OWNER — manual]` steps above. This skill
   stops at local folder edits; it does **not** touch the live server or DB.

---

## 8. Stop conditions (hard stops — report and wait)

- **The copy source is not confirmed** (§0.2) — never assume which build/template to copy.
- **The key map or the variable set cannot be agreed** (PHASE M) — do not copy against an
  unconfirmed map; a half-map fails silently.
- A file **fails `php -l`** or the remap audit leaves a stray old-build key / a changed DB
  column name / a leaked frontend key — stop at that batch, report, don't advance.
- The request asks the agent to **provision/migrate the DB or hit the live server** — those
  are `[OWNER — manual]` (§0.4); hand over the checklist and wait.
- The request asks to **edit the template's logic** (add/remove/restructure behavior) — out
  of scope for this skill (§0.3); that is a separate owner-directed change.
- The project layout can't be resolved to a writable `<project>/backend/` tree (§2).

---

## 9. Honesty about state (mirror `00_GOAL.md` §6)

The full **backend-copy path has not yet been run start-to-finish** in practice — treat this
procedure as designed-but-unproven until a real run validates it. Where a step is unknown,
**ask the owner** rather than invent, and **fail safe** (stop and ask, don't guess). The
first real run is expected to surface gaps; when it does, note them so the next build inherits
the fix.

---

*Skill: stand up a new build's backend by wholesale copy of the designated source + a
keys-and-variables remap — never a reimplementation. Sibling of
`create-extension-frontend`. The agent copies and validates offline; the owner provisions
the DB, hosts, deploys, and discloses.*
