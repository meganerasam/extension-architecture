# Backend Architecture — the canonical specification

**How the server handles a sync: the endpoints, the order in which a response is
assembled, how the heavy caches are generated, how the whole thing degrades under
outage, and how maintenance runs without a cron.**

Unlike the frontend, the backend is a **template to copy** (`00_INDEX.md` §1). This
document describes the one canonical server — the reference build is Ninja
(`ninja-adb21.php`), ported to Ghost (`ext-server.php`). A new build copies it
wholesale, keeps the function names, and remaps only the key names (`00_INDEX.md`
§6). References cite the reference code by `file:line` on purpose.

> **Companion documents.** The rules a response carries: `RULES_AND_PRIORITIES.md`.
> Identity resolution and the write surface: `USER_CREATION_AND_UPDATE.md`. The
> fraud/VM/behaviour layers: `FRAUD_DETECTION_V5.md`. The DB it reads/writes:
> `DB_SCHEMA.md`. Where the cache content comes from: `DATA_SOURCES.md`.

---

## 1. In plain English

The backend is **one POST endpoint plus a few static-ish asset URLs**. Every few
hours each client POSTs its identity and a little telemetry; the endpoint resolves
who they are, decides what they should get, and returns a JSON bundle of *data*.
The heavy blocklists and cosmetic filters do not travel in that response — they are
pre-generated cache files the client fetches from their own URLs only when a
version hash changes.

Three properties define a correct backend:

1. **It never fails hard.** A database outage returns `200` with the client's
   identity echoed and a jittered back-off, never a `500` — because a `500` turns
   into a fleet-wide retry storm.
2. **It never wipes a client.** The response is merged key-by-key; an existing
   user's per-user data is *omitted* on a degraded path, not overwritten.
3. **It maintains itself without a cron.** Money-path finalization and cache
   regeneration run **after** the response is flushed, on borrowed worker time,
   driven by inbound traffic.

Not every build has all of this. The backend ships in **capability generations
gated by the identity variant** (§8): a V1 build is network-only, while the fraud /
behaviour / finalization stack requires V2 — which is why some builds have it and
some do not. A build's generation is **fixed at creation and never migrated**: an
update must not touch an existing build's identity or uniqueness keys, or it
orphans every user already in the database (§8.3).

---

## 2. Backend files & endpoints

The backend is a **fixed set of PHP files copied wholesale between builds**
(`00_INDEX.md` §1). Copy the whole set; **exactly two files get a new name per
build** (§2.1); a handful of **variables** change (§2.3); everything else is
byte-identical. Use this section as the "did I copy everything / what do I rename"
checklist — a missing dependency here is a silent runtime failure.

> **Wholesale copy is the *create-a-new-build* operation.** When **creating** a new build you
> copy this entire set fresh — and **always ask which build to copy from; never assume** a
> specific one. When **updating an existing** build, its backend already exists — you do **not**
> re-copy it; you **augment** it (add only what's missing, matching the canonical function names
> and structure as closely as possible), and you **never assume the DB — ask the owner whether
> it's ready and read the actual schema snapshot** (`DB_SCHEMA.md` §7; update-extension GATE 0).
> When an update ships a schema migration alongside PHP that references the new columns, **apply
> the ALTER before uploading the PHP** — the canonical 2026-09 `conv_date`/`updated_at` migration
> is the standing example (its PHP reads `conv_date`; `DB_SCHEMA.md`).

### 2.1 The two files renamed per build

Only these two carry a product-specific filename. Every other file keeps its name.

| Role | Method | 12 Pro | North | Hunter | Wonder | Ninja | Ghost | 27 |
| :-- | :-- | :-- | :-- | :-- | :-- | :-- | :-- | :-- |
| **Sync endpoint** (the heartbeat — identity, risk/fraud, response assembly) | POST | `adbpro-202609.php` | `adbnorth_18.php` | `adshunter_2.php` | `wonder58.php` | `ninja-adb21.php` | `ext-server.php` | ⬜² |
| **Uninstall endpoint** (stamps churn `disabled`, redirects to feedback form, never fatal) | GET | ⬜¹ | `uninstall-adbnorth.php` | `ciao.php` | `goodbye.php` | `farewell.php` | `ext-uninstall.php` | ⬜² |

¹ **`12` (Gen 0)** — its post-uninstall redirect is handled outside this family rename
convention (owner-managed) and is **not pinned here** (`⬜`). Its sync endpoint is the known
`adbpro-202609.php` (build map).
² **`27`** is a **new build to create** (create-extension), not an update — its two endpoint
filenames do not exist yet, so `⬜` = to create.

The client hard-codes both names (the sync URL it POSTs to; the uninstall URL it
registers), so the frontend and these two filenames must agree. The sync endpoint
validates `extid` against `^[a-z]{32}$`.

