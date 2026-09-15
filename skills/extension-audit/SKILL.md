---
name: extension-audit
description: >-
  Verify a build end-to-end, offline, against the whole documentation set — the
  cross-cutting checks no single doc owns. Confirms the frontend↔backend↔DB key map lines
  up start to finish, scrubs for sibling-build mentions and monetization/injection remnants
  (active OR inert), checks artifact completeness (the db_schema snapshot is present, the two
  endpoint files match the frontend's hard-coded URLs, no -dev twins or runtime caches got
  copied, MV3-clean), and orchestrates each doc's conformance checklist filtered by the
  reachable-set rule. Emits a small PASS / FAIL / [OWNER — verify live] recap you review; then
  you unpack and smoke-test it yourself; then we write the build's rows into every
  per-extension table together. Read-only on the build — the agent audits offline and never
  loads a browser, packages, or deploys. Works on a new build or an existing one.
  Slash-only; run it as /extension-audit <project-folder> [scope].
argument-hint: <project-folder> [scope]
disable-model-invocation: true
---

# extension-audit

**Verify a whole build against the whole documentation set — the checks that fall *between*
the per-doc checklists.** Each specification's conformance checklist verifies one subject in
isolation; this skill owns what no single doc does: the **frontend↔backend↔DB seam**, the
**negative leak scrubs** (no siblings, no monetization/injection), and **artifact
completeness** — plus it orchestrates the per-doc checklists in one pass. It is the
**verifier** that completes the trio with the two builders (`create-extension-frontend`,
`create-extension-backend`).

**Architecture folder (source of truth, absolute path):**
`/Users/megane/Desktop/Dev/00 - Architecture/`
A `§` reference below is a section either in one of that folder's documents (when a document
is named, e.g. `00_INDEX.md` §6) or in **this skill** (a bare `§N`); the context makes clear
which. If that folder ever moves, update this one path.

---

## 0. Scope & the rules that outrank everything

### 0.1 What this skill is — a read-only, offline verifier

It **audits**; it does not build or fix. It reads the build's `extension/` and `backend/`
folders and the Architecture docs, runs the offline checks (§3), and emits a **recap** (§4).
Its **only** writes to disk are the per-extension **doc-table rows at the very end** (§6) —
and only after you have reviewed the recap and live-tested the build. It **never modifies the
build's code**, never provisions or touches the DB, never loads a browser, never packages or
deploys. Works on a **new** build or an **existing** one (a pre-submission gate or a health
check — the codified form of the ad-hoc full-build audit).

### 0.2 Offline-only — and honest about the boundary

Everything here is `[AGENT — offline]`: `grep`/static audits, `php -l`, key-map cross-reads,
`node`/logic simulations, reading logs/screenshots you paste in (`00_INDEX.md` §7). **The
runtime smoke test — load unpacked → navigate → observe — is `[OWNER — manual]`.** The audit
**cannot** confirm a behavior actually fires; it can only confirm the code/config/wiring that
should produce it. So every check lands in one of three buckets, never blurred:
**PASS** (proven offline) · **FAIL** (a concrete defect, with `file:line`) · **[OWNER — verify
live]** (correct on paper; only a live test confirms it).

### 0.3 The flow (what you asked for)

1. **`[AGENT — offline]`** run the audit passes (§3).
2. **`[AGENT — offline]`** emit the small PASS / FAIL / [OWNER — verify live] recap (§4).
   **← GATE: you review it.**
3. **`[OWNER — manual]`** if the recap is clean, you unpack and smoke-test the build yourself
   (§5); paste back what you saw.
4. **`[AGENT — offline]` + you, together** once you've live-confirmed, we draft and write the
   build's rows into every per-extension table (§6) — **after** the live test, because those
   cells record *verified* reality, not intentions.

---

## 1. Inputs

1. **`<project-folder>`** — the build to audit, e.g. `28 - <Name>` or `28`. Convention:
   `<project>/extension/` + `<project>/backend/`. If only one half exists (a frontend-only
   build, or a backend not yet created), audit what's present and mark the rest `n/a` with a
   reason — never assume the missing half passed.
