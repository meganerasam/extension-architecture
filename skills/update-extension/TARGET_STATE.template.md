# Target State — `<build>` (`<Gen 0 Legacy | V1 | V2-A | V2-B>`)

> The written picture of "what this build looks like when modernized." Drafted by
> `update-extension` PHASE T from the baseline + this build's per-extension rows + the
> sibling behaviors, then **confirmed/edited by the owner**. It is the yardstick for
> gap assessment (PHASE A) and the acceptance test at the end. Copy this template to
> `<project>/TARGET_STATE.md` and fill it in. Update it as the build converges.

- **Build:** `<n - Name>`  ·  **Codename:** `<...>`
- **Identity / generation (FROZEN):** `<variant>` — `<handle key>` on table `<table>`
- **Backend endpoint:** `<file.php>`  ·  **Monetization posture:** `<none | dormant | live>`
- **Drafted:** `<YYYY-MM-DD>`  ·  **Confirmed by owner:** `<yes/no + date>`

---

## 1. Frozen — identity, DB, dedup (NEVER touch)

*These are permanent by lineage; changing any orphans the user base. They are not
gaps.*

- Identity handle: `<...>`  ·  resolution/dedup: `<...>`
- Schema lineage: `<...>` (columns/indexes that define the variant)
- Fraud/telemetry rows that are `—` for this generation: `<...>`

## 2. Must-NOT-break invariants (checked every phase)

*The load-bearing behaviors. Any change is validated against these before advancing.*

- **Monetization path:** `<the exact chain — inject / redirect / mask / allow band>`
- `<other load-bearing behavior>`

## 3. Sibling behaviors to match (copy the LOGIC, diverge the CODE)

*What the finished build must behave like — matched to the proven siblings, in this
build's own structure/names.*

- [ ] `<behavior>` — like `<sibling>` (§ ref)
- [ ] ...

## 4. Definition of done — target per area

> **Seed this table from the FULL conformance surface — one row per baseline area — so
> nothing is silently dropped.** Do not hand-pick rows from memory (that is exactly how
> a build misses an item that mattered for it but not for the last one — e.g.
> the `recordVisit` timestamp for a build whose schema carries `visit`, or the acceptable-ads opt-out for a V2 build).
> Walk **`EXTENSION_COMPOSITION.md` §26** and every per-doc checklist
> (`RULES §9`, `DATA §6`, `DB_SCHEMA §8`, `USER_CREATION §12`, `FRAUD §19`,
> `BACKEND §10`) and put **every** area here, marking `n/a` where the area does not
> apply to this build/variant rather than omitting it.

*Each line is `current → target`; `n/a` = not applicable to this build (say why).*

| Area | § | Current | Target | Done? |
| :-- | :-- | :-- | :-- | :-: |
| Rule-ID / priority model | RULES §4, §8 | | | ☐ |
| User-tier priority (above blocks, below injection) | RULES §4.2 | | ~200/100 | ☐ |
| Local-first storage + purge/preserved set | EXT §3–5 | | | ☐ |
| Zero-flicker (2-layer, direct-read, structured **in this build's own format**) | EXT §11 | | | ☐ |
| Flicker not gated on injection allowlist | EXT §11.2 | | | ☐ |
| Cosmetics (four-layer) | EXT §10 | | | ☐ |
| Scriptlets static + CSP fallback; surrogates ship | EXT §12 | | | ☐ |
| Compiled bulk version-gated + **feed health** | DATA §4–6 | | | ☐ |
| Sync cadence (action-driven) | EXT §19 | | | ☐ |
| **Visit recording `{count, last-time}`** (cap ~50, counts-only wire) | EXT §18 | | | ☐ |
| Whitelist == paused (every path); removal-delta tracking | EXT §13 | | | ☐ |
| Static rulesets clean of injection-surface rules | RULES §9.1 | | | ☐ |
| Permissions minimal (drop unused) | EXT §2 | | | ☐ |
| No sibling/build-number/scheme leaks in shipped code | EXT §0.2 | | | ☐ |
| **MV3 clean — no remote code AND no external `<link>`/`<script src>`/`@import`/`url(http)`** | EXT §22.1 | | | ☐ |
| Degraded HTTP-200 survivable | EXT §23 | | | ☐ |
| YouTube skipper `[TOGGLEABLE off]` | EXT §14 | | | ☐ |
| Cookie-consent `[TOGGLEABLE off]` | EXT §15 | | | ☐ |
| Element picker `[OPTIONAL]` | EXT §16 | | | ☐ |
| Popup landing `[VARIANT A/B]` | EXT §17 | | | ☐ |
| **Acceptable-ads opt-out (state 2) reset** `[V2 only]` | EXT §24.2 | | | ☐ |
| Onboarding not reopened on update | EXT §20.2 | | | ☐ |
| *(add any per-doc checklist item not listed above — do not omit)* | | | | ☐ |

## 5. Explicitly OUT of scope (permanent gaps — do not close)

- `<the identity-coupled cells for this generation>`

## 6. Known deltas / notes

- `<anything the owner knows that the docs don't — business intent, quirks, history>`
