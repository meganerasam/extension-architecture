# Data Sources — the canonical specification

**Where the blocking data comes from: the upstream lists, how they are compiled
into what the extension ships statically versus what it fetches at runtime, and the
generation pipeline that keeps the fetched caches fresh.**

The data pipeline is part of the **backend template** (`00_INDEX.md` §1): the same
upstream sources and the same generators are reused for every build. A new build
copies the pipeline and points it at the shared repositories; only output paths /
key names are remapped.

> **The curated feeds are maintained externally by the owner.** The all-in-one blocklist, the
> compiled bulk rules, the generic/specific cosmetics, the scriptlets, and the all-in-one
> whitelist are **already curated upstream**; the backend **consumes them directly** and neither
> these docs nor `create-extension` generate or curate them. Treat curation as an owner-owned
> input, not a build step — the doc's scope starts where the backend reads the curated data.

> **Companion documents.** How the compiled rules are prioritized and applied:
> `RULES_AND_PRIORITIES.md` §3. How the caches are served and regenerated under
> load: `BACKEND_ARCHITECTURE.md` §4. How the client consumes each feed:
> `EXTENSION_COMPOSITION.md` §9–§12.

---

## 1. In plain English

The extension is only as good as its lists. There are four bodies of data —
**network blocklists, cosmetic filters, scriptlets, and surrogates** — and each
exists in **two forms**: a **static bundle** compiled into the extension at build
time (so it works offline from install), and, for the heavy network and cosmetic
data, a **server-generated cache** the client fetches and refreshes by version hash.

**Where the data ultimately comes from.** The ad-blocking data originates with
**uBlock Origin** — its filter lists (`uBlockOrigin/uAssets`) and its scriptlet +
redirect resources (`uBlockOrigin/uBlock`). Two processing paths flow from there:

- the **static bundle** is produced by **converting uBO into MV3 artifacts at build
  time** (§4.1);
- the **server caches** are mirrored from the shared **list factory**'s published
  `dist/` tree (repo `blocklist-v4` — itself fed by curated Google Sheets, the
  EasyList family, public hosts lists and fleet feedback; the whole system is
  documented in `BLOCKLIST_WHITELIST_SYSTEM.md`) on a 24 h server cadence (§4.2).

**A build must regenerate its static bundle from a recent, pinned uBO version — not
inherit a previous extension's frozen copy** (§4.1). Historically each new build
recycled the prior build's static files, so the whole family shipped uBO-as-of-years-
ago; regenerating from current uBO fixes that.

Everything is **data**, never code — the client compiles rules and CSS locally
(`EXTENSION_COMPOSITION.md` §22.1).

---

## 2. Upstream sources

Two origins. **uBO** is the root of the ad-blocking data and feeds the **static
bundle** (§4.1). The **list factory** — the private repo `meganerasam/blocklist-v4`
("Automated All In One List V2"), which replaced the four legacy `meganerasam/*`
repositories in 2026-09 — feeds the **server caches** (§4.2) through its published
`dist/` tree. The factory itself (its 15 curation sheets, the
curate→verify→compile pipeline, the precedence ladder, where to add a domain) is a
whole system with its own document: **`BLOCKLIST_WHITELIST_SYSTEM.md`** (visual
map: `la-fabrique-des-listes.html`). This section only lists what the backends
consume from it.

### 2.1 uBlock Origin (root of the static bundle)

| Source | Repo | Kind | Feeds |
| :-- | :-- | :-- | :-- |
| Filter lists (`filters/*.txt` + curated thirdparties) | `uBlockOrigin/uAssets` | ABP-syntax network + cosmetic + `##+js()` scriptlet-injection rules | the static DNR rulesets, cosmetic maps, and the scriptlet database |
| Scriptlet implementations (`scriptlets.js`) | `uBlockOrigin/uBlock` | the scriptlet **code** library | the MAIN-world runtime scriptlet library |
| Redirect / surrogate resources | `uBlockOrigin/uBlock` | inert stubs | the packaged surrogates + DNR redirect targets |

