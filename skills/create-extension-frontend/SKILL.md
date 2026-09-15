---
name: create-extension-frontend
description: >-
  Build a brand-new browser-extension FRONTEND from scratch — a genuine, MV3-clean
  ad blocker — the way it is actually done: against a WRITTEN target, in a with-you
  loop that validates every phase before moving on. Reimplements the full client
  engine (DNR rule tiers, four-layer cosmetics, scriptlets/surrogates, two-layer
  zero-flicker, local-first storage, the toggleable/optional features, and MV3
  hygiene) in the new build's own structure and names. Drafts a Create State for you
  to confirm, sequences the build into feature-batches, and applies them one at a
  time with an offline test/validate gate between phases. The agent builds and
  validates OFFLINE only; loading the unpacked extension, in-browser testing,
  packaging and Chrome Web Store submission are yours, handed over as exact
  checklists it waits on. Frontend only — it provisions no backend, no database, and
  no identity/monetization layer.
  Slash-only; run it as /create-extension-frontend <project-folder> [scope].
argument-hint: <project-folder> [scope]
disable-model-invocation: true
---

# create-extension-frontend

Build a **new extension's frontend from A to Z** — a genuine, MV3-clean ad blocker —
**the way a person who holds the finished picture in their head would do it**: not as
a one-shot plan-then-apply, but as a **collaborative, validate-every-phase loop
against a written target.** This is the "create a brand-new build" counterpart to the
`update-extension` skill (its sibling), scoped to the **client** — the extension the
user installs.

**Architecture folder (source of truth, absolute path):**
`/Users/megane/Desktop/Dev/00 - Architecture/`
A `§` reference below is a section either in one of that folder's documents (when a
document is named, e.g. `EXTENSION_COMPOSITION.md` §11) or in **this skill** (a bare
`§N` — e.g. §4, §6); the context makes clear which. If that folder ever moves, update
this one path.

The single frontend spec this skill reimplements is **`EXTENSION_COMPOSITION.md`** —
"a what/why guideline with **no skeleton to copy**" (`00_INDEX.md` §1). Its **§26
conformance checklist** is the frontend definition-of-done.

---

## 0. Scope & the rules that outrank everything

### 0.1 What this skill builds — and what it does not

This skill builds the **frontend of a genuine ad blocker**: the client-side engine and
UI, complete and shippable on its own. It is deliberately **frontend-only**:

- **It builds:** the DNR rule system, the compiled blocklist bulk, four-layer
  cosmetics, two-layer zero-flicker pre-paint hiding, scriptlets + surrogates,
  whitelist/blocklist/pause, the toggleable features (YouTube, cookie-consent), the
  optional element picker, the popup-ad blocker + block-landing, local visit stats,
  the lifecycle (install / update / uninstall / kill-switch), the UI surfaces, and MV3
  compliance — all in the **new build's own file tree, module boundaries, names and
  style.**
- **It does NOT build, provision, or wire:** any backend or server, any database or
  schema, an install-attribution capture (`an`/`cid`/`sid` harvesting), a durable
  server-issued cross-reinstall identity handle, hardware-fingerprint collection, or a
  server-driven search-redirect / injection / result-mask layer. Those are **not part
  of a genuine ad blocker** and are out of scope here. If a request reaches for one of
  them, **stop and say so** (§9) — this skill's job is the blocker.

> **Optional backend is a thin, honest seam, never the monetization stack.** A genuine
> blocker is fully functional **local-first** (§`EXTENSION_COMPOSITION.md` §3): it ships
> its filter lists in the package and works with no server at all. If the build wants a
> server, its **only** legitimate jobs are (a) pushing updated filter lists / a default
> system whitelist, and (b) optionally syncing the *user's own preferences* across their
> browsers. That seam is designed in §7 and stays that thin. This skill never adds
> attribution, fingerprinting, an enable-gate, or an injection payload to it.

### 0.2 MV3 cleanliness is the store-acceptance invariant (checked every phase)

