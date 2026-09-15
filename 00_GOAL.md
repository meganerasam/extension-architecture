# Architecture — the goal (charter)

**The one place that says what every build in this family is *for*, what "done" means,
and why two builds can look nothing alike in code yet be the same product underneath.**
This is the keystone: `00_INDEX.md` maps the documents; this states the target they all
serve. Every specification's per-extension table and conformance checklist is measuring a
build against what is written here. When a downstream doc and this one disagree about the
*goal*, this one is authoritative; when they disagree about a *fact*, the code wins and both
docs are wrong (fix them).

---

## 1. The goals — two, co-equal

Every build pursues **two goals at once**, and neither is subordinate to the other:

1. **A genuine, MV3-clean ad blocker.** Real DNR rule tiers, four-layer cosmetics,
   scriptlets/surrogates, structured zero-flicker. This is the product a user installs and the
   reason the build passes store review and earns a user base.
2. **A search-monetization path.** On an approved-market search, the server enables the build
   (`$script`), which injects the partner/AFS–YHS chain, redirects the search to the partner
   feed, and holds a **zero-flicker mask** over the native results until the redirect takes over;
   a server **allow band** keeps that traffic alive. "**Monetization must not break**" is a
   standing invariant, re-checked at every phase of any change — not only at the end.

A third, **supporting** layer serves both goals: a **self-protecting identity** — a stable
handle that survives reinstall (dedup + sync-mirror heal) so the user base isn't lost, plus, on
the generations that carry it, a fraud/telemetry stack whose job is to keep the monetized traffic
clean enough to stay paid. This layer is *in service of* the two goals, not a third goal.

> The two goals pull in tension — a blocker that also injects/redirects on search surfaces — and
> the whole architecture (the injection carve-out, the priority ladder, the protected-surface
> rules) exists to hold both true at once. A change that advances one goal by breaking the other
> is a regression, not progress.

## 2. The convergence rule — behavior converges, shape diverges

The **target** for any build is the **behavior of the latest *shipped* baseline** (today the
newest V2-B builds). "Modernized" means: *behaves like the latest baseline; is written like
nothing else in the family.*

- **Converge the behavior and logic** — the proven builds are the reference for *what the code
  does*; do not reinvent an algorithm and reintroduce its solved bugs.
- **Diverge the shape** — file tree, module boundaries, names, coding style, and every key name
  differ per build. The store flags near-identical extensions as related/plagiarized; structural
  sameness is a correlation risk (`00_INDEX.md` §1, `EXTENSION_COMPOSITION.md` §0.2).

The backend is the mirror image: **copied wholesale, keys remapped** — sameness there is safe and
desirable because store review never sees it (`00_INDEX.md` §1, §6).

## 3. What a build can actually reach — the reachable-set filter (**M2**)

The latest baseline is the **ceiling**. What a given build can *reach* is that ceiling **filtered
through the build's frozen substrate** — its identity generation and its DB schema, both fixed at
birth. For every baseline area, classify it into one of three bins:

- **Client-only** — touches no DB column (zero-flicker, cosmetics, priority ladder, scriptlets,
  MV3 hygiene, the client side of monetization). **Every build can reach it.** Converge to the
  baseline. *(This is the bulk of a legacy/older-build update.)*
- **Substrate-shaped** — reads or writes a DB column, or leans on the identity generation (visit
  recording, whitelist/block sync, acceptable-ads opt-out state, the fraud stack). **Reachable
  only if the build's existing schema already carries the column(s)**, and the data may have to be
  reshaped to what those columns allow. The right reference is the **sibling whose substrate
  matches** — not automatically the newest build.
- **Unreachable** — the frozen schema has no column for it. **Permanent `n/a`, not a gap.**

> **Decision B — the schema is frozen at birth (every generation).** Not just the identity/dedup
> columns: the **whole schema** is fixed at a build's creation. During an **update/modernization**
> you never add a feature column to chase a baseline behavior — a behavior needing an absent column
> is permanent `n/a`. Growing the schema is a **birth-time** act and belongs to **create-extension
> only** (§5). This is stronger than, and supersedes, the older "only identity columns are frozen"
> framing anywhere else in the folder.

**The gap test, made concrete (used by every conformance checklist):** for each area, ask *does
this build's existing schema carry the column(s) this behavior needs?* — **No ⇒ permanent `n/a`
(stop); Yes ⇒ closable, reshape to this build's own columns.**

**How the checklists apply the filter.** The backend specs' checklists are bucketed by generation;
run the bucket for the build's generation **and** the M2 filter over each item. Note that
`EXTENSION_COMPOSITION.md` §26 is bucketed **by topic** (not by generation) with inline `[V2 only]`
tags — for it, apply M2 per item rather than looking for a generation bucket.

**The concrete split — what every build shares vs. what's per-product.** The reachable-set model,
stated as the two lists every conformance checklist measures against:

- **Universal — every build must do this** *(identity-agnostic; reachable on every generation
  including Gen 0)*: the static blocking floor (static rules + uBO scriptlets + cosmetics that work
  with the backend down), the all-in-one static blocklist + version-gated compiled bulk, four-layer
  cosmetics, handling **both** static and dynamic feeds, zero-flicker (both halves), the system
  whitelist, own-domain self-allow at top priority, **no** monetization domains in the static files,
  **the monetization path itself** (`$script` → inject / redirect / mask / allow-band — same goal
  and mechanism on every generation; the `script` column exists in every schema), local-first
  storage, version-purge preserving identity + prefs, fail-safe on backend error, action-driven
  sync, client-side visit `{count, last-time}`, identity survival, MV3 cleanliness, phishing
  removed, no sibling/monetization/injection mentions in shipped code, and the standing requirement
  that each build have its **own** code shape / names / keys.
- **Per-product, kind 1 — free choices** *(any build can pick; not gated by generation)*: the
  identity variant chosen at birth, the **monetization posture** (none / dormant / live),
  popup-landing style (A/B), whether the element picker / cookie-blocker / YouTube skipper ship, the
  concrete code shape (file tree, names, keys, redirect/inject wire-format & style), and which
  optional features ship. This is the `00_INDEX.md` §5 "what differs per build" set.
- **Per-product, kind 2 — capability-gated by the identity generation** *(decision B; reachable only
  if the frozen schema carries the column)*: hardware fingerprint collection (`audio_fp` …) and the
  fraud/behaviour stack (**V2 only**), the durable acceptable-ads `script=2` opt-out (**V2 only** —
  needs a server-authoritative column), server-side persistence of visit **counts** (**V1/V2**, not
  Gen 0), and server-synced element-picker selectors `cosmeticDom` (**build 26+**). On a build whose
  schema lacks the column these are **permanent `n/a`**, never added.

> **Monetization is universal — it is in the first list, not generation-gated.** The generation
> gates only the identity mechanism and the kind-2 fraud/hardware capabilities. A Gen 0 or V1 build
> runs every universal item — **its monetization included** — and skips only the kind-2 items its
> schema cannot hold. A conformance item that reads as "monetization off / enable-gate inert" for a
> given generation is describing that generation's *enable mechanism*, never a lesser monetization
> goal.

## 4. The capability levels — what each generation is *allowed* to reach

A build's generation is permanent (never migrated — migration orphans the user base). It sets the
reachable ceiling for the substrate-shaped areas:

| Generation | Identity | Schema substrate | Fraud/telemetry | Reachable ceiling |
| :-- | :-- | :-- | :-- | :-- |
| **Gen 0 (Legacy, pre-family)** — 12 | AES `syncuid` handle | pre-family **ad-counter** schema (`conv 0–2`, no family user-data group, no `fingerprint`) | **none** — network-risk only | full frontend baseline (client-only); **not** the DB-backed family behaviors or any fraud — permanent `n/a` |
| **V1 (Gen 1)** — 21/22/23 | encrypted-UID handle | V1 family schema (`visit`, `whitelistedDom`, `blockedDom`, `instdom`, …; `conv 0–3`) | **none** — network-risk only | frontend baseline **+** the DB-backed family behaviors its schema carries; **not** the fraud/fingerprint stack — permanent `n/a` |
| **V2-A (Gen 2)** — 25 | HMAC `fingerprint` (`idx_fingerprint`), `device_sig` absent | V1 schema **+** hardware/telemetry columns, `device_hash`, fraud/behaviour cols, `conv '4'` | full network + hardware + behaviour stack | full baseline incl. fraud |
| **V2-B (Gen 3)** — 26/27 (**default for new builds**) | minted `fingerprint` token (`uq_fingerprint`), `device_sig` present | as V2-A | full stack | full baseline incl. fraud |

Detail for each lives in `USER_CREATION_AND_UPDATE.md` (§3, §11, §13), `DB_SCHEMA.md` (§7, §9),
and `FRAUD_DETECTION_V5.md` (§18). Recognizing a Gen 0 build (it passes every V1 test but has the
ad-counter schema) is in the `update-extension` skill §4.

## 5. Birth-time only — create-extension territory

Choosing the identity variant and the schema happens **once, at a build's birth** (create-extension;
default identity **V2-B**). That choice **locks the reachable set forever** (§3, decision B), so it
is the highest-stakes decision in creating a build.

The following belong **only** to birth-time / create-extension and must **never** appear in an
update/modernization run:

- Creating the user table (provisioning it from the canonical DDL, `DB_SCHEMA.md` §9).
- Adding columns — including "new columns behind a capability probe" — and any DB migration that
  changes the column set.
- Picking the identity generation, the handle key, and the per-build variable set.

*(These lines were removed from the `update-extension` skill's execute/deploy steps and live here
because they are birth-time acts, not modernization steps.)*

## 6. Proven vs. unproven — honesty about state

The frontend modernization loop has been exercised (the 2026-08 pass on build 12). The **full
end-to-end build/update procedure and the backend-copy path have not yet been run start-to-finish**
in practice — treat them as designed-but-unproven until a real run validates them. Say so rather
than presenting them as battle-tested; the first real create-extension run is expected to surface
gaps this charter can't predict.

---

*Charter / keystone. Unlike the seven specification documents, this file has no per-extension table
and no conformance checklist — it states the goal those tables and checklists measure against. Read
it first (`00_INDEX.md` §3), then the documents `00_INDEX.md` maps.*