### 2.2 The list factory's `dist/` (server-cache source since 2026-09)

| Source (in `blocklist-v4` `dist/`) | Kind | Feeds |
| :-- | :-- | :-- |
| `network/rules.json` | the FINISHED merged DNR ruleset (navigation redirects + tracker blocks + surgical EasyList + allow lanes; ID bands, budgets and whitelist carve-outs already applied) | mirrored verbatim by each backend's `generate_compiled_rules.php` shim — only the redirect substitution string is rebranded; **never re-compiled** |
| `cosmetic/generic.css` | generic cosmetic hide selectors (EasyList family) | the dynamic generic cosmetic feed (as a selector **array**) |
| `cosmetic/specific.json` + `cosmetic/unhide.json` | per-domain hide + unhide selectors | the dynamic per-domain specific feed (`{domain:{hide,unhide}}`; per-brand merge stays backend-side) |
| `traffic_quality/global.json` + `{market}.json` (24 files incl. `latam`/`apac`/`nordics` rollups) | essential domains not to break, per market | the system whitelist, cached per market (24 h). ⚠ v3 named the baseline `general_global.json` — v4 renamed it `global.json`; every swap must follow |
| `whitelist/` · `blocklist/` · `standalone/` · `catalog/` | the served whitelist flavours, flat navigation lists, the K–O standalone sheets, and the à-la-carte per-source layer | on-demand consumers (backend sync, inspection, composition) — see `BLOCKLIST_WHITELIST_SYSTEM.md` §13 |
| IDCAC banner list *(not from the factory)* | cookie-consent selectors | the cookie-blocker hide CSS (`EXTENSION_COMPOSITION.md` §15) |

*Historical note:* the pre-2026-09 sources — `blocklist.csv` ("Block List v3"),
the working-domains long list (capped to its last ~50k slice), `merged-dnr/dnr.json`
— came from the four legacy repos and were compiled ON the server. The factory
absorbed all of them; the 50k cap was replaced by ledger-based retention
(`BLOCKLIST_WHITELIST_SYSTEM.md` §12); `short.php`/`long.php` remain public legacy
endpoints only until an access-log check clears their retirement.

---

## 3. Static bundle vs. fetched cache

| Data | Shipped **static** in the package | Fetched **versioned** at runtime | Delivered **inline** in the sync response |
| :-- | :-- | :-- | :-- |
| Network blocklist | the all-in-one rulesets (compiled to static JSON, enabled by default) | the compiled **bulk** blocklist (IDs ≥ boundary) | the small per-user inline rules |
| Cosmetic — generic | `generic.css` + `filters-generic.css` | the dynamic generic feed (by version) | — |
| Cosmetic — specific | the specific selector maps | the dynamic specific feed (by version) | — |
| Zero-flicker directive | — | — | the typed directive (`EXTENSION_COMPOSITION.md` §11.2) |
| Scriptlets | the preset DB + runtime **code** | **code** never fetched (MV3); the **details-data** has an *optional dynamic layer* (dormant in Ghost/Ninja — §4.2) | the scriptlet-exclude list |
| Surrogates | the web-accessible stub resources | — | — |
| System whitelist | — | — | the system whitelist (rewritten each sync) |
| Cookie hide list + CMP runtime | bundled (feature off by default) | — | — |

**Why the split.** The static bundle is the floor that blocks from the first
millisecond and survives a dead backend; the fetched caches let the heavy,
fast-moving lists update without an extension release; the inline keys are the
small, per-user, per-sync data. Scriptlets and surrogates are **only ever static**
— fetching them would be remote code.

---

## 4. Generation & regeneration pipeline

Two pipelines produce the two forms of data: the **static bundle** is regenerated
from uBO at build time (§4.1); the **server caches** are generated on the server at
a 24 h TTL (§4.2).

### 4.1 The static bundle — regenerate from the latest uBO