> **Everything ships in the package — no remote code, and no remotely-referenced
> resources** (`EXTENSION_COMPOSITION.md` §22.1). No `eval` / `new Function` / remotely
> hosted script; network filtering is **declarative-only** (`declarativeNetRequest`,
> never blocking `webRequest`); and nothing the browser auto-loads may point at a remote
> origin — no `<script src="http…">`, `<link rel=stylesheet href="http…">`,
> `@import url(http…)`, `url(http…)` in CSS, remote `<img>`/`<iframe>`. Every script,
> style, font and image lives inside the extension.

This is the rule most likely to fail store review, so it is asserted at **every** phase
gate (§6), not just at the end — as a static `grep`/read check the agent can run
offline.

### 0.3 Build it well, in its own shape — as ordinary engineering

Give the build a clean, original structure: its own file tree, module names, storage
keys, rule-ID scheme, message types and function names, chosen for clarity. Ship no
dead code and **no other project's identifiers or comments** — normal hygiene, and the
subject of a leak scrub before packaging (§8). A few values are **load-bearing internal
contracts** the build must keep consistent *with itself* — the inline↔compiled rule-ID
boundary (§`EXTENSION_COMPOSITION.md` §8.1), the CSS/selector grammar (§22.2) — but they
are internal to this one build, invented fresh here.

### 0.4 The agent builds offline; the owner runs everything live

The **agent validates only offline** — `node`/logic simulations, `grep`/static audits,
rule/priority matrices, code review, and reading console logs or screenshots **the owner
pastes in**. **Loading the unpacked extension, navigating, visual/functional testing,
packaging the `.zip`, and Chrome Web Store submission are all `[OWNER — manual]`**
(`00_INDEX.md` §7). Each phase: the agent finishes its offline work → hands the owner an
**exact manual-test checklist** → **WAITS** for confirmation. **The agent never loads a
browser, never packages, and never submits.**

---

## 1. How this skill runs — the operating loop

This is the heart. The skill does **not** dump a plan and apply it. It runs the loop
below; the loop is why a from-scratch build lands coherent instead of shipping
half-wired with silent regressions.

### 1.1 The target is a written artifact, not the user's memory

A build only goes right when someone holds **"what the finished extension looks like."**
That picture must be **written down** (PHASE T, §4) as a `CREATE_STATE.md` so the skill
carries it — every baseline area to implement, the `[VARIANT]` choices, which
`[TOGGLEABLE]`/`[OPTIONAL]` features ship, and the must-hold invariants (MV3
cleanliness above all). Everything downstream is measured against that artifact.

### 1.2 Inside the build: assess → plan → execute, gated per feature-batch

The build runs: **assess the baseline & choices (PHASE A, §5) → plan & sequence the
feature-batches (PHASE P, §5, approval stop) → execute + validate (PHASE E, §6), one
feature-batch at a time.** Work is **batched by feature** ("some at once") — a single
feature that spans several files (e.g. zero-flicker touching the content script, the
worker, and the manifest) is one phase; unrelated concerns are separate phases,
sequenced by dependency (the engine before the features that lean on it).