2. **`[scope]`** *(optional)* — default: **the full audit (§3.1–§3.4).** A narrower scope runs
   only the named passes (e.g. `seam, leaks`).

Detect, don't assume: read the identity variant/generation, the handle/key prefix, the table
name, and the endpoint filenames from the build's **own** code (the `update-extension` §4
signals). The audit is only as sound as the build map it starts from — if detection is
ambiguous, that is itself a FAIL to surface, not a guess to paper over.

---

## 2. How this skill runs

Detect the build (§1) → run the four offline passes (§3) → emit the recap (§4, **STOP** for
your review) → you live-test (§5) → we write the doc rows (§6). Keep the user oriented: at the
recap and again after the write-back, append a one-line entry to `<project>/progress.txt`.

**Phase map:** §3 the offline passes · §4 the recap (review stop) · §5 owner live test ·
§6 the collaborative doc write-back · §7 stop conditions.

---

## 3. The offline audit passes

### 3.1 The seam — frontend ↔ backend ↔ DB, end to end  *(the check no single doc owns)*

For every keyed role in `00_INDEX.md` §6, confirm the three layers agree:

- **Frontend key ↔ backend wire key.** Each key the frontend stores/sends/echoes has a
  matching read/write on the backend under the **same per-build spelling** — the handle
  (`…_fp`), version, install-sites, visited-sites, sync/last-sync, data-version,
  acceptable-ads, user-allow/block/cosmetics, the hardware-signals wrapper, and the response
  cache keys (`rules`, system-whitelist, **the zero-flicker `…_local_css` directive**, cosmetic
  feeds, compiled-rules version, scriptlet excludes). A key present on one side and absent/
  misspelled on the other is a **FAIL** — a silently half-wired role.
- **`an` / `cid` / `sid` verbatim** on both sides (never renamed/prefixed).
- **Wire key ↔ DB column.** The remap is wire-layer only: the DB **column** names stay
  canonical; only the table name is per-build. A frontend spelling that leaked into a column
  read/write is a **FAIL**.
- **Endpoint agreement.** The frontend's hard-coded sync URL and the uninstall URL it registers
  match the two backend endpoint filenames (`BACKEND_ARCHITECTURE.md` §2.1). A mismatch is a
  **FAIL** (the client posts into the void).
- **Directive content match, not just the key.** Where both halves carry the zero-flicker
  directive, confirm the backend *sends* what the frontend *builds/targets* (host patterns,
  selectors, timing). **Whether the mask actually fires is `[OWNER — verify live]`.**

### 3.2 The negative scrubs — nothing that must NOT be there