**Goal:** produce each build's static files fresh from a **recent, pinned uBO
version**, rather than copying a previous build's frozen set. This is the same
conversion uBlock Origin performs for its own MV3 product, uBlock Origin Lite — so
it is a solved pipeline, not a bespoke one.

> **Reference implementation:** `00 - Blocklist automation/Automated Static Bundle/`
> — a build-time Node tool (not part of the server) that does exactly this: fetches
> pinned uAssets, converts network filters → DNR via `@adguard/tsurlfilter`, parses
> `##+js()` into a scriptlet DB, pulls the scriptlet code library + surrogates from
> `@adguard/scriptlets`, parses cosmetics, and enforces the §4.1.1 budget — emitting
> a neutral bundle (`rulesets/`, `scriptlets/`, `cosmetic/`, `surrogates/`) each
> build reshapes. Tested against current uAssets; bump its `config.json` `ref` to
> refresh from a newer uBO.

**Inputs (pin a commit/tag of each):** `uBlockOrigin/uAssets` (filter lists),
`uBlockOrigin/uBlock` (scriptlet implementations + redirect/surrogate resources).

**Converter — Route A (mirror uBOL).** uBOL (`uBlockOrigin/uBOL-home`, GPL-3.0) is
uBO's own MV3 build; it converts uAssets into MV3 DNR rulesets + scriptlets +
declarative cosmetic data. It is the closest match to this family's data lineage.

**Can the build run in PHP?** The **conversion core (ABP filter syntax → DNR) is
JavaScript** — uBOL reuses uBlock Origin's own filter compiler; **do not reimplement
it in PHP** (it is large, intricate, and would drift from uBO immediately). But
Route A has a PHP-native path, because the conversion does not have to run at *your*
build time:

- **A1 — consume uBOL's *compiled* output (PHP-only, recommended).** uBOL publishes
  its already-built MV3 artifacts (the `chromium/` build: DNR ruleset JSON,
  scriptlets, declarative cosmetic/scriptlet data). Fetch those, then do **all**
  re-partitioning, renaming, merging with the `meganerasam` rules, and limit
  enforcement **in PHP** — no Node at your build time, because uBO already ran the
  conversion. Bonus: you inherit uBOL's MV3-safe ruleset partitioning (see the
  budget below). Trade-off: you take uBOL's list selection/format and reshape it.
- **A2 — run uBOL's converter, orchestrated by PHP.** A PHP generator `proc_open`s
  the Node converter (for control over *which* uAssets lists and conversion options),
  then post-processes the JSON in PHP. Needs Node available at build time.

Either way the post-processing — dedupe, re-split into differently-named rulesets,
rename scriptlet code, merge the `meganerasam` network rules, enforce the budget
below — is comfortably PHP. *(Route B alternative, if ever wanted: AdGuard's
`@adguard/tsurlfilter` CLI / `@adguard/dnr-rulesets` + `@adguard/scriptlets`, or
`@eyeo/abp2dnr` for network-only — all Node, AdGuard lists not uBO.)*

**What each uBO construct becomes:**

| uBO filter / resource | Static artifact |
| :-- | :-- |
| network filter (`\|\|domain^$…`) | a DNR block/allow rule → the all-in-one static rulesets |
| `$redirect` / `$redirect-rule` | a DNR redirect rule **+** the referenced surrogate copied in |
| generic cosmetic (`##selector`) | the generic hide layer |
| specific cosmetic (`domain##selector`, `#@#`) | the per-domain specific map |
| procedural cosmetic (`#?#`, `:has-text()`, `:upward()`…) | **not DNR/CSS** — handled by the runtime cosmetic/scriptlet layer |
| scriptlet injection (`domain##+js(name,args)`) | an entry in the domain→`[scriptlet,args]` database |
| scriptlet code (`scriptlets.js`) | the MAIN-world runtime scriptlet library |
| redirect resources | the packaged surrogates |

