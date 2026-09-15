# Detection Redesign — Change Log

Chronological record of the detection-system rewrite (newest first).
See `FRAUD_DETECTION_V5.md` for the *current design*; this file is the "how we got here."

> Brought into the Architecture folder as the canonical detection history. Dated
> entries below sometimes cite the interim spec docs (`DETECTION_SPEC_V4.md`,
> `DETECTION_REDESIGN_SPEC.md`) by the name they had at the time — those are
> superseded by `FRAUD_DETECTION_V5.md` and referenced here only as history
> (`FRAUD_DETECTION_V5.md` header). **This is a post-mortem log, not current state: where two
> dated entries conflict — operational/cron/hosting state across the 2026-07-16→07-21 host
> migration, or a same-day policy reversal (e.g. the 07-12 ghost-payout flip) — the later-dated
> entry supersedes.** Start from `00_INDEX.md` for the folder map.

> **Detection is SHADOW-ONLY.** The enforcement flags in `config.php`
> (`BEHAVIOR_ENFORCE`, `DELAYED_CONVERSION`, `DEFER_CONVERT`) remain `false`, so **no
> detection-driven payout or block behavior has changed**. Only the shadow `decision`/`behavior_*`
> columns and the collection cadence are affected. All backups live in `_backups/`.
> **One deliberate exception:** the **STAGE 1 - 1 HOUR DELAY** feature (2026-07-09 entry) changes
> payout *timing* — same gate, same recipients (minus <1h uninstallers), fired ~1h late.
> Scope since 2026-07-21: **every ad network**, uniform 1h (was `an=gn` only at introduction, then
> `['*']` 07-12 → 07-16, then off 07-16 → 07-21). It is independent of the three detection flags.
> Grep `STAGE 1 - 1 HOUR DELAY` to find every code location.

---

## 2026-09-03 (afternoon) — ninja25 `conv_date`/`updated_at` migration authored · full Ninja⇄Ghost backend sync
Two-way convergence so builds 25 and 26 run the same fraud stack. **Nothing uploaded yet** —
strict deploy order at the end of this entry. All edits backed up first
(`25 …/backend/_backups/sync-ghost-fixes_2026-09-03/`, `26 …/backend/_backups/early-fraud-port_2026-09-03/`);
`php -l` clean on every changed file; independently diff-verified against backups (money-safety,
convention integrity, dormancy, line endings, scope).
- **Migration authored (NOT yet run):** `25 - Ninja Block/backend/docs/migrate_conv_date_updated_at_ninja25.sql`
  adds `conv_date` (after `conv`) + auto-maintained `updated_at` (last column) to `ninja25`,
  mirroring the canonical Ghost migration with ONE deliberate owner-chosen difference:
  **pre-migration rows get `conv_date = NULL`** ("true timestamp unknown" — no falsified data;
  ghost26 had backfilled `created_date`) and **`updated_at` = GREATEST(latest known date in the
  row)** instead of flat `created_date`. `@cutoff` captured before the ALTER protects rows
  inserted mid-migration; both columns are backfilled in ONE statement so the `conv_date`
  NULL-ing cannot bump `updated_at`. `ALGORITHM=INSTANT` fails loudly if not instant-eligible.