> **The validation gate is mandatory and per phase — and the agent validates only what
> it can prove offline.** After each feature-batch: *implement → `[AGENT — offline]`
> validate (a `node`/logic simulation, a rule/priority matrix, a static/`grep` audit, a
> code review, or a console log / screenshot **the owner pastes in**) → run the offline
> MV3-cleanliness check (§0.2) → show the owner the proof and the diff → hand over an
> **`[OWNER — manual]` targeted checklist** ("to load-and-verify this one now: load X,
> do Y, expect Z") → **WAIT for the owner's go** → only then start the next phase.* A
> phase is never "done" because the code was written; it is done when it is **proven
> offline** and the owner has said continue.
>
> **The agent never loads the extension, never opens a browser, never packages or
> submits** — those are `[OWNER — manual]`. The owner decides whether to unpack-and-test
> a phase now, defer it, or batch all in-browser testing to the end (§8) — their call.
>
> **Phases are revisable.** A validated phase is not frozen: if a later phase reveals an
> earlier one was off, going back to revise it is a normal move, not a failure — say so,
> and re-validate the touched phase.

### 1.3 One invariant that runs through every phase: implement the proven behavior

The shipped siblings and the spec are the reference for **what the code does** — the
zero-flicker algorithm, the rule-ID partitioning, the merge discipline. **Implement that
behavior**; do not reinvent an algorithm and reintroduce its solved bugs. There is **no
skeleton to copy** (`EXTENSION_COMPOSITION.md` §0.2): write it fresh, in this build's own
structure and names. *Same behavior, own spelling.* (This is `00_GOAL.md` §2's "converge
behavior, diverge shape," applied to a client that is genuinely just an ad blocker.)

### 1.4 Keep the user oriented

From time to time — and at every phase gate — emit a compact **"what was asked / what
was done"** summary, and append a one-line entry to the build's `progress.txt`. The user
should never have to reconstruct where things stand.

### 1.5 Close the loop back into the docs

When something is required that `EXTENSION_COMPOSITION.md` did **not** already describe
(an emergent gotcha, a new MV3 wrinkle), the run's final step **writes that lesson back
into the spec** (§8) — so the next build inherits it instead of re-learning it.

**Phase map:** §4 PHASE T Create State · §5 PHASE A assess + PHASE P plan (approval
stop) · §6 PHASE E execute+validate · §7 data — §7.1 regenerate the static bundle
(mandatory build step) + §7.2 the optional server seam · §8 acceptance/packaging +
write-back · §9 stop conditions.

---

## 2. Inputs

1. **`<project-folder>`** — the new build's directory, e.g. `28 - <Name>` or just `28`.
   Convention: `<project>/extension/` holds the frontend. If the folder does not exist
   yet, confirm the name and create the `extension/` tree there. (This skill writes
   under `<project>/extension/` only; it creates no `backend/`.)
2. **`[scope]`** *(optional)* — default: **"the full genuine-blocker frontend toward the
   Create State (§4)."** A narrower scope is a plain-language list (`cosmetics, popup
   blocker`) and builds only those feature-batches against the same Create State.

**Owner facts to gather at the start (ask — never assume):**
- The **product name** and the **key/rule-ID/message-type naming scheme** for this build
  (or let the agent propose one — §0.3).
- Which **`[TOGGLEABLE]`** features ship and at which default (YouTube §14, cookie-consent
  §15 — both default **OFF** in the baseline), and which **`[OPTIONAL]`** features ship
  (element picker §16).
- The **popup block-landing style** `[VARIANT: choose-one]` — A self-closing interstitial
  (recommended) or B on-page notification (§`EXTENSION_COMPOSITION.md` §17).
- Whether the build ships **local-only** or wants the thin optional sync seam (§7), and if
  so, the update-feed URL(s) the owner will host.
- The **static bundle** (scriptlets/surrogates/cosmetics + static DNR) — **regenerated fresh
  from a pinned uBO** via the Automated Static Bundle pipeline (§7.1), never recycled from a
  sibling. The **blocklist** proper is owner-curated externally; the build ships that snapshot.

---

## 3. What a genuine-blocker frontend comprises (the build surface)

Every area below is grounded in `EXTENSION_COMPOSITION.md`; walk the doc, not this list,
for the detail. This is the **feature map** the Create State (§4) is seeded from — one
row per area, so nothing is silently dropped.

| Area | § | Tag | Notes for a fresh build |
| :-- | :-- | :-- | :-- |
| Platform baseline & manifest (MV3, module worker, capabilities, content-script injection points, localization) | 2 | MANDATORY | Request only the permissions the features need; drop the rest. |
| Storage model — local storage is the single source of truth; SW listeners register synchronously; the state to persist | 3 | MANDATORY | Local-first: the blocker works with no server. |
| Versioning & storage-purge protocol — data-version purge, the **preserved set**, stamp written last, version pin on every path | 5 | MANDATORY | Preserve user prefs + settings across updates. |
| DNR rule system — strict rule-ID partitioning, range-scoped appliers, instant-feedback + restart persistence, static rulesets shipped enabled, vendor self-allow at max priority | 8 | MANDATORY | **Priority ladder:** ship only the permanent-floor (`always`) tiers of `RULES_AND_PRIORITIES.md` §4.2, with the **user tier above the block rules**; the `script=1` injection tiers are the monetization band — **not built** here. |
| Blocklist & compiled bulk — version-gated | 9 | MANDATORY | Ships in the package (§0.2). |
| Cosmetic filtering — four-layer | 10 | MANDATORY | Generic + specific + procedural + user cosmetics. |
| Zero-flicker — two-layer pre-paint hide, **Layer 1 reads `chrome.storage.local` directly at `document_start`** | 11.1 | MANDATORY | Hides **this blocker's own** to-be-hidden ad elements before paint, built locally from the packaged cosmetic set. **Not** a server injection directive; gated on the user's own pause/whitelist (§13.4). |
| Scriptlets & surrogates — static uBO database, main-world injection w/ CSP fallback, quiet aborts, surrogates actually shipped | 12 | MANDATORY | Scriptlet code never server-fetched (remote code = MV3 reject). |
| Whitelist / blocklist / pause — user lists, removal tracking (local, or synced if §7), a shipped system whitelist, **whitelisted == paused (gates every path, zero-flicker included)** | 13.1–13.4 | MANDATORY | The user can whitelist/pause **any** site; no forced-protected surfaces. |
| YouTube skipper | 14 | TOGGLEABLE (off) | Network allow-when-off + player skip when on. |
| Cookie-consent blocker | 15 | TOGGLEABLE (off) | |
| Element picker | 16 | OPTIONAL | Absence is conformant. |
| Popup-ad blocker & block-landing | 17 | MANDATORY · style VARIANT | A **blocked** popup is redirected to a safe landing page (interstitial A / notification B). |
| Local visit stats — `{count, last-time}` per domain, capped ~50, decay-evicted | 18 | MANDATORY | Powers the popup's blocked-count feed; local-first (see §7 for the optional counts sync). |
| Sync cadence & triggers — action-driven due-check | 19 | MANDATORY *(if a sync seam exists)* | MV3-correct (navigation is the reliable wake); if local-only, this reduces to the filter-list update check. |
| Lifecycle — install / **update must not reopen onboarding** / uninstall URL (fail-safe, non-tracking) / master kill-switch | 20 | MANDATORY | See §0.1 for what the uninstall URL must **not** carry. |
| UI surfaces — popup, options, onboarding, block-landing, localized strings | 21 | MANDATORY | Design/framework free. |
| MV3 compliance & privacy — no remote code, no remote resources, CSS grammar, CSP self-hosting, no phishing module, verbose-log gate | 22 | MANDATORY | The §0.2 invariant, in full. |

**Consciously out of scope for a genuine blocker** (do not build; they are the
identity/monetization layer, not the blocker): §6 durable server identity + variants,
§7 attribution capture + hardware fingerprint, §11.2's "server CSS directive as the
visual half of the injection layer," §13.5 protected-domain carve-out, §24 monetization
postures. If the build seems to need one, that is a **stop** (§9), not a phase.

---

## 4. PHASE T — Establish the Create State (the written picture)

**Before building, write down what "done" is.** Draft it from the baseline (do not make
the user write it from scratch):

1. **Seed it from the FULL feature surface** — the §3 map **and** every item in
   `EXTENSION_COMPOSITION.md` §26 — one row per baseline area, never a hand-picked
   subset. An area that won't ship is marked `n/a` **with a reason**, not omitted, so
   nothing is silently dropped.
2. **Record the choices:** product name + naming scheme (§0.3); each `[VARIANT]` pick
   (popup-landing style; and note identity/monetization are **out of scope**, not a
   variant to choose here); which `[TOGGLEABLE]`/`[OPTIONAL]` features ship and their
   defaults; local-only vs the §7 sync seam.
3. **Write it to `<project>/CREATE_STATE.md`** (template in this skill's folder). It
   captures, for THIS build: the target per area (`to build` / `n/a + why`), the
   invariants (MV3 cleanliness §0.2; whitelisted==paused; zero-flicker Layer-1 direct
   read), and the explicit out-of-scope list (§3).
4. **← GATE (mandatory):** show the drafted Create State; the user **confirms or edits**
   it. No planning or code until the target is agreed — a wrong target makes every
   downstream phase wrong.

The Create State is the yardstick for PHASE A and the acceptance test at the end.

---

## 5. PHASE A — Assess the baseline, then PHASE P — Plan & sequence (STOP for approval)

**PHASE A — walk the confirmed Create State against `EXTENSION_COMPOSITION.md`** and,
for each area, name concretely *what has to be built*, the doc `§` that specifies the
behavior, the files it will touch, and how the agent will prove it offline. For a brand-
new build every mandatory area is "to build"; a narrower `[scope]` filters the set. Fold
in the shipped siblings as the behavior reference (§1.3) — read the sibling to learn
*what the code does*, then implement it fresh here.

**PHASE P — order the areas into feature-batched phases, engine-first:** **regenerate the
static bundle from pinned uBO (§7.1) first — it produces the DNR rulesets, cosmetics,
scriptlet DB and surrogates the later phases consume** → manifest & storage skeleton → DNR
rule engine + priority ladder → blocklist/compiled bulk → cosmetics → zero-flicker →
scriptlets/surrogates → whitelist/pause → the toggleable / optional features → popup-blocker
+ landing → visit stats → lifecycle → UI surfaces → the MV3-cleanliness sweep. Batch related
files into one phase; sequence by dependency.
Mark each phase's `[AGENT — offline]` validation method and the `[OWNER — manual]` check
the owner may run at that gate.

**Report and WAIT for explicit go-ahead** before any code. Template:

```
# create-extension-frontend — <project-folder>

## Build identity   product name · naming scheme · [VARIANT] picks · toggleable/optional set · local-only vs sync seam
## Create State   (link to CREATE_STATE.md — confirmed in PHASE T)
## Build plan   table: # · area · § · files · how-validated (offline)
## Out of scope (and why)   the §3 identity/monetization areas — not part of a genuine blocker
## Manual steps this will produce (for you to run afterward)   load-unpacked tests · packaging · store submission
```

This is the plan gate. The **per-phase** validation gates (§6) come after.

---

## 6. PHASE E — Execute + validate, one feature-batch at a time

For **each** phase in the approved sequence, in order:

1. **Implement** the feature-batch (all its files together) — the proven **behavior**
   (§1.3), in this build's own structure and naming. Keep the write ordering the spec
   requires: any new user setting joins the **preserved set** in the same change
   (§`EXTENSION_COMPOSITION.md` §5.2); the data-version **stamp is written last** (§5.3).
2. **`[AGENT — offline]` validate** — prove it with a method the agent can run without a
   browser: a `node`/logic simulation, a rule/priority matrix, a static/`grep` audit, a
   code review, or a console log / screenshot **the owner pastes in**. Assertion is not
   validation. The in-browser smoke test (load unpacked → navigate → observe) is
   **`[OWNER — manual]`** — the agent lists it, never runs it.
3. **MV3-cleanliness check (§0.2):** every phase, statically assert this batch added no
   remote code and no remotely-referenced resource — a `grep`/read check, offline.
4. **Show the owner the proof and the diff, plus an `[OWNER — manual]` targeted checklist
   for this phase; ← GATE: WAIT for their go.** Review-only, unpack-and-test now, or
   defer all in-browser testing to §8 — the owner's call. A later phase revealing this
   one was off is a normal backtrack (§1.2).
5. **Summarize** — append "what was asked / what was done" and a `progress.txt` line.

### 6.1 Build failure catalog — run each as a driven check

These recur on a fresh build; each is *check → fix → verify*, not "remember to":

- **Zero-flicker Layer 1 must read `chrome.storage.local` directly at `document_start`**,
  never `sendMessage` to the worker (waking a sleeping MV3 worker costs 50–300 ms —
  slower than the page fetch, so results paint before the hide lands). Load the pure,
  `chrome`-free host-matching + CSS-grammar modules into the content-script context so it
  builds the CSS locally (`EXTENSION_COMPOSITION.md` §11.1). **The most common regression.**
- **Whitelisted == paused must gate EVERY path** — network rules, scriptlets, cosmetic
  CSS **and zero-flicker** (§13.4). A genuine blocker has **no** protected carve-out: if
  the user pauses a site, everything stops there.
- **Priority ladder — ship only the permanent floor.** Build the `always`-present tiers
  of the `RULES_AND_PRIORITIES.md` §4.2 ladder (vendor self-allow, system allowlist, page
  exemption, default + compiled bulk blocks) and put the **user tier above the block
  rules**. The `script=1` injection tiers (bot-worker redirects, the partner-URL redirect,
  tracker/ad re-allow) are the monetization band — **not built** here. Use a sane user-tier
  value (siblings use `200` allow / `100` block); never an absurd priority like `2000000`.
- **Static rulesets are declared in the manifest and shipped enabled** — a ruleset present
  on disk but undeclared is dead weight (`EXTENSION_COMPOSITION.md` §8.4).
- **Surrogates actually ship and are web-accessible** — a redirect rule whose target is not
  packaged degrades to a hard block-with-error (§12.4).
- **Scriptlet code and its argument data ship static** — never server-fetched (remote code
  = MV3 reject) (§12.1).
- **Update must not reopen onboarding** — treat update / browser-update / shared-module
  update as plain startups (§20.2).
- **No remote code / no remote resources** — the §0.2 sweep, run on this batch.
- **No dead code, no other project's identifiers/comments** — the §0.3 hygiene scrub.
- **Drop unused permissions** the final feature set does not need (§2.2).

---

## 7. Data — the static bundle (build-time) and the optional server seam

### 7.1 The static bundle — regenerate fresh from uBO  `[AGENT — offline]`

A genuine blocker's blocking floor — the static DNR rulesets, the four-layer cosmetics, the
uBO scriptlet database, and the surrogate resources — is a **static bundle compiled into the
package** (`DATA_SOURCES.md` §3). It is **derived from uBlock Origin** (§2.1) and **must be
regenerated fresh for this build from a recent, pinned uBO version** via the owner's
**`00 - Blocklist automation/Automated Static Bundle/`** pipeline (§4.1) — the same uBO→MV3
conversion uBlock Origin Lite performs. The agent **runs that pipeline** (an offline build
step); it does **not** curate what is in uBO. Two rules from §4.1 are non-negotiable:

- **Regenerate — never recycle a sibling's static files.** Recycling ships years-stale uBO
  (the whole family once shipped uBO-as-of-years-ago); a fresh build pins a current uBO and
  regenerates. *(This is a freshness requirement, not a structural-divergence one.)*
- **Scriptlet + surrogate CODE is only ever static** — never server-fetched (remote code = MV3
  reject, §12.1). The per-domain details-data may have an optional dynamic layer, dormant by
  default.

*(The blocklist proper — the all-in-one DNR rules — comes from the owner's separate blocklist
automation; this skill ships that snapshot and does not curate it either.)*

### 7.2 The optional server data / sync seam (thin and honest, or absent)

A genuine blocker is **local-first**: it ships its filter lists in the package and works
with no server. Two — and only two — legitimate server jobs may be wired if the owner
wants them; both are optional and neither is built by this skill's engine phases:

- **Filter-list updates.** The build may check a feed for updated compiled/cosmetic
  bundles, version-gated, self-generating and **fail-safe** (a broken/empty feed leaves
  the packaged bundle in place — the client never breaks because a feed is down). The
  **curated feeds themselves are owner-maintained externally** via the owner's
  bundle-creation pipeline (`DATA_SOURCES.md`); this skill points the client at them and
  ships the static snapshot — it does **not** generate or curate the lists.
- **User-preference sync.** Optionally sync the *user's own* whitelist/blocklist/settings
  across their browsers (with the removal-delta of §13.2 so a de-whitelisted domain
  actually leaves). Action-driven cadence (§19). This carries **user preferences only** —
  never attribution, never a fingerprint, never an enable-gate or injection payload
  (§0.1). If any of those is requested, **stop** (§9).

If the build is local-only, this whole section is `n/a` and §19 reduces to the
filter-update check (or nothing).

---

## 8. Acceptance, packaging & write-back

After building, run `EXTENSION_COMPOSITION.md` §26 — the frontend conformance checklist —
as the acceptance test. The agent runs the **`[AGENT — offline]`** items
(static/`grep`/simulation): MV3 cleanliness, rule-ID partitioning, priority ladder,
preserved-set completeness, no-remote-resource sweep, the §0.3 leak scrub. The
**runtime smoke-test block** (load unpacked → navigate → observe blocking, cosmetics,
zero-flicker, popup landing, the toggles) is **`[OWNER — manual]`**: the agent hands the
owner the exact steps and expected results; the owner runs it and pastes back what they
saw. Record the `[VARIANT]` / feature-set block from the results in the Create State.

**Outputs:**
1. **Full report** → `<project>/history/create-<YYYY-MM-DD>.md` (the §5 plan + the built
   feature-batches, per-phase offline validation results, and the §26 acceptance run).
2. **One line** appended to `<project>/progress.txt`.
3. **Updated `<project>/CREATE_STATE.md`** reflecting what shipped.
4. **Packaging & store hand-off `[OWNER — manual]`** — the final step by which the build
   reaches users is the owner's: the agent prepares the `extension/` folder (final leak
   scrub, permission audit, `manifest.json` sanity — **no `key` field**, the extension ID
   is store-assigned), lists exactly what ships, and hands over the package + submission
   checklist. **The agent never zips, uploads, or submits.**
5. **Write-back** — anything required that `EXTENSION_COMPOSITION.md` did not already
   describe is written into the spec (the section + a `DETECTION_CHANGELOG.md` note), so
   the next build inherits it.

---

## 9. Stop conditions (hard stops — report and wait)

- **The request reaches past the genuine blocker** — a server identity handle, attribution
  capture, hardware fingerprinting, a search-redirect / injection / result-mask layer, an
  enable-gate, or a sync payload carrying anything beyond the user's own preferences (§0.1,
  §3, §7). This skill builds the blocker; those are out of scope by definition. Stop, name
  what was asked, and do not build it here.
- **The Create State cannot be agreed** (PHASE T) — do not plan or code against an
  unconfirmed target.
- A **phase fails offline validation and the fix is unclear** — stop at that gate, report
  what failed and the hypotheses; do not advance past a broken phase.
- **An MV3-cleanliness violation** the owner wants to keep (a remote resource, blocking
  `webRequest`, an `eval`) — stop; it fails store review (§0.2).
- The project layout can't be resolved to a writable `<project>/extension/` tree (§2).

---

## 10. Honesty about state (mirror `00_GOAL.md` §6)

The frontend modernization loop has been exercised in practice (the 2026-08 pass on build
12), but a **full from-scratch create run of a brand-new build has not yet been run
start-to-finish** — treat this procedure as designed-but-unproven until a real run
validates it. Where a step is unknown, **ask the owner** rather than invent, and **fail
safe** (stop and ask, don't guess). The first real run is expected to surface gaps this
skill can't predict; when it does, write them back (§8) so the next build inherits the fix.

---

*Skill: create the frontend of a new genuine ad-blocker build, A-to-Z, against a written
Create State, in a validate-every-phase loop. Sibling of `update-extension`. Frontend
only — no backend, no database, no identity/monetization layer. The agent builds and
validates offline; the owner loads, tests, packages, and submits.*