**The one hard coupling rule:** regenerate the **scriptlet code library and the
`##+js()` database from the *same* uBO version, together.** Filters reference
scriptlets by name; a fresh database against an old library produces `+js()` calls
to scriptlets that do not exist (and vice-versa, dead code).

**Known, accepted limitations (the converter reports them):**

- **Conversion is lossy.** Procedural cosmetics and some `$csp` / `$redirect-rule` /
  regex `$options` do not map to DNR — which is fine, because those are exactly what
  the extension's MAIN-world scriptlet/cosmetic layer handles at runtime
  (`EXTENSION_COMPOSITION.md` §10, §12). Split the output: **network → DNR**,
  **procedural + scriptlets → the injection layer**.
- **DNR count limits.** MV3 caps how many rules you can ship — design against the
  budget in §4.1.1 below. uBOL already partitions the static side to fit, which is
  another reason to prefer A1.
- **License.** uAssets/uBO are **GPL-3.0**; the AdGuard packages carry their own
  (GPL/LGPL) terms. Deriving from them is standard for filter lists but carries the
  usual attribution/source obligations — a fact to record, independent of any
  per-build restructuring.

#### 4.1.1 MV3 rule budget — design against these

Current Chrome `declarativeNetRequest` limits, and what each bucket holds here:

| Bucket | Limit | Holds (this architecture) |
| :-- | :-- | :-- |
| Enabled static rules | **30,000** guaranteed across enabled rulesets (more via `getAvailableStaticRuleCount()`, a shared global pool) | the all-in-one static blocklist |
| Static rulesets | **100** declared / **50** enabled | the ruleset files |
| Dynamic **safe** rules (block / allow / allowAllRequests / upgradeScheme) | **30,000** | compiled bulk **blocks** + user/system allows |
| Dynamic **unsafe** rules (redirect, modifyHeaders) | **5,000** | compiled `main_frame` redirects + monetization redirects + CSP-strip modifyHeaders |
| Session rules | **5,000** | the instant-feedback user rules (`EXTENSION_COMPOSITION.md` §8.3) |
| Regex rules | **1,000** per type (static and dynamic counted separately) | keep regex minimal; prefer `urlFilter` over `regexFilter` |
| Per regex rule | **< 2 KB** compiled | avoid pathological single rules |

Design implications:

- **Keep *enabled* static rules within 30,000.** Probe `getAvailableStaticRuleCount()`
  before enabling optional rulesets; ship cookies/YouTube rulesets **disabled** so
  they cost nothing until toggled (`EXTENSION_COMPOSITION.md` §8.4). uBOL already
  partitions for this — the A1 reason again.
- **Unsafe dynamic is the tight one.** Every `redirect` and `modifyHeaders` across
  **both** the inline server rules *and* the compiled bulk shares the single **5,000**
  cap — the `main_frame`→interstitial redirects plus the monetization
  redirects/CSP-strips must all fit. Count *redirects specifically* when sizing the
  compiled list; the factory asserts its own budgets before publishing (≤ 30,000
  total · ≤ 5,000 unsafe · ≤ 1,000 regex — `BLOCKLIST_WHITELIST_SYSTEM.md` §10.4),
  but the inline server rules ride on top and share the same client-side caps.
- **Blocks are "safe"**, so the bulk blocklist draws on the roomy 30,000 dynamic-safe
  pool, not the 5,000 unsafe one.
- The compiled bulk is applied as **dynamic** rules (IDs ≥ the boundary,
  `RULES_AND_PRIORITIES.md` §3.1); the all-in-one is **static**. The two draw on
  different budgets above — size them independently.

**Per-build differentiation happens *after* regeneration, and is structural only.**
Re-partition into differently-named rulesets and restyle the scriptlet code per the
anti-template rule (`00_INDEX.md` §1) — but the filter **data** is inherently shared
with every ad blocker (all derive from EasyList/uAssets). The gain from this
pipeline is **currency** (latest uBO), not making the rules unique; correlation risk
lives in frontend *code structure*, not filter contents.