- **Ghost fixes → Ninja** (`generate_fraud_report.php`, `finalize_pending_installs.php`,
  `behavior_lib.php` — local `infomaniak/` mirror): all EIGHT unquoted `conv` comparisons in
  the fraud report quoted; fix B NULL guard `(conv NOT IN ('1','3') OR conv IS NULL)` on the
  deferred-convert write; fix C race guard `AND (conv = '0' OR conv IS NULL)` on the
  behavior-layer pay UPDATE; **BACKLOG BYPASS** on the 300s recent-run guard (a cron/manual
  run passes while `finalize_backlog.txt` exists); `conv_date` maintenance on EVERY conv
  write — plain `NOW()` at guaranteed transitions (forfeit/refuse/hygiene/expiry), IF-guard
  `conv_date = IF(conv <=> ?, conv_date, NOW())` (assigned BEFORE conv) at same-value-rewrite
  sites (S1 paid, deferred-convert, behavior pay, bot block). Ninja's bot block keeps `'4'`
  (the 2026-09-03 convention wins over Ghost's `'2'`) and gains the IF-guard with compare `'4'`.
- **early_fraud → Ghost** (`ext-server.php` v7→v8, `behavior_lib.php`): the SHADOW detection
  block ported **byte-identical** (verified by diff), all four `EARLY_FRAUD_FLIP` commented
  sites, Stage-2 Tier-3 `conv 2→4` + CONV CONVENTION comment, unpaid-bot write `'2'→'4'` with
  its existing `conv_date` IF-guard compare adapted to `'4'`, header version entry. Ghost's
  behavior flags verified false → same shadow no-op property as Ninja (one COUNT + token only).
  Flip-simulation lint reproduced independently on the ghost file.
- **Both endpoints + ninja dev twin:** stale Stage-2 preamble comment "Tier 3 blocks (conv=2)"
  corrected to "Tier 3 closes unpaid (conv=4; conv 2 reserved for evaluateRisk)".
- **STRICT DEPLOY ORDER (per DB_SCHEMA.md §5):**
  1. Run the migration SQL on live `ninja25` (phpMyAdmin), run its verification queries.
  2. Upload the three Ninja files (`ninja-adb21.php` + `ninja-adb21-dev.php` from the earlier
     entry, plus today's `finalize_pending_installs.php`, `behavior_lib.php`,
     `generate_fraud_report.php`) — endpoint+finalize ship as a pair.
  3. Upload the two Ghost files (`ext-server.php`, `behavior_lib.php`) — ghost26 already has
     the columns, no migration needed.
  If step 2 ever precedes step 1 by mistake: conv writes throw SQLSTATE 42S22 (unknown
  column), every site is inside try/catch, sweeps keep running and log errors — recoverable,
  but rows decided in that window lose their `conv_date`; don't do it.

## 2026-09-03 — `early_fraud` T0 detection (SHADOW) · pre-wired enforcement (commented) · conv-attribution convention (`25 - Ninja Block`)
Shipped in `25 - Ninja Block/backend/infomaniak/` (`ninja-adb21.php` + CRLF twin
`ninja-adb21-dev.php`, byte-identical edits, + `behavior_lib.php`). Backups:
`backend/_backups/early-fraud_2026-09-03/`. **Not yet uploaded to Infomaniak at the time of
this entry** — pair rule satisfied (finalize untouched). Grounding: the 2026-09-02 audit of the
`ninja25` dump against the ops-confirmed fraud set (708 installs, two operations: a Pakistan
device farm on AU sids 11443125/26/27(+sibling 28) and a Morocco anti-detect op on EU sids
11056035/11451037/11133827). Chronological replay, independently re-verified row-for-row:
structural-only real-time rules block 696/708 at 1.31% background hit.
- **SHADOW detection block** at the end of Step-5 tier assembly (`ninja-adb21.php:3202-3280`;
  token append `:3265-3273`): appends `|early_fraud:<reasons>` (`+`-joined) to `fraud_flags`.
  Reasons and measured performance (labels = confirmed-fraud set vs 34.7k background):
  - `tzdc` — `timezone_mismatch_network_vs_client` in `vm_type` AND `device_collisions >= 2`
    AND clean-net guard (not hard-proxy/vpn type, `proxy != 'yes'`, `risk = 0`, no level digit
    8): ~63% precision, ~70% of confirmed fraud. Either half alone is noise (tz 25%,
    collisions 5-13%) — the conjunction is the signal.
  - `fpvote` — anti-detect leak vote, >=2 of {1280x1024 + `touch_points` 0, Chrome major
    142-146 (hardcoded like the collision threshold; ages precision-safe, recall-decaying),
    `audio_fp` not `124.0434*`}; 2-of-3 needs a corroborator (prior-IP >= 2 or `127.0.0.1` in
    `instadom`); 3-of-3 fires alone.
  - `ipstack` — same exact IP with >= 3 installs in a rolling 30 days (farm stacked up to
    27/IP).
  One new indexed query (`ip`/30-day `COUNT`, same shape as the 7-day collisions count) — **no
  cron, no state tables**; windows age out in the `WHERE`. Whole block is fail-open
  (`try/catch` → no token, install proceeds) and gated on `has_fraud_columns` like the
  collisions count.
- **Length guard 218** on the append: 255 − 8 (`|s1_hold`) − 3 (worst rewrite `|s1_refused`)
  − 26 (longest refuse-reason append) — so every later `fraud_flags` UPDATE (forfeit / refuse /
  hygiene / sanitize bulk writes) stays under the `varchar(255)` even on a pathological row.
  Over-cap → token skipped, never truncated (`$earlyFraud` stays `''`, so a future flip stays
  in sync with the stored token).
- **SHADOW guarantee** (adversarially reviewed, diff-vs-backup): against the 2026-09-03 backups, the only reachable runtime
  deltas are the one `COUNT` and the token text. `conv`, `s1_hold`, response JSON and control
  flow are byte-identical; `$earlyFraud` has zero active consumers.
- **Enforcement pre-wired but COMMENTED — grep `EARLY_FRAUD_FLIP` (3 steps + 1 Stage-2
  guard):** (1/3) payout-gate condition `&& $earlyFraud === ''` (`:3312-3316`) — also covers
  the dormant instant-pay `else` branch, not just the marker; (2/3) `conv = 4` at the door,
  `(int)$conv === 0` precedence so a risk `conv=2` is never overwritten (`:3335-3340`); (3/3)
  `s1_hold` suppression in the Step-9 field line (`:3406-3410`); Stage-2 Tier-2 guard
  (`:3302-3304`, live only if a behavior flag ever flips). A flipped-on row is born `conv='4'`
  with no marker: invisible to the S1 money sweep (fails both predicates) and to the 24h
  hygiene (targets conv 0/NULL), still classified by the decision sweep (no conv filter,
  deliberate) so evidence keeps accruing. Flip is forward-only: shadow-period rows keep their
  `|s1_hold` and pay normally (optional one-time cleanup documented in the session notes). A
  fully-flipped copy of the file was lint-tested before shipping the commented lines.
- **Conv-attribution convention** (`FRAUD_DETECTION_V5.md` §14.1, `DB_SCHEMA.md` §3): `conv
  '2'` = `evaluateRisk` (level) blocks ONLY; `'4'` = every fraud/behavior/forfeit/expiry
  closure, detail in the tokens. Two dormant sites changed accordingly: Stage-2 Tier-3
  `$conv = 2 → 4` (`ninja-adb21.php:3298`; documented side effect: under a future Stage 2,
  tier3-blocked installs keep behavior collection — the cadence check tests `!== 2`;
  intentional) and the `behavior_lib.php` unpaid-bot write `SET conv = 2` → **quoted**
  `SET conv = '4'` — which also fixes a latent enum-index bug: the old unquoted `2` would
  have stored `'1'` (**PAID**) had `BEHAVIOR_ENFORCE` ever been enabled. Ghost's copies still
  write `'2'` — **port pending**.
- **Interaction matrix** (22 cases) agreed before implementation; keys: risk `conv=2` always
  wins (token still recorded); level digits 5/6 rows (conv 0, never held) unchanged — hygiene
  closes them at 24h as before; duplicate short-circuit repeats are never evaluated (existing
  behavior); the `-dev` twin writes to the same `ninja25` table (test rows carry the token
  too); the old-gen `ninja-adb.php` still writes untokened rows if old builds hit it — owner
  decision: to be retired, verify via access log then archive off the host.
- **Docs updated with this entry:** `FRAUD_DETECTION_V5.md` §0.2 (mechanism table row), §5.3
  (grammar, examples, marker rules, write-site snippet), §6.8 (pipeline step), §14.1 (conv
  rows + convention), **plus a line-ref re-anchor pass** (two rounds, independently reviewed): the Step-5
  insertion shifted `ninja-adb21.php` refs > 3196, and substantial pre-existing drift across
  §2.7/§3/§5.5/§6/§8/§11-§15 was re-anchored to the current file. Docs cite the current
  file (3,893 lines). Endpoint header version bumped 8 -> 9 (both twins). `DB_SCHEMA.md` §3 got the convention + the quoted-'4' note.
- **Rollout plan:** upload the three files; same-hour: one dev-endpoint synthetic install,
  check the token; daily: token rate expected ~1-3% of installs (a spike = do NOT flip);
  after 1-2 weeks of shadow, re-measure per-reason precision on a fresh dump before
  uncommenting anything. Rollback = restore the `.bak` files; the token is inert.

## 2026-09-02 — `conv_date` + `updated_at` columns · ENUM index-trap post-mortem · NULL + race guards (`26 - Ad Block Ghost`, now canonical)
Shipped in `26 - Ad Block Ghost` (`ext-server.php` on the `ghost26` table) — **the canonical
family backend from this entry forward**. Migration:
`backend/docs/migrate_conv_date_updated_at.sql`. Sibling-build status at the end of this entry.
- **Two new user-table columns** (both backfilled to `created_date` by the migration — the
  explicit `updated_at` assignment overrides its ON UPDATE clause, by design):
  - **`conv_date`** `datetime DEFAULT current_timestamp()`, placed right after `conv` — the
    moment `conv` received its CURRENT value. **No code writes it at INSERT**: the DEFAULT
    fires in the same statement as `created_date`'s DEFAULT and `NOW()` is per-statement, so
    `conv_date == created_date` exactly. That single property covers BOTH payout regimes with
    **no config gating**: pay-at-install (conv already 1/3 in the INSERT → conv_date =
    conversion time) and delayed conversion (conv 0 at insert, stamped by the later UPDATE).
    The only user-table INSERT is `insertUserRecord()` (`ext-server.php:2226`) — zero
    insert-path code change.
  - **`updated_at`** `datetime DEFAULT current_timestamp() ON UPDATE current_timestamp()`, the
    LAST column, maintained entirely by the DB: any UPDATE that changes at least one value
    bumps it, in any file, present or future; a no-change UPDATE does not. **Distinct from the
    extension-heartbeat column `updated`** (no auto-update clause; feeds the 36h-survival
    metrics — untouched).
- **UPDATE-side maintenance — every conv write now also maintains conv_date.** Sites whose
  WHERE guard (`conv='0' OR conv IS NULL`) guarantees a real transition use plain
  `conv_date = NOW()` (`finalize_pending_installs.php:232/280/384/570`). The 4 sites that can
  rewrite `conv` with its existing value use
  `SET conv_date = IF(conv <=> ?, conv_date, NOW()), conv = ?`
  (`finalize_pending_installs.php:315/538`, `behavior_lib.php:556/592`): NULL-safe compare,
  and **conv_date MUST be assigned before conv** — UPDATE SET evaluates left-to-right. A
  same-value rewrite therefore does NOT move conv_date, so the column stays truthful for
  forensics ("which code path set this conv, and when"). **Standing rule: any new conv write
  maintains conv_date with the same pattern.**
- **NO index on either column (deliberate):** conv_date is never filtered by live queries;
  updated_at is rewritten on every row touch — same "harmful index" category as the dropped
  `updated` index.
- **A. ENUM index trap (fixed) — retro-diagnosis of the 07-09 blacklist incident.** `conv` is
  `enum('0','1','2','3','4')`; documented MariaDB/MySQL behavior: an UNQUOTED number
  assigned/compared to an ENUM is treated as the enum INDEX, not the value — index 2 =
  value '1'.
  - `behavior_lib.php`'s bot block wrote `SET conv = 2`, which would have stored **'1'
    (PAID)** instead of '2' (blocked) — dead code in Stage 1 (flags false) but armed for
    Stage 2. Now quoted `'2'` (`behavior_lib.php:592`).
  - `generate_fraud_report.php` had EIGHT unquoted conv comparisons (ASN fraud-rate numerator
    :156, good cohort :202, bad cohort :220-224, per-network report :290, chargeback worklist
    :307): `conv = 2` matched '1' (paid), `conv IN (1,3)` matched '0','2'. Every conv-derived
    stat in `fraud_report.json` has been wrong for as long as those lines existed.
  - **The old ASN numerator (`is_vm=1 OR conv IN (2,4)`, unquoted) therefore counted PAID
    installs ('1' and '3') as fraud** — an ISP's "fraud rate" was roughly its paid-conversion
    rate, so the largest residential ISPs scored highest. That plausibly CAUSED the 2026-07-09
    residential-ASN blacklist incident (Comcast/AT&T/PLDT…; see the 07-09 Phase-0 entry
    below): the numerator was not even "the system's own verdicts" as diagnosed then — its
    conv half selected the paid cohort. The Phase-0 suspension stays correct and stays in
    force; this entry adds the mechanism that grew the list.
- **B. NULL three-valued logic:** finalize's deferred payout `WHERE conv NOT IN ('1','3')`
  silently skipped `conv IS NULL` rows (`NULL NOT IN (...)` is NULL, not true) — postback
  fired but never recorded in Stage 2. Now `(conv NOT IN ('1','3') OR conv IS NULL)`
  (`finalize_pending_installs.php:538`).
- **C. Pay-path race guard:** behavior_lib's pay UPDATE had `WHERE id=?` only; a terminal conv
  (2/4) set concurrently during processConversion's network wait could be overwritten. Now
  `AND (conv = '0' OR conv IS NULL)` (`behavior_lib.php:556`) — parity with the S1 payout
  write.
- **Expected effect + WATCH:** every conv-derived stat in `fraud_report.json` step-changes at
  the first post-fix run — the numbers were wrong BEFORE the fix, not after.
  `calibrated_risk_threshold` (the only fraud-report output consumed by the live request path —
  clamped 25–50, fallback 33, read at `ext-server.php:545`) can move on the first corrected
  run: **watch it**.
- **Deploy order:** ALTER first, PHP upload second (the PHP references conv_date). Files: the
  migration, then `finalize_pending_installs.php`, `behavior_lib.php`,
  `generate_fraud_report.php`.
- **Verified:** all three fixes adversarially reviewed; `php -l` clean on every touched file.
- **Sibling builds (recon 2026-09-02):** `25 - Ninja Block` (`backend/infomaniak/`, table
  `ninja25`, prod `stopads24`) carries the FULL pre-fix set: the same eight unquoted sites
  (`generate_fraud_report.php:156/202/220-224/290/307`), unquoted `SET conv = 2`
  (`behavior_lib.php:580`), no fix B (`finalize_pending_installs.php:514`), no fix C
  (`behavior_lib.php:547`), and no new columns (its schema dump is also stale: conv enum
  without '4'). **If 25 is still live, its fraud_report stats are computed with the wrong
  enum-index semantics fixed here.** Builds 08/21/22/23 (older monolithic backends): no fraud
  pipeline and no unquoted SQL conv literals — conv is always a bound prepared-statement
  param — and none has the new columns (23's schema doc confirms). Build 27's backend
  directory is still empty.

## 2026-08-31 — `12 - Ad Block Pro` documented as `Gen 0` (Legacy); frontend re-platformed onto baseline
- **Context:** 12 is a **pre-family ancestor** (AES `syncuid` handle on the `adbprocup11`
  schema — ad-counter columns, `conv 0–2`, `idx_*_ip/cid_script_updates`, no `fingerprint`
  and no fraud stack; network-risk only). It was previously **out of scope** in the update
  skill. It is still shipped and monetized (live YHS), so it is now a **first-class `Gen 0`
  (Legacy) generation** with a column in every per-extension table, handled through a new
  **legacy-modernization track** (`00_INDEX.md` §3): identity/DB/backend frozen forever, the
  **frontend** brought progressively to baseline. This is the template for the other
  pre-family builds (15/17/20/24).
- **Frontend modernization applied to 12 (identity untouched):** reserved-ID band model
  (user allow/block `1/2` + session `21/22`); **user-tier priority corrected `2000000` →
  `200/100`** (the old value sat *above* the injection band, letting a user allow-list away
  the monetized surface — `RULES_AND_PRIORITIES.md` §4.2); **structured zero-flicker** (hide +
  cloak/self-heal grammar, MV3-safe); **two-layer pre-paint fixed** — Layer 1 now reads
  `chrome.storage.local` directly at `document_start` (the worker round-trip was flashing on
  cold start); **flicker decoupled from the injection allowlist** (the whitelist-collapse bug —
  the backend re-added google/yahoo to `allowlist` after the protected-domain carve-out, so
  the mask was suppressed on the search surface); local-first storage (settle/heal/purge);
  action-driven sync; **`alarms` permission dropped**; static rulesets stripped of
  injection-surface rules; sibling/monetization references scrubbed from comments.
- **Spec lessons folded in (generic, from this port):** `EXTENSION_COMPOSITION.md` §11.1
  (content-script direct-read is the real fast path) and §11.2 (the allowlist trap) + §0.2
  (comment/scheme-scrub rule) + §26 (runtime smoke test); `RULES_AND_PRIORITIES.md` §4.2
  (safe user-tier value) / §5.4 (flicker↔allowlist) / §9.1 (static-ruleset injection audit);
  `DATA_SOURCES.md` §6 (feed-health check — 12's compiled feed was found broken/ungenerated).
- **No detection change.** 12 has no fraud stack; nothing here touches enforcement, and its
  identity/schema were not modified (doing so would orphan its user base).

## 2026-07-21 (night) — Response-first, traffic-driven maintenance: finalize runs IN-PROCESS
- **Incident that forced it:** the 07-13→07-16 migration to Infomaniak (no cron offered) left
  `finalize_pending_installs.php` with NO working scheduler for 5 days — the bg_run HTTPS
  self-call was eaten silently at the edge (it never read the response, so 404/403/CF-block
  looked identical to success). Unnoticed while payouts were at-install; the 07-21 global 1h
  hold made it a money outage within hours (zero postbacks, spawn stamp fresh, run stamp
  frozen at 07-16, browser hit = CF 524 but the run ground on server-side). Backlog was
  drained same day by repeated manual URL hits (atomic claims made that safe).
- **New contract (must hold on EVERY host, zero code change on migration):**
  `ninja-adb21.php` now ships the JSON response FIRST (`fastcgi_finish_request` /
  `litespeed_finish_request`; degraded ~2s budget if neither exists), then runs maintenance
  post-flush. Finalize executes **in-process** via `ninja_run_inline()` — a function-scope
  `require` (scope = variable isolation; config re-required inside to reset the enforcement
  flags createUser mutates per-network; `$pdo` NOT imported so finalize takes its standalone
  branch: own guarded connection, flock, run-stamp touch). No cron, exec, or self-HTTP on the
  money path. A cron (Hostinger later) runs the same file standalone, keeps the run stamp
  fresh, and the inline trigger stays dormant — cron present or absent, same code.
- **Budgeted slices (in finalize):** `$GLOBALS['NINJA_FPI_DEADLINE']` = slice deadline
  (12s detached / 2s degraded). Money sweep first: LIMIT 400 oldest-first, per-row deadline
  check BEFORE the atomic claim; early `return` skips hygiene/decision/expire when spent
  (decision sweep gets LIMIT 150 + per-row check when it does run). `finalize_backlog.txt`
  breadcrumb = slice stopped with work left → trigger re-fires at 60s instead of 1200s; the
  300s recent-run guard is bypassed in inline mode (trigger cadence + flock rate-limit instead).
- **bg_run is now VERIFIED** (reads the status line, 2s; fast non-2xx → error_log + false;
  silence → accepted-and-grinding → true; sends a User-Agent). Kept only for the fraud report
  (unguarded function names → not include-safe; stale `last_audit_run.txt` is the loud signal)
  and as first-chance for the funnel cache (inline fallback when detached — its helpers are
  guarded). Audit trigger got the two-stamp split: run stamp no longer pre-touched (it lied
  fresh for 5 days); `last_audit_spawn.txt` rate-limits attempts.
- **Observability:** every inline run error_logs its stats line; `last_finalize_run.txt`
  fresh = runs start; `finalize_backlog.txt` ABSENT = draining keeps up. Spawn stamps are
  attempt-records only — never read them as health. `last_finalize_spawn.txt` is vestigial.
- **Verified locally:** `php -l` both files; harness (unreachable-DB override) proved: no-deadline
  keeps the 300s guard (cron semantics intact); deadline bypasses it and reaches the guarded
  connect; a connect-failed run never touches the run stamp; flock never sticks; config values
  resolve correctly in function scope (`s1_delay_in_scope('gn')`=true, 3600s, allowlist=no).
- **Re-upload:** `ninja-adb21.php`, `finalize_pending_installs.php` (as a pair). Backups:
  `_backups/*.before-inline-scheduler.bak`.
- **Rollback:** restore both `.before-inline-scheduler.bak` files — returns to the 07-16
  self-call design (and its silent-failure mode; manual URL hits remain the stopgap).

## 2026-07-21 — GLOBAL uniform 1h payout hold re-activated (config-only)
- **What:** `$STAGE1_DELAY_CONV_AN = ['*']`, `$STAGE1_DELAY_CONV_SEC = 3600`,
  `$STAGE1_DELAY_CONV_SEC_AN = []` in `config.php` + `config-hostinger.php`. Every attributed
  install is held at install (`conv=0`, `'|s1_hold'` in `fraud_flags`) and paid by
  `finalize_pending_installs.php` at 1h. Difference vs the 07-12 rollout: **no per-network
  exception** — gn drops from 3h to the same 1h as everyone (the 3h was an S1 hold duration
  aligned to the Stage-2 checkpoint, only meaningful while gn was in `$BEHAVIOR_AN_ALLOWLIST`).
- **Deliberately NOT changed:** `$BEHAVIOR_ENFORCE` / `$DEFER_CONVERT` / `$DELAYED_CONVERSION`
  stay `false` (so `$pureShadow` stays true and the Stage-2 tier branches at
  `ninja-adb21.php:3243/3246` stay unreachable), and `$BEHAVIOR_AN_ALLOWLIST` stays `[]`.
  **This is timing only: nothing can be REFUSED.** The cron merge branch
  (`finalize_pending_installs.php:197-237`, `mergeConvV4` → `'|s1_refused'`) is unreachable while
  the allowlist is empty, so `$s1Stats['refused']` stays 0 by construction.
- **Prereq checked before flipping:** finalize cron confirmed alive (user, 2026-07-21). This is the
  standing dependency — with a global hold that one script carries 100% of payout traffic; the
  07-15/16 outage entries below are what happens when it stops.
- **Effective latency:** 60–75 min (1h floor + ≤15 min cron granularity).
- **Never held (unchanged):** `an=NULL` organic, `level !== '0'`, `conv=2`, missing `cid`.
- **In-flight rows:** untouched in both directions — the hold decision is taken once, at insert.
  Expect up to one pay window (72h) of mixed-state rows after the flip.
- **Expected reporting shift:** forfeits land as `conv=4`, which `generate_fraud_report.php:156`
  counts as fraud with no maturity filter; that moves the GOOD-cohort p95 feeding
  `calibrated_risk_threshold` (read live at `ninja-adb21.php:3202`). `generate_funnel_cache.php`
  and the dashboard paid-KPI (`>= 36 HOUR` scope) are unaffected.
- **Watch:** `$s1Stats['forfeited']` in the cron output — that counter is the only real measurement
  of the sub-1h uninstall rate (`disabled` is stored day-precision, so it is not derivable from
  existing rows).
- **Re-upload:** `config.php` only (all consumers read these as `global` at call time). Backups:
  `_backups/config.php.before-s1-1h-global.bak`, `_backups/config-hostinger.php.before-s1-1h-global.bak`.
- **Rollback:** the READY-TO-ROLLBACK block now in `config.php` (swap the 3 live lines for the 3
  commented ones) — new installs pay at install again, the cron still drains held rows.

## 2026-07-16 (night) — sanitize_fast_uninstalls.php: same-day uninstalls → conv=4
- **What:** one-shot, dry-run-default script closing every never-paid attributed row
  (conv 0/NULL, `an` set) whose `disabled` date equals its install date, as conv=4 with
  marker `'|conv4_fast_uninstall'`. Restores the conv taxonomy (0 = unattributed or
  in-flight ONLY) across ALL history — including rows before the 2026-07-13 forward-only
  hygiene cutoff (deliberate, scoped exception to that decision). `'|s1_hold'` rows
  excluded (finalize's jurisdiction). Idempotent (marker excluded from scope).
- **Money:** none moved; 3/3 panel — no default path pays these rows, and the close even
  BLOCKS a latent wrong payment by the legacy ninja-adb.php 3h sync payer (it doesn't
  check `disabled`). ORDER: run AFTER settling rescue `&uninstalled=1` (marked rows leave
  the rescue's reach; the dry run prints the overlap count).
- **Expected side effect:** fraud_report.json shifts (BAD cohort/per-ASN fraud counts grow
  retroactively) — reporting only; `calibrated_risk_threshold` derives from the GOOD
  cohort (conv 1/3) and cannot move.
- **Re-upload:** `sanitize_fast_uninstalls.php` (new file); delete from server when done.

## 2026-07-16 (night) — rescue_conv_20260715.php: one-time pay-at-install backfill
- **Why:** the outage + broken CLI spawn + early manual runs under the old 24h window left
  never-paid installs since 07-15 in mixed states: conv=0 no marker, conv=4 `'|s1_forfeit'`
  (forfeited by the manual runs), conv=4 `'|conv4_expired_unpaid'` (hygiene).
- **What:** new standalone script, DRY-RUN by default (`?run=dry`), pays with `?run=live`.
  Scope: created_date ≥ 2026-07-14 20:00 UTC (= 07-15 00:00 GMT+4) AND older than 3h,
  attributed (an+cid non-empty), level='0', never paid (conv 0/NULL; conv=4 forfeit/hygiene
  rows with `&forfeits=1` — recommended). Opt-in: `&uninstalled=1`, `&s1paid=1`.
  Safety: atomic `'|rescue0715'` claim (re-runnable, race-proof vs finalize/hygiene),
  `'|s1_hold'` rows excluded (finalize owns them), conv 1/2/3 untouchable at SELECT+claim,
  flock, progress heartbeat every 100 rows. 3/3 adversarial panel: no double-pay path;
  install path pays BEFORE insert so a resting conv=0 row provably never fired a postback.
- **Caveat:** processConversion pays by default when the 7thsense API is unreachable —
  run the live pass while the API is healthy. Delete the script from the server when done.
- **Re-upload:** `rescue_conv_20260715.php` (new file).

## 2026-07-16 (evening) — Pay window 72h, no delay floor, CLI spawn → HTTPS self-call
- **Field report:** with the lazy triggers live, both stamp .txt files appeared but held rows
  were NOT paid; a manual browser hit of `/finalize_pending_installs.php` DID pay them.
  Diagnosis: the CLI `php` spawned by exec() started the script (touched the stamps) then
  died at the unwrapped `new PDO` (CLI ini without pdo_mysql / wrong socket) — web PHP works.
- **`ninja-adb21.php`:** all three triggers now spawn via `ninja_bg_run()` — an async
  fire-and-forget **HTTPS self-call** to `ninja-block.com/<script>` (1s timeout, socket
  closed after headers), i.e. the exact environment the manual test proved working.
  Windows dev keeps the CLI popen.
- **`finalize_pending_installs.php`:** `ignore_user_abort(true)` added; standalone `new PDO`
  now wrapped — on connect failure it logs and returns WITHOUT touching the run stamp (a
  failed run no longer silences the fallback for a full window); run stamp now touched only
  after a working DB connection. Pay window default 259200 (**72h**, was 48h this morning,
  24h originally) via `$S1_HOLD_PAY_WINDOW_SEC`; S1 SELECT floor now derived
  (`pay window + 24h`) so aged-out rows still get a proper `'|s1_forfeit'`.
- **`generate_funnel_cache.php`:** `ignore_user_abort(true)` added (web-triggered runs).
- **`config.php` / `config-hostinger.php`:** `$S1_HOLD_PAY_WINDOW_SEC = 259200` (72h);
  `$STAGE1_DELAY_CONV_SEC = 0` and `$STAGE1_DELAY_CONV_SEC_AN = []` — leftover held rows
  are due IMMEDIATELY at the next sweep (no residual 1h/3h floor; full-drain user decision).
- **Re-upload:** `ninja-adb21.php`, `finalize_pending_installs.php`,
  `generate_funnel_cache.php`, live config. If new installs still get `'|s1_hold'` after
  the config upload, the LIVE config was not replaced (file must be named `config.php`).

## 2026-07-16 — S1-hold pay window 24h → 48h: rescue the maintenance-outage backlog
- **Why:** the 07-15 outage killed the finalize cron; held rows (`'|s1_hold'`, conv=0) blew
  past their 1h/3h deadline through no fault of their users. At 24h they would forfeit.
  User decision: any still-installed held row up to **48h** old gets PAID at the first
  catch-up run (aware some networks may not attribute postbacks fired >24h after click).
- **`finalize_pending_installs.php`:** forfeit test now `$ageSec > $s1PayWindowSec`
  (config `$S1_HOLD_PAY_WINDOW_SEC`, default 172800) instead of hardcoded 86400. The
  conv-hygiene sweep now EXCLUDES held rows still inside the pay window (they are in-flight,
  owned by the S1 sweep) — without that clause one failed S1 pass would let the bulk
  hygiene UPDATE forfeit rescuable 24-48h rows. NULL-fraud_flags rows stay closable
  (COALESCE). The 72h SELECT bound still covers the window; >72h stragglers keep being
  closed by hygiene (now that they are past 48h).
- **`config.php` / `config-hostinger.php`:** new `$S1_HOLD_PAY_WINDOW_SEC = 172800`.
- **Backups:** same-day `.before-lazy-triggers.bak` / `.before-s1-rollback.bak` still valid
  for rollback; this change is additive on top.
- **Re-upload:** `finalize_pending_installs.php` + the live config. To drain the backlog
  IMMEDIATELY (and see stats), open `/finalize_pending_installs.php` once in a browser —
  it runs standalone and prints `S1-delay sweep. Due: X (Paid: Y, ...)`.

## 2026-07-16 — FULL STAGE-1 ROLLBACK: everyone pays at install again (config-only)
- **What:** `$STAGE1_DELAY_CONV_AN` `['*']` → `[]` (no new installs held — the install gate at
  ninja-adb21.php fires `processConversion` immediately again) and `$BEHAVIOR_AN_ALLOWLIST`
  `['gn']` → `[]` (the V4-3 checkpoint merge — the only Stage-2 money path that was live —
  is off; held gn rows now drain via the plain Stage-1 level rule). The three behavior flags
  were already `false`. `$STAGE1_DELAY_CONV_SEC`/`_SEC_AN` left in place — inert with the AN
  list empty, still used to schedule the drain of rows already holding `'|s1_hold'`.
- **In-flight held rows:** nothing to do — the finalize cron selects by the `'|s1_hold'`
  marker, not by the list (designed rollback path), so existing holds pay at their original
  deadline (1h default, gn 3h) or forfeit on uninstall, then the feature is fully drained.
- **Shadow detection continues unchanged for everyone** — decision/behavior_* writes and the
  collection cadence are independent of both lists; only payout timing/gating reverted.
- **Backups:** `_backups/config.php.before-s1-rollback.bak`,
  `_backups/config-hostinger.php.before-s1-rollback.bak`.
- **Re-upload:** the config actually used by the live server — the repo has BOTH `config.php`
  (Infomaniak DB creds) and `config-hostinger.php` (Hostinger creds, deployed under the name
  `config.php`); both updated identically to stay in sync. No other file changed.

## 2026-07-16 — Lazy background triggers: cron-or-web fallback for the maintenance scripts
- **Why:** the 2026-07-15 incident showed the Hostinger cron can silently stop (maintenance);
  a traffic-driven fallback self-heals the moment requests return. Design: cron stays PRIMARY;
  ninja-adb21.php spawns the script in the **background** (detached process, output redirected —
  the user request never waits) only when the target's RUN stamp shows the cron missed its
  slot by more than the slack. **Never concurrent** (each script's own `LOCK_NB` flock kills
  racing duplicates); **one run per window** enforced by three layers: slack on the fallback
  deadline (dormant under normal cron jitter), a recent-run guard inside each script (a
  drifted cron arriving after a fallback run exits instead of re-running), and the flock.
- **Two stamps per target** (single-stamp v1 failed adversarial review — with the fallback TTL
  equal to the cron period and/or an end-touched stamp, the fallback raced the cron every
  window and cron drift > run duration produced double full runs):
  `last_<x>_run.txt` is touched only by the SCRIPT (finalize: at run start after winning its
  lock; funnel: after a successful cache write) and decides whether a run is due;
  `last_<x>_spawn.txt` is pre-touched by the endpoint before each spawn attempt and
  rate-limits attempts (stampede lesson) while still allowing retries after a failed run.
- **`ninja-adb21.php`:** new trigger block before the final `echo` —
  `finalize_pending_installs.php`: fires when run stamp >1200s stale (900s cron period +
  300s jitter slack; degraded cadence ~20 min), spawn attempts capped at 1/900s.
  `generate_funnel_cache.php`: fires from **01:15 GMT+4** (900s grace after the 01:00
  boundary, fixed `DateTimeZone('+04:00')` — independent of server/PHP default tz) while no
  run has SUCCEEDED since the boundary, retrying ≤1/1800s until one succeeds. The existing
  daily-audit trigger (`generate_fraud_report.php`, 6h TTL, no cron — the trigger IS its
  scheduler) kept byte-identical; the inline stampede-guarded funnel regen at the top of the
  file kept as last-resort backstop (>24h stale only).
- **`finalize_pending_installs.php`:** start-touches `last_finalize_run.txt` after winning its
  flock (healthy cron ⇒ stamp age ≈900s ⇒ fallback at 1200s stays dormant); recent-run guard
  skips if another run started <300s ago.
- **`generate_funnel_cache.php`:** end-touches `last_funnel_run.txt` after a **successful**
  `funnel_cache.json` write (any mode — cron, spawned, or inline fallback); a failed run
  leaves it stale so the web fallback retries same-day (≤1/1800s), NOT next-boundary.
  Recent-run guard skips standalone runs if a success happened <1800s ago AND the cache's
  mtime ≤ stamp mtime (i.e. the current funnel_cache.json is the one that run wrote) — so
  an out-of-band cache deletion (endpoint then pre-writes the empty skeleton, mtime newer
  than the stamp) is rebuilt immediately instead of serving empty funnel data for a day.
- **Money paths untouched:** no change to sweeps, gates, or postbacks — only to *when/how* the
  scripts get started. Idempotency (atomic `|s1_hold` claims etc.) unchanged.
- **Verified:** php -l all three · GMT+4 boundary math empirically tested (exact-copy harness,
  deliberately wrong default tz, 6 boundary cases) · 4-lens adversarial review (concurrency /
  hot-path regression / timing / docs), 34 agents; v1's 7 confirmed findings all addressed by
  the two-stamp + slack + recent-run-guard design above, re-verified.
- **Backups:** `_backups/ninja-adb21.php.before-lazy-triggers.bak`,
  `_backups/finalize_pending_installs.php.before-lazy-triggers.bak`,
  `_backups/generate_funnel_cache.php.before-lazy-triggers.bak`.
- **Re-upload:** `ninja-adb21.php`, `finalize_pending_installs.php`, `generate_funnel_cache.php`.
  Nothing to delete. Crons: keep finalize every 15 min; set/keep funnel daily at 01:00 GMT+4
  (= 21:00 server time if the server clock is UTC, which the DB clock is); no fraud-report cron
  needed (6h lazy trigger is its only scheduler, as before).

## 2026-07-13 — Dead-code cleanup: WOE fraud_score + per-campaign behavior_thresholds removed
- **Why:** the 2026-07-12 audit confirmed both subsystems are consumed by NOTHING for any
  decision. Detection runs entirely off the live tier + behavior + merge layers.
- **`generate_fraud_report.php`:** removed the Weight-of-Evidence model (`$woeWeights`,
  `$evaluateSignals`, the good/bad signal-count loops, the WOE log-ratio + intercept) and the
  entire per-row `fraud_score` scoring loop (old STEP 4). KEPT: STEP 1 device-hash backfill,
  STEP 2 clone clusters (now report-only), the cohorts, and **`calibrated_risk_threshold`**
  (the one value still consumed by the request path). `woe_weights` dropped from
  `fraud_cache.json`; report breakdown now uses live columns (bot_installs / non_converted /
  vm) instead of `fraud_score`. STEP banners renumbered 1–4.
- **`generate_funnel_cache.php`:** removed the per-campaign `behavior_thresholds` construction
  and its `funnel_cache.json` key (intentionally disconnected per spec §6 — `evaluateBehavior`
  ignores its `$thresholds` arg). KEPT: `global_funnel` / `by_campaign` (feed the
  funnel_only / zero_organic behavior signals) and the precision report.
- **DB columns `fraud_score` / `fraud_checked`:** now inert (frozen at last values). Left in
  place — dropping is unnecessary and risky; harmless as dead columns.
- **Cache files self-heal:** the live `fraud_cache.json` (still has `woe_weights`) and
  `funnel_cache.json` (still has `behavior_thresholds`) are rewritten without those keys on the
  next nightly run — no manual edit needed.
- **Verified:** php -l both · no orphaned references (woeWeights/scoringCount/compositeScore/
  behavior_thresholds all gone except comments + the schema probe).
- **Backups:** `_backups/generate_fraud_report.php.before-woe-cleanup.bak`,
  `_backups/generate_funnel_cache.php.before-thresholds-cleanup.bak`.
- **Re-upload:** `generate_fraud_report.php`, `generate_funnel_cache.php`. Nothing to delete.

## 2026-07-13 — Dashboard filter fix: Created-after / Win now compose with any cohort
- **Bug:** "Created after" (fromDate) and "Win" (days) were wired ONLY inside the `custom`
  cohort branch of `eraWhere()`, so with the default "CURRENT config" cohort they were
  silently ignored — a user filtering "gn since today" got gn (network filter works globally)
  but the whole cohort span instead of today. Network/Sub-id/Tier always worked (outside the
  era branch).
- **Fix (`dashboard_detection.php` eraWhere):** cohort boundaries, then an independent date
  narrowing that composes with EVERY cohort — `fromDate` always applies; `days` applies to
  present-open cohorts only (tier/all/custom), skipped for bounded legacy windows
  (v4/post/mid/pre) so "last N days" can't silently empty an old fixed span. Param order
  preserved (fromDate < an < sid). 6/6 builder tests. Small UI hint added under the form.
- **Re-upload:** `dashboard_detection.php`.

## 2026-07-13 — V4-3 LIVE FOR gn: full Stage-2 merge at the 3h checkpoint (canary)
- **User decision:** gn goes fully Stage 2 NOW (ahead of the formal validation read — the gn
  canary itself is the forward validation; rollback = remove 'gn' from the allowlist). All
  other networks unchanged: Stage 1 + 1h hold, uninstall forfeit only.
- **`finalize_pending_installs.php` — merge wiring in the hold sweep**, gated ONLY on
  `an_in_behavior_scope()` (`$BEHAVIOR_AN_ALLOWLIST = ['gn']`); the three legacy behavior
  flags stay false forever (sidesteps the audited flag-scoping asymmetry). Per gn row at its
  3h deadline: decision still pending → stays held (decision sweep terminalizes same run;
  merge fires next pass ~15 min later — never pay before the verdict) · terminal decision →
  `mergeConvV4(deriveTierFromRow(row), decision, flags)`: **pay** → existing atomic claim +
  processConversion · **refuse** (bot / T3-no-proof) → atomic `'|s1_hold'→'|s1_refused'` +
  `conv='4'` + merge reason appended (e.g. `|t1_bot_refused`). Ghosts PAID per policy.
  Proof read from behavior_flags (auth_login | popup_interaction); ghost from no_evidence
  (guaranteed on 0-sync rows since the audit fix). New sweep stat: Refused.
- **`behavior_lib.php` — popup_interaction hardened** (it now gates real T3 money):
  serialized-empty whitelist forms ('a:0:{}','N;','','null','[]') no longer count as
  interaction.
- **`dashboard_detection.php`** — Panel E header states gn is LIVE money, others simulation.
- **Verified:** php -l ×3 · classifier self-test 25/25 · 7/7 verdict-mapping tests on the
  real functions (bot→refuse, ghost→pay, T3±proof, pending→held).
- **Expected on gn:** refusals ≈ its non-uninstall bot rate (small today; the anti-adaptation
  shield) + rare T3-no-proof; conversions land ~3h15–3h30; watch `s1_refused` reasons and
  Panel E red-cell survival staying ~0%.
- **Backups:** `_backups/*.before-v43-gn.bak` (3 files).
- **Re-upload:** `finalize_pending_installs.php`, `behavior_lib.php`, `dashboard_detection.php`.

## 2026-07-13 — CONV HYGIENE (forward-only): attributed rows never REST at conv=0
- **User decision:** conv becomes a clean state machine — `0` = unattributed OR in-flight
  (held, deadline not reached) ONLY · `1/3` = processConversion only · `2` = install
  risk-block (or processConversion postback-failure — pre-existing dual meaning, kept) ·
  `4` = closed without ever reaching processConversion (forfeit / refusal / expiry; reason
  in decision + markers). **NO BACKFILL** — rows created before 2026-07-13 09:00 DB time
  (= 13:00 GMT+4) keep their historical values forever.
- **`finalize_pending_installs.php`:** (a) both forfeit branches of the hold sweep now also
  set `conv='4'` (guarded on conv 0/NULL); (b) new ungated CONV HYGIENE sweep: attributed
  rows still conv 0/NULL past 24h AND `created_date >= '2026-07-13'` → conv=4 +
  `|conv4_expired_unpaid` (catches level-5/6 gate failures, missing sub-values, level flips
  while held, claimed-but-unfired crash leftovers). Not enforcement — bookkeeping on money
  already not paid; deliberately independent of the behavior flags.
- **Note:** conv=4 rows feed the nightly report's observational `conv IN (2,4)` stats —
  correct (they are unpaid/bad traffic) and harmless (blacklists suspended).
- **V4-3 (gn Stage-2 merge) will reuse this vocabulary:** refuse (bot / T3-no-proof) → conv=4
  with merge reason. Queued for the validation checkpoint.
- **Backup:** `_backups/finalize_pending_installs.php.before-conv4.bak`.
- **Re-upload:** `finalize_pending_installs.php`.

## 2026-07-12 — FULL COHERENCE AUDIT (4-way + adversarial verify) · fixes · S1-DELAY PROGRESSIVE ROLLOUT
- **Audit verdict: COHERENT_WITH_CAVEATS (all 4 auditors).** Verified end-to-end: Step-5 tier
  == spec §1.1 (every condition has a matching reason, none can fire without the other);
  Stage-1 gate behaviorally identical to the historical gate; all money paths flag-gated and
  inert; Panel E sim ≡ mergeConvV4; TIER_EXPR ≡ deriveTierFromRow; F6/D LIKE predicates have
  no live substring collisions; device-hash v2 identical across ninja-adb21/generate_fraud_report;
  spec + changelog match deployed code.
- **CONFIRMED BLOCKER (fixed):** `no_evidence` was emitted only when score==0.0, so an
  attributed 0-sync ghost with a funnel landing tab (`instdom_funnel_only` +20) was counted
  as "unsure" instead of "ghost" by the dashboard/F8 (money unaffected — merge falls back to
  n_syncs==0). Fix: `classifier_v4.php` emits `no_evidence` on ANY 0-sync install. Ghost
  counts on the dashboard will step UP at deploy (more accurate, not worse traffic).
- **Latent-drift fix:** `behavior_lib.php` still carried a v1 (9-component) copy of
  computeDeviceHashFromRow whose enforce-path write would have poisoned v2 collision counts —
  synced to v2 (hash parity verified: identical output vs generate_fraud_report copy).
- **S1-DELAY PROGRESSIVE ROLLOUT (user decision):** `$STAGE1_DELAY_CONV_AN = ['*']` (ALL ad
  networks held; organic never in scope) · default `$STAGE1_DELAY_CONV_SEC = 3600` (1h) · new
  `$STAGE1_DELAY_CONV_SEC_AN = ['gn' => 10800]` (gn = full 3h checkpoint, Stage-2 timing
  canary). New `s1_delay_seconds($an)` helper + wildcard in `s1_delay_in_scope()`; sweep
  SELECTs at the minimum delay and enforces each row's own deadline (forfeit-on-uninstall
  checked before the due test, so uninstallers forfeit early). SAFE for every network:
  audit proved only ev/mg/m1/m2 postbacks carry a payout param (NOT ez — docs corrected) and
  `$po` is hardcoded 0 at install anyway, so delayed postbacks are bit-identical.
  **Expected effects:** conversions report ~1h late everywhere (gn ~3h); each network's
  conversion count drops by its <deadline fast-uninstall rate (that money was going to
  provably-departed installs); watch per-network `s1_forfeit` shares.
- **Caveats logged for later (not fixed now):** (a) sibling endpoints `ninja-adb.php` /
  `ninja-adb-dev.php` still contain the FULL pre-rework logic (old tier labels, v1 hash,
  lifetime collisions, no hold) writing to the same table — MUST be confirmed traffic-dead or
  removed from the server; (b) per-network flag scoping asymmetry (createUser resets only
  DELAYED_CONVERSION; DEFER_CONVERT never scoped) — harmless while flags are false, fix
  before V4-3; (c) `popup_interaction` feature counts serialized-empty whitelist forms —
  hardening candidate (it is the T3 proof signal); (d) soft-score ≥3 verdicts land in Tier-3
  `hard_vm` (`score_N:`) — now documented in spec §1.1, monitor via Panel A; (e) Panel E
  matured-scope now disclosed in its header.
- **Verified:** php -l ×6 · classifier self-test 25/25 · 7/7 s1-helper tests · cross-file
  hash parity · adversarial verify confirmed the blocker before fixing.
- **Backups:** `_backups/*.before-s1-rollout.bak` (6 files).
- **Re-upload:** `config.php`, `behavior_lib.php`, `finalize_pending_installs.php`,
  `classifier_v4.php`, `ninja-adb21.php`, `dashboard_detection.php`.

## 2026-07-12 — Ghost policy FINAL: ghosts are PAID — subcategory only (+F8 detail)
Correction of the entry below (its intermediate "ghost not paid, conv stays 0" state was
briefly deployed): **money principle = refuse only on POSITIVE evidence** (bot any tier /
T3 without proof). **ALL T1/T2 unknowns pay, ghosts included** — `ghost` (never synced) vs
`unsure` (synced, thin evidence) is an ANALYTICS SUBCATEGORY only (merge `reason`
`tX_ghost_paid` / `tX_unknown_paid`, Panel E KPI split, F8). conv semantics back to original:
paid 1/3 · decided refusal 4 · attributed installs terminal on conv at 3h. Known, accepted,
MONITORED cost: ghosts survive at 1.3%; the flip (ghost → refuse) stays a one-line change in
`mergeConvV4` if F8 forward data demands it. Self-test 25/25.
Also: **F8 detail upgrade** — uninstalled (day-precision, overall) separated from
stopped-syncing per horizon; every % now shows its n.
- **Re-upload:** `classifier_v4.php`, `dashboard_detection.php`.

## 2026-07-12 — UNIVERSAL 3h HOLD (blueprint+sim) · device-collision v2 · dashboard F8 + ERA fix
All SHADOW-side: no payout behavior changes until V4-3 flips the real hold.
- **Why (dashboard forward data, first clean cohort):** Panel E showed 419 attributed T1-bots
  (~4.8% of payouts) paid at install and provably gone by 3h — 100% of behavioral bots sit in
  Tier 1 (cross-layer "win case" row); ghost-unknowns (`no_evidence`, n=1,187) survive at 1.3%
  yet were mapped to PAY; `device_collisions` drove 14,789 Tier-2 rows (84% of T2) at −0.3
  survival lift (pure noise: LIFETIME count, threshold >0, hash blind to UA/screen/GL/audio).
- **1. `classifier_v4.php` — `mergeConvV4()` rewritten to UNIVERSAL HOLD:** nobody pays at
  install; money settles ONCE at 3h and NEVER moves after (user decision 2026-07-12: no late
  payments). bot → refuse (conv 4) in EVERY tier; T3 human+proof only; T1/T2 identical —
  human/evidence-unknown pay at 3h; **ghost-unknown → outcome `'none'`: NOT paid, conv rests
  at 0 forever** (no conversion event; decision stays terminal `deferred`; post-3h data is
  analysis-only). conv=4 is reserved for DECIDED refusals. The expire sweep never touches
  ghosts (it matches decision pending/NULL only). Return shape: `['pay','outcome','reason']`.
  Self-test extended: **24/24** (checks pay AND outcome).
- **2. `dashboard_detection.php`:**
  - **Panel E** simulates the new mapping (T1×bot now red REFUSE; new grey "NO PAY · ghost
    (conv stays 0)" KPI + cell class; ghost split via `behavior_flags LIKE '%no_evidence%'`).
  - **New Panel F8 "Deferred cohort"**: per tier × (ghost | evidence-unknown) — wake rate
    (synced after `decided_date`), 24h/36h outcomes split gone / active-quiet / active+human-
    signal (auth-marker in cumulative `visit` or popup columns), plus ghost wake-latency from
    `sync_log` MIN(server_ts) (avg + ≤6h/≤12h/≤24h shares). Answers "when does a user show
    himself" and sizes the 24h ghost-hold rescue.
  - **ERA_TIER fix**: the "trustworthy" cohort boundary was hardcoded to the Phase-0 upload
    (07-09 20:35), letting ~a day of pre-demotion labels pollute validation. Now auto-detected
    (first row carrying a Phase-2-only marker: `hard_proxy_type` / `extid_mismatch` /
    `tier2:vpn|proxy_flag|banned_provider|linux_ua`), cached in `era_tier.cache`.
- **3. Device-collision v2 (`ninja-adb21.php` + `generate_fraud_report.php`):**
  - Install-time count: `WHERE device_hash=? AND created_date >= NOW()-INTERVAL 7 DAY`
    (was lifetime, whole table); Tier-2 trigger `>= 2` (was `> 0`); reason string aligned.
  - `generateDeviceHash` v2: + `screen_res`, `gl_ext_count`, `audio_fp`, `ua_norm`
    (new `normalizeUaForDeviceHash`: OS family + browser major, Edge-before-Chrome priority).
    Mirrored byte-for-byte in `computeDeviceHashFromRow` + backfill SELECT extended.
  - ⚠️ v2 hashes don't match v1 rows: collision counts restart at deploy (quiet ~1 week);
    nightly clusters mix formats up to 30 days (undercount = conservative). Blacklist stays
    suspended anyway.
- **Verified:** `php -l` ×4 · mergeConvV4 self-test 24/24 · Step-5 harness 13/13 (incl.
  collision ≥2 → T2, single collision → T1) · hash tests (screen/browser change hash;
  identical input stable; edge/crios/firefox UA normalization).
- **Expected dashboard effect:** Panel E: T1×bot → REFUSE (red), new NO-PAY·ghost bucket ≈ ghost share;
  Tier 2 share collapses toward ~12–15% as v2 collisions replace noise; F8 fills as unknowns
  mature. Money simulation now previews the intended V4-3 policy through the validation window.
- **Backups:** `_backups/classifier_v4.php.before-hold.bak`,
  `_backups/dashboard_detection.php.before-hold.bak`, `_backups/ninja-adb21.php.before-devhash.bak`,
  `_backups/generate_fraud_report.php.before-devhash.bak`.
- **Re-upload:** `classifier_v4.php`, `dashboard_detection.php`, `ninja-adb21.php`,
  `generate_fraud_report.php`.

## 2026-07-10 — Tier demotions: Tier 3 = near-zero-FP evidence only — Phase 2
- **Why (data-driven):** proxycheck `type` breakdown over 30 days: Business 34,750 / Residential
  17,695 / Wireless 11,180 / **VPN 937** / HTTP 24 / HTTPS 6 / SOCKS4 2 — zero hosting/
  datacenter/tor traffic. The old Tier-3 test condemned all `conv==2` risk paths (VPN, proxy
  flag, banned provider, Linux) although their mature cohorts survive like Tier 1 (risk 72.2%,
  linux/other 81.4% vs Tier-1 78.1%).
- **What changed (Step 5 tier classification in `ninja-adb21.php` — LABELS ONLY):**
  - **Tier 3 now =** hard VM (`is_vm===1`, post-Phase-1 = hard traps only) · hard proxy
    infrastructure (`tor/socks*/http(s)/compromised/hosting/compute/datacenter` — evaluateRisk's
    type list **minus vpn**) · extension-ID mismatch (integrity) · manual device/ASN blacklists
    (cache ships empty since Phase 0; kept as an emergency lever). New reason strings:
    `hard_proxy_type:<type>`, `extid_mismatch` (replaces the blanket `risk_level:<level>`).
  - **Demoted to Tier 2** with explicit reasons: consumer VPN (`vpn`), proxycheck proxy flag /
    risk>threshold (`proxy_flag:<risk>`), banned-origin provider (`banned_provider`, read from
    level digit 8), desktop-Linux UA (`linux_ua` — the duplicate standalone Tier-3 condition is
    gone; coded once). Existing Tier-2 signals unchanged (borderline `physical:*` VM,
    minor risk score, device collisions).
- **NOT changed:** `evaluateRisk()`, `level`, `conv`, every conversion gate — byte-identical.
  VPN/Linux/banned-provider installs still fail the Stage-1 `level==="0"` payment gate exactly
  as before; only their tier LABEL changed. Paying any of them is a V4-3 merge decision.
- **Verified:** `php -l` clean; 12/12 scenario tests on the extracted live block (VPN→T2,
  hosting/SOCKS4→T3, linux→T2, hard VM→T3, old_laptop→T2, extid→T3, banned provider→T2,
  manual blacklist→T3, clean→T1).
- **Expected effect:** Tier 3 ≈ ~50 installs/30d (~0.08%): hard VM ~19 + hard proxy ~32 +
  extid ~2. Tier 2 absorbs VPN (~1.5%) + proxy-flagged + linux + banned-provider + Phase-1
  physical:* rows. Dashboard per-tier panels become meaningful from this deploy forward.
- **Validation clock RESTARTED:** V4-3's "≥4 days of clean forward data" gate counts from THIS
  deploy (2026-07-10). All tier-segmented validation filters `created_date >=` the upload
  timestamp; pre-deploy tier labels are legacy (blacklist era). First meaningful read ~day 5–6
  (36h outcome maturity). Spec updated: `DETECTION_SPEC_V4.md` §1.1 + §5 + §6.
- **Backups:** `_backups/ninja-adb21.php.before-tier-demote.bak`.
- **Rollback:** restore the backup.
- **Re-upload:** `ninja-adb21.php`.

## 2026-07-10 — VM evaluator de-fanged (old_laptop + timezone demoted, reducer fix) — Phase 1
- **Why (data-driven, follow-up to Phase 0):** of 1,144 `is_vm=1` verdicts in 30 days, 779 (68%)
  were `old_laptop`/`old_laptop_specs` and ~346 (30%) soft-score convictions with
  `timezone_mismatch` present in nearly every one; true hard traps fired only 19×. The mature
  (≥48h) VM cohort uninstalls at 15.6% (below Tier-1's 17.6%) and survives 36h at 71% — humans
  on old machines, not farm hardware.
- **What changed in `evaluateVirtualMachine()` (`ninja-adb21.php`, mirrored in `rerun-vm.php`):**
  1. `old_laptop` / `old_laptop_specs` no longer return `is_vm=1` — they return
     `is_vm=0, vm_type='physical:old_laptop[_specs]|<reasons>'`. The `physical:` prefix routes
     them to **Tier 2** (borderline_vm) instead of Tier 3; the tag stays queryable.
  2. `timezone_mismatch_network_vs_client` no longer scores (+2.0 removed) — observational
     reason only (proxycheck geolocation too coarse for mobile/CGNAT). Alone it now lands
     Tier 2 via the `physical:` prefix.
  3. Reducers (battery / RAM>4 / touch) apply at **any** score — the old `<= 3` cap made 3.5
     unfalsifiable while 3.0 could be rescued to 0. Evaders still keep their reasons in
     `vm_type` → Tier 2 → behavior decides.
  4. All hard traps byte-identical (webdriver, hypervisor GPU, VPS mismatch, RAM spoofs,
     headless UA, iOS battery). `ram_gb` column is FLOAT (verified) — spoof traps safe.
- **`rerun-vm.php`:** timezone ratchet REMOVED (it injected a fabricated mismatching
  `ip_timezone` for previously-flagged rows, so a one-time geolocation error could never clear).
  Safe to use as a backfill tool now; evaluator copy synced.
- **Verified:** 9/9 behavior tests on the extracted live function (old_laptop→physical/T2,
  tz observational, reducers at 3.5, all hard traps still convict, clean machine→physical).
  `php -l` clean on both files.
- **Money impact: none.** `is_vm` never gated Stage-1 payment; `level`/conv gates untouched.
- **Expected effect:** `is_vm=1` rate drops ~98% (to hard traps + rare reducer-less soft scores);
  Tier 2 grows (absorbs old_laptop / tz-mismatch / soft-GPU installs); with Phase 0, Tier 3
  shrinks to ~3% of installs (risk + linux + hard VM). Audit cohorts (good requires `is_vm=0`,
  bad includes `is_vm=1`) rebalance from tonight.
- **Optional backfill (NOT run yet — needs user go):** relabel historical mislabeled rows:
  `UPDATE ninja25 SET is_vm=0, vm_type=CONCAT('physical:', vm_type)
   WHERE is_vm=1 AND (vm_type LIKE 'old_laptop%' OR vm_type LIKE 'score_%');`
  (reversible in meaning — original trap name preserved inside vm_type; fraud_flags history
  deliberately NOT rewritten — the Phase-3 validation restart owns that axis.)
- **Backups:** `_backups/ninja-adb21.php.before-vm-defang.bak`,
  `_backups/rerun-vm.php.before-vm-defang.bak`.
- **Rollback:** restore both backups.
- **Re-upload:** `ninja-adb21.php` (rerun-vm.php only if/when used for backfill).

## 2026-07-09 — Tier-3 blacklist SUSPENSION (ASN + device) — Phase 0 of the tier-layer fix
- **Why (data-driven):** 69.7% of the last 30 days' installs (44,536/63,878) were Tier 3, and
  **90.3% of those carried `blacklisted_asn`** (40,216). The nightly audit's ASN "fraud rate"
  (`is_vm=1 OR conv IN (2,4)` — the system's own verdicts, not ground truth) had blacklisted the
  30 largest residential ISPs in our markets (Comcast, AT&T, Verizon, T-Mobile, BT, Bell, PLDT,
  Globe, Telkom…). Mature-cohort (≥48h) outcomes: ASN cohort uninstalls **14.7%** vs Tier-1
  **17.6%** and survives 36h at 69.5% (gap vs Tier 1 explained by SEA geography mix, not fraud);
  device-cluster-only cohort survives **81.4% — better than Tier 1** (popular hardware collides
  on identical hashes). Both lists were convicting ordinary humans at scale. VM+risk+linux
  (the remaining Tier-3 reasons) are ~4k rows, of which true hard traps fired only **19 times**.
  Timeline: Tier-3 share ratcheted 2.4% → 70.3% (06-22 → 07-02) as the nightly blacklist grew.
- **What changed:**
  - `generate_fraud_report.php` — Step 5a now emits **empty** `devices`/`asns` maps into
    `fraud_cache.json` (the request path can no longer stamp `tier3:blacklisted_asn` /
    `tier3:blacklisted_device`). The candidates are still computed and now land in
    `fraud_report.json` as `observed_device_clusters` / `observed_high_fraud_asns`
    (observation only). Section banners renamed **STAGE n → STEP n** ("Stage" is reserved for
    the detection Stage-1/Stage-2 concept).
  - `fraud_cache.json` — hand-emptied `devices` (232) and `asns` (30) for immediate effect
    (the edited cron keeps them empty from tonight).
- **NOT changed:** `level`, conv gates, payout behavior (Stage 1 pays exactly as before —
  the tier label never gated money in Stage 1); per-IP proxy/VPN/hosting detection in
  `evaluateRisk`; device-collision counting (`deviceCollisions > 0` remains a Tier-2 signal).
- **Re-blacklisting bar (if ever revived):** outcome-based numerators only (uninstall /
  0-sync / 36h survival), never `is_vm`/`conv`; residential/mobile ASNs capped at "suspicious".
- **Follow-ups (agreed plan):** Phase 1 = de-fang `evaluateVirtualMachine` (old_laptop +
  timezone_mismatch out of the verdict, reducer asymmetry fix — old_laptop family is 68% of
  VM verdicts, timezone_mismatch is in nearly every soft-score conviction). Phase 2 = tier
  demotions (VPN/Linux/device-collisions → Tier 2; prune banned-provider substrings). Phase 3 =
  restart the V4-2 forward-validation clock (per-tier panels were polluted) + fold the
  three-independent-layers model (hardware / network / behavior verdicts, fused once at the
  money merge) into V4-3.
- **Backups:** `_backups/generate_fraud_report.php.before-asn-suspend.bak`,
  `_backups/fraud_cache.json.before-asn-suspend.bak`.
- **Rollback:** restore both backups (blacklists resume next request / next nightly run).
- **Re-upload:** `generate_fraud_report.php`, `fraud_cache.json`.

## 2026-07-09 — STAGE 1 - 1 HOUR DELAY: gn payout held ~1h after install (canary)
- **What:** Stage-1 conversions for `$STAGE1_DELAY_CONV_AN` networks (**canary: `['gn']`** = Galaksion)
  are no longer fired at install. The install passes the **exact same Stop Ads gate** (`an` + `cid` +
  `level==='0'` + not conv=2), but instead of calling `processConversion`, conv stays `0` and the row is
  marked `|s1_hold` in **`fraud_flags`** (stable; `behavior_flags` is overwritten by the classifier every
  sync). The `finalize_pending_installs.php` cron fires the postback once the install is
  `$STAGE1_DELAY_CONV_SEC` (3600s) old. With the current ~15-min cron cadence, payment lands 60–75 min
  post-install. Marker lifecycle: `|s1_hold` → `|s1_paid` (postback fired, conv set) or `|s1_forfeit`
  (uninstalled before the deadline, or aged past 24h unpaid).
- **Deliberately NOT `$DELAYED_CONVERSION`:** flipping that flag would activate Stage-2 tier gating
  (Tier-3 gn → conv=2 hard block, Tier-2 gn → held for the 3h behavioral merge, Tier-1 gn → still paid
  at install) plus the 24h conv=4 expire sweep — that is V4-3, gated on forward validation. This feature
  is a flat timing shift with **zero detection coupling**: all three behavior flags stay `false`,
  `$pureShadow` stays true everywhere, V4-2 shadow data is unaffected (decision is conv-independent).
- **Files (grep `STAGE 1 - 1 HOUR DELAY`):**
  - `config.php` — new `$STAGE1_DELAY_CONV_AN = ['gn']`, `$STAGE1_DELAY_CONV_SEC = 3600`.
  - `behavior_lib.php` — new `s1_delay_in_scope($an)` helper (independent of `an_in_behavior_scope`;
    `$BEHAVIOR_AN_ALLOWLIST` stays reserved for the V4-3 enforcement canary).
  - `ninja-adb21.php` — Stage-1 gate in `createUser` holds instead of paying for scoped networks;
    `|s1_hold` appended to `fraud_flags` at insert.
  - `finalize_pending_installs.php` — new independent first sweep: selects by the `|s1_hold` marker
    (NOT by the an list, so rollback still drains held rows), skips/forfeits uninstalled rows, re-checks
    `level==='0'` (Stop Ads parity), then **atomic claim** (`|s1_hold`→`|s1_paid` UPDATE, `rowCount()===1`)
    before `processConversion(an,cid,sid,0)` — overlapping cron runs cannot double-fire (at-most-once).
    The existing 3–24h decision sweep and the expire sweep are untouched.
- **Known consequences (accepted):** installs uninstalling <1h are never paid (fast_uninstall = 0%
  survival per V4 data — this is the point); gn-reported conversions drop by the fast-uninstall rate and
  appear ~1h late on Galaksion's side (click_id postback, well within their window); dashboards show
  fresh gn installs (<1h) as conv=0. `po` is not persisted in the DB so the cron pays with `po=0` —
  free for gn (click_id-only postback); **persist po at install before expanding to ev/mg/m1/m2/ez**.
- **Rollback:** `$STAGE1_DELAY_CONV_AN = []` — new installs pay at install again; the cron drains
  already-held rows on its next passes. Full revert: restore `_backups/*.before-s1-delay.bak`.
- **Backups:** `config.php`, `ninja-adb21.php`, `behavior_lib.php`, `finalize_pending_installs.php`
  → `_backups/<file>.before-s1-delay.bak`.
- **Re-upload:** `config.php`, `behavior_lib.php`, `ninja-adb21.php`, `finalize_pending_installs.php`.

## 2026-07-09 — V4-2: behavior-only classifier goes live (SHADOW) + decision decoupled from conv
- **Design:** `DETECTION_SPEC_V4.md` (authoritative). `DETECTION_REDESIGN_SPEC.md` marked superseded.
- **New files (deployed, wired-in):** `classifier_v4.php` (behavior-only score + `mergeConvV4` + ≥2
  corroboration gate; self-test 20/20). `replay_v4.php` (read-only backtest; delete from server after use).
- **Modified live files:**
  - `behavior_lib.php` — `evaluateBehavior()` now delegates to `classifyBehaviorV4()` instead of the
    v3 `classifyBehavior()`. **The tier prior, `is_vm`, `device_collision`, `ip_rotation`, and
    `no_sync` no longer influence `decision`** — it is now a pure behavioral fact. (v3 `classifier_lib.php`
    kept only for `deriveTierFromRow()`.)
  - `finalize_pending_installs.php` — **dropped the `conv` filter from the resolution sweep** and the
    per-row paid/blocked skip + the `conv NOT IN ('1','3')` clauses on the decision UPDATEs, so paid
    installs also get a terminal decision (fixes the v3 "paid stuck at pending" invariant leak). No
    money logic touched — payout stays flag-gated + conv===0-guarded.
- **Backups:** `_backups/behavior_lib.php.before-v4-2.bak`, `_backups/finalize_pending_installs.php.before-v4-2.bak`
- **Why:** forward data (dashboard, 07-07→09) showed v3's `decision` was prior-driven, not behavioral
  (T3 prior +35 ≥ BOT_LINE +30 → terminal bot at first sync; 37% decided <30min; 1.6 avg syncs;
  non-monotonic score bands). Replay of v4 on 6/9 days restored strict monotonicity and confirmed the
  v3→v4 flow (74% of v3 "bots" were tier-only convictions → v4 human/unknown).
- **Corroboration gate:** a `human` verdict now needs ≥2 independent human signals (auth / popup /
  gov-edu / revisit / organic-breadth); a lone spoofable signal → `unknown`. Replay impact was small
  (~0.8% of humans) — it blocks the naive one-domain spoof, NOT the flood's multi-domain visit-spoof
  (that needs the future sync-accumulation feature).
- **Impact:** SHADOW-ONLY. All config flags remain `false`; no payout/block behavior changed. Only the
  `decision`/`behavior_*` columns change meaning going forward. The pre-existing >24h `pending` backlog
  is NOT cleared by this (still Step-4 backfill); the forward invariant self-heals as new installs
  resolve inside the 3-24h window.
- **Re-upload:** `behavior_lib.php`, `finalize_pending_installs.php`, `classifier_v4.php` (+ `replay_v4.php` if backtesting).

## 2026-07-07 — Collection cadence 5 min → 10 min (Option A)
- **File:** `ninja-adb21.php` — `BEHAVIOR_COLLECT_SYNC_MS` `300000 → 600000`.
- **Backup:** `_backups/ninja-adb21.php.before-cadence.bak`
- **Why:** at 5-min cadence, `BEHAVIOR_SYNC_CAP = 20` froze the sync_log-derived signals
  (`n_syncs`, `interval_cv`, `zero_activity_ratio`, `distinct_prefixes`) at ~100 min — only
  ~55% of the 3h window (both the sync_log INSERT and the fast-cadence hint sit in the same
  `underCap` gate in `logSyncArrival()`). `10min × 20 = 200min` now covers the full window.
- **Impact:** shadow-only. Did **not** invalidate prior data — key signals are cadence-robust
  (`interval_cv` is scale-invariant; `n_syncs` caps identically; organic signals accumulate).
  Done now so the entire forward-validation cohort runs on the final cadence.
- **Re-upload:** `ninja-adb21.php`.

## ~2026-07-04 — Step 2: decision lifecycle (`deferred` = "unknown", `pending` transient)
- **Files:** `behavior_lib.php`, `finalize_pending_installs.php`
- **Backups:** `_backups/behavior_lib.php.before-step2.bak`, `_backups/finalize_pending_installs.php.before-step2.bak`
- `behavior_lib.php`: added guarded `BEHAVIOR_WINDOW_SEC`; `runBehaviorLayer()` now
  **terminalizes** — undecided stays `pending` inside the 3h window, becomes terminal
  `deferred` past it; `decided_date` stamped on any terminal outcome.
- `finalize_pending_installs.php`: dropped the blanket **"0-sync → bot"**; a
  0-sync-but-not-uninstalled row now defers to the tier-aware classifier
  (Tier 1 → `deferred`, Tier 2/3 → `bot`), per the agreed install-prior policy (option c).
- **Decision:** reuse the existing `'deferred'` enum value as the terminal "unknown" —
  **zero schema change** (rename to `unknown` deferred to a later step).
- **Impact:** shadow-only. ⚠️ The existing **>24h `pending` backlog is NOT auto-cleared**
  (the cron sweep only covers 3–24h) — that's a future backfill (Step 4).
- **Deploy:** verified healthy in prod (no HTTP 500; `decision` populating).

## ~2026-07-04 — Step 1: scoring classifier
- **New:** `classifier_lib.php` (the model + self-test); `dry_run_classifier.php` and
  `validate_outcomes.php` (read-only analysis tools, CLI + token-gated web mode).
- **Modified:** `behavior_lib.php` — `evaluateBehavior()` delegates to `classifyBehavior()`;
  `extractBehaviorFeatures()` adds `tier` and `fast_uninstall`.
- **Backup:** `_backups/behavior_lib.php.2026-07-04.bak` (pre-Step-1 original).
- Replaced the binary if-rules with an **additive suspicion score**: tier prior
  (T1/T2/T3 = `0 / +15 / +35`) + weighted evidence; two thresholds
  (`bot ≥ +30`, `human ≤ −15`) → human / bot / unknown.
- `unknown` mapped to `deferred` to preserve caller vocabulary.
- **Tier-3 human hard gate:** requires `auth_session` OR `popup_interaction`
  (⚠️ spoofable — monitored via the forward survival delta; see SPEC §2).
- Self-test caught & fixed a double-penalty bug (`no_sync` + `zero_organic` stacking).
- **Impact:** shadow-only.

### Validation results (2026-07-04)
- Retrospective (15,379 rows): bots uninstall **12.8×** more than humans.
- **Decircularized** (`fast_uninstall` weight = 0, so uninstall cannot feed the label):
  bots still uninstall **7.2×** more (32.9% vs 4.6%). Airtight — the signal is real, not
  circular. `unknown` bucket uninstalls like humans (benign) → validates "Tier-2 pays unknown."
- Exposure: **~31% of PAID installs** fall in the bot bucket (the ROI case).
- ⏳ Still required: **forward** validation on clean post-deploy data before any enforcement.

---

## Current deployment state

> **⚠️ FROZEN HISTORICAL SNAPSHOT (state as of ~2026-07-09/07-12) — do NOT act on this.** These two
> sections ("Current deployment state" and "Not yet done"), including every `RE-UPLOAD now`
> directive, are a point-in-time record **superseded by the later entries above and by
> `FRAUD_DETECTION_V5.md`** (the live reference; §15–§16). Several items here shipped afterward —
> e.g. the V4-3 merge wiring went live 2026-07-13 then dormant 07-16 — so the file versions, the
> "not yet done" list, and the enforcement-flag plan below **no longer reflect the deployed
> system**. Read as history; never re-upload from this table. The V4-era vocabulary (V4-2/V4-3/Step
> N) maps to the current naming via `FRAUD_DETECTION_V5.md` §16.2.

| File | Status |
|---|---|
| `classifier_v4.php` | **deployed (shadow) — the LIVE classifier as of V4-2** |
| `classifier_lib.php` | deployed; now only `deriveTierFromRow()` is used (v3 model dormant) |
| `config.php` | **s1-delay version — RE-UPLOAD now** (`$STAGE1_DELAY_CONV_AN`/`_SEC`) |
| `behavior_lib.php` | **V4-2 + s1-delay version — RE-UPLOAD now** (delegates to v4; `s1_delay_in_scope`) |
| `finalize_pending_installs.php` | **V4-2 + s1-delay version — RE-UPLOAD now** (conv-independent resolution; held-payout sweep) |
| `ninja-adb21.php` | **tier-demote version — RE-UPLOAD now** (Tier 3 = hard evidence only; VPN/linux/proxy-flag/banned-provider → Tier 2; includes vm-defang + s1-delay) |
| `generate_fraud_report.php` | **blacklist-suspension version — RE-UPLOAD now** (Step 5a emits empty `devices`/`asns`; observation lists moved to `fraud_report.json`) |
| `fraud_cache.json` | **hand-emptied blacklists — RE-UPLOAD now** (immediate effect; cron keeps it empty from tonight) |
| `replay_v4.php` | read-only backtest tool (delete from server after use) |
| `dry_run_classifier.php` / `validate_outcomes.php` | legacy tools (delete from server after use) |

## Not yet done
- **V4-3 (Step 3)** — Stage-2 tier→`conv` money wiring in `ninja-adb21.php` using `mergeConvV4()`.
  **Money-moving; gated on forward validation of the V4-2 shadow data.**
- **Sync-accumulation human signal** — anti-spoof feature (organic evidence that GREW across the 3h
  window). Needs V4-2 forward sync histories first; calibrated forward. Answers the flood visit-spoof.
- **Step 4** — backfill the existing >24h `pending` backlog (decision axis).
- **Per-campaign `behavior_thresholds`** — intentionally disconnected; reconnect only if data demands.
- **Flip enforcement** — `config.php` flags `true` (canary via `$BEHAVIOR_AN_ALLOWLIST = ['gn']`) only after V4-3 + forward validation.