- **No sibling / scheme leaks in shipped frontend code** (`EXTENSION_COMPOSITION.md` §0.2):
  `grep` the shipped `extension/` for any other build's product name, codename, build number,
  endpoint, or key prefix, and for scheme words (the injection/monetization vocabulary, the
  partner-feed/`script`-flag/YHS references, a surrogate's real purpose). Any hit is a **FAIL**
  — a correlation or disclosure leak. *(Intent lives in `progress.txt`/`history/`, which must
  not ship inside `extension/`.)*
- **No monetization / injection remnants — active OR inert.** In both halves, `grep` for and
  flag: an active `injectGoogleScripts()` (or renamed equivalent) **call**; the injection DNR
  band (search→partner redirect, bot-worker redirects, CSP/header-strip, tracker/ad re-allow,
  analytics-swap — e.g. rule IDs in the `8011`/`8014`/`5005` families); a zero-flicker directive
  that hides the **organic results** container (an `opacity:0` + timed-reveal op over the
  results, not the ad slots) — the search-result **mask**; an active enable-gate / market-
  approval path; AFS/YHS host allows on the request path. **Active** remnants are a **FAIL**.
  **Inert** remnants (defined-but-uncalled functions, unused rule-ID arrays, commented blocks)
  are a **WARN** — not shipping harm, but exactly what a review or an acquirer's diligence
  flags; list them so the owner can decide. *(This pass is what catches a re-added `#res`/
  `#gevUs` mask or a stray redirect rule.)*

### 3.3 Artifact completeness & MV3 cleanliness

- **DB schema snapshot present:** a structure `.sql` exists under
  `<project>/backend/docs/db_schema/` for this build's table (`update-extension` §4.4). Absent
  ⇒ **FAIL** (or WARN if backend-not-yet-created) — a missing snapshot means nothing can verify
  the columns the wire writes to.
- **Backend copy hygiene:** both endpoint files present and named consistently with the
  frontend; the `-dev` twins and one-off rescue scripts were **not** copied
  (`BACKEND_ARCHITECTURE.md` §2.5); the runtime-generated caches were **not** committed (§2.4);
  every fixed-name dependency the endpoint `include`s is present (§2.2). A missing dependency is
  a **FAIL** (silent runtime failure).
- **MV3 clean** (`EXTENSION_COMPOSITION.md` §22.1): `grep` the shipped `extension/` for remote
  code (`eval`/`new Function`/remote script), blocking `webRequest`, and remotely-referenced
  resources (`<script src=http>`, `<link href=http>`, `@import url(http)`, `url(http…)` in CSS,
  remote `<img>`/`<iframe>`). Any hit is a **FAIL** — a store-review rejection.

### 3.4 Orchestrate the per-doc checklists (delegate — do not re-implement)

Run each document's conformance checklist, **filtered by the reachable-set rule**
(`00_GOAL.md` §3: Common items on every build; a `[V2]`/substrate-shaped item only where the
build's schema carries the column). Run only the `[AGENT — offline]` items; route the runtime
block to §5. Reference the docs; don't duplicate their content here:
`EXTENSION_COMPOSITION.md` §26 · `RULES_AND_PRIORITIES.md` §9 · `USER_CREATION_AND_UPDATE.md`
§12 · `FRAUD_DETECTION_V5.md` §19 · `BACKEND_ARCHITECTURE.md` §10 · `DB_SCHEMA.md` §8 ·
`DATA_SOURCES.md` §6. Any offline item that fails is a **FAIL** with its doc `§` + `file:line`.

> The per-doc passes MAY be fanned out — one reader per checklist against the real code — then
> merged into the one recap (§4).

### 3.5 Structural profile — descriptive, for the §25.2 row  *(informational, not a check)*

Read how this build **structures** each feature along the `EXTENSION_COMPOSITION.md` §25.2 axes —
file tree / modules, storage-key scheme, DNR-ruleset layout, scriptlet storage, zero-flicker
directive format, cosmetics format, surrogates/WAR, message-types — and record the *shape* (with
file evidence) for each. This produces the build's **structural profile**, which becomes its §25.2
entry at write-back (§6).

This pass is **purely descriptive and carries no verdict** — it is **not** a PASS/FAIL check and
**never** judges whether the build is "different enough" from siblings (that store-review framing is
out of scope, `create-extension-frontend` §0.3). It states *how* this build shapes each feature; where
that shape matches a sibling's, it says so plainly ("uniform with `<build>`"). Emit it as the recap's
**Structural profile** section (informational), separate from PASS/FAIL/WARN. *(The offline-only rule
still holds: describe what the code shows, don't infer runtime behavior.)*

---

## 4. The recap — small, trisected, then STOP

Emit a **compact** recap the owner can scan — not a wall of prose. One line per area, bucketed:

```
# extension-audit — <project-folder>   (variant/gen · handle · table · endpoints)

PASS  (proven offline)
  ✅ seam: all §6 keys agree FE↔BE↔DB · an/cid/sid verbatim · endpoints match frontend
  ✅ no sibling/scheme leaks · MV3 clean · schema snapshot present · deps complete
  ✅ <per-doc offline items that passed>

FAIL  (concrete defect — fix before shipping)
  ❌ <one line each> — file:line — what's wrong — which § it violates

OWNER — verify live  (correct on paper; only a live test confirms)
  ☐ <one line each> — load X, do Y, expect Z

WARN  (inert; your call)
  ⚠️ <inert monetization/injection remnants, unused deps, etc.>

Structural profile  (§3.5 — descriptive, no verdict; becomes the §25.2 row)
  • <feature> — <how this build shapes it> (uniform with <build> | own shape)
```

Give **`file:line` evidence for every FAIL and WARN**; PASS lines need no detail. **← GATE:
present the recap and WAIT.** If there are FAILs, the build is not ready — the owner fixes (or
re-runs a builder skill) and re-audits; the audit does not fix.

---

## 5. Owner live smoke test  `[OWNER — manual]`

When the recap is clean of FAILs, the owner unpacks and tests. The agent hands the exact
`[OWNER — verify live]` steps from §4 (the runtime block of `EXTENSION_COMPOSITION.md` §26 plus
any feed/endpoint check): load unpacked → navigate → observe blocking, cosmetics, zero-flicker,
the popup landing, the toggles; and, if a backend is live, a real sync returning 200 and the
feed URLs returning real data. The owner runs it and pastes back what they saw. **The agent
never runs this.**

---

## 6. PHASE W — Write the build into the docs (together, after the live test)

Only once the owner has live-confirmed: draft the new build's **row for every per-extension
table**, grounded in what was built + the live results, then the owner **confirms/edits** and
the agent writes them. This is the "close the loop into the docs" step, and it is done **after**
the live test on purpose — the cells record verified reality.

Tables to add the build's row to (one row each, cells grounded — never guessed):
`EXTENSION_COMPOSITION.md` **§25.1** (deviation matrix — conformance axis) **and §25.2**
(structural-variant map — the build's **structural profile** from §3.5, one entry per feature axis) ·
`RULES_AND_PRIORITIES.md` §8 · `USER_CREATION_AND_UPDATE.md` §13 · `FRAUD_DETECTION_V5.md` §18 ·
`BACKEND_ARCHITECTURE.md` §9 · `DB_SCHEMA.md` §7 · `DATA_SOURCES.md` §5 — plus the **build map**
(the `update-extension` skill §3 table: folder · codename · variant · generation · sync endpoint ·
handle key · DB table). Record the `[VARIANT]`/feature block from the live results.

> **§25.2 note:** the build's §25.2 entry is its **structural profile** (§3.5) — purely descriptive
> (how it shapes each feature), never a divergence-*adequacy* judgment. Where its shape matches a
> sibling's on an axis, record it honestly as "uniform with `<build>`" rather than inventing a
> difference — an honest uniform is a signal of where fresh divergence is still owed.

**← GATE:** show the drafted rows; the owner confirms or edits; then the agent writes them.
Append the final line to `<project>/progress.txt`. Anything the audit surfaced that the specs
didn't already describe (a new gotcha) is written back into the relevant section too.

---

## 7. Stop conditions (hard stops — report and wait)

- **Identity/variant detection is ambiguous** (§1) — surface it as a FAIL; do not audit against
  a guessed build map.
- **The recap has FAILs** — stop at the review gate; the owner fixes and re-audits. The audit
  does not modify the build.
- **The owner has not live-confirmed** — do not write any per-extension table row (§6); the
  cells must record verified reality.
- The project layout can't be resolved to `<project>/extension/` and/or `<project>/backend/`.
- The request asks the agent to **fix** the build, **provision the DB**, **package**, or **run
  the live test** — out of scope; the audit reports, the owner acts.

---

## 8. Honesty about state (mirror `00_GOAL.md` §6)

This audit can only prove what is checkable offline; a clean recap is **necessary, not
sufficient** — the owner's live smoke test is what confirms the build actually works. The full
create → audit → record path has **not yet been run start-to-finish**; treat it as
designed-but-unproven, **fail safe** (a WARN or [OWNER — verify] rather than a false PASS when
unsure), and **ask rather than guess**. Write back any gap the first real run surfaces.

---

*Skill: audit a whole build offline against the whole doc set — the seam, the leak scrubs, and
the completeness checks the per-doc checklists don't own, plus their offline items in one pass —
emit a small trisected recap, wait for the owner's review and live test, then record the build
into every per-extension table together. Read-only on the build; the owner tests and deploys.*