*(Some manual curation is expected — choosing which uAssets lists to include,
resolving conversion warnings, and re-partitioning — and is fine.)*

### 4.2 The server caches

The server generators (`generate_compiled_rules.php`, `generate_cosmetic_rules.php`,
and the scriptlet-details generator) turn the factory `dist/` sources (§2.2) into the
fetched caches:

- **Compiled rules** — since the 2026-09 v4 shims: MIRROR the factory's finished
  `dist/network/rules.json` (the DNR merge with the production ID bands of
  `RULES_AND_PRIORITIES.md` §3.1 already applied), rebranding only the redirect
  substitution string; substitute `__EXT_ID__` per client at serve time. The old
  three-source merge (short/long/dnr) is retired — never re-compile the artifact.
- **Cosmetic caches** — parse `generic.css` into a selector array; merge
  `specific.json` with `unhide.json` (matching unhide selectors cancel a hide, the
  rest become unhide exceptions); emit as data maps, never stylesheets.
- **Scriptlet-details cache (optional dynamic layer)** — `generate_scriptlet_details.php`
  builds `scriptlet_details_cache.json` + `_version.txt`: the *dynamic* scriptlet-DB
  layer, the scriptlet counterpart to the compiled-DNR and dynamic-cosmetic tiers
  (**data** — domain→scriptlet mapping; the scriptlet *code* is always static, MV3).
  Currently **dormant** in Ghost/Ninja (they ship scriptlets fully static and their
  endpoints emit no scriptlet version/url); the generator is part of the set and the
  layer re-enables by emitting the version/url (`BACKEND_ARCHITECTURE.md` §2.2).

**Operational guarantees** (detailed in `BACKEND_ARCHITECTURE.md` §4–§5): a **24 h
TTL** under a **single-flight lock**, **bounded** upstream fetches, a **version =
`md5`** stamp per cache, and **fail-safe** generation — an upstream failure keeps
the last-good cache and does not bump the version, so the fleet never re-downloads
an empty set.

---

## 5. Per-extension implementation

All builds draw from the same sources and pipeline; they differ only in which
cosmetic-format generation they run and whether surrogates actually ship. **Legend:**
✅ · ⚠️ · ❌ · ⬜ to copy.

| Data | § | 12 Pro | 21 North | 22 Hunter | 23 Wonder | 25 Ninja | 26 Ghost | 27 |
| :-- | :-- | :-- | :-- | :-- | :-- | :-- | :-- | :-- |
| All-in-one static blocklist | 3 | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| Compiled bulk (versioned) | 4 | ⚠️¹ | ✅ | ✅ | ✅ | ✅ | ✅ | ⬜ |
| Cosmetic generic + specific (data) | 4 | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| Zero-flicker directive as structured data | EXT §11.2 | ✅ | ✅ | ✅ | ⚠️ raw string | ✅ | ✅ | ✅ |
| Static scriptlet DB (no fetch) | 3 | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| Surrogates actually shipped | 3 | ✅ | ⚠️ server-hosted | ✅ | ❌ | ❌ | ✅ | ✅ |
| IDCAC cookie list | 2 | ❌ | ❌ | ❌ | ⚠️ realm | ❌ | ✅ | ✅ |
| YouTube static ruleset (ships disabled) | 4.1.1 | ✅ | ❓ | ❓ | ❓ | ❓ | ❓ | ⬜ |

**Notes.** Historically each build recycled a **frozen uBO snapshot** from the prior build's
static files (tracing back years), so scriptlets/cosmetics went stale. That is now **superseded
by the automated static-bundle creation** pipeline (§4.1) — used for 12 — which regenerates the
bundle from the latest uBO. Regenerating is a non-identity change, safe to apply on an update
(`00_INDEX.md` §3); a build not yet re-generated still carries the old frozen snapshot. Wonder's
generic pre-hide also ships as a server-built raw CSS string rather than a
compiled-locally directive (`EXTENSION_COMPOSITION.md` §11.2 note); Wonder and Ninja
reference surrogate redirect targets that are not packaged, so those redirects
degrade to hard blocks (`RULES_AND_PRIORITIES.md` §2) — fix on any port.
12 additionally ships a fourth static ruleset — **YouTube** (8 block rules, declared `"enabled": false`, toggled at runtime by its YouTube mode) — the disabled-until-toggled pattern of §4.1.1; the sibling ❓ cells in that row are unaudited.