### 2.2 Fixed-name dependencies (copy verbatim)

Same filename in every build. **Gen** marks which generation ships the file
(`00_INDEX.md` §5; generations in §8 below): **all** = every build; **V2** = Gen 2/3 (fraud stack) only —
a V1 build omits the fraud/behaviour files entirely.

| File | Role | HTTP? | Gen |
| :-- | :-- | :-- | :-- |
| `config.php` | DB creds + all secrets + table name + flags (§2.3). The one file every other includes. | — | all |
| `compiled_rules.php` | Serves the compiled bulk blocklist; per-client `__EXT_ID__` substitution; `ETag`/`304`. | GET `?extid=` | all |
| `generate_compiled_rules.php` | **Since the 2026-09 v4 shims:** a thin MIRROR of the list factory's finished `dist/network/rules.json` (rebrand of the redirect target + last-good fallback + hard abort — it no longer compiles anything; `BLOCKLIST_WHITELIST_SYSTEM.md` §17.2). Each brand needs its OWN substitution string with the target page WAR-listed. | — | all |
| `generate_cosmetic_rules.php` | Builds the cosmetic caches (`cosmetic_css_cache.*`, `cosmetic_specific_cache.*`) served as static files — its 3 source URLs point at the factory's `dist/cosmetic/` since the v4 shims; the per-brand merge stays backend-side. | — | all |
| `generate_scriptlet_details.php` | Builds the **dynamic scriptlet-details cache** (`scriptlet_details_cache.json` + `_version.txt`) — the scriptlet analogue of the compiled-DNR and dynamic-cosmetic layers (this is *data* — which domain gets which scriptlet; the scriptlet **code** is always static, MV3). **Posture is per-build:** Ghost & Ninja currently ship scriptlets fully static and their endpoint emits **no** scriptlet version/url, so the layer is **dormant** there — but the generator is part of the set and the layer re-enables by emitting the version/url, exactly like the DNR/cosmetic dynamic tiers. | — | all |
| `short.php` | **Legacy** popup/malicious blocklist source (curated list → `main_frame` redirects) — superseded by the factory artifact; stays a PUBLIC endpoint until an access-log check proves no external callers, then retire. | — | all |
| `long.php` | **Legacy** auto-collected ad/tracker source (capped slice → block + redirects) — same retirement path as `short.php`; its role is absorbed by the factory (whose ledger retention replaced the 50k cap). | — | all |
| `finalize_pending_installs.php` | Stage-1 payout finalization + decision/expiry sweeps; drained post-flush by the scheduler (§6). | (self-run) | V2 |
| `behavior_lib.php` | Behaviour layer: feature extraction, `runBehaviorLayer`, the enforcement flags. | — | V2 |
| `classifier_lib.php` | Classifier support library. | — | V2 |
| `classifier_v4.php` | The v4 behaviour classifier (score → `decision`). | — | V2 |
| `generate_fraud_report.php` | Daily offline audit → `fraud_cache.json` + `fraud_report.json`. **`conv` is an ENUM — every SQL literal touching it must be a quoted string** (an unquoted number binds by enum *index*: `conv = 2` matched `'1'` = paid); canonical since the 2026-09 Ghost fix, which corrected eight unquoted comparisons in this file. Because the pre-fix stats were wrong, every conv-derived number in both output files **step-changes on the first corrected run** — watch `calibrated_risk_threshold` (the one report output the live request path consumes; clamped 25–50, fallback 33) after that run. Details: `FRAUD_DETECTION_V5.md`. | — | V2 |
| `generate_funnel_cache.php` | Funnel/analytics cache. | — | V2 |
| `dashboard_detection.php` | Detection dashboard (ops view). | GET | V2 |
| `dry_run_classifier.php`, `validate_behavior.php`, `validate_outcomes.php`, `_index_diagnostics.php`, `_test_identity.php` | Diagnostics / validation tools — not on the request path, but part of the set. | (CLI/GET) | V2 |

Fraud internals for the V2 files: `FRAUD_DETECTION_V5.md`. Cache generation detail:
§4 + `DATA_SOURCES.md`.

### 2.3 Per-build variables (change these, nothing else)

Everything product-specific lives in a few places — mostly `config.php`:

