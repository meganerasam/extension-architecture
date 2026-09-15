# Session brief — build the `create-extension` skill

> **⚠️ SUPERSEDED (2026-09-03) — historical context only.** This brief scoped a single
> `create-extension` skill provisioning identity/DB/backend and carrying the search-monetization /
> injection layer. **That path was not built.** What shipped instead is a **de-monetized trio** —
> **`create-extension-frontend`** (a genuine ad-blocker frontend), **`create-extension-backend`**
> (copy a de-injected backend template + remap keys), and **`extension-audit`** (verify end-to-end +
> record into the docs). None of them builds a monetization/injection layer. **Use those three
> skills.** There is deliberately **no `create-extension/SKILL.md`** — this folder holds only this
> brief, kept for history.

*Hand this to a fresh session. It is self-contained, but everything it asserts is grounded in the
Architecture folder — read the docs and verify; do not trust this brief blindly.*

---

## Your task

Build a new, complete **`create-extension`** skill: the "create a brand-new browser-extension
build from A to Z" counterpart to the existing **`update-extension`** skill. Author it as
`skills/create-extension/SKILL.md`, matching the structure, rigor, and voice of
`skills/update-extension/SKILL.md` (that skill is your gold-standard template — the sibling of
what you're writing).

**You edit docs/skill files only. Do NOT touch any live extension, backend, server, or DB.**

## What this family is (one paragraph)

A family of near-identical ad-block browser extensions built on a deliberate **frontend↔backend
asymmetry**: the **frontend is reimplemented fresh per build** (different structure/names/keys, to
avoid Chrome Web Store correlation/plagiarism flags), while the **backend is copied wholesale from
a prior build and only its keys/variables are remapped** (the server is never seen by store
review, so sameness is safe). Every build pursues **two co-equal goals** — a genuine MV3 ad blocker
**and** a search-monetization path — with a self-protecting identity/sync layer serving both.

## Read these first — the hardened source of truth

Architecture folder (absolute): `/Users/megane/Desktop/Dev/00 - Architecture/`

1. **`00_GOAL.md` — THE CHARTER. Read it first.** The two co-equal goals; converge-behavior /
   diverge-shape; the **reachable-set filter + decision B**; the per-generation capability levels
   (Gen 0 / V1 / V2-A / V2-B); and the **universal-vs-per-product** split. Everything you build
   serves this.
2. **`00_INDEX.md`** — §1 the frontend↔backend asymmetry; **§4 the A-Z build procedure** (your
   spine); **§6 the canonical key mapping** (what makes "copy the backend, change the keys" safe);
   **§7 the `[AGENT — offline]` / `[OWNER — manual]` actor-tag convention**.
3. **`EXTENSION_COMPOSITION.md`** — the whole frontend to reimplement; its **§26 conformance
   checklist** is the frontend definition-of-done.
4. **`USER_CREATION_AND_UPDATE.md`** (§11 identity chooser, default **V2-B**),
   **`BACKEND_ARCHITECTURE.md`**, **`DB_SCHEMA.md`**, **`FRAUD_DETECTION_V5.md`** — the backend to
   copy + remap.
5. **`DATA_SOURCES.md`** — the data/cache pipeline.
6. **`skills/update-extension/SKILL.md` — the STRUCTURAL TEMPLATE.** Reuse its written-target-first
   approach, its per-phase validation gate, its phase map, its stop conditions, its actor model,
   and its write-back-to-docs step. `create-extension` is its sibling.

Your **project memory** (`MEMORY.md`) also carries a `create-extension-plan` note plus the key
decisions (`reachable-set-schema-freeze`, `scope-boundary-injection-layer`, `docs-hardening-pass`)
— read them.

## Honest state — this path has NEVER been run

The full A-Z build / backend-copy path has **never been exercised end-to-end** (`00_GOAL §6`). You
are writing it against the owner's **real answers**, not guessing. Where a step is unknown, **ASK
the owner** — never invent. The skill you produce is **unproven until a real create run validates
it**; say so in the skill (mirror `00_GOAL §6`'s honesty caveat), and write it to **fail safe and
ask rather than guess**.

## Non-negotiable invariants (honor exactly — they are already in the docs)

- **Validation model.** The **agent validates only offline** — `node`/logic simulations,
  `grep`/static audits, code review, reading logs/screenshots the owner pastes in. **ALL**
  browser/live/deploy steps (load unpacked, navigate, visual test, upload, DB migration, store
  submission, any live-server hit) are **`[OWNER — manual]`**. Each phase: the agent finishes its
  offline work → hands the owner an **exact manual-test checklist** → **WAITS** for confirmation.
  **The agent never browser-tests and never deploys.**
- **Decision B — the birth-time choice locks the reachable set forever.** At creation you **choose
  the identity variant + schema** (default **V2-B**); that choice **permanently fixes what the
  build can ever reach** (`00_GOAL §3/§4/§5`). It is the **highest-stakes decision in the whole
  flow** — make it a prominent, owner-confirmed gate. (Growing a schema later is forbidden; that is
  what `update-extension` enforces.)
- **Two co-equal goals + universal-vs-per-product** (`00_GOAL §3`). **Monetization is universal** —
  a per-build posture/owner-toggle, **not** generation-gated. The generation gates only the
  identity mechanism and the V2 fraud/hardware capabilities.
- **Frontend↔backend asymmetry** (`00_INDEX §1, §6`): frontend **reimplemented fresh** (diverge
  structure/names/keys); backend **copied wholesale**, keeping function names, remapping only keys
  + the per-build variable set.
- **The boundary.** You document and wire the **ad-block engine** and the client's contract
  obligations. The **monetization/injection/tracking layer** you keep correct and protect
  ("monetization must not break") but **do not build out**; its live testing is the owner's.

## Always-ask rules (never assume)

- **Always ask which build's backend to copy from** — never assume Ninja/Ghost or any specific one.
- **Always ask whether the target DB is provisioned, and read the actual
  `<project>/backend/docs/db_schema/*.sql` snapshot** — never assume a column/table exists.
- **Uninstall form:** the agent ensures the post-uninstall redirect is copied into the server files
  and **re-pointed to THIS build** (not a leftover from the copied-from build); the **owner** owns
  and verifies the live form.
- **Curated data feeds** (blocklist, cosmetics, scriptlets, the all-in-one whitelist) are
  **owner-maintained externally**; the static bundle is produced by the owner's **automated
  bundle-creation pipeline**. `create-extension` does **not** generate or curate them.
- **No manifest `key`** (the extension ID is store-assigned). **Post-release monitoring is out of
  scope.**

## The A-Z lifecycle the skill must cover (see `00_INDEX §4`)

Gate every phase; actor-tag every step (`[AGENT — offline]` vs `[OWNER — manual]`):

1. **Name & keys** — product name + a unique key prefix; **remap every** storage/request/response
   key from the canonical set (`00_INDEX §6`); only `an`/`cid`/`sid` stay verbatim.
2. **Frontend** — reimplement fresh against `EXTENSION_COMPOSITION.md` in the build's own
   structure/names; run its §26 checklist (agent runs the offline items; the runtime smoke test is
   `[OWNER — manual]`).
3. **Identity `[GATE — owner confirms]`** — pick V1 / V2-A / V2-B (default **V2-B**,
   `USER_CREATION §11`); wire the client obligations (`EXT §6`). **This locks the reachable set —
   confirm with the owner before proceeding.**
4. **Backend** — copy the whole canonical set **wholesale** (**ask which build**), rename the two
   endpoint files (`BACKEND §2.1`), set the per-build variables (db config, ext-number, endpoint
   filenames, host/asset URLs, market list, a freshly-generated identity key), and remap key names
   to the frontend's posted keys. Keep canonical function names.
5. **DB provisioning `[OWNER — manual]`** — create the build's table from the canonical DDL
   (`DB_SCHEMA §9`) **before first sync**. A missing table fails **silently** as a degraded
   HTTP-200, not a visible error. Ask if the DB is ready; work from the snapshot. Apply any ALTER
   **before** uploading PHP that references new columns.
6. **Hosting bridge `[OWNER — manual]`** — host choice, initial upload, TLS, file permissions.
7. **Data** — point the pipeline at the shared curated sources; ship the static bundles (produced
   by the owner's automated pipeline).
8. **Distinctive choices** — monetization posture (owner toggle), popup-landing style
   `[VARIANT A/B]`, which optional features ship, and the per-build redirect/inject wire-format
   (`EXT §11.2`, §17).
9. **Packaging & Chrome Web Store submission `[OWNER — manual]`** — the final step by which the
   build reaches users; the agent prepares the package and lists what shipped, and **never uploads
   or submits**.
10. **Verify** — run each doc's conformance checklist, **filtered by the reachable-set rule**
    (`00_GOAL §3`); agent runs the offline items, the owner runs the runtime smoke test + the live
    end-to-end checks.

## Structure it like `update-extension`

Reuse: **write the target first** (a `CREATE_STATE.md` — the definition of done for this new
build), then **assess → plan (STOP for approval) → execute + validate one feature-batch at a
time**, with the mandatory per-phase gate (implement → `[AGENT — offline]` validate → hand the
owner a `[OWNER — manual]` checklist → **WAIT**). Add a prominent **GATE 0 = confirm the birth-time
identity + schema choice** (it's permanent). Keep the **stop conditions** and the
**write-lessons-back-to-the-docs** step. Sequence **monetization work last**, and never live-test
it (owner only).

## Owner facts to gather (ask, don't assume)

1. Which build's backend is the canonical copy source for this new build?
2. Is the target DB provisioned? Provide the schema snapshot (`docs/db_schema/*.sql`).
3. Confirm the complete per-build variable set (db config, ext-number, the two endpoint filenames,
   host/asset URLs, market list, freshly-generated identity key, remapped key names).
4. The new build's product name, key prefix, market list, identity variant, and launch
   monetization posture.

## Constraints

- **Docs/skill edits only.** Do NOT touch any live extension, backend, server, or DB.
- The agent **never** browser-tests or deploys — those are `[OWNER — manual]`, handed over as
  checklists it waits on.
- **Ground every claim in the actual files; flag what you infer vs. verify.**
- **Report first, get approval before writing the full skill.** Read the docs, draft the skill's
  outline + the create-lifecycle plan, and **stop at a plan gate for the owner's confirmation**
  before authoring the whole `SKILL.md` — exactly as `update-extension` does.

## Deliverable

`skills/create-extension/SKILL.md` (+ a `CREATE_STATE.template.md` if useful), matching the quality
and structure of `skills/update-extension/SKILL.md`.