¹ **12's compiled-bulk client wiring is conformant (version-gated lazy fetch,
applied direct-to-engine), but its upstream feed was found broken** during the
2026-08 pass — the source sheet the cache is generated from had been deleted, so the
feed URL returned an HTML "not found" page instead of a rule array, and the
inline single-flight cache had never been generated on the host. The client fails
safe (keeps last-good / applies nothing), so it *looks* fine on the surface. This is
exactly why §6 now carries an explicit **feed-health** item: a broken or ungenerated
feed is invisible from the extension alone.

---

## 6. Conformance checklist

> **All Common — no V1/V2 split.** The data layer (blocklists, scriptlets,
> surrogates, cosmetics, zero-flicker directives) is **generation-agnostic**: it is
> byte-identical whether the build is V1 or V2, and every item below applies to
> every build. The identity generation changes the *backend* and *schema*, never the
> filter/asset data. (The fraud stack's own inputs are V2-only and live in
> `FRAUD_DETECTION_V5.md`, not here.)

The data layer is conformant when:

- [ ] The static bundle is **regenerated from a pinned recent uBO version** (uAssets + uBlock), not inherited from a previous build — §4.1.
- [ ] The scriptlet **code library and the `##+js()` database come from the same uBO version** — §4.1.
- [ ] The rules fit the **MV3 budget**: enabled static ≤ 30k, dynamic **unsafe** (redirect + modifyHeaders across inline *and* compiled) ≤ 5k, dynamic safe ≤ 30k, regex minimal — §4.1.1.
- [ ] The all-in-one blocklist ships **static and enabled by default**; the floor works offline — §3.
- [ ] The compiled bulk blocklist is **fetched by version hash**, applied direct-to-engine, never `script`-gated — §3, §4.2.
- [ ] Generic + specific cosmetics travel as **data** (selector arrays / maps), compiled locally — never a remote stylesheet — §3, §4.
- [ ] Scriptlet **code** and surrogates are **bundled static** — never fetched; the scriptlet **details-data** may run as an optional dynamic layer (dormant by default) — §3, §4.2.
- [ ] Every referenced surrogate/redirect target is actually **packaged and web-accessible** — §4.1, `RULES_AND_PRIORITIES.md` §2.
- [ ] Server-cache generation is single-flight, bounded, versioned by `md5`, and fail-safe (last-good on upstream failure) — §4.2.
- [ ] **Feed health** — `[OWNER — manual]` **(verify live, not from the extension):** every feed URL the client fetches — compiled rules, generic + specific cosmetics — returns **real data** (a JSON rule array / selector map), **not** an empty set, an HTML error page, or a 404; the inline cache **self-generates on first request** (single-flight), so a fresh host is not serving an empty feed. Because the client fails safe, a broken upstream or an ungenerated cache is **invisible from the extension** — check the URLs directly — §4.2, §5 note ¹.
- [ ] The long auto-collected list's cap is a **documented** choice (older entries age out) — §2.
- [ ] Output paths / key names are remapped per build; the upstream sources are shared — §1.

---

*Data-sources reference. The static bundle is regenerated from the latest uBlock
Origin (§4.1); the server caches are mirrored from the list factory's `dist/`
(§4.2 — the factory itself: `BLOCKLIST_WHITELIST_SYSTEM.md`). Rule priorities →
`RULES_AND_PRIORITIES.md`; cache serving → `BACKEND_ARCHITECTURE.md`; client
consumption → `EXTENSION_COMPOSITION.md`. Start from `00_INDEX.md`.*