- **`config.php`:** DB name / user / password; **table name** (`adbnorth21` … `ghost26`, `DB_SCHEMA.md`); the **identity key** (V1 AES / V2-A HMAC — **freshly generated per build, never shared**, since a shared key lets one build decrypt/derive another's handles; V2-B needs no issuance key, its handle is a random token, §7); proxycheck key; payout-provider token; **product number**; approved-market list; approved store ext-id prefix; the backend host (`BG_HOST`); asset base URL; the search-monetization type token (`…RD<product-number>`) and injection hosts.
- **The sync endpoint file:** its own filename (§2.1); the **uninstall filename** it registers on the client; the compiled-rules URL + cosmetic-cache filenames it points the client at; the product number baked into `typetag`.
- **The uninstall endpoint file:** its own filename (§2.1); for **V1**, the identity key **must match** the sync endpoint's (so it can decrypt the handle); the feedback-form URL.

### 2.4 Generated at runtime (do not copy — they rebuild)

`compiled_rules_cache.json`, `cosmetic_css_cache.*`, `cosmetic_specific_cache.*`,
the `*_version.txt` md5 stamps, `fraud_cache.json`, `fraud_report.json`,
`allowlist_cache_<market>.json`, and the finalize/run stamp files. These are outputs
of the generators (§4); a fresh build starts empty and regenerates them.

### 2.5 Do NOT copy

The `-dev` / legacy twins (`ninja-adb21-dev.php`, `wonder58-dev.php`, `ninja-adb.php`,
North's `adbnorth.php`/`adbnorth-dev.php`) and one-off rescue/migration scripts
(`rerun-vm.php`, `rescue_conv_*.php`, `sanitize_fast_uninstalls.php`, `replay_v4.php`,
host-specific `config-hostinger.php`, `_dbcheck.php`, `_porttest.php`). The `-dev`
twins force `script=1`/`level=0` and are **not** what store builds call (`00_INDEX.md`
per-doc notes).

---

## 3. Request → response, in order

A sync request is handled in a fixed sequence (`ninja-adb21.php` `processUserLogic`).
**This is the full Gen 2/3 (V2) flow; a V1 (Gen 1) build runs the subset** — the
`[V2]`-tagged parts below collapse to network-risk-only, with no hardware, VM/tier,
behaviour, or `WAIT` (§8). The universal spine is: parse → resolve identity → risk →
update row → assemble → defer maintenance.

1. **Parse** the JSON body — identity handle, telemetry (visited map, install
   sites), user allow/block lists, acceptable-ads flag, and — **`[V2]` first-run /
   migration only** — the raw hardware signals (a V1 payload carries none).
2. **Resolve identity** — `USER_CREATION_AND_UPDATE.md` §5/§6. All V2 paths end
   `EXISTING` / `CREATE` / `WAIT` / `ABORT`; the cascade differs by variant:
   **`[V2-A]`** (the `ninja-adb21.php` reference) token echo → **derived-token
   recompute** → cid+sid → IP+`an`+instdom → create; **no device-signature tier** (`USER_CREATION_AND_UPDATE.md` §6.3).
   **`[V2-B]`** (Ghost) token echo → attribution (cid-guarded) → **`device_sig`**
   single-candidate → create (`USER_CREATION_AND_UPDATE.md` §6.4).
   **`[V1]`** decrypt the handle to identify the row (else dedup by cid/sid, then
   IP+`an`+instdom); **no `WAIT`, no device signature** — a first request always
   resolves to `EXISTING` or `CREATE`.
3. **Evaluate risk** — **network risk → `level`** (all builds, `FRAUD_DETECTION_V5.md`
   §3). **`[V2]` additionally:** hardware → VM/tier, and behaviour is recorded (not
   decided here). **`[V1]`:** network risk only — no VM, tier, or behaviour layer.
4. **Update the row** — `updates++`, write `level`/`script`, user lists, timestamps
   (`USER_CREATION_AND_UPDATE.md` §7). **`[V2]`** merges `visit` non-destructively
   (cumulative history). **`[V1]`** overwrites `visit` wholesale (snapshot, no
   accumulation) and touches no hardware/fraud/behaviour columns — it has none.
   The canonical `conv_date` / `updated_at` columns (shipped in Ghost 2026-09, now
   family spec — `DB_SCHEMA.md`) cost the endpoint **zero code**: the insert stamps
   `conv_date` via its column DEFAULT (same statement as `created_date`, so the two
   start equal) and the DB maintains `updated_at` on any value-changing UPDATE;
   only code that **writes `conv`** maintains `conv_date` (`FRAUD_DETECTION_V5.md`).
5. **Assemble the response** (`RULES_AND_PRIORITIES.md` §7 step 3): base inline rules →
   **if `script == 1`** the injection bundle, **if `script == 2`** the explicit
   reset payload → unconditional default block + chunked system allowlist → the
   cosmetic/zero-flicker directives → the version + URL pointers for the lazily
   fetched assets. Plus: the identity handle, the fingerprint/data versions, the
   sync interval, and the merged visited map. *(Same for both generations; the
   `script` machinery is present even where monetization is dormant, `RULES §5`.)*
6. **Register nothing else synchronously** — heavy work is deferred to §6.

The response is a flat JSON object of data keys (`00_INDEX.md` §6 lists the roles).
CORS reflects the request `Origin` with credentials enabled; origin enforcement
against approved store ids exists but has shipped commented-out in at least one
build — a documented debt.

---

## 4. Cache generation

The bulk assets are generated server-side, not authored inline, and served from
their own URLs so the sync response stays small.

> **The 2026-09 shift:** rule/cosmetic content is now produced by ONE shared
> factory — the `blocklist-v4` repo ("Automated All In One List V2"), fully
> documented in `BLOCKLIST_WHITELIST_SYSTEM.md` (+ its visual map
> `la-fabrique-des-listes.html`). The backend generators became thin **mirrors**
> of the factory's finished `dist/` artifacts; nothing on the server compiles
> lists anymore. Provenance questions ("why is this domain blocked?") are
> answered there, not here.

- **Compiled rules** (`generate_compiled_rules.php`) — MIRRORS the factory's
  `dist/network/rules.json` (10k+ finished rules: ID bands, budgets, whitelist
  carve-outs already applied), rebranding only the redirect substitution string to
  the brand's own WAR-listed page. Never re-compile the artifact — it is finished
  (re-running the old STEP-3 logic re-creates solved bugs and ships `[]` on fetch
  failure). Guards: last-good local mirror, hard abort under 1,000 rules, rebrand
  count must match the redirect count. The old last-50k cap is GONE — replaced by
  the factory's ledger-based retention.
- **Cosmetic caches** (`generate_cosmetic_rules.php`) — the generic hide list and
  the per-domain specific/unhide map, emitted as **data** (selector arrays / maps),
  never a stylesheet; sources = the factory's `dist/cosmetic/`, per-brand merge
  kept backend-side.
- **Versioning** — each cache has a `*_version.txt` = `md5` of the file, written
  **only after a verified successful build** (it doubles as the success marker).
  The client re-fetches only when the hash changes; the sync response carries the
  current hash + URL.

Every generator runs on a **24 h staleness check keyed on the SUCCESS markers**
(the `*_version.txt` files — never a pre-touched cache mtime, which once froze a
failed build for 24 h), with a **30-minute attempt stamp** rate-limiting retries
and a **single-flight lock** (§5); all **fail-safe**: an upstream fetch failure
returns without rewriting the cache or bumping the version, so a bad refresh keeps
the last-good data rather than shipping an empty set. ⚠ Per-market allowlists: the
factory renamed the baseline file `general_global.json` → `global.json` — every
brand swap must follow or the baseline 404s, masked ~24 h by the market cache.

---

## 5. Outage hardening

The lessons of three real outages (a database-grant failure that became a retry
storm; a cron-less migration that silently killed the money path; the 2026-09
StopAds connection-cap storm) are baked in as mandatory behaviour. **Since the
2026-09-10 fleet pass, items 1–4 are FAMILY SPEC on every brand's prod AND `-dev`
endpoint** (Ninja · Ghost · Wonder · North · StopAds — 8 files patched in lockstep;
backups per backend in `_backups_hardening_2026-09-10/`).

1. **Degraded HTTP-200.** On any PDO failure or uncaught throwable, a
   `set_exception_handler` returns `200`, not `500`, with: the client's identity
   echoed; the **data-version pin unchanged** (a wrong value purges the fleet); the
   file-served asset versions/URLs; a **jittered ~3–4 h back-off**
   (`10800000 + rand(0, 3600000)`) on the brand's OWN sync-interval key
   (`ninja_sync` / `ghost_sync` / `wondercycle` / `backrotation` / `core_sync` —
   **remap this key when copying**); and baseline rules **for new installs only**,
   served from the hoisted product-neutral helpers `db_down_response()` /
   `default_blockdom()` / `default_excluded_domains()` (single source — the inline
   duplicate arrays were removed; pure file reads, no DB, no network). For an
   existing install it **omits `rules`**, relying on the client's key-by-key merge
   to preserve applied rules (`EXTENSION_COMPOSITION.md` §23.3). **Every bodyless
   `exit` on the request path routes through `db_down_response()`** — a real client
   flow must never receive an empty body.
2. **Fatal safety net** (`register_shutdown_function`, family spec 2026-09-10).
   The exception handler cannot see compile-time fatals, OOM or timeouts; the
   shutdown handler catches those and — deliberately NOT calling
   `db_down_response()` (measured headroom at shutdown is too small) — writes the
   smallest valid answer that still brakes the fleet: `{"<brand-sync-key>": N}`
   with the jittered back-off (`ninja-adb21.php` header notes, §70–80).
3. **Canonical client-IP block** — byte-identical across the five family brands
   since 2026-09-10: explode the `X-Forwarded-For` comma-list, validate each
   candidate, honor the CF header, safe fallback. It replaced two real bugs: a raw
   `getenv` chain that **fatals on a proxied comma-list** (Ninja/Ghost/StopAds —
   and with handler-equipped backends the crash would have been a *silent*
   degraded 200: no installs recorded, nobody alerted), and validate-without-explode
   that collapsed **every proxied user onto `0.0.0.0`** (Wonder — silently
   destroying per-IP dedup and fraud). Copy it verbatim; never re-derive it.
4. **Single-flight generators, keyed on SUCCESS.** Each inline cache regen is
   wrapped in a non-blocking `flock(LOCK_EX | LOCK_NB)`, with staleness keyed on
   the **success markers** (`compiled_rules_version.txt` / the oldest of the two
   cosmetic `*_version.txt` — written only by a verified successful build) plus a
   **30-minute attempt stamp** (`last_rules_regen_attempt.txt`,
   `ninja-adb21.php:4059`) rate-limiting retries. **No pre-touch** — the old
   pre-touched-cache-mtime scheme made a *failed* build look fresh for 24 h (the
   week-frozen-cache scar); the fleet was re-keyed off it in 2026-09 (Ninja is
   where the doctrine came from). One deliberate exception: the scriptlet-details
   trigger (uBO uAssets, not the factory) keeps the old mtime scheme.
5. **Bounded fetches.** All upstream fetches use a `stream_context` timeout (~10 s);
   `PDO::ATTR_TIMEOUT` ~5 s. A slow upstream can never hang a worker. (Truly
   fleet-wide since 2026-09-10 — North's two unbounded `curl_exec` calls were the
   last, now bounded 5 s/10 s with `die()` → `return false`.)
6. **Flip-flop guard.** If a specific/unhide fetch or decode fails, return early and
   keep the previous cache + version instead of writing an empty merged cache that
   would churn the `md5` and force the whole fleet to re-download.
7. **Tmp-orphan sweep.** `unlink` any `*_cache_*.tmp` older than ~1 h so a crashed
   generation cannot leak.
8. **Non-fatal uninstall.** `display_errors=0`, short PDO timeout, connect failure
   redirects to the feedback form anyway, the `disabled` write wrapped in
   `try/catch(Throwable)` — the redirect is the user contract, the write is
   bookkeeping (`USER_CREATION_AND_UPDATE.md` §9).

**StopAds variant (rules ride INSIDE the sync response):** its fallback shape is to
**omit the `rules` key entirely** when the factory artifact is unavailable —
`chrome.storage.local.set` merges key-by-key, so the client keeps its stored array
and the packaged 69k-rule static engine covers fresh installs; sending an **empty**
`rules` array would wipe the fleet's dynamic rules (the exact silent-downgrade
failure the doctrine exists to prevent). Bandwidth is gated server-side by a
`corerulesversion` echo (no extension release needed).

**Known residual hole (documented, deliberately deferred):** every brake above
works only **when PHP actually runs**. The client backs off only on a 2xx with
parseable JSON — a host/CDN **429**, an infra-level 500 (php-fpm pool exhausted),
or a dead webserver never reaches PHP, so the client keeps hammering: the
self-sustaining loop behind the StopAds "[2002] Operation not permitted" storm.
Three closure options are on file (a Cloudflare custom response answering the rate
limit with `200 + {"<sync-key>": N}` per brand · loosening the limiter now that the
brakes live in code · a one-line client back-off on non-2xx, which needs a release
per brand). Related one-liner worth adding when touching a sibling:
`ignore_user_abort(true)` exists in Ninja only — the other brands' generators lack
it (low impact, they are silent, but it is the stated assumption of Ghost's own
comments).

---

## 6. Response-first, in-process scheduling

The live hosts have **no cron**, so maintenance cannot be a separate job — yet the
maintenance work must still run. **Split by generation:** cache regeneration
(compiled rules, cosmetics) is **universal**; the **money path** — finalization
(`finalize_pending_installs.php`), the fraud report, the funnel cache — is **`[V2]`
only**. A V1 (Gen 1) build's post-flush work is **cache regen alone** (it has no
finalize, no fraud sweeps). The contract:

1. **Ship the response first** — `fastcgi_finish_request()` /
   `litespeed_finish_request()` (or an `ob_end_flush` fallback under a ~2 s degraded
   budget) so the client is answered before any maintenance runs.
2. **Run maintenance post-flush, in-process** — a function-scope `require` of the
   maintenance entrypoints (`finalize_pending_installs.php` for held payouts, the
   fraud report, the funnel cache, cache regen), with config **re-required inside**
   for enforcement-flag isolation.
3. **Gate on staleness with atomic claims** — a maintenance slice runs only when its
   stamp file is stale (`last_finalize_run.txt` > ~1200 s, or > 60 s when a
   `finalize_backlog.txt` marker exists). Atomic claims make repeated manual URL
   hits safe for draining a backlog. **Cache-regen slices use the §5.4 doctrine**:
   staleness keyed on the SUCCESS markers (`*_version.txt`), a 30-minute attempt
   stamp, non-blocking flock — never a pre-touched cache mtime.
4. **Verify any self-HTTP call** — if a maintenance step calls back over HTTPS, it
   **reads the status line** and returns an honest boolean; a silently-eaten
   404/edge-block must not look like success (the exact failure that caused a 5-day
   money outage).

Inbound traffic is therefore the scheduler — the model the family is standardizing on
(Ninja/Ghost). Where a low-frequency cron happens to exist it only supplements — keeping the
stamps fresh and the inline path dormant — but new work does **not** rely on cron.

---

## 7. Configuration & secrets

A single `config.php` holds the DB credentials, the identity key, the proxycheck.io
key, and any payout-provider token. The proxycheck risk key and the payout token are **all-builds** (network risk → `level` and install-time conversions run on V1 too); only the *held/finalized-payout* machinery (§6) is V2-only. The identity key differs by variant:
**V1** an AES encryption key (encrypts the whole handle); **V2-A** an HMAC key
(derives the handle); **V2-B** needs **no identity key for issuance** — the handle is
a random token — though a key may still exist only for the legacy-decrypt branch of
its uninstall endpoint. These are per-build secrets — **remap them when copying the
backend**, and **never reuse the same identity key across two builds** (it would let
one build decrypt/derive another's handles). The reference builds hard-code these in
`config.php`; treat that as the value to relocate, not to replicate verbatim.

---

## 8. Backend generations & capability dependencies

The backend is one template, but it exists in **capability generations gated by the
identity variant** — the backend's equivalent of the frontend's identity choice.
**A build's generation is chosen once, at creation, and is fixed for the life of
that build.** The fraud stack needs hardware signals; hardware needs V2 identity; so
a V1 build (Gen 1) has no material to run fraud on — and that is **not a deficiency
to correct but a permanent, correct property** of that build. The generations are a
**fleet** fact (different builds were born on different identities), never an upgrade
ladder inside one build. Why it can never be an upgrade: §8.3.

### 8.1 The generations

| Generation | Identity (`USER_CREATION_AND_UPDATE.md` §3) | What the backend can do | Example builds |
| :-- | :-- | :-- | :-- |
| **Gen 0 — legacy (pre-family)** | **Legacy** (AES `syncuid` blob, no hardware) | user record + dedup on a **pre-family schema** (ad-counter columns, `conv 0–2`, `idx_*_ip/cid_script_updates`); network risk → `level`; monetization injection (may be **dev-forced** rather than gated); `conv` fixed at creation. **No** family user-data group. Backend often *derived* from a family backend (12's is Ninja-derived) but kept on the legacy identity/schema. | 12 (15/17/20/24 to add) |
| **Gen 1 — network-only** | **V1** (encrypted blob, no hardware) | user record + dedup (cid/sid, ip+`an`+instdom); network risk → `level`; monetization injection with an **inert** enable gate; `conv` fixed at creation | North, Hunter, Wonder |
| **Gen 2 — fraud stack** | **V2-A** (HMAC handle) | all of Gen 1, **plus** hardware/VM detection, `device_hash`, install tiers, the behaviour layer + `decision`, finalization sweeps, a **live** enable gate (aging/behaviour-driven 0→1), a representable opt-out (`script = 2`), the sync journal | Ninja |
| **Gen 3 — fraud stack + identity redesign** | **V2-B** (minted token) | all of Gen 2, **plus** a separate stable `device_sig` (identity) beside the volatile `device_hash` (fraud), IP removed from identity, the token recovery cascade | Ghost, 27 |

Gen 2 and Gen 3 have the **same fraud capabilities**; they differ only in identity
mechanics. So "has fraud" = **Gen 2 or higher** = "collects hardware" = "is V2".

### 8.2 The dependency chain — why features can't be cherry-picked

Each capability rides on prerequisites in three layers at once. A feature whose
prerequisites are not all present **does nothing** if bolted on alone.

| Backend capability | Needs identity | Needs client | Needs schema |
| :-- | :-- | :-- | :-- |
| Network risk / `level` | V1+ | — | network columns |
| Hardware / VM detection | **V2** | hardware collection (`EXTENSION_COMPOSITION.md` §7.2) | hardware + `is_vm`/`vm_type` columns (behind the capability probe) |
| Install tiers | **V2** | hardware (network already present) | `device_hash`, tier in `fraud_flags` |
| Behaviour layer / `decision` | **V2** | the visit journal + faster pending cadence | behaviour columns + the sync-log table |
| Live enable gate (0→1) | **V2** | aging / updates / visit tracking already sent | `updates` / `visit` / `enabled` columns |
| Representable opt-out (`script = 2`) | **V2** (server-authoritative state) | the acceptable-ads toggle, sent each sync | `acceptable_ads_disabled` column |
| Held / finalized payouts (`conv`) | **V2** | — | `conv` / `decided_date` + the post-flush scheduler (§6) |
| `device_sig` recovery | **V2-B** | stable-hardware subset | `device_sig` column + `uq_fingerprint` |

**Why V1 cannot run any of the fraud rows:** it collects no hardware (nothing to
evaluate) and its state lives in a client-rewindable blob (nothing
server-authoritative to hold a `decision` or a durable opt-out). Its enable gate
exists in code but is **inert** by construction (`USER_CREATION_AND_UPDATE.md` §7
row 3, §11).

### 8.3 Generation is fixed at birth — never migrate a live build's identity

**The identity mechanism is frozen for the life of a build. An update must never
change the identity handle, its format, the uniqueness / dedup keys, or their
columns and indexes.** Every existing user row is keyed by that identity; change it
and every stored handle stops resolving — the entire existing user base is orphaned
in the database. That is a fleet-wide crash, and it is exactly the outcome this rule
exists to prevent.

Concretely:

- **A V1 build stays V1.** It works — and is *meant* to work — without a fraud layer,
  forever (21, 22 and 23 run this way). Its fraud/behaviour columns being absent is a
  correct permanent property, not a gap to close. **Never add the fraud stack to a V1
  build.**
- **You do not "upgrade" a build to a higher generation.** A higher-generation
  product is a *different build*, born on V2 with its own fresh user base — not a
  migration of an existing one.
- **What an update may safely change:** anything that does **not** touch the
  identity / uniqueness contract — frontend structure, rules, cosmetics, scriptlets,
  zero-flicker, the element picker, YouTube / cookie handling, UI. Bring an existing
  build up to the latest *feature* baseline freely; leave its identity mechanism
  exactly as it was born.
- **The only identity-adjacent operation that is safe** is a fingerprint-*version*
  refresh **within** a V2 build that re-collects telemetry while leaving the identity
  handle itself byte-for-byte unchanged (`EXTENSION_COMPOSITION.md` §5.5). It backfills
  columns; it never re-keys a user. A V1 build has nothing of the sort and needs none.

**Capability vs enforcement (the one axis that *is* safe to move):** a build *born*
on Gen 2/3 can still run with monetization **none / dormant** and its fraud layer in
**shadow** — *having* a capability is separate from *enforcing* it
(`RULES_AND_PRIORITIES.md` §5, `FRAUD_DETECTION_V5.md` §0). That is the only sense in
which a build's behaviour is dialled up or down over time. **The generation itself
never moves.**

---

## 9. Per-extension implementation

All builds share this architecture; they differ only in posture. **Legend:**
✅ present · ⚠️ partial/legacy · ❌ absent · ⬜ to copy.

| Aspect | § | 12 Pro | 21 North | 22 Hunter | 23 Wonder | 25 Ninja | 26 Ghost | 27 |
| :-- | :-- | :-- | :-- | :-- | :-- | :-- | :-- | :-- |
| **Backend generation** | 8 | **Gen 0 (Legacy)** | Gen 1 (V1) | Gen 1 (V1) | Gen 1 (V1) | Gen 2 (V2-A) | Gen 3 (V2-B) | Gen 3 (V2-B) |
| Degraded HTTP-200 hardening | 5 | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ⬜ |
| Single-flight generators + bounded fetch | 4–5 | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ⬜ |
| Flip-flop guard + tmp sweep | 5 | ⚠️ | ✅ | ✅ | ✅ | ✅ | ✅ | ⬜ |
| Response-first in-process scheduler | 6 | ⚠️ | ⚠️ | ⚠️ | ⚠️ | ✅ | ✅ | ⬜ |
| Compiled-rules `__EXT_ID__` + ETag/304 | 2 | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ⬜ |
| Monetization posture (capable — **on/off is an owner toggle**, not tracked⁴) | 3 | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ⬜ |
| Live endpoint | — | `adbpro-202609.php` | `adbnorth_18.php` | `adshunter_2.php` | `wonder58.php` | `ninja-adb21.php` | `ext-server.php` | *(empty)* |

**Notes.** ⁴ **Monetization on/off is an owner-controlled business toggle** — flipped at will,
not tracked per-build here; every built backend is monetization-**capable**. Beware the `-dev.php` twins (`ninja-adb21-dev.php`, `wonder58-dev.php`,
`adbpro-202609-dev.php`): they contain a "DEV PURPOSE ONLY" block that force-sets
`script=1`/`level=0` and are **not** what store builds should call — always verify
against the prod file. **12's *prod* file (`adbpro-202609.php`) currently also
hardcodes `$script=1`/`$pcmarket` in a DEV block** — a manual dev-force override (owner
toggle), not the organic gate; verify the prod file to read the current mechanism, and don't
treat any build's on/off as documented state. 12's backend is **Ninja-derived** (it still references `ninja_fp` in
comments) but runs on the Gen 0 identity/schema — treat it as Gen 0, not as a family
backend to re-sync. North keeps two dead legacy endpoints on a different priority
scale; never reason across that boundary. The response-first scheduler was
retrofitted into Ninja/Ghost after the cron-less outage; older builds (incl. 12)
still lean on inline single-flight generation where no cron exists.

---

## 10. Conformance checklist

Run the **Common** items for every build, then add the block for the build's
generation (§8). Never run a V2 item against a V1 build — it will false-fail.

### 10.1 Common — every build

- [ ] Every file in the set is **copied** (§2.2); the **two endpoint files are renamed** (§2.1) to match the client; per-build **variables set** (§2.3), incl. a **freshly generated identity key** (never shared) — §2.
- [ ] Generation **matches the identity variant** and is **fixed at birth**; no update ever changes an existing build's identity/uniqueness keys or their columns/indexes (would orphan every user) — §8.
- [ ] DB failure / uncaught error → **HTTP 200**, identity echoed, **data-version pin unchanged**, jittered back-off, baseline rules for new installs only — §5.1.
- [ ] The degraded response is **identity-conditional**: it omits `rules` (+ per-user keys) for existing installs — §5.1.
- [ ] Cache generators are single-flight (`flock` + pre-touch), fetches bounded (~10 s), generation fail-safe (last-good on failure); flip-flop guard + tmp-orphan sweep — §4, §5.2–§5.5.
- [ ] The uninstall endpoint **redirects even when its `disabled` write fails**, and its
  **redirect target is present in the server file and re-pointed to *this* build** — not a
  leftover pointing at the copied-from build's form *(agent — verify in code)*. `[OWNER — manual]`
  the **live feedback form actually exists** and its content matches this extension (the owner
  owns/creates the form; the agent only checks the code points at the right place) — §2.1, §5.6.
- [ ] Response ships **before** maintenance (`*_finish_request`); **cache regen** runs post-flush, gated by atomic staleness claims — §6.
- [ ] `extid` validated (`^[a-z]{32}$`); compiled rules carry per-client `__EXT_ID__` + `ETag`/`304`; response assembly follows the fixed order — §2, §3.
- [ ] Secrets remapped per build; the identity key never shared — §7.

### 10.2 V1 (Gen 1) only

- [ ] The fraud / VM / behaviour / finalization files are **absent**, and the enable gate is **inert** — `script = 1` is reached only via a seeded/echoed flag, never auto-promotion — §2.2, §8.
- [ ] `visit` is written **wholesale** each sync (snapshot; no merge, no cumulative history) — §3.
- [ ] There is **no `script = 2` opt-out path** (V1 state is client-rewindable) — §8, `RULES_AND_PRIORITIES.md` §5.

### 10.3 V2 (Gen 2/3) only

- [ ] Finalization / the money path (`finalize_pending_installs.php`, fraud report, funnel cache) runs post-flush; any **self-HTTP maintenance call reads the status line** and returns an honest boolean — §6, §6.4.
- [ ] Every SQL literal compared to or assigned into `conv` is a **quoted string** (`conv` is an ENUM; an unquoted number binds by enum *index*) — `generate_fraud_report.php`, `behavior_lib.php` — §2.2; fraud detail `FRAUD_DETECTION_V5.md` §19.
- [ ] `visit` merge is **non-destructive** (cumulative; a deserialise failure cannot wipe history) — §3.
- [ ] The injection bundle / `script = 2` reset payload obey `RULES_AND_PRIORITIES.md` §5 — §3.
- [ ] The fraud/behaviour conformance items in `FRAUD_DETECTION_V5.md` §19 pass.

---

*Backend architecture reference. Copy it, remap the keys (`00_INDEX.md` §6), keep
the functions. Rules → `RULES_AND_PRIORITIES.md`; identity → `USER_CREATION_AND_UPDATE.md`;
fraud → `FRAUD_DETECTION_V5.md`; schema → `DB_SCHEMA.md`; data → `DATA_SOURCES.md`.*
