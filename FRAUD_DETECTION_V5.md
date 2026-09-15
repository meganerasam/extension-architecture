# Fraud Detection V5 — Live System Reference

> **What this document is.** A complete, standalone reference for the fraud/VM/behavioural
> detection system as it actually runs in `ninja-adb21.php` and its dependencies. Where V4 was a
> design rationale, this is a reference manual: every threshold, every config value, every state
> marker, every branch. A reader with only this document should be able to predict what the
> endpoint does with a given request.
>
> **Source of truth.** `infomaniak/ninja-adb21.php` and the files it loads. Where a code comment
> and the code disagree, the code wins and this document follows the code.
>
> **Supersedes** `DETECTION_SPEC_V4.md` (kept on disk as history) and `DETECTION_SYSTEM.md`
> (v3-era, superseded on 2026-07-09/10 and no longer accurate). `DETECTION_CHANGELOG.md` remains
> the chronological record of *how we got here* and is not superseded.
>
> Last verified against the live files: **2026-08-24**.
>
> **Two version axes, don't conflate them:** **"V5"** is *this document's* (spec) version — it
> supersedes the V4-era spec docs. **"v4"** (`classifier_v4.php`, "v4-2", the `V4-1..V4-5`
> roadmap in §16.2) is the *classifier/detection-logic* version — a separate axis that V5 simply
> documents. A "v4" classifier under a "V5" spec is expected, not a contradiction.
>
> **Canonical update (2026-09-02).** The `conv_date` / `updated_at` columns, the quoted-enum
> `conv` fix, the NULL-safe deferred-payout guard and the pay-path race guard shipped in
> `26 - Ad Block Ghost` (`ext-server.php` on the `ghost26` table) and are **canonical for the
> family from that date** (`DETECTION_CHANGELOG.md` 2026-09-02; migration
> `migrate_conv_date_updated_at.sql`). Where this document describes that delta, the `file:line`
> citations are the **Ghost** copies of `finalize_pending_installs.php` / `behavior_lib.php` /
> `generate_fraud_report.php`; all other citations remain the Ninja reference lines above, and
> SQL quotes keep the reference table name `ninja25` (Ghost substitutes `ghost26`). Ninja's
> local mirror was synced to the fix set on 2026-09-03 (upload pending); its live host
> still runs the pre-fix pipeline (§18).

---

## Table of contents

| § | Section | |
|---|---|---|
| 0 | [Live state at a glance](#0-live-state-at-a-glance) | start here |
| 1 | [Architecture — three layers, two axes](#1-architecture--three-layers-two-axes) | the model |
| 2 | [Hardware layer — `evaluateVirtualMachine()`](#2-hardware-layer--evaluatevirtualmachine) | layer 1 |
| 3 | [Network layer — risk lookup and the `level` string](#3-network-layer--risk-lookup-and-the-level-string) | layer 2 |
| 4 | [Identity signals — device, fingerprint, duplicates](#4-identity-signals--device-fingerprint-duplicates) | layer 2 |
| 5 | [Tier assembly at install](#5-tier-assembly-at-install) | fusion |
| 6 | [The install-time money path](#6-the-install-time-money-path-what-happens-today) | **money today** |
| 7 | [Behavior layer — features and the v4 classifier](#7-behavior-layer--features-and-the-v4-classifier) | layer 3 |
| 8 | [Decision lifecycle, collection and cadence](#8-decision-lifecycle-collection-and-cadence) | layer 3 |
| 9 | [Finalization — the money and decision sweeps](#9-finalization--the-money-and-decision-sweeps) | **money today** |
| 10 | [The Stage-2 merge — `mergeConvV4()`](#10-the-stage-2-merge--mergeconvv4-designed-not-active) | **dormant** |
| 11 | [Runtime envelope and scheduling](#11-runtime-envelope-and-scheduling) | how it executes |
| 12 | [Invariants — claimed vs actually enforced](#12-invariants--claimed-vs-actually-enforced) | read before trusting |
| 13 | [Configuration reference](#13-configuration-reference) | every knob |
| 14 | [State vocabulary](#14-state-vocabulary) | every column and marker |
| 15 | [Operations — deploy, promote, roll back, diagnose](#15-operations--deploy-promote-roll-back-diagnose) | runbook |
| 16 | [Validation protocol and roadmap](#16-validation-protocol-and-roadmap) | what's next |
| 17 | [What changed from V4](#17-what-changed-from-v4) | migration notes |
| 18 | [Per-extension fraud implementation](#18-per-extension-fraud-implementation) | fleet map |
| 19 | [Conformance checklist](#19-conformance-checklist) | run before shipping |

---

## 0. Live state at a glance

Two sentences, because everything else depends on getting these right:

**Classification is v4, running in full shadow. Money is Stage 1, with a global timing-only 1-hour
payout hold.**

| Axis | Live state | Set by |
|---|---|---|
| Classifier | **v4-2** — behavior-only, no tier prior | `behavior_lib.php:430` → `classifyBehaviorV4()` |
| Enforcement | **OFF** — `$BEHAVIOR_ENFORCE`, `$DEFER_CONVERT`, `$DELAYED_CONVERSION` all `false` | `config.php:39-41` |
| Stage 2 (refusals) | **OFF** — `$BEHAVIOR_AN_ALLOWLIST = []` | `config.php:46` |
| Payout timing | **held 1 h, every network** — `$STAGE1_DELAY_CONV_AN = ['*']`, `_SEC = 3600` | `config.php:68-69` |
| Payout rescue window | **72 h** — `$S1_HOLD_PAY_WINDOW_SEC = 259200` | `config.php:89` |
| Scheduler | **inbound traffic** — post-flush inline slices; the live host has no cron | `ninja-adb21.php:3647`+ |

### 0.1 The one distinction everybody gets wrong

There are two independent switches and they are routinely confused.

| | `$STAGE1_DELAY_CONV_AN` | `$BEHAVIOR_AN_ALLOWLIST` |
|---|---|---|
| Controls | **WHEN** a payout fires | **WHETHER** a payout can be refused |
| Live value | `['*']` — every ad network | `[]` — no network |
| Wildcard support | yes, `'*'` | **no** — exact match only (`an_in_behavior_scope()`) |
| Effect today | postback fired ~1 h late instead of at install | none — the refusal branch is unreachable |
| Same recipients as before? | **yes**, identical gate | n/a |

The 1-hour hold is **Stage 1**. It changes timing and nothing else: same gate, same recipients,
minus only the users who uninstall inside the hour (`|s1_forfeit`). **No install can be refused
today.** Stage 2 requires putting a network into `$BEHAVIOR_AN_ALLOWLIST`; see §10.8 and §15.3.

### 0.2 What actually blocks money today

Shadow mode does **not** mean nothing blocks. Two install-time mechanisms refuse payment (and
neither is the tier layer); a third — `early_fraud` — detects in shadow and refuses nothing yet:

| Mechanism | Effect | Where |
|---|---|---|
| `evaluateRisk()` sets `conv = 2` on risk digits `3`,`4`,`7`,`8`,`9` | hard block at install, in shadow | §3.2 |
| `level !== "0"` | never paid, at any layer, ever | §3.3 |
| `early_fraud` T0 detection (2026-09-03, builds 25+26) | **SHADOW — blocks nothing yet.** Appends `\|early_fraud:<reasons>` to `fraud_flags` at install; enforcement exists as commented `EARLY_FRAUD_FLIP` lines (`ninja-adb21.php:3206-3284` active block; flip sites `:3316-3320`, `:3339-3344`, `:3410-3414`) | §5.3, changelog 2026-09-03 |

The **tier** (1/2/3), by contrast, is a pure label today — it gates nothing while `$pureShadow`
is true (§5.5). A hard VM that is `level = 0` with an `an` and a `cid` still converts. A consumer
VPN, which is only *Tier 2*, does not — because `vpn` sets risk digit `3`. Read §3.6 before
drawing any conclusion from a tier label.

### 0.3 How to read this document

- §§2–4 are the **input layers** — what the system measures.
- §§5–6 are the **install path** — what happens in the first request, including all money movement
  that occurs today.
- §§7–8 are the **behaviour layer** — what the system observes over 3 hours, and what it records.
- §9 is the **backstop** — the sweeps that finish every install.
- §10 is **designed but dark**. Every claim in it is prefixed accordingly.
- §§11–14 are **reference** — runtime, invariants, config, vocabulary.
- §§15–16 are **operational**.

Anything marked **DORMANT** or **NOT ACTIVE TODAY** describes code that exists and is tested but
cannot execute under the current configuration. Anything in a `> **Defect**` blockquote is a known
bug in the live code, documented rather than silently corrected.

---

## 1. Architecture — three layers, two axes

### 1.1 Two axes that must never fuse

The system answers two different questions and stores them in two columns. Fusing them was the
central defect of v3.

| Column | Question | Values |
|---|---|---|
| `decision` | **Who is this user?** — a behavioural fact | `pending` (transient) · `human` · `bot` · `deferred` (= terminal "unknown") |
| `conv` | **What did we do?** — a money record | `0` open/unattributed · `1`/`3` paid · `2` blocked/postback-failed · `4` closed-not-paid |

The dependency is one-way: **`conv` may read `decision`; `decision` never reads `conv`.** The
finalization sweep enforces this by resolving decisions with *no conv filter at all* (§9.6), which
is what fixed v3's leak where paid-but-silent installs never terminalized.

Full value semantics for both columns: §14.1 and §14.2.

### 1.2 Three layers

| Layer | Question | Signals | When measured | Feeds |
|---|---|---|---|---|
| **Hardware** (§2) | VM or physical machine? | GPU renderer, RAM, cores, battery, touch, WebGL, audio, screen, UA traps, webdriver | at install, **and retroactively on sync** when `diagnostic_date` is empty | the **tier** |
| **Network** (§3–4) | Suspicious infrastructure or identity? | proxycheck type/proxy/risk/ASN, provider keywords, extension-ID integrity, desktop-Linux UA, device-hash collisions | at install | the **tier** *and* `level` |
| **Behaviour** (§7–8) | Does it act like a human? | syncs, organic domains, revisits, auth markers, popup use, uninstall, farm-pattern collision | 0 → 3 h, 10-min cadence | **`decision`** |

> **Correction vs V4.** V4's layer table said hardware is measured "at install". It is also
> re-evaluated on the sync path (§2.7), and a retroactive `is_vm` change does **not** recompute the
> tier label already written to `fraud_flags`. Rows carrying `is_vm = 1` next to `tier1:high_trust`
> are expected, not corrupt.

### 1.3 Where the layers meet

- **Tier (1/2/3)** is the fusion of hardware + network, computed **once, at install** (§5). It is
  a label today.
- **`decision`** is behaviour only. It never reads the tier. `farm_collision` stays behavioural —
  it is a *browsing-pattern* collision, not an infrastructure signal.
- The tier is designed to act exactly once, at the **money merge** (§10) — which is dormant.
- `unknown` is stored as `'deferred'` in the DB, reusing the existing enum value (zero schema
  change).

*Why the layers are kept apart: a bot can fake the hardware or the behaviour, but rarely both
convincingly over time. Layer 1 is instant but spoofable; layer 3 is slow but hard to fake. Fusing
them into one score — as v3 did, by injecting a tier prior into the behaviour score — made
hardware suspicion unfalsifiable and terminalized 37% of installs inside 30 minutes, before any
behaviour existed to measure.*

### 1.4 The staged rollout model

Three stages, switched by the flags in §13.1 and scoped per ad network by
`$BEHAVIOR_AN_ALLOWLIST`.

| Stage | Flags | Payout timing | Behaviour layer |
|---|---|---|---|
| **1 — collect** *(live)* | all `false` | at install (currently held 1 h) | shadow: records `decision`, moves no money |
| **2 — refuse bots** | a flag `true` + network in allowlist | at the checkpoint | acts on `bot` verdicts and the T3 proof gate |
| **3 — withhold & decide** | `$DELAYED_CONVERSION` too | held until a verdict | full pipeline |

**Organic installs (`an = NULL`) are never paid** in any stage — there is no partner to owe a fee.
Their signals are still recorded for analytics. A network not in the allowlist always behaves as
Stage 1 regardless of the global flags; the per-row re-derivation in the sweep enforces this
(§9.6).

---

## 2. Hardware layer — `evaluateVirtualMachine()`

The hardware layer is a single pure function, `evaluateVirtualMachine(array $data): array` (`ninja-adb21.php:833`). It takes a flat array of device telemetry and returns a verdict pair. It touches no database, performs no I/O, and has no dependency on config flags — the same input always produces the same output. Its verdict is written to the `is_vm` / `vm_type` columns and is consumed in two decision sites: the `is_vm` Tier-3 test at `ninja-adb21.php:3151` and the `vm_type` `'physical:'` Tier-2 test at `ninja-adb21.php:3180`.

> *Rationale: the hardware layer answers "is this a real machine?" independently of the network layer and the behavior layer; the three verdicts are fused once, at tier classification.*

**NOT ACTIVE TODAY as a payment gate.** `is_vm` has never gated Stage-1 payment. Under full shadow (`$BEHAVIOR_ENFORCE = $DEFER_CONVERT = $DELAYED_CONVERSION = false`), the tier labels this function feeds are observational only — see `ninja-adb21.php:3193-3197`. A `webdriver_automation` hard-VM install that is `level === '0'` with a valid `an` + `cid` still converts and still gets paid.

### 2.1 Contract

**Signature:** `function evaluateVirtualMachine(array $data): array` — `ninja-adb21.php:833`.

**Return shape:** `['is_vm' => 0|1, 'vm_type' => string]`. Never null, never throws — every path including the `catch` returns this shape (`ninja-adb21.php:1079-1081`).

**Dual sourcing.** The same function is called with two structurally different arrays: fresh `$fingerprintData` posted by the extension (`$dataArray['ninja_fingerprint_data']`, `ninja-adb21.php:376`) and a whole `SELECT *` DB row (`$existingUser`). This works because the DB column names were chosen to match the frontend key names one-for-one. The only key that is never a column is `ip_timezone`, which every call site injects manually from `getRiskData()`.

| Field | Key(s) read | Line | Normalization | Null (rule skipped) when |
|---|---|---|---|---|
| GPU renderer | `gpu_renderer`, fallback `renderer` | 833 | `strtolower(trim((string)…))`, default `''` | `''` — the `$renderer !== ''` guards at 885 / 943 skip the rule |
| Screen resolution | `screen_res` | 834 | `strtolower(trim(…))`, default `''` | `''` |
| Client timezone | `timezone` | 835 | `strtolower(trim(…))`, default `''` | `''` |
| User agent | `user_agent` | 836 | `strtolower(trim(…))`, default `''` | `''` |
| RAM (GB) | `ram_gb`, fallback `deviceMemory` | 841-842 | `(float)` if numeric | value is `null`, `'unknown'`, `''`, or non-numeric |
| CPU cores | `cpu_cores` | 844-845 | `(int)` if numeric | same sentinel set |
| Touch points | `touch_points` | 847-848 | `(int)` if numeric | same sentinel set |
| Color depth | `color_depth` | 850-851 | `(int)` if numeric | same sentinel set |
| Battery charging | `bat_charging` | 854-862 | strict allow-list → `true` for `true`/`'true'`/`'1'`/`1`; `false` for `false`/`'false'`/`'0'`/`0` | anything else, incl. `'unknown'` and `''` |
| Battery level | `bat_level` | 864-865 | `(float)` if numeric | `'unknown'`, `''`, non-numeric |
| WebDriver flag | `webdriver_present` | 870 | `isset(…) ? (int) : 0` | absent/`NULL` → treated as `0`, not null |
| WebGL extension count | `gl_ext_count` | 972 | `isset(…) ? (int) : null` | absent/`NULL` → `null` |
| Audio fingerprint | `audio_fp` | 979 | raw, no cast | absent → `null` |
| Network timezone | `ip_timezone` | 989 | `(string)`, default `''` | `''` |

**Null skips the rule — it never counts against the install.** Every trap and every soft signal is guarded by an explicit `!== null` test, so missing telemetry produces a *cleaner* verdict, not a dirtier one. A DB row from before a column existed evaluates as `physical` on those fields.

> **Note** `webdriver_present` is the one exception to null-skip: `isset()` on a `NULL` column returns `false`, which collapses to `0` (not-automated). Absent WebDriver telemetry is read as "no automation present".

**Two derived OS predicates**, computed once at `ninja-adb21.php:842-843` from the lowercased UA and reused throughout:

- `$isLinuxDesktop = (strpos($ua,'linux') !== false && strpos($ua,'android') === false)`
- `$isDesktopOS = (strpos($ua,'windows') !== false || strpos($ua,'macintosh') !== false || $isLinuxDesktop)`

Note `$isDesktopOS` does **not** exclude ChromeOS (`cros` UAs contain `linux`), unlike the separate `$isDesktopLinuxUA` used for tier classification at `ninja-adb21.php:3119-3123`.

### 2.2 STEP 1 — hard traps (`ninja-adb21.php:871-938`)

Each trap returns immediately with `is_vm => 1`. Order matters: the first match wins, and nothing downstream (soft score, reducers, legacy-hardware tags) can rescue the install.

| # | Line | Condition (verbatim) | Threshold | `vm_type` | Rationale |
|---|---|---|---|---|---|
| 1a | 871 | `$webdriver === 1` | exact `1` | `webdriver_automation` | *`navigator.webdriver` is set only by an automation engine.* |
| 1b | 877 | `$ram <= 2.0 && $cores >= 8` | ≤2 GB **and** ≥8 cores | `vps_hardware_mismatch` | *No consumer machine ships 8 cores with 2 GB; that ratio is a sliced VPS.* |
| 1c | 883-887 | `strpos($renderer, $kw) !== false` for any `$kw` in `$hardGPU`, and `$renderer !== ''` | substring | `gpu_hypervisor` | *The hypervisor names itself in the WebGL renderer string.* |
| 1d-i | 893 | `$ram > 32` | >32 | `ram_spoof_max_cap` | *`navigator.deviceMemory` is spec-capped at 32; anything above is fabricated.* |
| 1d-ii | 898 | `!in_array($ram, [0.25, 0.5, 1, 2, 4, 8, 16, 32])` | set membership (loose) | `ram_spoof_power_2` | *Browsers round `deviceMemory` down to a fixed power-of-two ladder; off-ladder values are hand-set.* |
| 1d-iii | 903-905 | `$isFirefox \|\| $isSafari` (and `$ram !== null`) | any RAM value | `ram_spoof_unsupported_browser` | *Firefox and Safari do not implement `deviceMemory` at all, so any value from them is injected.* |
| 1d-iv | 912-917 | `$ram > 8` **and** `preg_match('/(?:chrome\|crios\|headlesschrome)\/(\d+)\./', $ua, $matches)` **and** `(int)$matches[1] < 147` | Chrome < 147 reporting >8 GB | `ram_spoof_old_version` | *Chrome only raised the reported ceiling above 8 in v147; an older build claiming 16 or 32 is spoofed.* |
| 1e | 923-925 | `$isIos && $batCharge !== null && $batLevel !== null` | both battery fields present | `ios_battery_trap` | *iOS/Safari removed `getBattery()`; battery data on an iOS UA means the UA is a lie.* |
| 1f | 929-933 | `strpos($ua, $kw) !== false` for any `$kw` in `$hardUA` | substring | `headless_ua_signature` | *Self-identifying automation UA.* |

**`$hardGPU` (`ninja-adb21.php:887`), verbatim and complete:**

```
'vmware', 'virtualbox', 'vbox', 'xen', 'qemu', 'parallels', 'hyper-v', 'hyperv'
```

**`$hardUA` (`ninja-adb21.php:933`), verbatim and complete:**

```
'headlesschrome', 'phantomjs', 'selenium', 'puppeteer', 'playwright', 'cypress'
```

**Allowed RAM ladder (`ninja-adb21.php:902`), verbatim and complete:** `[0.25, 0.5, 1, 2, 4, 8, 16, 32]`.

Three consequences of the 1d block that are easy to miss:

- **`ram_gb = 0` is a hard VM.** `0` is not on the ladder, so `deviceMemory: 0` — a common headless/spoofed default — returns `ram_spoof_power_2`. Only `null`-normalizing sentinels (`null`, `'unknown'`, `''`, non-numeric) skip the block; a numeric zero does not.
- **The cap check is deliberately ordered before the ladder check** (`ninja-adb21.php:896` comment). Both would fire on `ram_gb = 64`; running the cap first yields the more specific `ram_spoof_max_cap`.
- **The Firefox/Safari rule is broader than "spoof".** It fires on *any* non-null `$ram`, including a perfectly plausible `8`. `$isSafari` is `strpos($ua,'safari') !== false && strpos($ua,'chrome') === false && strpos($ua,'chromium') === false && strpos($ua,'crios') === false && strpos($ua,'edg') === false` (`ninja-adb21.php:908`), so a genuine Firefox or desktop-Safari install that reports any `ram_gb` at all is condemned as a hard VM — it is a browser-engine consistency check, not a numeric-plausibility check. Any extension build that populates `ram_gb` with a constant on non-Chromium engines would flag its entire Firefox/Safari cohort.

### 2.3 STEP 2 — soft score (`ninja-adb21.php:940-1013`)

`$score` starts at `0` (int) and `$reasons` at `[]`. Every signal that fires appends a token to `$reasons`; the joined token list becomes the tail of `vm_type` regardless of the final verdict.

| Signal | Line | Points | Exact condition | Reason token |
|---|---|---|---|---|
| Software renderer | 941-948 | **+2** | `$renderer !== ''` and `$renderer` contains any `$softGPU` keyword (`break` after first — scores once) | `soft_gpu:<keyword>` |
| Mobile UA, zero touch | 951-955 | **+2** | `$isMobileUA && $touch !== null && $touch === 0`, where `$isMobileUA = strpos($ua,'android') !== false \|\| strpos($ua,'mobile') !== false` | `mobile_no_touch` |
| Suspicious core count | 960-963 | **+1** | `$cores !== null && ($cores < 2 \|\| ($cores % 2 !== 0 && $isDesktopOS))` | `odd_cores:<n>` |
| Low color depth | 966-969 | **+1** | `$colorDepth !== null && $colorDepth < 24` | `low_color_depth:<n>` |
| Low WebGL extension count | 972-976 | **+1.5** | `$glExtCount !== null && $glExtCount > 0 && $glExtCount < 20 && $isDesktopOS` | `low_webgl_extensions:<n>` |
| Invalid audio stack | 979-983 | **+1.5** | `$audioFp === 'unsupported' \|\| $audioFp === 'error' \|\| $audioFp === '0'` (strict string comparison) | `invalid_audio_stack:<value>` |
| Timezone mismatch | 989-992 | **0** | `$tz !== '' && $ipTz !== '' && !timezonesEffectivelyEqual($tz, $ipTz)` | `timezone_mismatch_network_vs_client` |
| VM screen resolution | 997-1000 | **+1** | `$screenRes !== '' && in_array($screenRes, ['800x600','1024x768'], true)` | `vm_resolution:<res>` |
| Desktop on UTC | 1005-1008 | **+1** | `$isDesktopOS && ($tz === 'utc' \|\| $tz === 'gmt' \|\| strpos($tz,'etc/') === 0)` | `desktop_utc_timezone` |

**`$softGPU` (`ninja-adb21.php:945`), verbatim and complete:**

```
'swiftshader', 'llvmpipe', 'microsoft basic', 'mesa offscreen', 'webgl_disabled', 'webgl disabled'
```

**The timezone-mismatch signal scores ZERO today but still changes routing.** Since 2026-07-10 it adds no points (`ninja-adb21.php:989-996`) — but it still appends `timezone_mismatch_network_vs_client` to `$reasons`, and a non-empty `$reasons` list turns the STEP-5 fallthrough return into `'physical:' . $vmType` instead of the bare `'physical'` (`ninja-adb21.php:1077`). Because Tier 2 is tested with `strpos($vmResult['vm_type'], 'physical:') === 0` (`ninja-adb21.php:3180`), an otherwise spotless install whose only finding is a timezone mismatch is **demoted from Tier 1 to Tier 2**. Zero points is not zero effect.

> *Rationale (`ninja-adb21.php:990-992`): proxycheck geolocation is too coarse for mobile carriers and CGNAT, and the 2026-07-09 audit found this flag in nearly every soft-score false positive — so it was demoted to observational rather than deleted.*

`timezonesEffectivelyEqual()` (`ninja-adb21.php:791-803`) fails safe in three ways: either side empty returns `true`, a case-insensitive exact match returns `true`, an unresolvable `DateTimeZone` returns `true`. Otherwise it compares current UTC offsets, so `Europe/Paris` vs `Europe/Madrid` is *not* a mismatch.

### 2.4 STEP 3 — reducers (`ninja-adb21.php:1015-1041`)

All three reducers are inside a single `if ($score > 0)` block (`ninja-adb21.php:1021`) and are evaluated independently — a maximum of −3.

| Reducer | Line | Value | Exact condition | Reason token |
|---|---|---|---|---|
| Live discharging battery | 1019-1022 | **−1** | `$batCharge === false && $batLevel !== null && $batLevel >= 0.05 && $batLevel <= 0.97` | `reducer_battery` |
| Adequate RAM | 1025-1028 | **−1** | `$ram !== null && $ram > 4.0` | `reducer_adequate_ram` |
| Physical touch screen | 1031-1034 | **−1** | `$touch !== null && $touch > 0` | `reducer_touch` |

The battery reducer was cut from −2 to −1 and given the `0.05 … 0.97` plausibility window; a pinned `1.0` or `0.0` level no longer earns the discount.

**Reducers apply at ANY score > 0.** The pre-2026-07-10 code capped them at `$score <= 3`, which was asymmetric: a 3.5 score (reachable with a single 1.5-point signal plus a 2-point one) was unfalsifiable — no amount of physical-hardware evidence could touch it — while a plain 3 could be rescued all the way to 0. The comment at `ninja-adb21.php:1017-1020` records the trade explicitly: spoof risk is accepted, because an install that shaves itself below the VM line keeps every reason token in `vm_type`, so the `physical:` prefix still routes it to Tier 2 and the behavior layer decides.

`$score = max(0, $score)` floors the result at zero (`ninja-adb21.php:1041`), so reducers can never produce a negative score or offset a future signal.

### 2.5 STEP 4 — verdict (`ninja-adb21.php:1043-1048`)

```php
$vmType = implode('|', $reasons);
if ($score >= 3) {
    return ['is_vm' => 1, 'vm_type' => 'score_' . $score . ($vmType ? ':' . $vmType : '')];
}
```

The line is a hard `>= 3` on the **post-reducer** score. `$vmType` is built at line 1044, *after* STEP 3, so reducer tokens (`reducer_battery` and friends) appear in the string of a VM verdict too.

**`vm_type` is `score_<float>`, and fractional scores are routine.** Two signals carry 1.5 points (`low_webgl_extensions`, `invalid_audio_stack`), so `score_3.5` is an ordinary value; PHP's float-to-string conversion also renders a float `4.0` as `score_4`. Anything bucketing on the `vm_type` string must parse the number, not match a fixed set of literals — `explode(':', …)[0]` (the pattern `rerun-vm.php:328-329` uses for its category breakdown) yields distinct buckets `score_3`, `score_3.5`, `score_4`, `score_4.5`, … rather than one `score_*` bucket.

### 2.6 STEP 5 — legacy-hardware tags (`ninja-adb21.php:1050-1077`)

**NOT A VM VERDICT.** Since 2026-07-10 every STEP-5 return carries `is_vm => 0`. *Rationale (`ninja-adb21.php:1051-1053`): the mature old-laptop cohort survives 36h at ~71% and uninstalls at 15.6% — below Tier 1 — i.e. humans on old machines, not farm hardware.*

The whole block is gated on `if ($isDesktopOS)` (`ninja-adb21.php:1054`).

| Outcome | Line | Condition | Returned `vm_type` |
|---|---|---|---|
| `old_laptop` | 1063-1065 | `$isOldChrome \|\| $isAncientGPU` | `'physical:old_laptop' . ($vmType ? '\|' . $vmType : '')` |
| `old_laptop_specs` | 1068-1070 | `$ram !== null && $ram <= 4 && $cores !== null && $cores <= 4 && $screenRes === '1280x720'` | `'physical:old_laptop_specs' . ($vmType ? '\|' . $vmType : '')` |
| bare physical | 1073 | fallthrough | `'physical' . ($vmType ? ':' . $vmType : '')` |

- `$isOldChrome` — set when `preg_match('/Chrome\/(\d+)\./', $ua, $chromeMatches)` matches and `(int)$chromeMatches[1] <= 109` (`ninja-adb21.php:1057-1061`). *Chrome 109 was the last version for Windows 7/8.1.*
- `$isAncientGPU = (strpos($renderer, 'hd graphics 300') !== false || strpos($renderer, 'direct3d9ex') !== false)` (`ninja-adb21.php:1065`).

**The `'physical:' prefix contract.** Tier classification tests `strpos($vmResult['vm_type'], 'physical:') === 0` (`ninja-adb21.php:3180`) and labels the row `borderline_vm:<vm_type>` (`ninja-adb21.php:3192`). The colon is load-bearing: a completely clean install with an empty `$reasons` list returns the bare string `'physical'`, which does **not** match `'physical:'` and therefore stays eligible for `tier1:high_trust`. Every non-empty reason list, including a zero-point timezone mismatch, flips it to Tier 2.

**The three return shapes use inconsistent separators.** `old_laptop` and `old_laptop_specs` join their reason tail with `|`; the bare-physical return joins with `:`. So you get `physical:old_laptop|soft_gpu:llvmpipe` but `physical:soft_gpu:llvmpipe`. Any parser that splits `vm_type` on `:` must cope with both; splitting on the *first* colon and then on `|` is the only shape that handles all four return families (`score_*`, `physical:old_laptop*`, `physical:*`, `detection_error:*`).

> **Defect** The STEP-5 old-Chrome branch is dead code. `$ua` was lowercased at `ninja-adb21.php:840` (`strtolower(trim(...))`), but the regex at `ninja-adb21.php:1057` is `'/Chrome\/(\d+)\./'` — capital `C`, no `i` modifier. It can never match, so `$isOldChrome` is always `false` and `old_laptop` is only ever reachable via `$isAncientGPU`. The equivalent STEP-1d regex at `ninja-adb21.php:917` is correctly lowercase (`/(?:chrome|crios|headlesschrome)\/(\d+)\./`) and does match, which is why the same file appears to handle Chrome versions correctly elsewhere. The identical defect exists in the mirror at `rerun-vm.php:256`. Consequence: the `physical:old_laptop` tag is under-emitted, and those installs fall through to `old_laptop_specs` or to bare `physical`.

### 2.7 When it runs — call sites

Eight call sites in `ninja-adb21.php`, one of which is the install path; the other seven are retroactive or migration re-evaluations on the sync path.

| # | Line | Path | Guard | Writes |
|---|---|---|---|---|
| 1 | **3074** | **Install** — new row in `createUser()` | none (always runs) | `$vmResult` → the `is_vm` / `vm_type` fields of the INSERT (`ninja-adb21.php:3394-3395`), and `is_vm===1` drives the Tier-3 test at `3151` |
| 2 | 2725 | duplicate-fingerprint branch of `createUser()` | `empty($existingUser['diagnostic_date'])` (2721) | `UPDATE … SET is_vm, vm_type, diagnostic_date = NOW()` (2726-2728) |
| 3 | 2798 | duplicate-CID/SID branch — schema migration | `empty($existingUser['audio_fp']) && is_array($fingerprintData) && !empty($fingerprintData['audio_fp'])` (2783) | UPDATE at 2801-2819, plus local write-back at 2826-2827 |
| 4 | 2842 | duplicate-CID/SID branch — retroactive | `empty($existingUser['diagnostic_date']) \|\| $existingUser['diagnostic_date'] < '2026-05-25 00:00:00'` (2835) | UPDATE at 2843-2845, plus local write-back at 2846-2848 |
| 5 | 2946 | duplicate-IP branch — schema migration | same condition as #3 (2931) | UPDATE at 2949-2967, local write-back 2974-2975 |
| 6 | 2990 | duplicate-IP branch — retroactive | same condition as #4 (2983) | UPDATE at 2991-2993, local write-back 2994-2996 |
| 7 | 3476 | main sync path (`$coredetails` = known fingerprint) — schema migration | `$fpNeedsMigration && doesUserNeedFingerprintUpdate($existingUser, $fingerprintData)` (3463) | UPDATE at 3479-3499, local write-back 3502-3510 |
| 8 | 3523 | main sync path — retroactive | `empty($existingUser['diagnostic_date'])` (3517) | UPDATE at 3524-3526, **no** local write-back |

Every retroactive site injects the network timezone first (`$existingUser['ip_timezone'] = $riskData['timezone'] ?? ''`) so the observational timezone check has an input, and each is wrapped in `try { … } catch (Exception $eVm) { errorLog($pdo, $ip, 'vm-detect-update-error', …); }`.

Two behavioural asymmetries between the retroactive sites:

- **The re-run window differs.** Sites #2 and #8 fire only when `diagnostic_date` is empty. Sites #4 and #6 *also* re-fire for any row whose `diagnostic_date` predates `2026-05-25 00:00:00`. The same install can therefore be re-evaluated or not depending on which duplicate branch caught it.
- **Sites #2 and #8 do not write back into `$existingUser`.** The row in the database is updated, but the in-memory array used by `processUserLogic()` and `runBehaviorLayer()` later in the same request still holds the pre-evaluation `is_vm` / `vm_type`. Sites #4 and #6 do write back (2828-2830, 2976-2978).

**A retroactive `is_vm` flip does NOT recompute the tier label.** The tier string is computed only inside the new-install branch of `createUser()` (`ninja-adb21.php:3150-3204`) and written exactly once, by the INSERT at `ninja-adb21.php:3414` / `insertUserRecord()` (`1809-1815`). None of the seven retroactive/migration sites touches `fraud_flags`; the only other writer is the dormant enforcement block in `behavior_lib.php:566-604`, which *appends* a `|behavior_bot…` token and never rewrites the `tierN:` prefix. So a row that was clean at install and is later re-evaluated into a hard trap ends up carrying `is_vm = 1` alongside `fraud_flags = 'tier1:high_trust'`.

> **Note** Any query or dashboard that buckets installs by tier will disagree with a query that buckets by `is_vm`. `is_vm` is current; `fraud_flags` is a snapshot of install time. Neither is wrong — they answer different questions. Reconcile on `is_vm` when you want today's hardware verdict, on `fraud_flags` when you want what the system believed when the payout decision was made.

### 2.8 `rerun-vm.php` — table-wide backfill

`rerun-vm.php` carries a byte-for-byte copy of `evaluateVirtualMachine()` (`rerun-vm.php:47-281`) and of `timezonesEffectivelyEqual()` (`rerun-vm.php:30-42`). Diffing the two function bodies shows **no logic differences** — only shortened comments. The dead old-Chrome regex is duplicated at `rerun-vm.php:256`.

What it does (`rerun-vm.php:283-350`):

1. `SELECT * FROM ninja25 ORDER BY id ASC` (`rerun-vm.php:290`) — every row, unbuffered iteration.
2. For each row, `evaluateVirtualMachine($row)` (`rerun-vm.php:312`).
3. **Unconditionally** `UPDATE ninja25 SET is_vm = :is_vm, vm_type = :vm_type, diagnostic_date = NOW() WHERE id = :id` (`rerun-vm.php:301, 315-319`) — there is no change detection despite the `// Check if VM status changed` comment; every row is rewritten and every `diagnostic_date` is refreshed.
4. Prints totals and a category breakdown keyed on `explode(':', $result['vm_type'])[0]` (`rerun-vm.php:327-346`).

Because DB rows carry no `ip_timezone` column and the script injects none, the timezone check is skipped on every row of a backfill — `$ipTz` is `''`, so the condition at line 994 short-circuits. The old "timezone ratchet", which fabricated a mismatching `ip_timezone` for previously-flagged rows so a one-time geolocation error could never clear, was removed on 2026-07-10 (`rerun-vm.php:306-309`).

> **Note** `rerun-vm.php` has **no authentication guard of any kind**. There is no token check, no secret, no `REMOTE_ADDR` allowlist, no CLI-only gate — it detects CLI vs web only to choose a line-break character (`rerun-vm.php:284-285`). Anyone who can reach its URL triggers a full-table rewrite of `is_vm`, `vm_type` and `diagnostic_date`, refreshing `diagnostic_date` on every row and thereby permanently disabling the `empty(diagnostic_date)` retroactive re-evaluation guards at call sites #2 and #8 in §2.7. It also runs unbounded over the whole table with one UPDATE per row. Treat it as a manual, offline maintenance tool: it should not be deployed to the live document root.

### 2.9 The `detection_error:` fail-open

> **Defect** The catch-all at `ninja-adb21.php:1079-1081` returns
> `['is_vm' => 0, 'vm_type' => 'detection_error:' . substr($e->getMessage(), 0, 100)]`.
> Two things go wrong at once. First, `is_vm` is `0`, so an evaluator crash reads as "physical" — the function fails **open**. Second, the `vm_type` prefix is `detection_error:`, not `physical:`, so it fails the Tier-2 test at `ninja-adb21.php:3180` (`strpos(..., 'physical:') === 0`) just as it fails the Tier-3 test at `3151`. An install whose hardware evaluation threw an exception therefore lands in **`tier1:high_trust`** — the *most* trusted bucket — purely because the layer that was supposed to judge it crashed. The same fail-open exists in the mirror at `rerun-vm.php:278-280`, where it additionally overwrites the row's previously-good `vm_type` with the error string.
>
> A machine that reliably crashes the evaluator is a machine that reliably earns Tier 1. The minimal correction is to return the `physical:` prefix (routing the install to Tier 2, where behavior decides) rather than a prefix that matches no tier test at all.

This mirrors the `lookup_failed` fail-open in the network layer: two independent layers both degrade toward "clean and payable" when they break, and nothing alerts on either.

### 2.10 Worked example

**Payload** (posted as `ninja_fingerprint_data`, plus `ip_timezone` injected at `ninja-adb21.php:3073`):

```json
{
  "user_agent": "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/140.0.0.0 Safari/537.36",
  "gpu_renderer": "ANGLE (Google, Vulkan 1.3.0 (SwiftShader Device (LLVM 16.0.0)))",
  "ram_gb": 8, "cpu_cores": 4, "touch_points": 0, "color_depth": 24,
  "gl_ext_count": 12, "audio_fp": "124.04347527516074",
  "bat_charging": false, "bat_level": 0.62,
  "screen_res": "1920x1080", "timezone": "Europe/Paris",
  "ip_timezone": "America/New_York"
}
```

| Stage | Evaluation | Running score |
|---|---|---|
| Normalize (833-865) | `$renderer` lowercased contains `swiftshader`; `$ua` contains `windows` → `$isDesktopOS = true`; `$ram = 8.0`, `$cores = 4`, `$touch = 0`, `$colorDepth = 24`, `$batCharge = false`, `$batLevel = 0.62` | — |
| 1a-1c | webdriver absent → `0`; `8 <= 2.0` false; `swiftshader` is **not** in `$hardGPU` | pass |
| 1d | `8 > 32` false; `8` is on the ladder; UA contains both `safari` and `chrome` → `$isSafari = false`, `$isFirefox = false`; `8 > 8` false | pass |
| 1e-1f | not iOS; no `$hardUA` keyword | pass |
| 2 · soft GPU (941) | `swiftshader` matched | **+2** = 2 |
| 2 · mobile_no_touch (951) | UA has neither `android` nor `mobile` → skip | 2 |
| 2 · odd_cores (960) | `4 < 2` false, `4 % 2 !== 0` false | 2 |
| 2 · low_color_depth (966) | `24 < 24` false | 2 |
| 2 · low_webgl_extensions (973) | `12 > 0 && 12 < 20 && $isDesktopOS` | **+1.5** = 3.5 |
| 2 · invalid_audio_stack (980) | not `unsupported`/`error`/`0` | 3.5 |
| 2 · timezone mismatch (990) | Paris (+02) ≠ New York (−04) → token recorded | **+0** = 3.5 |
| 2 · vm_resolution (997) | `1920x1080` not in list | 3.5 |
| 2 · desktop_utc (1006) | not UTC/GMT/`etc/` | 3.5 |
| 3 · reducer_battery (1019) | `false === false`, `0.62` inside `[0.05, 0.97]` | **−1** = 2.5 |
| 3 · reducer_adequate_ram (1025) | `8.0 > 4.0` | **−1** = 1.5 |
| 3 · reducer_touch (1031) | `0 > 0` false | 1.5 |
| 4 · verdict (1042) | `1.5 >= 3` false → not a VM | — |
| 5 · legacy tags (1050) | `$isDesktopOS` true; old-Chrome regex is the dead branch (§2.6) so `$isOldChrome = false`; `$isAncientGPU = false`; `8 <= 4` false → fallthrough | — |

**Return (1073):**

```
['is_vm' => 0,
 'vm_type' => 'physical:soft_gpu:swiftshader|low_webgl_extensions:12|timezone_mismatch_network_vs_client|reducer_battery|reducer_adequate_ram']
```

**Downstream:** `is_vm === 1` is false, so the Tier-3 test at `ninja-adb21.php:3151` does not fire. `strpos($vmType, 'physical:') === 0` is true, so `$isTier2` is true (`ninja-adb21.php:3180`) and the row is stamped `fraud_flags = 'tier2:borderline_vm:physical:soft_gpu:swiftshader|…'` (`ninja-adb21.php:3192-3199`). **Payment is unaffected:** with all three behavior flags false, `$pureShadow` is true (`ninja-adb21.php:3292`), the tier branches at `3291` and `3300` are skipped, and the install runs the ordinary Stop Ads / Wonder conversion gate exactly as a Tier-1 install would. Two 1.5-point signals short of the line, a headless-Chrome software renderer, and a timezone that does not match the IP — and today the install still gets paid, with the evidence recorded and nothing else.

---

## 3. Network layer — risk lookup and the `level` string

The network layer is a single external lookup (proxycheck.io) whose result is compressed into one
string column, `level`. `level` is written once at install (`ninja-adb21.php:3366`) and is **never
recomputed** for an existing row — `processUserLogic()` reads it straight back out of the DB
(`ninja-adb21.php:2561`) and hands it to `updateUserRecord()` unchanged.

> *Rationale: `level` is the historical "Stop Ads / Wonder" gate. Every payout site in the codebase still defers to it, so it behaves as a permanent, install-time verdict rather than a live signal.*

### 3.1 getRiskData — the proxycheck.io v2 call

`getRiskData($ip)` — `ninja-adb21.php:1564-1595`.

| Property | Value |
|---|---|
| Endpoint | `http://proxycheck.io/v2/$ip?key={$proxycheck_key}&vpn=1&asn=1&risk=2` (`ninja-adb21.php:1567`) |
| Scheme | **plain HTTP**, not HTTPS |
| Key | `$proxycheck_key` from `config.php` (global) |
| `vpn=1` | return connection `type` / `proxy` classification |
| `asn=1` | return `asn`, `provider`, `organisation`, `hostname`, `isocode`, `timezone` |
| `risk=2` | return the full numeric `risk` score (0-100) |
| Transport | `callAPI('GET', …)` — `ninja-adb21.php:1085-1115` |
| Timeout | `CURLOPT_TIMEOUT = 5`, `CURLOPT_CONNECTTIMEOUT = 3` (`ninja-adb21.php:1107-1108`) |
| Failure signal | `curl_exec()` returns `false` → `callAPI` returns `null` (`ninja-adb21.php:1111-1114`) |

Fields the system consumes from `$response[$ip]`:

| Key | Used by |
|---|---|
| `provider` | digit 8 (banned-provider), `provider` column |
| `proxy` | digit 7, Tier-2 `proxy_flag` |
| `type` | digit 3, Tier-3 `isHardProxyType`, Tier-2 `vpn` |
| `risk` | digit 7, Tier-2 `proxy_flag` / `minor_risk_score` |
| `isocode` | `$pcmarket` (falls back to `$CfMkt`) — `ninja-adb21.php:3063` |
| `organisation` | digit 6 |
| `hostname` | digit 5 |
| `asn` | ASN blacklist lookup, `asn` column |
| `timezone` | injected as `ip_timezone` into `evaluateVirtualMachine()` (`ninja-adb21.php:3073`) |
| `lookup_failed` | `evaluateRisk()` early-return guard |

When `callAPI` returns `null`, every field is set to `null` and `'lookup_failed' => true`
(`ninja-adb21.php:1568-1580`). On any parsed response `'lookup_failed' => false`
(`ninja-adb21.php:1593`) — including a response whose body contains no `$ip` key at all (see §3.4).

`getRiskData()` is also called on non-install paths: retroactive VM detection and schema migration
in the dupFP / dupCID / dupIP branches (`ninja-adb21.php:2723`, `2795`, `2839`, `2943`, `2987`,
`3473`, `3520`). Those calls only use `timezone` (and `asn` on migration); they never touch `level`.

### 3.2 The `level` digit table

`evaluateRisk($riskData, $userAgent)` — `ninja-adb21.php:1610-1698`. It appends digits to a string
in fixed order and returns `['level' => …, 'conv' => …]`. If no digit is appended, `level` becomes
the literal `"0"` (`ninja-adb21.php:1694-1696`). `conv` is `0` unless a blocking digit set it to `2`.

| Digit | Trigger (exact) | Ref | Sets `conv=2`? |
|---|---|---|---|
| `4` | `$storeExtId !== '' && $shortExtId !== $storeExtId && $shortExtId !== $storeExtIdBing` | `1612-1617` | **Yes — BLOCKING** |
| `3` | `type` (lowercased) is exactly one of `vpn`, `tor`, `socks`, `socks4`, `socks5`, `http`, `https`, `compromised serv`, **or** contains `hosting` / `compute` / `data center` / `datacenter` | `1620-1637` | **Yes — BLOCKING** |
| `5` | `hostname` contains `google` (case-insensitive) **and** `provider !== 'Google Fiber Inc.'` | `1640-1643` | No — advisory |
| `6` | `organisation` contains `google` **and** does not contain `google fiber` | `1645-1648` | No — advisory |
| `7` | `proxy === 'yes'` **or** `(int)risk > $calibratedRiskThreshold` (default 33) | `1650-1656` | **Yes — BLOCKING** |
| `8` | `provider` matches the banned-origin list **and** `provider !== 'Google Fiber Inc.'` | `1666-1672` | **Yes — BLOCKING** |
| `9` | UA contains `linux` **and not** `android` **and not** `cros` (desktop Linux) | `1673-1683` | **Yes — BLOCKING** |
| `0` | no digit appended | `1685-1687` | n/a |

> **Note** Digits 5 and 6 are advisory in `conv` terms only. They still land in the `level` string, and `level !== "0"` blocks payment everywhere (§3.3) — so in practice a Google hostname or organisation is just as fatal to a payout as a hard block. `finalize_pending_installs.php:367-368` names them explicitly as a source of permanently-unpaid rows.

The banned-provider list, verbatim (`ninja-adb21.php:1667-1672`):

```php
$bannedOrigin = [
    'google', 'amazon.com', 'amazon technologies', 'firefox', 'microsoft', 'avast', 'mozilla',
    'as138544', 'asc-al', 'quadranet', 'cipherkey', 'clouvider', 'iastate',
    'multifi', 'readyserver', 'staicovici', 'plateau', 'aureon', 'yahoo',
    'metronet', 'texas', 'northside independent school district',
    'datacamp limited', 'the optimal link corporation', 'hot mobile ltd.'
];
```

Matching runs through `array_in_string($provider, $bannedOrigin)` (`ninja-adb21.php:739-748`),
which is `stripos($string, $value) !== false` per entry: **case-insensitive substring**, not
whole-word and not exact. Any provider string containing one of these fragments anywhere matches.

> **Note** The false positive is acknowledged in the code itself at `ninja-adb21.php:1678`: *"'texas' contains false-positive risk for standard Texas residential users (like Texas communications)."* Any provider named `Texas …` — an ordinary residential ISP — takes digit 8 and can never be paid.

Two carve-outs exist, both for Google Fiber: digit 5 and digit 8 both re-check
`$provider !== 'Google Fiber Inc.'` (exact string), and digit 6 re-checks
`strpos(strtolower($org), 'google fiber') === false`.

### 3.3 The money invariant — `level !== "0"` means never paid

**This is the single hardest rule in the system.** An install whose `level` column is anything other
than the string `"0"` is never converted, at any point in its lifetime, by any layer. Four
independent enforcement sites implement it, and each one re-reads `level` rather than trusting an
upstream decision.

| # | Site | Ref | Code | Effect |
|---|---|---|---|---|
| 1 | Install gate (`createUser`) | `ninja-adb21.php:3320` | `if (!is_null($an) && !is_null($cid) && $level === "0" && $conv !== 2)` | `processConversion()` (or the Stage-1 hold) is only reachable inside this branch. Non-`"0"` → `conv` stays 0, no postback, no `\|s1_hold` marker. |
| 2 | Behavioral layer, `human` verdict | `behavior_lib.php:541-542` | `$levelClean = ((string)($install['level'] ?? '') === '0'); if ($payable && !$uninstalled && $levelClean)` | A behaviorally-confirmed human on a dirty `level` is still never paid. **DORMANT** — the enclosing block requires `$BEHAVIOR_ENFORCE \|\| $DELAYED_CONVERSION` (`behavior_lib.php:529`), both `false`. |
| 3 | Stage-1 hold sweep | `finalize_pending_installs.php:294` | `if ((string)($held['level'] ?? '') !== '0') { $s1Stats['held']++; continue; }` | A held row whose `level` is dirty is left held and ages out to forfeit at `$S1_HOLD_PAY_WINDOW_SEC` (72h). **This one is LIVE** — it runs on every finalize pass. |
| 4 | Deferred benefit-of-the-doubt convert | `finalize_pending_installs.php:524-525` | `$levelClean = ((string)($row['level'] ?? '') === '0'); if (!empty($BEHAVIOR_ENFORCE) && !empty($DEFER_CONVERT) && $levelClean)` | **DORMANT** — requires both `$BEHAVIOR_ENFORCE` and `$DEFER_CONVERT`, both `false`. |

Comparison semantics matter: site 1 uses a bare strict `===` against the string `"0"` (no cast — `evaluateRisk()` always returns a string), while site 2 uses strict `===` after an explicit
`(string)` cast; sites 3 and 4 use `!==` / `===` likewise. There is no loose comparison, so a `NULL`
`level` column is **not** clean — it casts to `''`, which is `!== '0'`, and the row is never paid.

> **Note** `level` also gates two non-money paths, using **loose** comparison: `shouldEnableScript()` requires `$level == '0'` (`ninja-adb21.php:1871`) and the Yahoo/Google script injection requires `$level == 0` (`ninja-adb21.php:2640`). Those are monetisation-script decisions, not postbacks; do not confuse them with the four sites above.

### 3.4 Failure modes

> **Defect** `lookup_failed` fails **OPEN**. `evaluateRisk()` returns `['level' => '0', 'conv' => 0]` immediately when `lookup_failed` is truthy (`ninja-adb21.php:1615-1618`), *before* any digit is evaluated. A proxycheck outage — DNS failure, connection refused, or simply exceeding the 5s `CURLOPT_TIMEOUT` — therefore makes every install in that window look perfectly clean and fully payable. It also skips the desktop-Linux UA check (digit 9), which needs no network data at all. There is no counter, no alarm and no retry; the only trace is that `provider` / `type` / `risk` / `asn` are all `NULL` on those rows.

> **Defect** A second, quieter failure mode: an HTTP-200 **non-answer** is not flagged as `lookup_failed`. `callAPI()` returns `null` only when `curl_exec()` returns `false` (`ninja-adb21.php:1111`). A proxycheck response of `{"status":"denied","message":"…"}` (quota exhausted, key disabled, rate-limited) is a successful 200 with a body, so `getRiskData()` takes the parse path and every `$response[$ip][…] ?? null` resolves to `null` — with `'lookup_failed' => false` (`ninja-adb21.php:1593`). `evaluateRisk()` then runs normally against all-null data: `type` is `''` (no digit 3), `hostname` is `''` (no 5), `organisation` is `''` (no 6), `proxy` is not `'yes'` and `risk` casts to `0` (no 7), `provider` is `''` (no 8). Result: `level = "0"`, fully payable, and *indistinguishable in the DB* from a genuinely clean lookup. Unlike the `lookup_failed` path this one is not even conceptually handled anywhere.

> **Defect** `$storeExtIdBing = ''` (`ninja-adb21.php:355`) while `$storeExtId = 'ppfa'` (`ninja-adb21.php:354`). `$shortExtId = $extid ? substr($extid, 0, 4) : null` (`ninja-adb21.php:368`), so an install that sends **no `extid` at all** has `$shortExtId === null`. The digit-4 condition (`ninja-adb21.php:1621-1623`) is all strict comparisons: `'ppfa' !== ''` is true, `null !== 'ppfa'` is true, `null !== ''` is true. Every extid-less install therefore takes digit 4 **and** `conv = 2`, and is simultaneously labelled Tier 3 `extid_mismatch` by the identical condition at `ninja-adb21.php:3145-3147`. A missing field is treated as proof of a side-loaded build.

### 3.5 calibrated_risk_threshold

| Aspect | Detail |
|---|---|
| Default | `33` — hardcoded at `ninja-adb21.php:102` and re-defaulted at every read site |
| Written by | `generate_fraud_report.php:237-246`, emitted to `fraud_cache.json` at `generate_fraud_report.php:269-281` |
| Derivation | 95th percentile of the GOOD cohort's own `risk` scores, clamped to `[25, 50]`: `$idx = (int)floor($goodTotal * 0.95); max(25, min(50, $riskScores[$idx] ?? 33))` |
| Guard | Only recalculated when `$goodTotal >= 10 && $badTotal >= 10`; otherwise stays at the literal `33` |
| Loaded by | `ninja-adb21.php:215-223` — `$GLOBALS['calibrated_risk_threshold'] = $cacheData['calibrated_risk_threshold'] ?? 33`. If `fraud_cache.json` is missing, the file-load block is skipped entirely and the value stays `33`. |
| Read at | `ninja-adb21.php:1660` (digit 7) and `ninja-adb21.php:3173` (Tier-2 `proxy_flag` / `minor_risk_score` band) |

What it gates:

- **Digit 7 / `conv=2`** — `risk > threshold` is a hard payout block (`ninja-adb21.php:1662`).
- **Tier-2 `proxy_flag:<score>`** — `risk > threshold` (`ninja-adb21.php:3174-3176`), observational label.
- **Tier-2 `minor_risk_score:<score>`** — `risk > 0 && risk <= threshold` (`ninja-adb21.php:3185`), observational label.

> *Rationale: it is the only value `generate_fraud_report.php` still feeds back into the request path — the WOE model and per-row `fraud_score` were removed 2026-07-13, and the device/ASN blacklists were suspended 2026-07-09 (`generate_fraud_report.php:263-273`).*

> **Note** `fraud_cache.json` is not present in the repository checkout; it is generated on the live host. Absent the file, the request path runs on the `33` default with empty blacklists — which is exactly the intended state (§4.5).

> **WATCH (2026-09-02).** The GOOD/BAD cohort queries that feed the derivation carried **unquoted** `conv` comparisons until the quoted-enum fix (canonical: `generate_fraud_report.php:202` good cohort, `:220-224` bad cohort — see §4.5 and `DETECTION_CHANGELOG.md` 2026-09-02): `conv IN (1,3)` selected by enum **index**, i.e. values `'0'`,`'2'`, so the cohorts — and the 95th-percentile input — were built from the wrong populations. Every conv-derived stat in `fraud_report.json` **step-changes at the first post-fix run** (the numbers were wrong BEFORE, not after), and `calibrated_risk_threshold` — the only fraud-report output the live request path consumes — can move on that first corrected run. The `[25, 50]` clamp and the `33` fallback bound the blast radius; watch the emitted value anyway (§16.3).

### 3.6 Tier label vs. money — the relationship readers get wrong

Tier assignment (`ninja-adb21.php:3107-3204`) and the payout gate (`ninja-adb21.php:3286-3337`) are
**two separate mechanisms reading overlapping inputs**. The tier is a label written to
`fraud_flags`; the money is decided by `level` / `conv`. In full shadow mode
(`$pureShadow === true`, since all three behavior flags are `false`) the tier label has **zero**
effect on `conv` — the code goes straight to the `else` branch at `ninja-adb21.php:3310`.

The counter-intuitive consequence: several signals demoted to Tier 2 in the Phase-2 relabelling
still hard-block the payout via `level`, while the flagship Tier-3 signal (a hard VM) does not.

| Signal | Tier label | `level` digit | Payable today? |
|---|---|---|---|
| VPN connection type (`type === 'vpn'`) | Tier 2 (`vpn`) | `3` + `conv=2` | **No — blocked** |
| Proxy flag / `risk > threshold` | Tier 2 (`proxy_flag`) | `7` + `conv=2` | **No — blocked** |
| Banned provider keyword | Tier 2 (`banned_provider`) | `8` + `conv=2` | **No — blocked** |
| Desktop-Linux UA | Tier 2 (`linux_ua`) | `9` + `conv=2` | **No — blocked** |
| Hard proxy type (tor/socks/hosting/datacenter) | Tier 3 (`hard_proxy_type`) | `3` + `conv=2` | **No — blocked** |
| Extension-ID mismatch | Tier 3 (`extid_mismatch`) | `4` + `conv=2` | **No — blocked** |
| **Hard VM (`is_vm === 1`)** | **Tier 3 (`hard_vm`)** | **none** | **YES — paid normally** |
| Blacklisted device / ASN | Tier 3 | none | **YES — paid** (lists are empty anyway, §4.5) |
| Device collisions >= 2 | Tier 2 (`device_collisions`) | none | **YES — paid** |
| Borderline VM (`physical:*`) | Tier 2 (`borderline_vm`) | none | **YES — paid** |
| Google hostname / organisation | *no tier contribution* | `5` / `6`, no `conv` | **No — blocked by the level invariant** |

> *Rationale: `level` predates the tier system and was never rewired when Phase 2 (2026-07-10) demoted the circumstantial network signals. The comment at `ninja-adb21.php:3111-3113` is explicit — the relabelling is "LABELS ONLY: evaluateRisk / level / conv gates are byte-identical".*

> **Note** Read the last three rows together: a virtual machine that arrives on a clean residential IP with a valid `extid` gets a loud `tier3:hard_vm` label in `fraud_flags` **and gets paid**, whereas an ordinary human on a Texas ISP gets a mild `tier2:banned_provider` label **and never gets paid**. The label is not the decision.

## 4. Identity signals — device, fingerprint, duplicates

Two independent SHA-256 hashes are derived from the client telemetry, and they answer different
questions. `fingerprint` answers *"is this the same install?"* — it is keyed with `an`/`cid`/`sid`/`ip`
and is the client's identity token, echoed back as `ninja_fp`. `device_hash` answers *"is this the
same physical machine?"* — hardware only, deliberately **not** keyed with tracking params, so it can
see one machine reappear across campaigns.

### 4.1 Device hash v2

`generateDeviceHash(array $fpData, ?string $canvasHash)` — `ninja-adb21.php:1516-1535`. Plain
`hash('sha256', …)` over the components joined with the `'|||'` separator. No HMAC key.

| # | Component prefix | Source | Default when absent |
|---|---|---|---|
| 1 | `cpu_cores:` | `$fpData['cpu_cores']` | `unknown` |
| 2 | `ram_gb:` | `$fpData['ram_gb']` | `unknown` |
| 3 | `gpu_vendor:` | `$fpData['gpu_vendor']` | `unknown` |
| 4 | `gpu_renderer:` | `$fpData['gpu_renderer']` | `unknown` |
| 5 | `lang:` | `$fpData['lang']` | `unknown` |
| 6 | `color_depth:` | `$fpData['color_depth']` | `unknown` |
| 7 | `timezone:` | `$fpData['timezone']` | `unknown` |
| 8 | `canvas_hash:` | `$canvasHash` argument | `unknown` |
| 9 | `touch_points:` | `$fpData['touch_points']` | `0` |
| 10 | `screen_res:` | `$fpData['screen_res']` | `unknown` — **v2 addition** |
| 11 | `gl_ext_count:` | `$fpData['gl_ext_count']` | `unknown` — **v2 addition** |
| 12 | `audio_fp:` | `$fpData['audio_fp']` | `unknown` — **v2 addition** |
| 13 | `ua_norm:` | `normalizeUaForDeviceHash($ua)` | — **v2 addition** |

The `$canvasHash` argument is computed by the caller, not inside the function
(`ninja-adb21.php:3077-3079`): `hash('sha256', $fpData['canvas_data'])`, but only when
`canvas_data` is set **and** is not the literal string `'error'`; otherwise `null` → component 8
becomes `canvas_hash:unknown`. The UA is `$fpData['user_agent']`, falling back to
`$_SERVER['HTTP_USER_AGENT']`, falling back to `''` (`ninja-adb21.php:1518`).

**The UA normalizer** — `normalizeUaForDeviceHash(?string $ua)` at `ninja-adb21.php:1492-1514` —
returns `"<os>-<browser><major>"`, e.g. `windows-chrome141`, `mac-edg139`, `android-other`.

- OS, first match wins, on the lowercased UA: `windows` → `windows`; `android` → `android`;
  `cros` → `cros`; `linux` → `linux`; `iphone` **or** `ipad` → `ios`; `macintosh` → `mac`;
  otherwise `other`. *The `android`-before-`linux` ordering is deliberate — Android UAs contain "linux".*
- Browser, first match wins, in the fixed order `['edg', 'crios', 'firefox', 'chrome']`, matched with
  `preg_match('/' . $b . '\/(\d+)/', $u, $m)` against the **already lowercased** UA. Result is the
  token plus the captured major version, e.g. `edg139`. No match → `other`.
  *The order matters: Edge UAs contain `Chrome/` before `Edg/`.*

> *Rationale (from `ninja-adb21.php:1489-1491`): the raw UA string is too volatile to hash, but OS family + browser + major version adds real entropy — a farm cloning one machine shares it, while two real users on "identical" laptops rarely also share screen/GL/audio.*

**The v1/v2 break** (`ninja-adb21.php:1510-1515`, deployed 2026-07-12). Components 10-13 did not
exist in v1. Because they are inside the hashed string, **v2 hashes do not match v1 rows**: collision
counts restart from the deploy moment, and the nightly clone clustering mixes the two formats for up
to 30 days (an undercount — the conservative direction). Two consumers must therefore stay
component-identical to `generateDeviceHash`:

- `generate_fraud_report.php:75-93` (`computeDeviceHashFromRow`) — the backfill/cluster path. It is
  currently component-identical, including the shared `normalizeUaForDeviceHash`.
- The 7-day collision query below, which compares live hashes against stored ones.

### 4.2 The 7-day collision query

`ninja-adb21.php:3080-3095`, inside `createUser` and only reached on the true-new-install path.

```sql
SELECT COUNT(*) FROM `{$GLOBALS['table']}`
WHERE `device_hash` = ? AND `created_date` >= NOW() - INTERVAL 7 DAY
```

The count includes rows from the same device regardless of `an` / `cid` / `sid` / `ip`, and does not
exclude the row being created (which is inserted afterwards, at `ninja-adb21.php:3418`).

The result feeds exactly one consumer, the Tier-2 test at `ninja-adb21.php:3186`:

```php
$deviceCollisions >= 2   // reason string: 'device_collisions:' . $deviceCollisions
```

Threshold is `>= 2` prior same-hash installs within the window. This is a **label only** — it
contributes no `level` digit and no `conv` value, so it has no effect on payment (§3.6).

> *Rationale (`ninja-adb21.php:3085-3090`): v1 used an unbounded LIFETIME count, which made every common config collide as the table grew — 14,789 Tier-2 rows, 84% of all Tier 2, with a survival lift of -0.3 (pure noise) — and same-device reinstalls from a new IP/campaign self-collided. A real clone farm shows a BURST of same-hash installs, not slow accumulation.*

The whole signal is wrapped in `if ($GLOBALS['has_fraud_columns'])` (`ninja-adb21.php:3084`).

> **Defect** `$HAS_FRAUD_COLUMNS = false` in `config.php` silently disables far more than it appears to. `$GLOBALS['has_fraud_columns']` is set from that flag at `ninja-adb21.php:101` with no schema probe (the per-request `SHOW COLUMNS` was removed 2026-07-17). With it false: (a) `$deviceCollisions` stays `0`, so the `device_collisions` Tier-2 reason can never fire; (b) `insertUserRecord()` skips the `device_hash`, `fraud_score` and `fraud_flags` columns entirely (`ninja-adb21.php:1806-1818`) — the hash is computed and thrown away, and **no `fraud_flags` value is ever INSERTed**. Consequence (b) is a money bug: the Stage-1 hold marker `'|s1_hold'` (`ninja-adb21.php:3414`) lives in `fraud_flags`, and the finalize sweep selects held rows by `fraud_flags LIKE '%|s1_hold%'` (`finalize_pending_installs.php:200`). With the global 1h hold active (`$STAGE1_DELAY_CONV_AN = ['*']`), every install is held, no marker is written, the sweep finds nothing, and **every payout is silently lost** — with no error anywhere, because `conv` legitimately stays `0`. The flag is `true` today; treat flipping it as a payout kill-switch, not a schema toggle.

### 4.3 Fingerprint hash — an identity token, not a detection signal

`generateFingerprintHash($fpData, $an, $cid, $sid, $ip = null)` — `ninja-adb21.php:1462-1485`.
Returns `null` immediately if `$fpData` is falsy or not an array.

Unlike the device hash this is an **HMAC**: the key is `hash('sha256', $aes_key, true)` (the raw
binary digest of `$aes_key` from `config.php`), and the digest is
`hash_hmac('sha256', implode('|||', $components), $hmacKey)` (`ninja-adb21.php:1482-1483`).

Components, in order (`ninja-adb21.php:1467-1481`):

| # | Prefix | Default |
|---|---|---|
| 1-7 | `cpu_cores:`, `ram_gb:`, `gpu_vendor:`, `gpu_renderer:`, `lang:`, `color_depth:`, `timezone:` | `unknown` |
| 8 | `canvas_data:` — the **raw** canvas payload, *not* the SHA-256 of it | `unknown` |
| 9 | `touch_points:` | `0` |
| 10 | `audio_fp:` | `unknown` |
| 11 | `an:` | `none` |
| 12 | `sid:` | `none` |
| 13 | `cid:` | `none` |
| 14 | `ip:` | `unknown` |

Note the two deliberate divergences from the device hash: it hashes `canvas_data` raw (device hash
uses the pre-hashed `canvas_hash`), and it omits `screen_res`, `gl_ext_count` and `ua_norm` entirely.

Components 11-14 are what make it an identity token rather than a fraud signal.

> *Rationale (`ninja-adb21.php:1458-1461`): including the IP ensures two users with identical hardware and no tracking params (organic installs) still get distinct hashes.*

The corollary is that a **collision is never treated as suspicious**. It is proof of same-install, and
the code says so at `ninja-adb21.php:1537-1539`: the fingerprint hash is the user's identity token, so
a match always means "this is the same user" — no secondary checks. Its uses are:

- Stored in the `fingerprint` column and returned to the client as `ninja_fp` — the value the
  extension sends on every subsequent sync (`ninja-adb21.php:3421-3422`, `3440`).
- The primary lookup key on the sync path (`ninja-adb21.php:3449`).
- Recomputed and rewritten on schema migration, when new telemetry (`audio_fp`, `webdriver_present`,
  `gl_ext_count`) arrives for a row that lacked it (`ninja-adb21.php:2782`, `2930`, `3461`).
- Backfilled from stored DB columns when a matched row has no `fingerprint` value
  (`ninja-adb21.php:3036-3052`, `3424-3439`) — note this reconstruction passes `canvas_hash` into the
  `canvas_data` slot and omits `audio_fp`, so a backfilled hash is **not** byte-identical to one
  computed from a live request.

### 4.4 The duplicate short-circuits

`createUser()` runs three duplicate checks in sequence before it ever reaches the risk lookup. Each
one that hits **returns from `createUser` early**, and the entire block from Step 3 onward — the
proxycheck call, `evaluateRisk`, `evaluateVirtualMachine`, the device-hash collision query, the
blacklist checks, tier assembly, and the conversion gate — is never executed.

| Order | Check | Matches on | Function | Short-circuit |
|---|---|---|---|---|
| 1 | `dupFP` | `fingerprint = ?`, `ORDER BY id ASC LIMIT 1` | `checkDuplicateFingerprint` — `ninja-adb21.php:1540-1562` | `ninja-adb21.php:2716-2762`, `return $fingerprintHash` at `2761` |
| 2 | `dupCID` | `cid = ? AND sid = ?` (both must be non-empty) | `checkDuplicateCIDSID` — `ninja-adb21.php:1375-1394` | `ninja-adb21.php:2768-2907` |
| 3 | `dupIP` | `ip = INET6_ATON(?) AND (an <=> ?) AND instdom = ?`, `ORDER BY id ASC LIMIT 1` | `checkDuplicateIP` — `ninja-adb21.php:1396-1430` | `ninja-adb21.php:2911-3058` |

Details worth knowing:

- **dupFP** only runs when `$fingerprintData` is a non-empty array (`ninja-adb21.php:2711`). Its
  branch does a retroactive VM evaluation if `diagnostic_date` is empty (`ninja-adb21.php:2719-2732`,
  which is where the only `getRiskData()` call on this path lives), runs `processUserLogic`,
  increments `duplicate`, calls `updateUserRecord` / `logSyncArrival` / `runBehaviorLayer`, logs
  `dupFP-rebuilding-in-createUser`, and returns.
- **dupCIDSID** returns `['found' => false]` immediately if either `$cid` or `$sid` is falsy
  (`ninja-adb21.php:1377-1379`). The detection query selects on `cid` **and** `sid`, but the
  subsequent row fetch inside the branch selects on `cid` alone
  (`ninja-adb21.php:2773`: `WHERE cid = ? ORDER BY id ASC LIMIT 1`). If that fetch finds nothing, the
  code logs `dupCIDSID-userNotFound` and the request **`exit`s** (`ninja-adb21.php:2905-2906`) — no JSON response, and the dupIP check is never reached.
- **dupIP** uses the NULL-safe `<=>` operator on `an` so that organic (`an IS NULL`) installs match
  each other, and compares `instdom` against `serialize($instadom)` (`ninja-adb21.php:1398`). If the
  detection query found a row but the branch's own re-fetch does not, the request **`exit`s**
  (`ninja-adb21.php:3056-3057`) after logging `dupIP-userNotFound` — no JSON response at all.
- `$_GLOBALS['dupIPRowId']` is stamped at `ninja-adb21.php:2927` for downstream use.

> **Note** The consequence is structural, not incidental: **a returning or duplicate install is never re-classified and never re-evaluated for payment.** Its `level`, `conv`, `is_vm`, `fraud_flags` and `device_hash` are whatever was written at the original install. `processUserLogic` reads `level` and `conv` straight off the existing row (`ninja-adb21.php:2561-2562`) and its Step 1.5 comment (`ninja-adb21.php:2573-2576`) states the rule plainly: *"conv is decided once at install … and is NOT mutated on sync."* An install that was mis-evaluated during a proxycheck outage (§3.4) stays clean forever; an install that took a spurious digit 8 stays unpayable forever.

> **Note** Because dupFP fires before Step 3, a repeat install from a device whose hardware, `an`, `cid`, `sid` and `ip` all match an existing row never contributes to `device_collisions` — the collision query is on the other side of the short-circuit. The clone signal only sees devices arriving with *different* tracking params or IPs, which is exactly the farm behaviour it targets, but it means the count is not a count of installs from a device.

### 4.5 The manual blacklists

Two maps, initialised empty at `ninja-adb21.php:103-104` and overwritten from `fraud_cache.json` at
`ninja-adb21.php:215-223`:

```php
$GLOBALS['blacklisted_devices'] = $cacheData['devices'] ?? [];
$GLOBALS['blacklisted_asns']    = $cacheData['asns'] ?? [];
```

If the file is missing or does not decode to an array, both stay `[]`.

Each is consulted at `ninja-adb21.php:3084-3091`, and each accepts **two data shapes**:

| Shape | Test | Ref |
|---|---|---|
| Map — hash/ASN as the **key**, reason string as the value | `array_key_exists($deviceHash, …)` / `array_key_exists($riskData['asn'], …)` | `3081`, `3085` |
| Flat list — hash/ASN as a **value** | `in_array($deviceHash, …, true)` / `in_array($riskData['asn'], …, true)` (strict) | `3082`, `3086` |

The device check is additionally guarded by `$deviceHash` being truthy; the ASN check by
`!empty($riskData['asn'])`. A hit sets `$isDeviceBlacklisted` / `$isAsnBlacklisted`, which are
Tier-3 conditions (`ninja-adb21.php:3152-3153`) producing the `blacklisted_device` /
`blacklisted_asn` reasons (`ninja-adb21.php:3153-3154`).

**NOT ACTIVE TODAY.** `generate_fraud_report.php:269-276` emits both keys as empty arrays on purpose:

```php
$cacheData = [
    'devices' => [],
    'asns' => [],
    'calibrated_risk_threshold' => $calibratedRiskThreshold,
    'generated_at' => date('Y-m-d H:i:s')
];
```

The candidate lists are still computed and still written — to `fraud_report.json` as
`observed_device_clusters` and `observed_high_fraud_asns` (`generate_fraud_report.php:334-335`) —
for observation only. Device clusters are gathered with `HAVING cnt >= 3` over 30 days
(`generate_fraud_report.php:129-136`) and promoted to candidate at `count >= 5`
(`generate_fraud_report.php:254-258`); ASN candidates need `cnt >= 50` and a >= 75% flag rate
(`generate_fraud_report.php:158-175`).

> *Rationale (`generate_fraud_report.php:260-267`, suspension dated 2026-07-09): both lists were convicting ordinary users. The blacklisted-ASN cohort was 90% of all Tier 3 and uninstalled LESS than Tier 1 (14.7% vs 17.6%); the device-cluster-only cohort survived 36h BETTER than Tier 1 (81.4% vs 78.1%). The ASN "fraud rate" numerator was the system's own verdicts (`is_vm` / `conv` 2/4), not ground truth, so it blacklisted the largest residential ISPs — Comcast, AT&T, PLDT, BT.*

> **Retro-diagnosis (2026-09-02).** The mechanism was worse than "own verdicts". Until the quoted-enum fix, the numerator's `conv` comparisons were **unquoted numbers**, and `conv` is `enum('0','1','2','3','4')` — documented MariaDB/MySQL behavior reads an unquoted number as the enum **INDEX** (index 2 = value `'1'`, index 4 = value `'3'`). `is_vm = 1 OR conv = 2 OR conv = 4` therefore counted **PAID installs (`'1'`/`'3'`) as fraud**: an ISP's "fraud rate" was roughly its paid-conversion rate, so the largest residential ISPs scored highest — which **plausibly CAUSED** the 2026-07-09 residential-ASN blacklist incident. The comparisons are quoted since 2026-09-02 (canonical: `generate_fraud_report.php:156`; the same fix quotes all EIGHT sites — `:202`, `:220-224`, `:290`, `:307`). The Phase-0 suspension stays correct and stays in force; the outcome-based-numerator prerequisite in the Note below still applies before any re-arming.

> **Note** These maps are an **emergency lever only**: hand-populating `devices` or `asns` in `fraud_cache.json` re-arms Tier 3 for those keys on the next request (the file is re-read per request, no restart needed). Doing so has no payout effect while `$pureShadow` is true, since Tier 3 does not reach `conv` in shadow mode (§3.6). Per the code's own warning, do not re-wire the ASN list into the request path unless it is rebuilt on OUTCOME-based numerators (uninstall / 0-sync / 36h survival) with residential and mobile ASNs capped at "suspicious".

---

## 5. Tier assembly at install

The tier is computed in **Step 5** of `createUser()` (`ninja-adb21.php:3107-3204`), immediately after `evaluateRisk()` (`ninja-adb21.php:3066`) and `evaluateVirtualMachine()` (`ninja-adb21.php:3074`) have run and after the device hash / 7-day collision count has been resolved (`ninja-adb21.php:3064-3082`). Its only output is the string `$fraudFlags`, which is written verbatim into the `fraud_flags` column at Step 9.

> The tier is the fusion of the **hardware** layer (`is_vm` / `vm_type`) and the **network** layer (proxycheck `type` / `risk` / `proxy`, `level` digits, blacklists, device collisions) into one install-time label. Behavior is deliberately absent from it — behavior is a separate, later fact.

### 5.1 The tier predicates as evaluable pseudo-code

Six inputs are pre-computed just above the predicates:

| Variable | Definition | Ref |
|---|---|---|
| `$uaLower` | `strtolower($_SERVER['HTTP_USER_AGENT'] ?? '')` | `ninja-adb21.php:3118` |
| `$isDesktopLinuxUA` | `strpos($uaLower,'linux')!==false && strpos($uaLower,'android')===false && strpos($uaLower,'cros')===false` | `ninja-adb21.php:3105-3110` |
| `$tierTypeLower` | `strtolower((string)($riskData['type'] ?? ''))` | `ninja-adb21.php:3128` |
| `$isHardProxyType` | `$tierTypeLower !== ''` AND one of: `=== 'tor'`, `=== 'socks'`, `=== 'socks4'`, `=== 'socks5'`, `=== 'http'`, `=== 'https'`, `=== 'compromised serv'`, `strpos(...,'hosting')!==false`, `strpos(...,'compute')!==false`, `strpos(...,'data center')!==false`, `strpos(...,'datacenter')!==false` | `ninja-adb21.php:3129-3141` |
| `$isExtIdMismatch` | `$GLOBALS['storeExtId'] !== '' && $GLOBALS['shortExtId'] !== $GLOBALS['storeExtId'] && $GLOBALS['shortExtId'] !== $GLOBALS['storeExtIdBing']` | `ninja-adb21.php:3145-3147` |
| `$isDeviceBlacklisted` | `$deviceHash` truthy AND present as key or value in `$GLOBALS['blacklisted_devices']` | `ninja-adb21.php:3098-3101` |
| `$isAsnBlacklisted` | `!empty($riskData['asn'])` AND present as key or value in `$GLOBALS['blacklisted_asns']` | `ninja-adb21.php:3102-3105` |

*The hard-proxy list is `evaluateRisk()`'s type list (`ninja-adb21.php:1629-1642`) **minus `vpn`** — consumer VPN is common among ad-block users, so it is circumstantial, not hard.*

> **Note** Both blacklist maps default to `[]` (`ninja-adb21.php:103-104`) and are refilled only from `fraud_cache.json` (`ninja-adb21.php:219-220`). `generate_fraud_report.php` has emitted **empty** `devices`/`asns` maps since 2026-07-09 (Phase 0), so `blacklisted_device` / `blacklisted_asn` are **NOT ACTIVE TODAY** — they remain a manual emergency lever only.

The predicate chain, in evaluation order:

```text
isTier3 =  vmResult.is_vm === 1          # ninja-adb21.php:3151
        || isHardProxyType               # ninja-adb21.php:3138
        || isExtIdMismatch               # ninja-adb21.php:3139
        || isDeviceBlacklisted           # ninja-adb21.php:3154  (map empty today)
        || isAsnBlacklisted              # ninja-adb21.php:3155  (map empty today)

if (isTier3):
    fraudFlags = 'tier3:' + join('|', tier3Reasons)
    # SHORT-CIRCUIT: isTier2 stays false; NO Tier-2 reason is ever evaluated or recorded
else:
    riskScore               = (int)(riskData.risk ?? 0)                    # :3154
    calibratedRiskThreshold = GLOBALS.calibrated_risk_threshold ?? 33      # :3155
    isVpnType               = (tierTypeLower === 'vpn')                    # :3156
    isProxyFlagged          = (riskData.proxy === 'yes')
                              || (riskScore > calibratedRiskThreshold)     # :3157-3158
    isBannedProvider        = strpos(level, '8') !== false                 # :3159

    isTier2 =  strpos(vmResult.vm_type, 'physical:') === 0                 # :3162
            || isVpnType                                                   # :3163
            || isProxyFlagged                                              # :3164
            || isBannedProvider                                            # :3165
            || isDesktopLinuxUA                                            # :3166
            || (riskScore > 0 && riskScore <= calibratedRiskThreshold)      # :3167
            || deviceCollisions >= 2                                       # :3168

    if (isTier2): fraudFlags = 'tier2:' + join('|', tier2Reasons)          # :3181
    else:         fraudFlags = 'tier1:high_trust'                          # :3184
```

Three consequences of the structure, all load-bearing:

- **The short-circuit is total.** A Tier-3 install that is *also* on a VPN with `risk=40` records `tier3:hard_proxy_type:...` only. The Tier-2 evidence is lost, not appended. `$isTier2` is initialized `false` at `ninja-adb21.php:3160` and is never reached inside the Tier-3 branch.
- **`$isBannedProvider` reads the `level` string, not `$riskData`.** It is `strpos($level,'8')` — digit 8 written by `evaluateRisk()` (`ninja-adb21.php:1677`). Tier assembly therefore depends on `evaluateRisk()` having already run.
- **Tier 1 is a residual**, never a positive test: it is exactly "neither Tier 3 nor Tier 2".

`$calibratedRiskThreshold` defaults to `33` (`ninja-adb21.php:102`) and is overridden from `fraud_cache.json` when present (`ninja-adb21.php:221`). The same value is used inside `evaluateRisk()` (`ninja-adb21.php:1660`), so `proxy_flag` and `level` digit 7 always agree.

### 5.2 Complete reason-token table

Tokens are appended to `$reasons` in **fixed source order** and joined with `|`. The order below is the order they appear in a real string.

| # | Tier | Token (with value suffix) | Fires when | Ref |
|---|---|---|---|---|
| 1 | 3 | `hard_vm:<vm_type>` | `$vmResult['is_vm'] === 1`. Suffix is the raw `vm_type`, e.g. `score_4:soft_gpu:swiftshader\|vm_resolution:800x600\|desktop_utc_timezone` | `ninja-adb21.php:3164` |
| 2 | 3 | `hard_proxy_type:<type>` | `$isHardProxyType`. Suffix is `$tierTypeLower` (already lowercased), e.g. `hosting`, `socks4`, `compromised serv` | `ninja-adb21.php:3165` |
| 3 | 3 | `extid_mismatch` | `$isExtIdMismatch` — the install is not the store build. No suffix | `ninja-adb21.php:3166` |
| 4 | 3 | `blacklisted_device` | `$deviceHash` in `$GLOBALS['blacklisted_devices']`. **NOT ACTIVE TODAY** (map ships empty) | `ninja-adb21.php:3167` |
| 5 | 3 | `blacklisted_asn` | `$riskData['asn']` in `$GLOBALS['blacklisted_asns']`. **NOT ACTIVE TODAY** (map ships empty) | `ninja-adb21.php:3168` |
| 1 | 2 | `borderline_vm:<vm_type>` | `vm_type` starts with `physical:`. Suffix is the whole `vm_type`, e.g. `physical:old_laptop`, `physical:timezone_mismatch_network_vs_client`, `physical:old_laptop_specs\|low_color_depth:16` | `ninja-adb21.php:3192` |
| 2 | 2 | `vpn` | `$tierTypeLower === 'vpn'`. No suffix | `ninja-adb21.php:3193` |
| 3 | 2 | `proxy_flag:<risk>` | `proxy === 'yes'` **or** `risk > threshold`. Suffix is `$riskScore` — note the token name says "flag" but the suffix is the *score*, so `proxy_flag:0` is a legitimate value (proxy=yes, risk 0) | `ninja-adb21.php:3194` |
| 4 | 2 | `banned_provider` | `level` contains digit `8`. No suffix | `ninja-adb21.php:3195` |
| 5 | 2 | `linux_ua` | desktop Linux UA (not Android, not ChromeOS). No suffix | `ninja-adb21.php:3196` |
| 6 | 2 | `minor_risk_score:<risk>` | `0 < riskScore <= threshold`. Suffix is `$riskScore` | `ninja-adb21.php:3197` |
| 7 | 2 | `device_collisions:<n>` | `$deviceCollisions >= 2`. Suffix is the count | `ninja-adb21.php:3198` |
| — | 1 | `high_trust` | residual; the only Tier-1 token, always alone | `ninja-adb21.php:3202` |

`$deviceCollisions` is `COUNT(*)` of rows with the same `device_hash` and `created_date >= NOW() - INTERVAL 7 DAY` (`ninja-adb21.php:3090-3092`). *A real clone farm shows a burst of same-hash installs; the old unbounded lifetime count just accumulated popular hardware.* The count is only queried when `$GLOBALS['has_fraud_columns']` is true (`ninja-adb21.php:3084`) — with the flag off, `$deviceCollisions` stays `0` and the token can never fire.

Both `proxy_flag:<risk>` and `minor_risk_score:<risk>` can never co-fire on the score alone (their ranges are disjoint), but `proxy === 'yes'` with `0 < risk <= 33` fires **both**, producing `proxy_flag:12|minor_risk_score:12`.

### 5.3 The `fraud_flags` string grammar

```text
fraud_flags := tier_prefix reasons [ '|early_fraud:' ef_reasons ] [ '|s1_hold' ]
tier_prefix := 'tier3:' | 'tier2:' | 'tier1:'
reasons     := reason ( '|' reason )*        # Tier 1: always exactly 'high_trust'
ef_reasons  := ef_reason ( '+' ef_reason )*  # 'tzdc' | 'fpvote' | 'ipstack'  (2026-09-03, builds 25+26)
```

The tier part is built at `ninja-adb21.php:3169` / `:3199` / `:3202`. The `|early_fraud:` marker (2026-09-03) is appended right after tier assembly by the shadow detection block (`ninja-adb21.php:3269-3277`, `+`-joined reasons, guarded by a 218-char cap so every later marker rewrite stays under the `varchar(255)` — see the changelog entry for the cap math). The `|s1_hold` suffix is **not** part of tier assembly — it is appended at write time in the Step 9 field array (the commented line above it is enforcement flip 3/3):

```php
'fraud_flags' => $fraudFlags . ($s1DelayHold ? '|s1_hold' : '')   // ninja-adb21.php:3414
```

Real strings produced by the live code:

```text
tier1:high_trust|s1_hold
tier2:vpn|proxy_flag:57
tier2:borderline_vm:physical:old_laptop|linux_ua|minor_risk_score:11
tier2:borderline_vm:physical:timezone_mismatch_network_vs_client|device_collisions:5|early_fraud:tzdc|s1_hold
tier3:hard_vm:score_4:soft_gpu:swiftshader|vm_resolution:800x600|desktop_utc_timezone|hard_proxy_type:hosting
```

Two grammar hazards follow from the flat `|` separator:

- `hard_vm:` and `borderline_vm:` suffixes may themselves contain `|` (the VM evaluator joins its own reasons with `|` — `ninja-adb21.php:1044`, `:1068`, `:1073`). **Reason tokens are therefore not safely splittable on `|`.** Match on known token prefixes instead.
- Marker matching is substring-based everywhere downstream (`finalize_pending_installs.php:200`, `:234`, `:303`). The lifecycle markers are chosen so no marker is a substring of another: `|s1_hold` → `|s1_paid`, `|s1_forfeit`, or `|s1_refused`. The `|early_fraud:` marker (2026-09-03) obeys the same rule — verified against every existing token family, including `vpn` vs `fpvote` — and is **immutable once written**: nothing rewrites it; it is the permanent record that T0 detection fired. Consumers match `LIKE '%early_fraud%'`.

Marker lifecycle, all rewrites done by `finalize_pending_installs.php`:

| From | To | Meaning | Ref |
|---|---|---|---|
| `\|s1_hold` | `\|s1_paid` | atomic claim, then the postback fires | `finalize_pending_installs.php:301-303` |
| `\|s1_hold` | `\|s1_forfeit` | uninstalled, or older than `$S1_HOLD_PAY_WINDOW_SEC` (259200 = 72h) | `finalize_pending_installs.php:231-234` |
| `\|s1_hold` | `\|s1_refused` | Stage-2 merge said do-not-pay. **NOT ACTIVE TODAY** — gated on `an_in_behavior_scope()`, and `$BEHAVIOR_AN_ALLOWLIST = []` | `finalize_pending_installs.php:276-283` |

### 5.4 Principle and history

**Strictness proportional to false-positive rate.** Tier 3 is reserved for evidence a human essentially cannot produce; anything merely *correlated* with fraud is Tier 2, where the behavior layer is designed to get the final say at the merge. Carried from the superseded V4 spec, one line per rule:

| Rule | Rationale |
|---|---|
| Hard VM traps stay Tier 3 | *Webdriver, hypervisor GPU, VPS mismatch, RAM spoofs, headless UA, iOS battery — a real user's browser does not emit them.* |
| `tor` / `socks*` / `http(s)` / `compromised` / hosting / compute / datacenter stay Tier 3 | *Residential ad-block users do not install from datacenter infrastructure.* |
| `vpn` is Tier 2, not Tier 3 | *Consumer VPN is common among ad-block users; its mature cohort survives 36h at 72.2% against Tier 1's 78.1%.* |
| `extid_mismatch` stays Tier 3 | *Distribution integrity, not fraud detection — the build is not ours.* |
| `linux_ua` is Tier 2 | *The mature Linux/other cohort survives 36h at 81.4% — better than Tier 1.* |
| `borderline_vm:physical:*` is Tier 2 | *Old-laptop and timezone-mismatch cohorts uninstall at 15.6%, below Tier 1's 17.6% — humans on old machines.* |
| `device_collisions` needs ≥2 in 7 days | *Popular hardware collides on identical hashes; lifetime counting made 84% of Tier 2 with a −0.3 survival lift.* |
| Blacklists are manual-only | *The nightly generator scored ASNs on the system's own verdicts and blacklisted the 30 largest residential ISPs.* |

> Historical note (2026-07-09/10, three phases). **Phase 0** suspended the nightly ASN/device auto-blacklists: 69.7% of installs were Tier 3 and 90.3% of those carried `blacklisted_asn`; `generate_fraud_report.php` now emits empty maps and the candidates land in `fraud_report.json` as observation only. **Phase 1** de-fanged `evaluateVirtualMachine()`: `old_laptop` / `old_laptop_specs` return `is_vm=0` with a `physical:` prefix, `timezone_mismatch_network_vs_client` stopped scoring (+2.0 removed), and score reducers now apply at any score. **Phase 2** rewrote Step 5 into its present shape — Tier 3 reduced to hard VM + hard proxy + extid + manual blacklists, with `vpn` / `proxy_flag` / `banned_provider` / `linux_ua` demoted to Tier 2 and given explicit reason tokens (`hard_proxy_type:<type>` and `extid_mismatch` replaced the old blanket `risk_level:<level>`). All three phases were **labels only**: `evaluateRisk()`, `level` and every conversion gate were byte-identical before and after. Tier labels stamped before the Phase-2 deploy are legacy and must not feed tier-segmented analysis. *(Retro-diagnosis 2026-09-02: the Phase-0 numerator was not merely self-confirming — its unquoted enum comparisons made the `conv` half select the PAID cohort; see §4.5.)*

### 5.5 What the tier does NOT do today

**The tier is a LABEL. It gates nothing.**

Both tier-gating branches of the conversion decision are guarded by `!$pureShadow` (`ninja-adb21.php:3295`, `:3304`), and `$pureShadow` is unconditionally true on the live host (see §6.1). Consequences:

| Statement | Live truth |
|---|---|
| Tier 3 sets `conv = 4` (fraud-detection closure; **was `2` until 2026-09-03** — conv-attribution convention, §14.1) | **NOT ACTIVE TODAY** — branch is `if (!$pureShadow && $isTier3)` (`ninja-adb21.php:3295`) |
| Tier 2 defers the payout to the behavior layer | **NOT ACTIVE TODAY** — branch is `elseif (!$pureShadow && $isTier2)` (`ninja-adb21.php:3304`) |
| Tier changes `ninja_sync` (3h for T3, 10min for T2) | **NOT ACTIVE TODAY** — and overwritten anyway (§6.3) |
| Tier is written to `fraud_flags` and read by the dashboard, the report generators and `replay_v4.php` | **ACTIVE** |

> **Note** "The tier blocks nothing" is not "nothing blocks at install." **`level` does.** The Stage-1 gate requires `level === "0"` (`ninja-adb21.php:3320`), and `evaluateRisk()` writes a non-`0` level for extid mismatch (digit `4`), hard/VPN proxy types (`3`), Google hostname (`5`), Google org (`6`), proxy/risk (`7`), banned provider (`8`) and desktop Linux (`9`). A `tier3:hard_proxy_type:hosting` install is unpayable — not because it is Tier 3, but because its `level` contains `3`. See **§3.6** for the full `level` semantics and the enforcement sites.

The practical overlap: every Tier-3 reason except `hard_vm` and the two dormant blacklists also writes a `level` digit, and `vpn`, `proxy_flag`, `banned_provider` and `linux_ua` do too. The tier reasons that carry **no** payment consequence today are exactly `hard_vm:*`, `blacklisted_device`, `blacklisted_asn`, `borderline_vm:*`, `minor_risk_score:*` and `device_collisions:*`.

---

## 6. The install-time money path (what happens today)

### 6.1 `$pureShadow` and the three branches

```php
$pureShadow = empty($BEHAVIOR_ENFORCE) && empty($DEFER_CONVERT) && empty($DELAYED_CONVERSION);
// ninja-adb21.php:3292
```

All three flags are `false` in `config.php` (`$BEHAVIOR_ENFORCE`, `$DEFER_CONVERT`, `$DELAYED_CONVERSION` — `config.php:39-41`), so **`$pureShadow` is `true` on every live request**. It is doubly guaranteed: `createUser()` also force-clears `$DELAYED_CONVERSION` for any network outside the behavior allowlist (`ninja-adb21.php:2685-2687`), and `$BEHAVIOR_AN_ALLOWLIST = []` (`config.php:46`) makes `an_in_behavior_scope()` return `false` for every `an` (`behavior_lib.php:223-227`).

The same expression is computed in `logSyncArrival()` (`ninja-adb21.php:2526`) and in `finalize_pending_installs.php:563`.

| Branch | Condition | `$conv` | `ninja_sync` set in-branch | Live? | Ref |
|---|---|---|---|---|---|
| A — Stage-2 hard block | `!$pureShadow && $isTier3` | `4` *(was `2`; 2026-09-03 convention)* | `10800000` (3h) | **NOT ACTIVE TODAY** | `ninja-adb21.php:3295-3303` |
| B — Stage-2 deferred | `!$pureShadow && $isTier2` | `0` | `600000` (10 min) | **NOT ACTIVE TODAY** | `ninja-adb21.php:3304-3309` |
| C — Stage-1 gate (any tier) | else | `0`, then `2` if `$riskEvaluation['conv'] == 2`, then possibly `processConversion()`'s return | `10800000` (3h) | **ALWAYS** | `ninja-adb21.php:3310-3337` |

Branch C is the only reachable branch. `$conv` starts at `0` (`ninja-adb21.php:2690`), is re-zeroed at `:3312`, and is set to `2` when `evaluateRisk()` returned `conv == 2` (`ninja-adb21.php:3313-3315`) — that is, when `level` picked up digit `4`, `3`, `7`, `8` or `9`.

### 6.2 The Stage-1 conversion gate, verbatim

```php
if (!is_null($an) && !is_null($cid) && $level === "0" && $conv !== 2) {   // ninja-adb21.php:3320
    if (s1_delay_in_scope($an)) {                                        // ninja-adb21.php:3330
        $s1DelayHold = true;                                             // ninja-adb21.php:3331
    } else {
        $conv = processConversion($an, $cid, $sid, $po);                  // ninja-adb21.php:3333
    }
}
```

All four conjuncts must hold to reach a payout path at all:

| Conjunct | Meaning | Failure outcome |
|---|---|---|
| `!is_null($an)` | an ad network is attributed (from POST body or the `an` cookie, `ninja-adb21.php:2699-2701`) | organic install — `conv` stays `0`, no marker, never paid |
| `!is_null($cid)` | a click id exists to post back | `conv` stays `0`, never paid |
| `$level === "0"` | strict string comparison — **any** risk digit disqualifies | `conv` stays `0` (or `2` if `evaluateRisk` said so) |
| `$conv !== 2` | `evaluateRisk()` did not already condemn | already `2` |

*`level !== "0"` means the install is never paid, at this site and at every other enforcement site — including the finalize sweep's parity check (`finalize_pending_installs.php:294-297`).*

`s1_delay_in_scope()` (`behavior_lib.php:237-241`) returns true when `$an` is non-empty and `$STAGE1_DELAY_CONV_AN` contains `'*'` or the exact `$an`. Live config is `$STAGE1_DELAY_CONV_AN = ['*']` (`config.php:68`), so **the diversion is taken for every attributed install**. Today, `processConversion()` at `ninja-adb21.php:3333` is **dead code on the install path** — nothing pays at install.

> **Note** This is the **STAGE-1 DELAY, which is TIMING ONLY.** It is not Stage 2. Same gate, same recipients, same postback URL, same `po` — the postback simply fires ~1h later from `finalize_pending_installs.php` instead of inline. Stage 2 (the ability to *refuse* an install) lives behind `$BEHAVIOR_AN_ALLOWLIST`, which is `[]` — **STAGE 2 IS OFF and no install can be REFUSED today**. `$STAGE1_DELAY_CONV_SEC = 3600` (1h hold), `$STAGE1_DELAY_CONV_SEC_AN = []` (no per-network override), `$S1_HOLD_PAY_WINDOW_SEC = 259200` (72h payable window). Per-network duration resolution is `s1_delay_seconds()` (`behavior_lib.php:248-252`).

> **Note** The live host (Infomaniak) has **no cron**. `finalize_pending_installs.php` runs **in-process, post-flush**, driven by inbound traffic (`ninja-adb21.php:3647-3811`). With a global hold in force, that in-process runner carries 100% of payout traffic: if traffic stops, held rows stop being paid and age out to `|s1_forfeit` at 72h. Any prose describing "the 15-min cron" as the live scheduler is wrong.

### 6.3 The `ninja_sync` overwrite

> **Defect** All three branches of the decision block set `$jsonResponse['ninja_sync']` (`ninja-adb21.php:3303` = 10800000, `:3309` = 600000, `:3336` = 10800000), and a few lines later it is overwritten unconditionally:
> ```php
> if ($pureShadow || (int)$conv !== 2) {
>     $jsonResponse['ninja_sync'] = BEHAVIOR_COLLECT_SYNC_MS;   // ninja-adb21.php:3350-3352
> }
> ```
> Since `$pureShadow` is always `true`, the left disjunct alone makes this fire on **every** install. `BEHAVIOR_COLLECT_SYNC_MS` is `600000` (`ninja-adb21.php:87`). **The live `ninja_sync` value returned at install is therefore always `600000` (10 minutes), regardless of tier or `conv`.** The three per-branch assignments above are dead stores. The intent was Stage-1 "collect from everyone, including hard-blocked" — but the code as written also nullifies the Stage-2 cadences it was meant to preserve.

### 6.4 `processConversion()` — full contract

`ninja-adb21.php:1706-1768`. Signature `processConversion($an, $cid, $sid, $po)`.

> **Defined twice in the same deployment.** In **both** reference backends (Ninja
> and Ghost — each a modular layout: endpoint + `behavior_lib.php` +
> `finalize_pending_installs.php` + crons) the money functions (`processConversion`,
> `convPixel`, `sendConversion`, the 7thSense payout call) appear in **two** files:
> in the **sync endpoint** (`ninja-adb21.php` for Ninja / `ext-server.php` for Ghost
> — the copies §6 cites) **and** in the shared **`behavior_lib.php`**, whose
> definitions are wrapped in `if (!function_exists(...))` guards (`behavior_lib.php:792`)
> so the library can run **standalone** under the finalize sweep and the crons. That
> guarded library copy is what §13/§14 cite (e.g. the 7thSense call: endpoint
> `ninja-adb21.php:1722`, library `behavior_lib.php:793`). Both citations are
> correct — the endpoint copy runs on the sync path, the library copy under finalize.

**Guards** (both return `0` — "not converted", never paid):

| Guard | Condition | Ref |
|---|---|---|
| Organic | `empty($an)` | `ninja-adb21.php:1708-1711` |
| No attribution | `empty($cid)` | `ninja-adb21.php:1712-1717` |

*Centralized here so every caller — install, behavior layer, finalize — is protected identically.*

**The 7thsense API call** (`ninja-adb21.php:1721-1736`):

| Property | Value |
|---|---|
| Host | `https://api.7thsense.media` |
| URL | `$host . "/api/v1/payouts/$sid/conversion"` — note the path key is `sid`, not `cid` |
| Method | GET (no `CURLOPT_POST`) |
| Auth | header `Authorization: Bearer $token_7thsense` (`config.php:30`) |
| Timeouts | `CURLOPT_TIMEOUT` 10, `CURLOPT_CONNECTTIMEOUT` 5 |

**`$conversionValue` resolution** (`ninja-adb21.php:1738-1754`):

| Response | `$conversionValue` |
|---|---|
| `curl_exec` returns `false` (network failure/timeout) | `1` — **fails toward paying** |
| JSON has `statusCode` and it is `!== 200` | `1` — **fails toward paying** |
| anything else | `(int)$response` — the raw body cast to int, so a non-numeric body yields `0` |

**Return values:**

| Return | Produced by | Meaning for `conv` |
|---|---|---|
| `0` | `empty($an)` or `empty($cid)` guard; or `(int)$response === 0` falling through to `return $conversionValue` | not converted |
| `1` | `$conversionValue == 1` **and** `sendConversion(...) == 1` (`ninja-adb21.php:1757-1759`) | paid — postback returned HTTP 200 |
| `2` | `$conversionValue == 1` **and** `sendConversion(...) != 1` (`ninja-adb21.php:1760-1761`) | invalid ad network, or postback did not return 200 |
| `3` | `$conversionValue == 3` (`ninja-adb21.php:1763-1765`) | special conversion flag |
| other int | `return $conversionValue` (`ninja-adb21.php:1767`) | passthrough of the API's numeric body |

`$po` is declared `$po = 0;` at `ninja-adb21.php:2694` with the comment "Payout placeholder", is never reassigned anywhere in `createUser()`, and is **not a column in `insertUserRecord()`** (`ninja-adb21.php:1775-1782`) — it is never persisted. The finalize sweep consequently pays with a literal `0` (`finalize_pending_installs.php:310-312`), which is bit-identical to what the install path would have sent. *If real payout values are ever wired at install, `po` must be persisted before the delayed postbacks stay in scope.*

### 6.5 `sendConversion()` — the per-network postback map

`ninja-adb21.php:1164-1354`. A flat `if/elseif` chain on `$an`; each branch builds `$url` and calls `convPixel($url)` (`ninja-adb21.php:1121-1162`, cURL GET, `FOLLOWLOCATION` on, timeout 10 / connect 5, returns the HTTP status code). At the end: `$validConv = ($convRet == '200') ? 1 : 0` (`ninja-adb21.php:1347-1352`). An `$an` matching **no** branch leaves `$convRet = 0` → returns `0` → `processConversion()` returns `2`.

| `an` | Network (per source comment) | Endpoint | Payout param | Ref |
|---|---|---|---|---|
| `pa` | PropellerAds | `http://ad.propellerads.com/conversion.php?aid=666178&pid=&tid=65828&visitor_id={cid}&zoneid={sid}` | — | `:1170-1174` |
| `at` | Adterra | `http://www.pbterra.com/name/nrattaz/at?subid_short={cid}` | — | `:1175-1179` |
| `am` | AdMaven | `http://rtb-internal-3499555.us-east-1.elb.amazonaws.com/pixel?unique_req={cid}` | — | `:1180-1184` |
| `gc` | Grand Clicks | `https://indlyment-stuador.com/postback?cid={cid}&payout=1` | `payout=1` (**hardcoded literal**, not `$po`) | `:1185-1189` |
| `ra` | RichAds | `http://xml.auxml.com/log?action=conversion&key={cid}` | — | `:1190-1196` |
| `pc` | PopCash | `https://ct.popcash.net/click?aid=257975&clickid={cid}&siteid={sid}` | — | `:1197-1201` |
| `ac` | Adcash | `https://www.imcounting.com/et/event.php?advertiser=140794&cid={cid}&id=2ee368` | — | `:1202-1208` |
| `ar` | AdRight | `https://xml.fstsrv9.com/conversion?c={cid}&count=1&value=1` | `value=1` (**hardcoded literal**) | `:1209-1214` |
| `un` | Unicorner | `http://yslqczldaxcy.unicornpride123.com/postback.php?sid={cid}&payout=1&count=1&value=1` | `payout=1`, `value=1` (**hardcoded literals**) | `:1215-1219` |
| `ro` | RollerAds | `https://eu.rollerads.com/conversion/{cid}/aid/10526/073788ab573297ca` | — | `:1220-1224` |
| `ua` | unGads | `https://ungads-postback.com/c/2CBtXhKUOTUCGoQAFS5HxA?subid={cid}` | — | `:1225-1229` |
| `zp` | ZeroPark | `http://zp-postback.com/zppostback/032a02d1-8374-11e8-a025-0e41d0acbc1a?cid={cid}` | — | `:1230-1235` |
| `pr` | PrimeRevenue | `https://offers.primerevenues.com/postback?clickid={cid}` | — | `:1236-1240` |
| `ax` | Adterra (second slot) | `http://www.pbterra.com/name/nrattaz/at?subid_short={cid}` — **identical URL to `at`** | — | `:1241-1245` |
| `ca` | Clickadu | `http://sconvtrk.com/conversion/2e5e0278a8c6486d6673edafd1cc27a952b7751b/?visitor_id={cid}&aid=232578` | — | `:1246-1250` |
| `vi` | Vimmy | `http://vmtrck.com/cnvproxy.php?cnv_id={cid}` | — | `:1251-1255` |
| `jr` | (comment says "vimmy") | **no HTTP call** — `$convRet = '200'` is assigned literally | — | `:1256-1259` |
| `m1` | Media buyer | `https://trk.datapass.space/postback?cid={cid}&payout={po}` | **`payout={po}`** | `:1260-1265` |
| `m2` | Media buyer | `https://b.datapasss.life/click.php?cnv_id={cid}&payout={po}` | **`payout={po}`** | `:1266-1270` |
| `ph` | PusHub | `https://xml.pushub.net/conversion?c={cid}` | — | `:1271-1275` |
| `wn` | WakeNet | `http://rep.pe-wok.biz/track_cou.php?cid={cid}` | — | `:1276-1280` |
| `ez` | EzMob | `https://brencesubiltime.com/postback?cid={cid}` | — (click-id only) | `:1281-1285` |
| `c1` | Clickadu 2 | `http://sancontr.com/conversion/0dfff22c690a8a5c7a5f96c26e7909c6c557e8ff/?visitor_id={cid}&aid=243625` | — | `:1286-1290` |
| `an` | AdNation | `https://syndication.optimizesrv.com/tag.php?goal=2a79ea27c279e471f4d180b08d62b00a&skey=2be0fe7a85069c83bb3f1b42a8020108aa6d893b&tag={cid}` | — | `:1291-1295` |
| `xr` | Xandr | `http://sspx-router.adnxs.net/sspx?id=1751305&sspdata={cid}&order_id={cid}` | — | `:1296-1300` |
| `tb` | Taboola (comment says "xander") | `https://trc.taboola.com/actions-handler/log/3/s2s-action?click-id={cid}&name=Install_extension&revenue=&currency=&orderid={cid}` | `revenue=` sent **empty**, `currency=` empty | `:1301-1305` |
| `mg` | MGID (comment says "xander") | `https://a.mgid.com/postback/847756?c={cid}&e=install&r={po}` | **`r={po}`** | `:1306-1310` |
| `za` | Zemanta | `https://p1.zemanta.com/v2/p/s2s/110954/install/?postbackid={cid}` | — | `:1311-1315` |
| `oa` | Onclicka | `https://tracking.onclicka.com/in/postbacks/?token=wOC6Qh&click_id={cid}` | — | `:1316-1320` |
| `gn` | Galaksion | `http://postback.info/postback.php?cid=37221&click_id={cid}` | — | `:1321-1325` |
| `hs` | HilltopAds | `https://postback.hilltopads.com/close/?token={cid}&advertiserId=106995` | — | `:1326-1330` |
| `ev` | Evadav | `https://evadav.com/phpb?click_id={cid}&payout={po}` | **`payout={po}`** | `:1331-1335` |
| `ad` | Adstand | `https://psb-handler.xyz/p?uid=660&subid={cid}` | — | `:1336-1340` |

Summary of payout-bearing postbacks:

| Class | Networks |
|---|---|
| Interpolate `$po` (always `0` today) | `ev`, `mg`, `m1`, `m2` |
| Hardcoded payout/value literals independent of `$po` | `gc` (`payout=1`), `un` (`payout=1&value=1`), `ar` (`value=1`) |
| Empty revenue field | `tb` |
| No payout param at all | every other network, `ez` included |

*Because `$po` is `0` at install and `0` in the finalize sweep, a delayed postback is byte-identical to an install-time one for every network in the table.*

> **Note** `jr` returns a synthetic `'200'` with no outbound request (`ninja-adb21.php:1257`), so `processConversion('jr', ...)` always returns `1` and the install is recorded as paid without any network being notified. `at` and `ax` post to the same URL, so a per-network revenue split cannot be derived from the postback itself.

### 6.6 Step 6 / Step 9 record preparation

Step 6 (`ninja-adb21.php:3354-3357`) computes the creation timestamp and the tag:

```php
$creationdate = time();
$typetag = date('W', $creationdate) . substr(date("Y", $creationdate), -1) . '-' . ($an ?? 'xx') . '-25';
```

| Component | Source | Example |
|---|---|---|
| ISO week number, zero-padded | `date('W')` | `34` |
| last digit of the year | `substr(date('Y'),-1)` | `6` |
| `-` + ad network, or `xx` when `$an` is null | `($an ?? 'xx')` | `-gn` / `-xx` |
| `-25` | fixed suffix | `-25` |

A live value: `346-gn-25` (week 34 of 2026, network `gn`). Organic installs get `346-xx-25`.

Step 7 serializes the install-tab domains: `serialize($instadom)` (`ninja-adb21.php:3360`).

Step 9 builds the field array (`ninja-adb21.php:3363-3415`). Three fields deserve explicit statement:

| Field | Value | Ref |
|---|---|---|
| `acceptable_ads_disabled` | `($acceptableAds === false) ? date('Y-m-d') : null`. `$acceptableAds` is hardcoded `true` at `ninja-adb21.php:378` (the read from `$dataArray['ninja_acceptable_ads']` is commented out), so this column is **always NULL** in practice | `ninja-adb21.php:3403` |
| `fraud_score` | literal `null` — **dead column**, nothing in `createUser()` ever computes a score | `ninja-adb21.php:3405` |
| `fraud_flags` | `$fraudFlags . ($s1DelayHold ? '|s1_hold' : '')` | `ninja-adb21.php:3414` |

`insertUserRecord()` (`ninja-adb21.php:1773-1826`) inserts a fixed 41-column base list (`ip` written through `INET6_ATON(?)`, `diagnostic_date` as `NOW()`), and appends three more columns **only** behind a capability gate:

```php
if (!empty($GLOBALS['has_fraud_columns'])) {   // ninja-adb21.php:1806
    $cols[] = 'device_hash'; $cols[] = 'fraud_score'; $cols[] = 'fraud_flags';
    ...
}
```

`$GLOBALS['has_fraud_columns']` is derived from the `$HAS_FRAUD_COLUMNS` config flag (`ninja-adb21.php:101`, `config.php:36` = `true`). The per-request `SHOW COLUMNS` schema probe was removed on 2026-07-17 (`ninja-adb21.php:40`, `:209`).

> **Canonical (2026-09-02) — `conv_date` and `updated_at` are deliberately NOT in the INSERT column list.** On a migrated schema (§14.6) both fire by column `DEFAULT current_timestamp()` in this same statement, and `NOW()` is evaluated once **per statement**, so `conv_date == created_date` exactly. That one property covers both payout regimes with **no config gating**: pay-at-install (`conv` already 1/3 in the INSERT ⇒ `conv_date` = conversion time) and delayed conversion (`conv` 0 at insert, stamped by the later UPDATE). `insertUserRecord()` is the only INSERT into the user table (canonical: `ext-server.php:2240`) — the insert path needed **zero code change**.

> **Defect** `$HAS_FRAUD_COLUMNS = false` is documented as a safe degradation ("the code then degrades fraud logic off, exactly as the old probe did" — `config.php:32-35`). **It is not safe while the Stage-1 global hold is on.** With the flag off, `fraud_flags` is not in the INSERT column list (`ninja-adb21.php:1805-1816`), so the `|s1_hold` marker never reaches the database. The install is still diverted away from `processConversion()` at `ninja-adb21.php:3330-3332` — the gate does not consult the flag — so no postback fires at install either. The finalize sweep selects exclusively on `fraud_flags LIKE '%|s1_hold%'` (`finalize_pending_installs.php:200`), finds nothing, and the row is never paid, never forfeited, and never even counted. **Every payout is silently lost, with no error and no marker to audit against.** The same flag also disables the device-collision query (`ninja-adb21.php:3084`), so `device_collisions:<n>` stops firing. Do not flip this flag while `$STAGE1_DELAY_CONV_AN` is non-empty.

> **Defect** `$storeExtIdBing = ''` (`ninja-adb21.php:355`) and `$shortExtId = null` whenever the request carries no `extid` (`ninja-adb21.php:367`). PHP's `!==` is type-strict, so `null !== 'ppfa'` and `null !== ''` are both true: `$isExtIdMismatch` fires (`ninja-adb21.php:3145-3147`), the install is labelled `tier3:extid_mismatch`, and `evaluateRisk()` appends digit `4` with `conv = 2` (`ninja-adb21.php:1621-1626`). An install that merely failed to report its extension id is condemned as a distribution-integrity violation and can never be paid.

> **Defect** `evaluateVirtualMachine()`'s catch-all returns `['is_vm' => 0, 'vm_type' => 'detection_error:' . substr($e->getMessage(), 0, 100)]` (`ninja-adb21.php:1080`). That `vm_type` does **not** start with `physical:`, so neither `$isTier3` (needs `is_vm === 1`) nor the `borderline_vm` predicate (`ninja-adb21.php:3180`) fires. An exception inside the evaluator lands the install in `tier1:high_trust`. **The VM layer fails OPEN.**

> **Defect** `getRiskData()`'s `lookup_failed` path makes `evaluateRisk()` return `['level' => '0', 'conv' => 0]` immediately (`ninja-adb21.php:1614-1617`), before any type/proxy/provider/UA test runs. A proxycheck outage therefore makes **every** install look clean and payable — `level === "0"` satisfies the Stage-1 gate at `ninja-adb21.php:3320`. **The network layer fails OPEN.**

> **Defect** The old-Chrome branch of STEP 5 in `evaluateVirtualMachine()` runs `preg_match('/Chrome\/(\d+)\./', $ua, $chromeMatches)` (`ninja-adb21.php:1057`) against a `$ua` that was lowercased at `ninja-adb21.php:840` (`strtolower(trim(...))`). The pattern is case-sensitive and the haystack contains `chrome/`, so it **never matches**: `$isOldChrome` is permanently `false` and `physical:old_laptop` can only ever be reached through the `$isAncientGPU` test (`hd graphics 300` / `direct3d9ex`, `ninja-adb21.php:1065`). **Dead branch.**

### 6.7 `mergeConvV4` is never called from `ninja-adb21.php`

`mergeConvV4()` is defined in `classifier_v4.php:158` (guarded by `function_exists` at `:130`). Its only production call site is `finalize_pending_installs.php:274`.

- `ninja-adb21.php` does not `require` `classifier_v4.php` at all — its includes are `config.php` (`:72`) and `behavior_lib.php` (`:74`). The symbol is nevertheless loaded transitively — `behavior_lib.php:11` requires `classifier_v4.php` — it is simply never called from the endpoint.
- The only other references are the offline backtest `replay_v4.php:140` and the self-test in `classifier_v4.php:235`.
- **The money merge is unreachable today.** Its single call site is wrapped in `if (function_exists('an_in_behavior_scope') && an_in_behavior_scope($held['an']) && function_exists('mergeConvV4'))` (`finalize_pending_installs.php:255-256`), and `an_in_behavior_scope()` returns `false` for every network while `$BEHAVIOR_AN_ALLOWLIST = []`. The `|s1_refused` marker it would write (`finalize_pending_installs.php:276-283`) is therefore never produced.

Wiring `mergeConvV4` into the install path is the V4-3 change and has not been made. **NOT ACTIVE TODAY.**

### 6.8 Decision flow, install request to marker

```text
install request (an, cid, sid, extid, fingerprint)
  |
  +-- getRiskData(ip) ............................ proxycheck
  |      lookup_failed? --> level '0', conv 0  [FAILS OPEN, see 6.6]
  |
  +-- evaluateRisk() ............................. level digits 3,4,5,6,7,8,9 ; conv 0|2
  +-- evaluateVirtualMachine() ................... is_vm / vm_type
  +-- device_hash + 7-day collisions ............. gated on has_fraud_columns
  |
  +-- STEP 5 tier ................................ tier3 | tier2 | tier1  -> $fraudFlags
  |      (LABEL ONLY -- gates nothing, see 5.5)
  |
  +-- EARLY_FRAUD detection (2026-09-03) ......... may append '|early_fraud:tzdc+fpvote+ipstack'
  |      SHADOW: label only, gates nothing; one extra COUNT (ip/30d); fail-open try/catch
  |      enforcement pre-wired but COMMENTED (grep EARLY_FRAUD_FLIP)
  |
  +-- decision block  ($pureShadow == true, ALWAYS)
         |
         +-- an null OR cid null? .............................. conv 0, no marker  -> NEVER PAID
         +-- level !== "0" (or evaluateRisk conv==2)? .......... conv 0 or 2        -> NEVER PAID
         +-- an && cid && level==="0" && conv!==2
                |
                +-- s1_delay_in_scope(an)? .... TRUE for every an (list = ['*'])
                |        -> $s1DelayHold = true ; conv stays 0
                |        -> fraud_flags gets '|s1_hold'
                |        -> paid ~1h later by finalize (in-process, traffic-driven)
                |
                +-- (else) processConversion() ................ DEAD PATH TODAY
  |
  +-- ninja_sync := 600000 unconditionally  [ninja-adb21.php:3350-3352]
  +-- Step 6/9 -> insertUserRecord()
```

| Install shape | `level` | tier label | `conv` at INSERT | `fraud_flags` written | Outcome |
|---|---|---|---|---|---|
| Clean, attributed (`an`+`cid`) | `0` | `tier1:high_trust` | `0` | `tier1:high_trust\|s1_hold` | held; paid at ~1h by finalize, marker → `\|s1_paid` |
| VPN, attributed | `3` (+ maybe `7`) | `tier2:vpn\|...` | `2` | `tier2:vpn\|...` | never paid; no marker |
| Old laptop, clean network, attributed | `0` | `tier2:borderline_vm:physical:old_laptop` | `0` | `...\|s1_hold` | held; paid at ~1h — Tier 2 does **not** block today |
| Hosting IP, attributed | `3` | `tier3:hard_proxy_type:hosting` | `2` | `tier3:hard_proxy_type:hosting` | never paid |
| Hard VM, clean network, attributed | `0` | `tier3:hard_vm:score_4:...` | `0` | `...\|s1_hold` | **held and paid** — the tier does not gate; only `level` does |
| No `extid` sent, attributed | `4` | `tier3:extid_mismatch` | `2` | `tier3:extid_mismatch` | never paid (see the `$storeExtIdBing` defect) |
| Organic (`an` null) | `0` | any | `0` | tier label, **no marker** | never paid; `typetag` shows `-xx-` |
| Held row, uninstalled before deadline | — | — | `0` | `\|s1_hold` → `\|s1_forfeit` | forfeited (`farewell.php` sets `disabled`) |
| Held row, older than 72h | — | — | `0` | `\|s1_hold` → `\|s1_forfeit` | forfeited at `$S1_HOLD_PAY_WINDOW_SEC` |

> **Note** The straggler/expiry sweep in `finalize_pending_installs.php` is wrapped in `if (!$pureShadow)` (`finalize_pending_installs.php:563-564`) and therefore **NEVER RUNS TODAY**. The V4 guarantee of "zero non-terminal installs past 24h" is not enforced; non-terminal rows can persist indefinitely. Separately, `runBehaviorLayer()` terminalizes `human`/`bot` at **any** age, not only past 3h, and a terminal verdict stamps `decided_date` — which makes `logSyncArrival()` stop writing `sync_log` for that install (`ninja-adb21.php:2519`, `:2515`), cutting off the evidence stream the classifier depends on.

---

## 7. Behavior layer — features and the v4 classifier

The behavior layer answers one question — *does this install act like a human?* — from three inputs only: the merged `visit` map, the install-tab `instdom` list, and the `sync_log` rows. It is implemented in two files: `behavior_lib.php` (feature extraction, string tables, orchestration) and `classifier_v4.php` (the pure scoring function). Both are live and run on every sync today, in **FULL SHADOW** — the verdict is written to `decision` / `behavior_score` / `behavior_flags`, and moves no money.

### 7.1 The principle

`decision` is a **behavioural fact**. `classifyBehaviorV4()` takes no `$tier` argument by design, and reads no `conv`, no `level`, no `is_vm`, no `device_hash`, no IP-rotation input (`classifier_v4.php:52-59`). Hardware and network suspicion live in the tier, and the tier acts once, later, at the money merge (`mergeConvV4`, `classifier_v4.php:158`). If behaviour said nothing, the honest verdict is `unknown` — the classifier never converts absence of evidence into guilt. This is the single structural correction v4 made over v3, where the tier prior (`tier_prior[3] = 35`, `classifier_lib.php:23`) alone exceeded `bot_line = 30` and terminalised Tier-3 installs as bots before any behaviour existed.

> *Why it matters operationally: because `decision` never reads `conv`, the shadow dataset stays valid as a label even though Stage-1 pays every install at (or 1h after) install.*

### 7.2 Feature extraction

`extractBehaviorFeatures($install, $syncRows, $funnel, $pdo, $table)` — `behavior_lib.php:256-417`. It emits exactly twenty keys. Callers: `runBehaviorLayer` (`behavior_lib.php:477`), `finalize_pending_installs.php:485`, `generate_funnel_cache.php:279`, `replay_v4.php:112`.

| Feature key | Type | Exact derivation | Threshold / constant | Consumed by live classifier? |
|---|---|---|---|---|
| `organic_core` | string[] | every key of `safeUnserializeVisit($install['visit'])`, lowercased+trimmed, dropping empty, `is_generic()` and `is_funnel_domain()` hits; then `sort()`ed (`behavior_lib.php:268-284`) | — | **No** — internal only (feeds `behavior_hash`) |
| `organic_core_size` | int | `count($organicCore)` (`:283`) | — | Yes |
| `has_revisit` | bool | true if any surviving organic domain has `$count > 1` in the `visit` map (`:277-279`) | `> 1` | Yes |
| `auth_session` | bool | any organic-core domain satisfies `matches_auth_marker()` (`:287-293`) | — | Yes |
| `gov_edu_corp` | bool | any organic-core domain matches `/\.(go\.id\|ac\.id\|gov\|edu)$\|instructure\.com\|okta\.com\|sharepoint/i` (`:297`) | — | Yes |
| `popup_interaction` | bool | `whitelistedDom` **or** `blockedDom` is not one of the empty forms (`:307-311`) | see §7.5 | Yes |
| `vm_hardflag` | bool | `(int)$install['is_vm'] === 1` (`:313`) | — | **No — DEAD SIGNAL** |
| `behavior_hash` | string\|null | `hash('sha256', implode('|', $organicCore))` when `count >= 3`, else `null` (`:315-319`) | `>= 3` | **No** — persisted to the `behavior_hash` column, and read back by the collision SQL |
| `instdom_size` | int | `count(safeUnserializeInstdom($install['instdom']))` (`:321-322`) | — | **No — DEAD SIGNAL** |
| `instdom_size_bucket` | string | `'0'` / `'small'` / `'medium'` / `'large'` (`:323-331`) | `0` · `<=5` · `<=20` · `>20` | **No — DEAD SIGNAL** (only `generate_funnel_cache.php:300-309` cohorts read it) |
| `instdom_has_login` | bool | any install-tab domain matches `matches_auth_marker()` **or** `/login\|signin\|signup\|oauth/i` (`:333-339`) | — | **No — DEAD SIGNAL** (cohorts only, `generate_funnel_cache.php:311`) |
| `instdom_funnel_only` | bool | `instdom_size > 0` and **every** install-tab domain is `is_funnel_domain()` (`:341-351`) | — | Yes (`funnel_only`, +20) |
| `farm_collision` | bool | DB lookup, §7.6 (`:353-365`) | ±30 min | Yes |
| `n_syncs` | int | `count($syncRows)` (`:367`) | — | Yes |
| `interval_cv` | float | stddev/mean of consecutive `server_ts` gaps (`:369-384`) | — | Yes (metronome) |
| `zero_activity_ratio` | float | fraction of sync rows with `new_domains === 0` (`:386-392`) | — | Yes (metronome) |
| `distinct_ips` | int | count of distinct non-empty `src_ip` binaries (`:394-409`) | — | **No — DEAD SIGNAL** |
| `distinct_prefixes` | int | first 2 bytes of a 4-byte `src_ip`, first 4 bytes of a 16-byte one (`:399-407`) | IPv4 /16, IPv6 /32 | **No — DEAD SIGNAL** (v3's `ip_rotation`, removed for wrong sign) |
| `tier` | int | `deriveTierFromRow($install)` — `classifier_lib.php:160-170` (`:413`) | — | **No — DEAD SIGNAL** in the classifier; `finalize_pending_installs.php:273` recomputes it independently for the (dormant) merge |
| `fast_uninstall` | bool | `!empty($install['disabled'])` (`:414`) | none — §7.10 | Yes (+30) |

**Dead signals computed on every sync and never consumed by `classifyBehaviorV4()`:** `vm_hardflag`, `instdom_size`, `instdom_size_bucket`, `instdom_has_login`, `distinct_ips`, `distinct_prefixes`, `tier`, and the `organic_core` array itself. Seven of the eight are pure cost; `tier` costs a `fraud_flags` string scan per call.

> **Note** `distinct_ips` / `distinct_prefixes` operate on the raw `INET6_ATON()` binary written by `logSyncArrival` (`ninja-adb21.php:2550`). Any value whose length is neither 4 nor 16 contributes to `distinct_ips` but not to `distinct_prefixes`.

### 7.3 The string tables that gate the human signals

**GENERICS** — `behavior_lib.php:17-24`, verbatim:

```
google.com
youtube.com
chromewebstore.google.com
bing.com
```

`is_generic()` (`behavior_lib.php:62-75`) lowercases, trims, strips a single leading `www.`, then compares with `===`. **Matching is EXACT, not suffix.** Consequences that matter:

| Domain | Generic? | Effect |
|---|---|---|
| `google.com`, `www.google.com` | yes | excluded from organic core |
| `mail.google.com` | **no** | counts as organic **and** is an auth marker → `auth_session` |
| `docs.google.com`, `drive.google.com`, `accounts.google.com` | **no** | same |
| `news.google.com`, `translate.google.com` | **no** | counts as ordinary organic breadth |
| `m.youtube.com`, `www.bing.com` | no / yes | `m.youtube.com` counts as organic; `www.bing.com` is stripped to `bing.com` and excluded |

**AUTH_MARKERS** — `behavior_lib.php:132-151`, verbatim:

```
web.whatsapp.com
mail.google.com
docs.google.com
drive.google.com
accounts.google.com
outlook.*
instagram.com
facebook.com
x.com
reddit.com
tiktok.com
linkedin.com
chatgpt.com
canva.com
discord.com
netflix.com
```

`matches_auth_marker()` (`behavior_lib.php:153-173`) lowercases and trims, then for every marker except `outlook.*` accepts an exact match **or** a `'.' . $marker` suffix match. `outlook.*` is special-cased at `:157-159` as `$domain === 'outlook.com' || substr($domain, 0, 8) === 'outlook.'` — a **prefix** test.

Asymmetries this creates:

| Case | Result | Reason |
|---|---|---|
| `www.facebook.com` | match | suffix rule `.facebook.com` |
| `outlook.live.com`, `outlook.office365.com` | match | `outlook.` prefix |
| `www.outlook.com` | **no match** | prefix is `www.outl`; the suffix rule is skipped for this marker |
| `outlook.fr`, `outlook.anything` | match | prefix rule is TLD-blind |
| `notfacebook.com` | no match | suffix rule requires the leading dot |
| `$domain === 'outlook.com'` clause | redundant | already covered by the `substr(...,0,8)` test |

**gov / edu / corp** — the exact regex, `behavior_lib.php:297`:

```php
preg_match('/\.(go\.id|ac\.id|gov|edu)$|instructure\.com|okta\.com|sharepoint/i', $dom)
```

Only the first alternative is anchored (`$` binds to that alternative alone) and it requires a leading dot. The other three are **unanchored substring** matches.

| Domain | Match | Why |
|---|---|---|
| `harvard.edu`, `whitehouse.gov`, `kemdikbud.go.id`, `ui.ac.id` | yes | anchored suffix |
| `ox.ac.uk`, `www.gov.uk`, `hmrc.gov.uk` | **no** | ends in `.uk` — the anchored branch misses every non-US national academic/government TLD (`.ac.uk`, `.gov.uk`, `.edu.au`, `.ac.jp`, `.gouv.fr`, `.edu.cn`) |
| `contoso.sharepoint.com` | yes | unanchored `sharepoint` |
| `mysharepointfake.ru` | yes | unanchored — false positive |
| `notinstructure.com.evil.net` | yes | unanchored — false positive |

> *Rationale for the loose corp half: `sharepoint` / `okta.com` / `instructure.com` are managed-identity portals a bot farm has no reason to visit; the false-positive surface was accepted because the signal is only −35 and the corroboration gate (§7.9) requires a second, independent signal anyway.*

### 7.4 Funnel handling

**`normalize_funnel_domain($domain)`** — `behavior_lib.php:77-94`. Lowercase/trim, split on `.`, then:

1. While `count($parts) > 2` and `$parts[0]` contains a digit (`/\d/`), shift it off. *Strips rotating numeric subdomains: `cdn3.track7.zerodrifts.com` → `zerodrifts.com`.*
2. If exactly 2 parts remain and `$parts[0]` **ends** in digits (`/\d+$/`), return `$parts[0]` **alone — TLD dropped**: `rbtvplus18.com` → `rbtvplus18`. *Collapses a campaign that rotates TLDs on a fixed numbered base.*

Otherwise the joined domain is returned. Rule 2 is why the funnel lists contain bare, dotless tokens.

**`is_funnel_domain($domain, $funnelPatterns)`** — `behavior_lib.php:96-130`. Returns false immediately on an empty domain or non-array pattern list. Per pattern (lowercased/trimmed, empties skipped):

| Pattern shape | Test |
|---|---|
| contains no `.` (e.g. `rbtvplus18`) | `in_array($pattern, explode('.', $domain), true)` — matches the token as **any whole label** of the domain (`:111-114`) |
| contains a `.` (e.g. `zerodrifts.com`) | exact match, or `'.' . $pattern` suffix match (`:116-124`) |

**`funnel_cache.json` contract.** Written atomically (`.tmp` + `rename`) by `generate_funnel_cache.php:444-457`; that script is the only writer. Shape:

```json
{
  "global_funnel": ["zerodrifts.com", "rbtvplus18"],
  "by_campaign": { "gn": ["..."], "pa": ["..."] },
  "generated_at": 1753000000
}
```

Read once per request at `ninja-adb21.php:231-233` into `$GLOBALS['funnel']`; a missing file yields `[]`, and the request path **never** regenerates it (regeneration is the background self-call at `ninja-adb21.php:3813-3834`, which also fires when the file is missing). `extractBehaviorFeatures` merges `global_funnel` with `by_campaign[$an]` via `array_unique(array_merge(...))` (`behavior_lib.php:261-263`).

> **Note** When `funnel_cache.json` is MISSING (or an empty skeleton), `$funnelPatterns === []`, `is_funnel_domain()` returns false for everything, and **two signals silently disable**: (1) `instdom_funnel_only` can never be true, so `funnel_only` (+20) is dead; (2) campaign landing domains are no longer stripped from the organic core, so `organic_core_size` inflates and `zero_organic` (+20) stops firing — while `organic_domain_each` (−5 each) starts paying out on funnel traffic. The failure is silent and biases the classifier toward *human*. Since the live host has **no cron**, the only regenerator is the traffic-driven background spawn.

A third key, `behavior_thresholds`, is still read by `behavior_lib.php:482-484` and `finalize_pending_installs.php:487-490` but has **not been written since 2026-07-13** (`generate_funnel_cache.php:348-352`) and is ignored downstream anyway (§7.12). `$thresholds` is therefore always `[]`.

### 7.5 popup_interaction

The exact test — `behavior_lib.php:307-311`:

```php
$popupEmptyForms = ['', 'a:0:{}', 'N;', 'null', '[]'];
$features['popup_interaction'] = (
    !in_array(trim((string)($install['whitelistedDom'] ?? '')), $popupEmptyForms, true)
    || !in_array(trim((string)($install['blockedDom'] ?? '')), $popupEmptyForms, true)
);
```

True when **either** column holds something that is not an empty container. The hardening dated **2026-07-13** added `'a:0:{}'`, `'N;'`, `'null'` and `'[]'` to the list: before it, a stored serialized-empty array (`a:0:{}`) was "not empty string" and therefore scored −25 as proof the user had opened the popup — a free human signal for every install whose extension posted an empty list. The comment at `:303-305` marks this as a T3 PROOF signal (it is one of the two proofs `mergeConvV4` accepts, `classifier_v4.php:170`), which is why it was hardened rather than left approximate.

Encodings accepted, and where they come from:

| Stored value | Source | `popup_interaction` |
|---|---|---|
| `NULL` → `(string)` `''` | live writer when the list is empty: `ninja-adb21.php:379-384` sets the variable to `null` | false |
| `["a.com","b.net"]` | live writer, `json_encode()` of a non-empty array | **true** |
| `[]` | JSON empty array (legacy / other writers) | false |
| `null` | JSON null literal | false |
| `a:0:{}` | PHP-serialized empty array (legacy rows) | false |
| `N;` | PHP-serialized null (legacy rows) | false |
| `a:1:{i:0;s:5:"a.com";}` | PHP-serialized non-empty (legacy rows) | **true** |
| `{}` | empty JSON **object** | **true** — not in the empty-forms list |

`runBehaviorLayer` overlays the incoming request values before extraction, but only when they are non-`null` (`behavior_lib.php:469-474`), so the stored column is otherwise authoritative.

> **Note** `popup_interaction` is **non-monotonic**. `updateUserRecord` writes `whitelistedDom` / `blockedDom` unconditionally on every sync (`ninja-adb21.php:2485-2486`, bound at `:2502-2503`), and the value is `null` whenever the incoming list is empty (`:379-384`). A user who adds two sites and later clears them therefore has the column reset to `NULL`, and the next evaluation **loses** the −25 and one of the five corroboration signals. A verdict that was `human` at 40 minutes can be `unknown` at 90 minutes. Terminal verdicts are never reopened (`behavior_lib.php:456-458`), so which value survives is a race with the terminalisation moment.

### 7.6 behavior_hash and farm_collision

`behavior_hash` = `hash('sha256', implode('|', $organicCore))` where `$organicCore` has already been `sort()`ed (`behavior_lib.php:281`, hash at `:316`). It is computed **only when `count($organicCore) >= 3`**; below that it is `null` (`:315-319`). *Two installs that each visited only `example.com` are not a farm; three-domain agreement is the smallest pattern worth calling identical.* The separator is a single `|` and the input is the deduplicated, sorted, generic- and funnel-stripped core — so the hash is order-independent and campaign-independent.

`farm_collision` (`behavior_lib.php:353-365`) runs only when `$pdo !== null` **and** `behavior_hash` is non-empty:

```sql
SELECT `id` FROM `{$table}`
 WHERE `behavior_hash` = ?
   AND `id` != ?
   AND `created_date` BETWEEN DATE_SUB(?, INTERVAL 30 MINUTE) AND DATE_ADD(?, INTERVAL 30 MINUTE)
 LIMIT 1
```

Bound parameters, in order: the hash, `$install['id']`, `$install['created_date']`, `$install['created_date']` — i.e. a **±30 minute window centred on this install's own creation time**, a different `id`, an identical hash.

Two dependencies that decide whether it can ever fire:

- The peer row must **already have persisted** its `behavior_hash`. That column is written only by the `UPDATE` in `runBehaviorLayer` (`behavior_lib.php:515-526`) and by the finalize writes (`finalize_pending_installs.php:506-520`). An install that never syncs never writes a hash and is invisible to the query, no matter how many twins it has.
- Detection is therefore **asymmetric in time**: the first install of a pair sees no peer and scores 0; the second sees the first and scores +50. A farm burst is caught from its second member onward, and only if the first member's sync landed within the ±30-minute window.

`$pdo` is passed by every live caller, so the guard at `:353` is only exercised by `replay_v4.php` in `--no-farm` mode (and unconditionally by `simulate_stopads.php:195`) (`replay_v4.php:112`).

### 7.7 Sync-derived signals

All four are computed from `$syncRows`, ordered `server_ts ASC` (`behavior_lib.php:461-463`).

| Signal | Derivation | 0 syncs | 1 sync |
|---|---|---|---|
| `n_syncs` | `count($syncRows)` (`:367`) | `0` | `1` |
| `interval_cv` | `$gaps[i] = strtotime(ts[i]) - strtotime(ts[i-1])` for `i = 1..n-1`; `cv = stddev/mean`, population stddev, computed only when `count($gaps) > 0` **and** `$mean > 0` (`:369-384`) | `0.0` (initialised, never touched) | `0.0` — zero gaps exist |
| `zero_activity_ratio` | `count(rows with (int)new_domains === 0) / count($syncRows)`, guarded by `count > 0` (`:386-392`) | `0.0` | `0.0` or `1.0` |
| `distinct_prefixes` | distinct 2-byte (IPv4) / 4-byte (IPv6) prefixes of `src_ip` (`:394-410`) | `0` | `0` or `1` |

Two degenerate cases worth knowing:

- `interval_cv = 0.0` is the value for "perfectly regular" **and** for "not enough data". The metronome detector is protected from the second reading only by its `n_syncs >= 6` precondition.
- If `$mean <= 0` — every sync recorded within the same second, so all gaps are 0 — the stddev branch is skipped and `interval_cv` stays at its initialised `0.0`, which is `< metro_max_cv`. Six same-second syncs with no new domains therefore satisfy all three metronome conditions.

### 7.8 The classifier

`ninja_v4_weights()` — `classifier_v4.php:16-49`, reproduced exactly:

| Key | Value | Group |
|---|---|---|
| `farm_collision` | `50` | bot-like (+) |
| `fast_uninstall` | `30` | bot-like (+) |
| `metronome` | `25` | bot-like (+) |
| `zero_organic` | `20` | bot-like (+) |
| `funnel_only` | `20` | bot-like (+) |
| `auth_login` | `-40` | human-like (−) |
| `gov_edu_corp` | `-35` | human-like (−) |
| `popup_interaction` | `-25` | human-like (−) |
| `organic_revisit` | `-20` | human-like (−) |
| `organic_domain_each` | `-5` | human-like (−) |
| `organic_domain_cap` | `-40` | human-like (−) |
| `bot_line` | `30` | threshold |
| `human_line` | `-15` | threshold |
| `human_min_signals` | `2` | corroboration gate |
| `human_breadth_min` | `3` | corroboration gate |
| `metro_min_syncs` | `6` | metronome param |
| `metro_max_cv` | `0.15` | metronome param |
| `metro_min_idle` | `0.8` | metronome param |

`classifyBehaviorV4(array $f, ?array $w = null)` — `classifier_v4.php:59-127`. `$S` starts at `0.0` (`:62`). Steps, in evaluation order:

| # | Line | Condition | Effect |
|---|---|---|---|
| 1 | `:68` | `!empty($f['farm_collision'])` | `S += 50`; flag `farm_collision` |
| 2 | `:69` | `!empty($f['fast_uninstall'])` | `S += 30`; flag `fast_uninstall` |
| 3 | `:71` | `$nSyncs > 0 && $organicSize === 0` | `S += 20`; flag `zero_organic` |
| 4 | `:73-78` | `$nSyncs >= 6` **and** `interval_cv < 0.15` **and** `zero_activity_ratio > 0.8` | `S += 25`; flag `metronome` |
| 5 | `:80` | `!empty($f['instdom_funnel_only'])` | `S += 20`; flag `funnel_only` |
| 6 | `:83` | `!empty($f['auth_session'])` | `S += -40`; flag `auth_login` |
| 7 | `:84` | `!empty($f['popup_interaction'])` | `S += -25`; flag `popup_interaction` |
| 8 | `:85` | `!empty($f['gov_edu_corp'])` | `S += -35`; flag `gov_edu_corp` |
| 9 | `:87-89` | `$organicSize > 0` | `S += max(-40, -5 * $organicSize)`; flag `organic_domains:<N>` |
| 10 | `:90-93` | inside step 9: `has_revisit` **and** `$organicSize >= 2` | `S += -20`; flag `organic_revisit` |
| 11 | `:101` | `$nSyncs === 0` | flag `no_evidence` — **score unchanged** |
| 12 | `:103-106` | `S >= 30` → `bot`; elseif `S <= -15` → `human`; else `unknown` | sets `$decision` |
| 13 | `:113-124` | only if `$decision === 'human'` — the corroboration gate, §7.9 | may downgrade to `unknown` |
| — | `:126` | returns `['decision','score' => round($S,2),'flags']` | |

Details that are easy to get wrong:

- **`zero_organic` requires `n_syncs > 0`** (`:71`). A 0-sync ghost has `organic_core_size === 0` but must not be penalised for it — v3's separate `no_sync` (+15) was removed as empirically wrong-signed, and this precondition is what stops the same absence being scored twice.
- **The metronome is a triple conjunction**, all three params from the weight table: `metro_min_syncs = 6`, `metro_max_cv = 0.15`, `metro_min_idle = 0.8`. Note the defaults inside the reads: a missing `interval_cv` defaults to `1.0` (fails the test) and a missing `zero_activity_ratio` to `0.0` (fails the test) — the detector fails closed on absent features.
- **The organic cap is a floor on a negative sum**: `max($w['organic_domain_cap'], $w['organic_domain_each'] * $organicSize)` = `max(-40, -5N)`. Both operands are negative, so `max()` picks the *less negative* one. N=1 → −5, N=4 → −20, N=8 → −40, N=50 → −40. Breadth stops paying after the eighth organic domain.
- **`organic_revisit` needs `organic_core_size >= 2`** (`:90`), not merely `has_revisit`. A single domain visited twenty times is a refresh loop, not browsing breadth.
- **`no_evidence` is unconditional on 0 syncs** (`:101`) — it is emitted regardless of score. The comment at `:98-100` records why: the earlier `&& $S === 0.0` guard suppressed the flag on exactly the attributed ghosts that carry install-time priors (`funnel_only` +20, or a popup value present at install), which are the rows the dashboard ghost subcategory and the merge most need to count.

### 7.9 Thresholds and the corroboration gate

Two lines, evaluated bot-first (`classifier_v4.php:103-106`):

| Condition | Verdict |
|---|---|
| `S >= bot_line` (`30`) | `bot` |
| `S <= human_line` (`-15`) | `human` |
| otherwise | `unknown` |

A `human` verdict then faces the gate (`classifier_v4.php:113-124`). Five **independent human-evidence types** are counted:

| # | Test (`classifier_v4.php`) |
|---|---|
| 1 | `!empty($f['auth_session'])` — `:115` |
| 2 | `!empty($f['popup_interaction'])` — `:116` |
| 3 | `!empty($f['gov_edu_corp'])` — `:117` |
| 4 | `!empty($f['has_revisit']) && $organicSize >= 2` — `:118` |
| 5 | `$organicSize >= $w['human_breadth_min']` (`3`) — `:119` |

If the count is below `human_min_signals = 2`, the decision is set to `unknown` and the flag `human_gate_uncorroborated:<count>` is appended (`:120-123`).

**The gate downgrades to `unknown` and NEVER to `bot`.** Absence of corroboration is doubt, not guilt; what an uncorroborated user is worth is the merge's decision, not the classifier's. Concretely: one faked `mail.google.com` visit yields `auth_login` (−40) plus `organic_domains:1` (−5) = −45, comfortably past `human_line`, but only **one** signal type — so the verdict is `unknown`, not `human`. Adding `popup_interaction` or a second organic revisit or a third organic domain flips it to `human`.

> *What the gate does not stop: a multi-domain visit spoof. An install listing several fabricated auth/organic domains clears a count gate by construction. The intended answer is a future sync-accumulation signal (organic evidence that grew gradually across the window), which does not exist in the code today.*

### 7.10 fast_uninstall

Derivation, in full — `behavior_lib.php:414`:

```php
$features['fast_uninstall'] = !empty($install['disabled']);
```

`disabled` is a **DATE** column, written as `date('Y-m-d')` by `farewell.php:79` (fingerprint path) and `farewell.php:89` (legacy encrypted-uid path) when the uninstall page is hit. Nothing else writes it.

**There is no time bound.** The feature is a bare emptiness test on a date column — it does not compare `disabled` to `created_date`, and it does not consult `BEHAVIOR_WINDOW_SEC`. **Any** uninstalled row scores **+30** and flags `fast_uninstall`, whether the user left eleven minutes or eleven weeks after installing. The name is a description of the intended cohort, not of the test. Since +30 equals `bot_line` exactly, an uninstalled install with no human evidence is `bot` on the strength of that one feature.

> **Note** This makes `fast_uninstall` circular whenever the verdict is used to predict retention: the label is partly *made of* the outcome. `replay_v4.php:17-18, 72, 203` exists for this reason — it prints a second, decircularised block with `fast_uninstall` weighted `0` alongside the normal one. Read the decircularised numbers when judging classifier quality.

### 7.11 behavior_flags — the token vocabulary

`runBehaviorLayer` persists `implode('|', $flags)` into the `behavior_flags` column (`behavior_lib.php:511`, written at `:515-526`); `finalize_pending_installs.php:504-520` does the same. The separator is a single `|`, with **no leading delimiter** on the classifier-produced portion.

Tokens produced by `classifyBehaviorV4`, in the order they can appear:

| Token | Emitted at | Meaning |
|---|---|---|
| `farm_collision` | `classifier_v4.php:68` | ±30-min identical-organic-core peer |
| `fast_uninstall` | `:69` | `disabled` is set (any age) |
| `zero_organic` | `:71` | synced at least once, organic core empty |
| `metronome` | `:77` | ≥6 syncs, cv < 0.15, idle > 0.8 |
| `funnel_only` | `:80` | every install-tab domain is a funnel domain |
| `auth_login` | `:83` | `auth_session` |
| `popup_interaction` | `:84` | popup lists non-empty |
| `gov_edu_corp` | `:85` | gov/edu/corp regex hit |
| `organic_domains:<N>` | `:89` | N = `organic_core_size`, only when N > 0 |
| `organic_revisit` | `:92` | revisit with core size ≥ 2 |
| `no_evidence` | `:101` | `n_syncs === 0` |
| `human_gate_uncorroborated:<N>` | `:122` | gate fired; N = signals counted (0 or 1) |

Tokens appended later by finalize, each with an explicit leading `|`:

| Token | Site | Status |
|---|---|---|
| `\|finalize_fast_uninstall_bot` | `finalize_pending_installs.php:468-469` | active — appended when an undecided, uninstalled row is terminalised as `bot` |
| `\|finalize_expired_unpaid` | `finalize_pending_installs.php:570` | **NOT ACTIVE TODAY** — inside `if (!$pureShadow)` (`:563-564`), and all three config flags are false |

This string is a **load-bearing contract**, parsed by substring on both sides:

- `finalize_pending_installs.php:265-271` reconstructs merge features from it — `auth_session` from `strpos($bfM, 'auth_login')`, `popup_interaction`, `no_evidence`, and a synthetic `n_syncs` of `0` or `1` derived from `no_evidence`. (That block sits behind `an_in_behavior_scope()`, so it is **NOT ACTIVE TODAY** — `$BEHAVIOR_AN_ALLOWLIST = []`.)
- `dashboard_detection.php:77` builds its `$PROOF` predicate as `LIKE '%auth_login%' OR LIKE '%popup_interaction%'`, and `:140`, `:186`, `:210-214` count every token above with `LIKE '%token%'`.

> **Note** Because both readers match on substrings, renaming or reordering a token silently breaks the dashboard and the merge with no error. One collision already exists: `finalize_fast_uninstall_bot` **contains** `fast_uninstall`, so `dashboard_detection.php:140`'s `LIKE '%fast_uninstall%'` counter includes rows that were tagged by finalize's heuristic branch and never scored by the classifier at all.

### 7.12 evaluateBehavior()

`behavior_lib.php:420-439` — the only adapter between the classifier and its callers:

```php
$result   = classifyBehaviorV4($features);
$decision = $result['decision'] === 'unknown' ? 'deferred' : $result['decision'];
return ['decision' => $decision, 'flags' => $result['flags'], 'score' => $result['score']];
```

| Classifier output | Value returned to callers | DB `decision` column |
|---|---|---|
| `human` | `human` | `human` (terminal) |
| `bot` | `bot` | `bot` (terminal) |
| `unknown` | `deferred` | `pending` while age ≤ 10800 s, else `deferred` (`behavior_lib.php:497-510`) |

The `unknown → 'deferred'` rename exists purely so every pre-existing caller (`runBehaviorLayer`, `finalize_pending_installs.php`) keeps its vocabulary with zero schema change. Note the split of responsibility: `evaluateBehavior` maps the *word*; `runBehaviorLayer` decides whether that word is transient (`pending`) or terminal (`deferred`) from the install's age against `BEHAVIOR_WINDOW_SEC` (10800 s).

**The `$thresholds` argument is accepted and IGNORED.** `function evaluateBehavior($features, $thresholds = [])` at `:421` never references `$thresholds` in its body; the comment at `:429` states it is present "for signature compatibility". Callers still assemble it — `behavior_lib.php:480-484` and `finalize_pending_installs.php:487-490` both read `$funnel['behavior_thresholds'][$an]` — and `generate_funnel_cache.php:371` passes `$defaultThresholds` with the inline note `// 2nd arg ignored by design`. The producing side was deleted on 2026-07-13 (`generate_funnel_cache.php:348-352`), so the key is no longer even written. **Per-campaign thresholds are permanently disconnected**; every install in every campaign is judged by the global constants in `ninja_v4_weights()`.

### 7.13 Changing the weights

The weights are **code constants** returned by `ninja_v4_weights()` (`classifier_v4.php:16-49`). There is no config file entry, no DB row, and no funnel-cache key that can move them. The only override path is the second parameter `?array $w = null` on `classifyBehaviorV4` (`classifier_v4.php:59-60`) — and **no live caller passes it**: `evaluateBehavior` calls the function with one argument (`behavior_lib.php:430`). The sole user of the parameter is the offline backtester (`replay_v4.php:115-116`, where `$wDecirc` zeroes `fast_uninstall`). Treat it as a testing hook, not a runtime lever.

**Change procedure**

1. Edit the literal in `ninja_v4_weights()`, `classifier_v4.php:18-48`. Change one line at a time — the design goal is to move a line, not ten rules.
2. Run the built-in self-test: `php classifier_v4.php` (`classifier_v4.php:190-243`, 13 classify cases + 12 merge cases). It exits non-zero on any failure. Expect to have to update the expectations if the change is intentional.
3. Run the read-only backtest: `php replay_v4.php` — and read the **decircularised** block (`replay_v4.php:203`), not the headline block, because `fast_uninstall` is outcome-contaminated (§7.10).
4. **`[OWNER — manual]`** Deploy `classifier_v4.php` (steps 1–3 — the weight edit and the CLI self-test/backtest — are agent-runnable offline; this deploy to the live host is the owner's). No DB migration and no cache invalidation is required.

**Rollback:** restore the previous literals and redeploy. Nothing else is needed *going forward* — but note what rollback does **not** undo:

> **Note** Scores and flags already written to `behavior_score` / `behavior_flags` / `decision` are **not** recomputed. `runBehaviorLayer` returns immediately on an install whose `decision` is already `human` or `bot` (`behavior_lib.php:456-458`), and a terminal `deferred` is never reopened (`:503-504`). A weight change therefore takes effect only on installs that are still `pending` or undecided; the dataset ends up mixed, with rows scored under two different scorecards and nothing in the row recording which. If a weight change is material, stamp the deploy time in `DETECTION_CHANGELOG.md` and segment any subsequent analysis by `created_date`.

---

## 8. Decision lifecycle, collection and cadence

This section covers how an install moves from `pending` to a terminal `decision`, what data is collected while it is undecided, and what cadence the extension is told to sync at. Everything here runs today **in full shadow**: the classification path is live and writes `decision` / `behavior_score` / `behavior_flags` / `behavior_hash` on every sync, while every money side-effect inside `runBehaviorLayer()` is gated behind flags that are all `false`.

### 8.1 The observation window and cadence constants

| Constant | Value | Meaning | Defined at |
|---|---|---|---|
| `BEHAVIOR_COLLECT_SYNC_MS` | `600000` (10 min) | `ninja_sync` interval handed to the extension while the install is still collecting | `ninja-adb21.php:87` |
| `BEHAVIOR_WINDOW_SEC` | `10800` (3 h) | Observation window: the age past which an undecided install is force-resolved | `ninja-adb21.php:88`, `behavior_lib.php:15` |
| `BEHAVIOR_SYNC_CAP` | `20` | Maximum `sync_log` rows written per install | `ninja-adb21.php:89` |

The arithmetic is the design intent, reproduced from the code comment at `ninja-adb21.php:87`: *10 min x `BEHAVIOR_SYNC_CAP`(20) = 200 min, covers the full 3h window* (180 min). The cap is therefore not the binding constraint on a well-behaved client — the 3 h age gate closes first, with 20 min of headroom for missed or late syncs.

All three are declared with `if (!defined(...))` guards. `BEHAVIOR_WINDOW_SEC` is deliberately defined **twice** — once in the endpoint (`ninja-adb21.php:88`) and once in the shared library (`behavior_lib.php:15`) — so that `runBehaviorLayer()` has it when invoked from `finalize_pending_installs.php`, which does not load `ninja-adb21.php`.

> The comment at `behavior_lib.php:13-14` calls the second definition "so the cron ... still has it available". On the live Infomaniak host there is no cron: finalize runs in-process post-flush, driven by inbound traffic. The guard is still load-bearing — it covers the standalone/CLI path — but read "cron" as "the finalize pass" everywhere in these comments.

### 8.2 `sync_log` — the collection gate

`logSyncArrival()` (`ninja-adb21.php:2516`) is the only writer of `sync_log`. It writes a row only when **all four** conditions hold (`ninja-adb21.php:2529`):

| # | Condition | Code | Source of the value |
|---|---|---|---|
| 1 | `!$decided` | `!empty($existingUser['decided_date'])` inverted | `ninja-adb21.php:2519` |
| 2 | `$ageSec <= BEHAVIOR_WINDOW_SEC` | age from `created_date`, `PHP_INT_MAX` if absent | `ninja-adb21.php:2520` |
| 3 | `$underCap` | `(int)$existingUser['updates'] < 20` | `ninja-adb21.php:2521` |
| 4 | `!$skipHardBlocked` | `(!$pureShadow && conv === 2)` inverted | `ninja-adb21.php:2527` |

`$pureShadow = empty($BEHAVIOR_ENFORCE) && empty($DEFER_CONVERT) && empty($DELAYED_CONVERSION)` (`ninja-adb21.php:2526`). Today all three flags are `false`, so `$pureShadow` is **true** and condition 4 is always satisfied — **hard-blocked `conv=2` installs are still collected from**. *Stage-1 blocks are observational and may be wrong, so the code deliberately gathers maximum data* (`ninja-adb21.php:2523-2525`).

Inside the gate, two things happen, in order:

1. `$GLOBALS['jsonResponse']['ninja_sync'] = BEHAVIOR_COLLECT_SYNC_MS;` — the 10-minute collection cadence (`ninja-adb21.php:2532`). This is the *only* place on the sync path that sets the fast cadence today; the `runBehaviorLayer` overrides that would follow it are all dormant (see 8.8).
2. The `INSERT` (`ninja-adb21.php:2550`).

Columns written:

| Column | Value | Computation |
|---|---|---|
| `install_id` | `$installId` | — |
| `server_ts` | `NOW()` | server clock, not client |
| `src_ip` | `INET6_ATON($ip)` | request IP for this sync |
| `visit_size` | `count($mergedVisits)` | number of distinct domains in the merged `visit` map (`ninja-adb21.php:2534`) |
| `new_domains` | `max(0, $visit_size - $previous_visit_size)` | `$previous_visit_size` read from the most recent `sync_log` row for this install, `ORDER BY id DESC LIMIT 1` (`ninja-adb21.php:2537-2544`) |
| `visit_hash` | `substr(md5(implode('|', $sorted_keys)), 0, 16)` | keys of `$mergedVisits`, `sort()`ed, joined with `|`, MD5, first 16 hex chars (`ninja-adb21.php:2546-2548`) |

`new_domains` is clamped at 0, so a shrinking domain set records `0` rather than a negative delta. `visit_hash` is a set hash — it changes only when the *set* of domains changes, not when per-domain counts change.

The whole body is wrapped in `try { } catch (\Throwable $e)` which routes to `errorLog($pdo, $ip, 'sync_log_error', ...)` (`ninja-adb21.php:2552-2556`), so a collection failure is silent to the client.

> **Note** — `$existingUser` is the row snapshot taken *before* `updateUserRecord()` bumped `updates`. The cap therefore tests the pre-sync counter: rows are logged while the stored counter is 0..19, giving exactly 20 rows.

### 8.3 Terminalization — the real rule

`runBehaviorLayer()` maps the classifier verdict to the `decision` column at `behavior_lib.php:501-509`. `evaluateBehavior()` (`behavior_lib.php:421`) is a thin wrapper over `classifyBehaviorV4()` that renames the classifier's `unknown` to `deferred` (`behavior_lib.php:430-431`); only that renamed `deferred` is ever provisional.

| Classifier verdict | Stored `decision` was… | Age in-window (`<= 10800 s`) | Age past-window (`> 10800 s`) | Terminal? |
|---|---|---|---|---|
| `human` | anything except `human`/`bot` | `human` | `human` | **Yes — at any age** |
| `bot` | anything except `human`/`bot` | `bot` | `bot` | **Yes — at any age** |
| `deferred` (classifier `unknown`) | `deferred` (already terminal) | `deferred` (preserved) | `deferred` (preserved) | Yes |
| `deferred` (classifier `unknown`) | `pending` / `NULL` / anything else | `pending` | `deferred` | No / Yes |
| `human` or `bot` | already `human` or `bot` | *early-return, nothing written* | *early-return, nothing written* | Already terminal |

Three consequences worth stating plainly:

- **`human` and `bot` are terminal at ANY age.** The `pastWindow` test at `behavior_lib.php:499` is evaluated but is *not* consulted for confident verdicts — the `if ($decision === 'human' || $decision === 'bot')` branch (`behavior_lib.php:501`) fires first and unconditionally. A 4-minute-old install can be stamped terminal.
- **Only `deferred`/`unknown` is provisional**, and only while in-window. Once `$pastWindow` is true it is written as terminal `deferred` — the "3 h forced resolution".
- **An already-terminal `deferred` is preserved** by `behavior_lib.php:503-504`, *but only against another `deferred`*. Because the `human`/`bot` branch is tested first, a later sync that produces `human` or `bot` **overwrites** a stored `deferred`. The `deferred` row is not protected — it is merely not re-opened by more undecidedness.

`$isTerminal = ($dbDecision !== 'pending')` (`behavior_lib.php:510`). The persist statement stamps `decided_date` with a SQL `CASE`:

```
`decided_date` = CASE WHEN ? = 1 AND `decided_date` IS NULL THEN NOW() ELSE `decided_date` END
```

(`behavior_lib.php:520-523`). So `decided_date` is set on the **first** terminal write and never moved afterwards, even if `decision` is later overwritten (`deferred` -> `bot`).

`finalize_pending_installs.php` uses the identical `CASE` idiom in its own terminal writes — the fast-uninstall bot path (`finalize_pending_installs.php:473`) and the behavioural bot path (`finalize_pending_installs.php:508`).

### 8.4 Defect: a terminal verdict permanently stops collection

> **Defect / consequence** — Stamping `decided_date` (`behavior_lib.php:520-523`) trips the `!$decided` gate in `logSyncArrival()` (`ninja-adb21.php:2519`, tested at `ninja-adb21.php:2529`). From that request onward, **no further `sync_log` row is ever written for that install**, and the extension stops being handed `BEHAVIOR_COLLECT_SYNC_MS` from `ninja-adb21.php:2532`.
>
> Because 8.3 makes `human` and `bot` terminal at **any** age, an install classified confidently at minute 4 stops contributing sync data for the remaining 176 minutes of its own observation window. Its `n_syncs`, `interval_cv`, `zero_activity_ratio`, `distinct_ips` and `distinct_prefixes` features (`behavior_lib.php:367-410`) are frozen at whatever the early verdict saw, and they can never be revisited with more evidence — the early-return at `behavior_lib.php:456` also blocks re-evaluation.
>
> This is the same failure mode v4 set out to remove from v3 (the "decided early, blind thereafter" trap). V4-2 removed it for the *undecided* population — `deferred`/`pending` rows keep collecting until the window or the cap closes — but it **survives intact for confident verdicts**, which are precisely the rows a chargeback or a false-positive audit would most want history on.

### 8.5 Feature drift after the window

`visit` is cumulative. `updateUserRecord()` merges each sync's `$domCount` into the stored map with `max($existingCount, (int)$count)` per domain and re-serialises the whole map (`ninja-adb21.php:2459-2464`, `ninja-adb21.php:2494`). Nothing ever truncates it, and the merge is **not** gated by window, cap, or `decided_date` — it runs on every sync of every install, forever.

`extractBehaviorFeatures()` reads that column directly (`behavior_lib.php:265`). So every `visit`-derived feature — `organic_core_size`, `organic_core`, `has_revisit`, `auth_session`, `gov_edu_corp`, and `behavior_hash` itself (`behavior_lib.php:316`) — reflects **all browsing to date**, not browsing inside the 3 h window.

The practical effect: a row re-evaluated after the window (the finalize 3–24 h sweep at `finalize_pending_installs.php:409-418`, or any later sync that reaches `runBehaviorLayer`) can see post-window browsing that no in-window evaluation could have seen. The same install can flip from `pending` to `human` on evidence that arrived at hour 5.

> This applies to the **LIVE classifier**, not only to replay. `replay_v4.php` inherits the same drift because it reads the same cumulative column, but the drift originates in production, on the live path. Two evaluations of one install at different wall-clock times are not comparable on `visit`-derived features. Only the `sync_log`-derived features (`behavior_lib.php:367-410`) are genuinely windowed, and only because collection stops.

### 8.6 The terminal early-return

```php
if (isset($install['decision']) && in_array($install['decision'], ['human', 'bot'], true)) {
    return;
}
```

`behavior_lib.php:455-458`. This is the second statement in the function, after the row load (`behavior_lib.php:448-453`, which also returns early if the row is missing). It runs **before** the `sync_log` fetch, before feature extraction, and before classification.

Consequences: a `human` or `bot` install costs one `SELECT` per sync and nothing more; its `behavior_score` / `behavior_flags` / `behavior_hash` are frozen at the values written by the deciding request; and no enforcement side-effect can ever fire for it again. Note the asymmetry with `deferred` — a terminal `deferred` row does **not** early-return. It runs the full classification on every sync and is re-written as `deferred` by `behavior_lib.php:503-504`, which is exactly how a stored `deferred` can be overwritten to `human`/`bot` later.

### 8.7 Always-persisted columns vs flag-gated writes

**Always written**, on every non-early-returned invocation, by the single `UPDATE` at `behavior_lib.php:515-526`:

| Column | Value |
|---|---|
| `behavior_score` | `$evaluation['score']` from `classifyBehaviorV4()` |
| `behavior_flags` | `implode('|', $flags)` (`behavior_lib.php:511`) — full overwrite, not append |
| `behavior_hash` | `$features['behavior_hash']` — SHA-256 of the sorted organic-core domain list, or `null` when the organic core has fewer than 3 domains (`behavior_lib.php:316-318`) |
| `decision` | `$dbDecision` per the table in 8.3 |
| `decided_date` | `NOW()` on the first terminal write only; otherwise left as-is |

This write is unconditional — it is the shadow-mode product, and it is what makes the system useful today.

**Flag-gated**, and therefore not written today:

| Write | Gate | Line |
|---|---|---|
| `conv` = payout result, + `conv_date` (IF-guarded, canonical 2026-09-02) | `$BEHAVIOR_ENFORCE \|\| $DELAYED_CONVERSION` | ninja `behavior_lib.php:529`, `:555-556` (= canonical Ghost `:530`, `:556-557`; synced 2026-09-03, upload pending) |
| `flagged`, `device_hash`, `fraud_flags` | `$BEHAVIOR_ENFORCE` | `behavior_lib.php:566`, `:581-584`, `:595-599` |
| `conv = '4'` (bot block — **quoted**; `'4'` since the 2026-09-03 conv-attribution convention) + IF-guarded `conv_date` (compare `'4'`) — both builds since the 2026-09-03 sync, uploads pending | `$BEHAVIOR_ENFORCE` | ninja `behavior_lib.php:595-599`; Ghost `:594-598` |
| `ninja_sync` overrides | `$BEHAVIOR_ENFORCE` / `$DELAYED_CONVERSION` | `behavior_lib.php:560`, `:602`, `:616` |

Note that `behavior_flags` is a full overwrite here, whereas `finalize_pending_installs.php:469` *appends* (`($row['behavior_flags'] ?? '') . '|' . $reason`) for its fast-uninstall path. The two writers do not use the same convention.

### 8.8 Enforcement side-effects — **NOT ACTIVE TODAY**

Everything in this sub-section is behind `$BEHAVIOR_ENFORCE` (`config.php:39`) or `$DELAYED_CONVERSION` (`config.php:41`), both `false`. It is documented for completeness and for the day a flag flips.

**A. Human pay path** — **DORMANT**, gate `$BEHAVIOR_ENFORCE || $DELAYED_CONVERSION` (`behavior_lib.php:529`).

Fires only when `$decision === 'human'` and all three guards pass (`behavior_lib.php:537-542`):

| Guard | Test | Purpose |
|---|---|---|
| `$payable` | `(int)($install['conv'] ?? 0) === 0` | blocks re-paying `conv` 1/3, un-blocking hard-fraud `conv=2`, and re-converting expired `conv=4` |
| `!$uninstalled` | `empty($install['disabled'])` | an uninstalled install is never paid |
| `$levelClean` | `(string)($install['level'] ?? '') === '0'` | Stop Ads parity |

> **`level !== "0"` means the install is NEVER paid.** This holds at *every* enforcement site, and `behavior_lib.php:541` is one of them: a behaviourally-perfect human on a non-zero `level` does not get paid here, because Stop Ads would never have paid it at install either.

On pass: `processConversion($install['an'], $install['cid'], $install['sid'], 0)` — note the hardcoded `po = 0` (`behavior_lib.php:544`), with a fallback of `$conv = 1` if the function is not loaded (`behavior_lib.php:546`, the finalize-standalone case). The result is written through the canonical statement (2026-09-02) `UPDATE … SET conv_date = IF(conv <=> ?, conv_date, NOW()), conv = ? WHERE id = ? AND (conv = '0' OR conv IS NULL)` (canonical Ghost `behavior_lib.php:556-557`; ninja `:555-556` since the 2026-09-03 sync) — `conv_date` is assigned **before** `conv` (UPDATE SET evaluates left-to-right, so the NULL-safe compare sees the OLD value and only a real transition moves the stamp), and the `WHERE` conv guard is **race parity with the S1 payout write**: `$payable` was read from a stale row snapshot and `processConversion` can block for seconds on the network, so without it a terminal `conv` (2/4) set concurrently by a sweep would be silently overwritten. `$install['conv']` is then updated locally, and `ninja_sync` is restored to `10800000` (3 h) (`behavior_lib.php:560`).

**B. Bot branch** — **DORMANT**, gate `$BEHAVIOR_ENFORCE` alone (`behavior_lib.php:566`), deliberately independent of `$DELAYED_CONVERSION`.

`$alreadyPaid = in_array((int)($install['conv'] ?? 0), [1, 3], true)` (`behavior_lib.php:568`). `device_hash` is taken from the row or recomputed via `computeDeviceHashFromRow($install)` (`behavior_lib.php:569-571`). The fraud marker is appended only if not already present (`behavior_lib.php:574-576`):

| State | Marker | Write |
|---|---|---|
| Paid (`conv` 1 or 3) | `'\|behavior_bot_post_pay:' . implode(',', $flags)` | `flagged = CURDATE()`, `device_hash`, `fraud_flags`. **`conv` is never touched** — evidence only, for the post-hoc chargeback report (`behavior_lib.php:581-584`) |
| Unpaid | `'\|behavior_bot:' . implode(',', $flags)` | `conv = '4'` (**quoted** — 2026-09-03 convention) + IF-guarded `conv_date` (compare `'4'`; ninja SQL `behavior_lib.php:595-598`, Ghost `:594-597` — synced 2026-09-03, uploads pending), plus `flagged = CURDATE()`, `device_hash`, `fraud_flags` |

> **The value must stay quoted (fixed 2026-09-02; ninja writes `'4'` since 2026-09-03).** `conv` is `enum('0','1','2','3','4')`, and MariaDB/MySQL reads an **unquoted** number assigned to an ENUM as the enum **INDEX** — index 2 = value `'1'`, index 4 = value `'3'`. The pre-fix `SET conv = 2` would have stored `'1'` (**PAID**); an unquoted `4` would store `'3'` (also paid). Dead code in Stage 1 (flags false), but armed for Stage 2. Canonical rule in §14.1. Both builds stamp `conv_date` via the IF-guard, comparing `'4'` (a same-value rewrite must not move the stamp) — aligned in the 2026-09-03 sync (uploads pending).

**C. Deferred cadence override** — **DORMANT**, gate `$BEHAVIOR_ENFORCE` (`behavior_lib.php:601-602`). `$decision === 'deferred'` sets `ninja_sync = 600000` (10 min). This is a no-op in practice even when enabled, since `logSyncArrival()` already set the same value at `ninja-adb21.php:2532` — it matters only for rows where the collection gate was closed.

**D. `DELAYED_CONVERSION` fast re-sync** — **DORMANT**, gate `$DELAYED_CONVERSION` (`behavior_lib.php:607`). Re-reads `conv` and `decision` from the DB *after* all the writes above (`behavior_lib.php:608-610`), and if the install is unpaid (`conv` not in `[1, 3]`) and not a bot, sets `ninja_sync = 300000` (5 min) (`behavior_lib.php:616`). This is the last cadence writer in the function and therefore wins over A and C.

> **Note** — none of this is the Stage-1 delay. The Stage-1 delay (`$STAGE1_DELAY_CONV_AN = ['*']`, `$STAGE1_DELAY_CONV_SEC = 3600`) is **timing only**: same gate, same recipients, postback ~1 h late. It lives at `ninja-adb21.php:3330` and is fired by `finalize_pending_installs.php`. Stage 2 — the refusal machinery in this sub-section — is **off**, because `$BEHAVIOR_AN_ALLOWLIST = []`. No install can be REFUSED today. See 8.10.

### 8.9 Call sites and ordering

| # | File:line | Context | `runBehaviorLayer` at |
|---|---|---|---|
| 1 | `ninja-adb21.php:2748` | `createUser` — duplicate-fingerprint rebuild path | `:2750` |
| 2 | `ninja-adb21.php:2866` | `createUser` — duplicate cid/sid rebuild path | `:2868` |
| 3 | `ninja-adb21.php:3017` | `createUser` — duplicate-IP rebuild path | `:3019` |
| 4 | `ninja-adb21.php:3544` | main existing-user sync path (the common case) | `:3546` |
| 5 | `finalize_pending_installs.php:500` | post-flush finalize sweep, `human` branch only | `:500` |

The four endpoint call sites share an identical shape:

1. `updateUserRecord(...)` returns `$mergedVisits` and persists the merged `visit` and the incremented `updates`.
2. `$jsonResponse['ninja_visited_sites'] = $mergedVisits;`
3. **`logSyncArrival($pdo, $id, $ip, $existingUser, $mergedVisits);`** — always first.
4. `runBehaviorLayer(...)` inside a `try` / `catch (\Throwable $eBehavior)`.

The ordering is load-bearing: `logSyncArrival` must run **before** `runBehaviorLayer` so that this request's sync row exists in `sync_log` before the classifier fetches the history at `behavior_lib.php:461-463`. It also means `logSyncArrival` sees the *pre-request* `decided_date` from `$existingUser`, so a terminal verdict reached in this very request does not retroactively suppress this request's own sync row — it suppresses the next one.

The `catch` at every site is:

```php
} catch (\Throwable $eBehavior) {
    errorLog($pdo, $ip, 'behavior-layer-exception', $eBehavior->getMessage());
}
```

(`ninja-adb21.php:2751-2753`, `:2869-2871`, `:3020-3022`, `:3547-3549`).

> **Note** — this swallows the exception into the `errors` table (`errorLog`, `ninja-adb21.php:805`) and lets the response continue normally. A behaviour-layer failure is **invisible in the HTTP response** and invisible in the row: `decision` stays whatever it was, no flag is raised on the install itself. The only trace is a row in `errors` with `type = 'behavior-layer-exception'`. Monitor that type; silence there is the only evidence the layer is running at all.

Call site 5 is different: `finalize_pending_installs.php` classifies the row itself first (`:481-496`) and delegates to `runBehaviorLayer` **only** on a `human` verdict, precisely because that function owns the payout side-effect. `bot` and `deferred` verdicts are written by finalize's own SQL (`:502-508` and `:512+`) without re-entering the behaviour layer. Before each row, finalize re-derives the three flags per-network (`:450-453`), so a non-allowlisted network — which today is *every* network — gets `$BEHAVIOR_ENFORCE = $DELAYED_CONVERSION = $DEFER_CONVERT = false` even if the global config were flipped on.

> **Note** — the hard-guarantee straggler/expiry sweep at `finalize_pending_installs.php:552-579` is wrapped in `if (!$pureShadow)` (`:563-564`), where `$pureShadow` is derived from the same three flags. It **never runs today**. Any claim that the system guarantees "zero non-terminal installs past 24 h" is not enforced in the live configuration.

### 8.10 Scope helpers

Three helpers in `behavior_lib.php` decide *who* is in scope for what. They are deliberately independent of each other.

| Helper | Line | Reads | Returns true when |
|---|---|---|---|
| `an_in_behavior_scope($an)` | `behavior_lib.php:223` | `$BEHAVIOR_AN_ALLOWLIST` | `!empty($an) && in_array($an, $list, true)` |
| `s1_delay_in_scope($an)` | `behavior_lib.php:237` | `$STAGE1_DELAY_CONV_AN` | `!empty($an) && (in_array('*', $list, true) \|\| in_array($an, $list, true))` |
| `s1_delay_seconds($an)` | `behavior_lib.php:248` | `$STAGE1_DELAY_CONV_SEC_AN`, `$STAGE1_DELAY_CONV_SEC` | — returns an integer |

**`an_in_behavior_scope`** is the **Stage-2/3 enforcement switch** — the bot/Tier-3 *refusal* canary. `$BEHAVIOR_AN_ALLOWLIST = []` (`config.php:46`), so it returns `false` for every input. Both `!empty($an)` and strict `in_array` mean `an = NULL` (organic), `an = ''` and `an = '0'` are all out of scope by construction. Live effect: **Stage 2 is off; no install can be REFUSED today.** Its call sites — `ninja-adb21.php:2685`, `ninja-adb21.php:3456`, `finalize_pending_installs.php:255`, `finalize_pending_installs.php:450` — all take the Stage-1 branch on every request.

**`s1_delay_in_scope`** is the **STAGE-1 DELAY** — *timing only*. `$STAGE1_DELAY_CONV_AN = ['*']` (`config.php:68`), so it returns `true` for every non-empty `an`. At `ninja-adb21.php:3330` it sets `$s1DelayHold = true` instead of calling `processConversion`, leaving `conv = 0` and relying on the `'|s1_hold'` marker plus `finalize_pending_installs.php` to fire the postback later.

> **The two are constantly confused. They are not the same gate.** `s1_delay_in_scope` changes **WHEN** the Stage-1 postback fires — same gate, same recipients, ~1 h late. `an_in_behavior_scope` changes **WHO** gets paid at all, and it is empty. A `['*']` Stage-1 delay list does not put anyone into Stage 2. The code says this itself at `behavior_lib.php:233-235`: the delay list is *"Deliberately INDEPENDENT of an_in_behavior_scope()/$BEHAVIOR_AN_ALLOWLIST"*.

**`s1_delay_seconds`** resolution order, exactly (`behavior_lib.php:249-252`):

1. If `$an` is non-empty **and** `isset($STAGE1_DELAY_CONV_SEC_AN[$an])` -> return `(int)` that per-network value.
2. Else -> return `(int)$STAGE1_DELAY_CONV_SEC` if set.
3. Else -> return the hardcoded literal `3600`.

Today `$STAGE1_DELAY_CONV_SEC_AN = []` (`config.php:70`), so step 1 never hits and every network resolves to `$STAGE1_DELAY_CONV_SEC = 3600` (1 h) via step 2. The historical `['gn' => 10800]` override from the 2026-07-12 rollout is retired. Note that the per-network map uses `isset()`, not `in_array` — a key with value `0` would return `0` (immediate release), while a key mapped to `null` would fall through to the default.

**The `'*'` asymmetry.** `s1_delay_in_scope` explicitly tests `in_array('*', $list, true)` (`behavior_lib.php:240`). `an_in_behavior_scope` does **not** — it is `in_array($an, $list, true)` only (`behavior_lib.php:226`), pure exact-match. `config.php:49` states this outright: *"No '*' support here: an_in_behavior_scope() is exact-match only."* Putting `'*'` into `$BEHAVIOR_AN_ALLOWLIST` would enable Stage 2 for exactly one network — a literal `an` of `'*'`, which does not exist — and nothing else. To enable Stage 2 you must enumerate the networks by code.

**The 72 h pay window.** `$S1_HOLD_PAY_WINDOW_SEC = 259200` (`config.php:89`) is not consulted by any of these three helpers; it bounds how long a held row stays rescuable by the finalize S1 sweep, and it appears as the exclusion clause in the conv-hygiene bulk `UPDATE` (`finalize_pending_installs.php:391-392`) so that one failed sweep pass cannot forfeit an in-flight held row.

> **Defect** — the whole Stage-1 hold depends on the `'|s1_hold'` marker actually reaching `fraud_flags`. If `$HAS_FRAUD_COLUMNS` (`config.php:36`, surfaced as `$GLOBALS['has_fraud_columns']` at `ninja-adb21.php:101`) were `false`, `fraud_flags` is not INSERTed, the marker is never written, the finalize sweep — which selects on `LIKE '%|s1_hold%'` — never finds the row, and **every payout is silently lost** for as long as the global hold is on. There is no counter-check: `conv` stays `0`, which is indistinguishable from a legitimately-held row.

---

## 9. Finalization — the money and decision sweeps

`finalize_pending_installs.php` is the post-install backstop: it fires the held Stage-1 postbacks, closes conv bookkeeping, and forces a terminal `decision` on installs that are 3-24h old. `farewell.php` (uninstall checkpoint) and `sanitize_fast_uninstalls.php` (manual one-shot) sit on the same money path and are documented here with it.

The file runs its work in exactly this order: bootstrap/guards → **Sweep 1** (s1_hold money) → money-first early return → **conv hygiene** → **decision sweep** → **straggler/expiry** (dormant) → stats.

> The whole file is written so that a partial run is always safe: every sweep is either a bulk idempotent UPDATE or is protected by an atomic claim, so stopping anywhere loses nothing but time.

> **Canonical (2026-09-02): every `conv` write in this file also maintains `conv_date`** (§14.1). The four writes whose `WHERE` guard (`conv = '0' OR conv IS NULL`) guarantees a real transition use plain `conv_date = NOW()` — forfeit, Stage-2 refuse, hygiene, expiry (canonical: `finalize_pending_installs.php:232`, `:280`, `:384`, `:570`). The two payout persists, which can rewrite `conv` with its existing value, use `SET conv_date = IF(conv <=> ?, conv_date, NOW()), conv = ?` with `conv_date` assigned **before** `conv` (canonical: `:315`, `:538`). The SQL quotes in 9.3-9.7 show the canonical shapes.

### 9.1 Two run modes: standalone and INLINE

The mode is decided by one global, `$GLOBALS['NINJA_FPI_DEADLINE']`, read into `$fpiDeadline` at `finalize_pending_installs.php:57`.

| | Standalone (cron / manual / CLI) | INLINE (post-flush slice) |
|---|---|---|
| Trigger | external scheduler or a browser hit | `ninja_run_inline('finalize_pending_installs.php', …)` — `ninja-adb21.php:3810` |
| `$fpiDeadline` | `null` | `microtime(true) + max(1.0, $budgetSec)` — `ninja-adb21.php:3761` |
| Budget | none (`set_time_limit(300)`, `finalize_pending_installs.php:31`) | **12.0s** when the response detached, **2.0s** when it did not — `ninja-adb21.php:3810` |
| `$fpiRowLimit` | `0` = unlimited | `400` — `finalize_pending_installs.php:58` |
| Money sweep rows | all due | `ORDER BY created_date ASC LIMIT 400` — `:206` |
| Hygiene rows | all matching | `LIMIT 5000` — `:395` |
| Decision sweep rows | all in window | `ORDER BY created_date ASC LIMIT 150` — `:419` (literal 150, not `$fpiRowLimit`) |
| 300s recent-run guard | enforced | bypassed — `:91` |
| Stats output | `echo` to stdout | `echo` captured by `ob_start()` and `error_log`ged, first 400 chars — `ninja-adb21.php:3763-3774` |

**On the live host only the INLINE path runs.** Infomaniak has no cron; the trigger is inbound traffic: `ninja-adb21.php:3806-3811` fires the inline run when `last_finalize_run.txt` is older than 1200s, or older than 60s while `finalize_backlog.txt` exists.

> **Note** `$isStandalone` (`:64-65`) is *not* the mode switch — it is true whenever `$pdo` is unset in scope, which includes inline runs (`ninja_run_inline` deliberately does not declare `$pdo` global, `ninja-adb21.php:3749-3753`). So an inline slice also takes the flock, opens its own guarded connection, touches the run stamp, and prints its stats line. Only `$fpiDeadline` distinguishes the two worlds.

**Per-row deadline checks.** Money sweep: `if ($fpiDeadline !== null && microtime(true) >= $fpiDeadline) { $fpiBudgetHit = true; break; }` at `:214-217` — placed **before** the atomic claim so a budget stop can never strand a claimed-but-unpaid row. Decision sweep: `if ($fpiDeadline !== null && microtime(true) >= $fpiDeadline) { break; }` at `:442`.

**Money-first early return** (`:354-361`): after Sweep 1, if `$fpiDeadline !== null` and (`$fpiBudgetHit` or `microtime(true) >= $fpiDeadline - 1.0`), the file `return`s. Hygiene, decision and expiry are shadow bookkeeping and are skipped for that slice. *The 1.0s margin exists because the sweeps below open with bulk statements that cannot be interrupted mid-flight.*

**Breadcrumbs** (`:333-341`): `finalize_backlog.txt` is touched when the slice hit its budget **or** `$s1Stats['due'] >= $fpiRowLimit` (the LIMITed SELECT came back full); otherwise it is unlinked and `last_finalize_success.txt` is touched. It is deliberately **not** touched from the sweep's `catch` (`:342`), so an unknown state preserves whatever signal already exists.

### 9.2 Guards, in order

| # | Guard | Line | What it proves | Live behaviour |
|---|---|---|---|---|
| 1 | Recent-run guard: `$fpiDeadline === null && file_exists($__fpiRunStamp) && (time() - filemtime($__fpiRunStamp)) < 300` → `return` | `:91-96` | No run *started* in the last 300s (catches a cron-vs-fallback drift duplicate arriving after the earlier run already finished and released the lock) | **Bypassed** — inline mode always has a deadline; cadence is set by the trigger instead |
| 2 | `flock($__fpiLockFp, LOCK_EX \| LOCK_NB)` on `finalize_pending_installs.lock` | `:97-101` | No run is *currently* in progress; single-flights concurrent attempts | Active. The handle is scope-local, so the lock releases when `ninja_run_inline` returns |
| 3 | Guarded connect: `new PDO(...)` inside `try/catch`, `PDO::ATTR_TIMEOUT => 10` | `:107-121` | A working DB connection exists *before* anything claims a run happened | Active; on failure it logs and `return`s, leaving the stamp stale so the next request retries |
| 4 | `@touch($__fpiRunStamp)` (`last_finalize_run.txt`) | `:126` | A run started **and** reached a working DB | Active — this stamp is what keeps the 1200s trigger dormant |

> *The connect-then-stamp ordering is the fix for "runs that started but never applied": an unwrapped `new PDO` after the touch made a broken environment look like a healthy 15-minute cadence.*

Guards 2-4 all sit inside `if (!isset($pdo))`. `$table` is resolved at `:128` as `$GLOBALS['table'] ?? ($table ?? 'ninja25')`; live it is `ninja25` (`ninja-adb21.php:93`) — the in-code comment `prod = stopads24` does not describe this deployment. The funnel cache is loaded from `funnel_cache.json` into `$GLOBALS['funnel']` if absent (`:142-150`).

### 9.3 Sweep 1 — the s1_hold money sweep

This is the only sweep in the file that moves money, and it is **live today**. It is Stage-1 money on the Stage-1 gate, fired late; it is **not** Stage 2 and it ignores `$BEHAVIOR_ENFORCE` / `$DEFER_CONVERT` / `$DELAYED_CONVERSION` entirely.

Resolved constants (`:172-192`):

| Variable | Source | Live value |
|---|---|---|
| `$s1DelaySec` | `$STAGE1_DELAY_CONV_SEC` (`config.php:69`) | `3600` |
| `$s1MinDelay` | `min($s1DelaySec, …$STAGE1_DELAY_CONV_SEC_AN)` | `3600` (`$STAGE1_DELAY_CONV_SEC_AN = []`, `config.php:70`) |
| `$s1PayWindowSec` | `$S1_HOLD_PAY_WINDOW_SEC` (`config.php:89`) | `259200` (72h) |
| `$s1SelectFloorSec` | `$s1PayWindowSec + 86400` | `345600` (96h) |
| `$fpiRowLimit` | inline | `400` |

Selection is by the `'|s1_hold'` marker written at install (`ninja-adb21.php:3414`), **not** by `$STAGE1_DELAY_CONV_AN` — so emptying that list still drains rows already held.

```sql
SELECT `id`, `an`, `cid`, `sid`, `level`, `disabled`, `created_date`,
       `decision`, `behavior_flags`, `fraud_flags`, `is_vm`
FROM `ninja25`
WHERE `fraud_flags` LIKE '%|s1_hold%'
  AND (`conv` = '0' OR `conv` IS NULL)
  AND `created_date` < NOW() - INTERVAL 3600 SECOND
  AND `created_date` > NOW() - INTERVAL 345600 SECOND
ORDER BY `created_date` ASC LIMIT 400
```
(`:197-206`; the two `INTERVAL` values are `{$s1MinDelay}` and `{$s1SelectFloorSec}`, interpolated as integers.)

> *The lower bound is the pay window **plus 24h** so a row that just aged out is still fetched and gets a proper `'|s1_forfeit'`, instead of silently dropping out of this sweep and being closed later by the generic hygiene marker.*

Per-row order (`:210-326`):

| Step | Line | Condition | Effect |
|---|---|---|---|
| 0 | `:214` | slice budget spent | `$fpiBudgetHit = true; break` |
| 1 | `:218` | — | `$s1Stats['due']++`; `$ageSec = time() - strtotime(created_date)` |
| 2 | `:229` | `!empty($held['disabled']) \|\| $ageSec > 259200` | **forfeit** (below) |
| 3 | `:242-246` | `$ageSec < s1_delay_seconds($an)` (live: 3600) | `held++`, skip |
| 4 | `:255-289` | `an_in_behavior_scope($an)` | mergeConvV4 branch — **DORMANT**, see 9.4 |
| 5 | `:294-297` | `(string)$held['level'] !== '0'` | `held++`, skip — it ages out to forfeit |
| 6 | `:300-307` | — | **atomic claim** |
| 7 | `:309-318` | claim won | `processConversion($an, $cid, $sid, 0)` then persist `conv` + `conv_date` |

The forfeit branch (canonical: `:229-235`) — note it is tested *before* the due test, so an uninstalled row forfeits early:

```sql
UPDATE `ninja25`
   SET `fraud_flags` = REPLACE(`fraud_flags`, '|s1_hold', '|s1_forfeit'),
       `conv` = '4',
       `conv_date` = NOW()
 WHERE `id` = ? AND `fraud_flags` LIKE '%|s1_hold%'
   AND (`conv` = '0' OR `conv` IS NULL)
```
(Plain `NOW()` is correct here: the `WHERE` guard admits only rows still at `conv` 0/NULL, so this is always a real transition — §14.1.)

The atomic claim (`:300-307`) — `'|s1_hold'` → `'|s1_paid'` **before** the postback:

```sql
UPDATE `ninja25`
   SET `fraud_flags` = REPLACE(`fraud_flags`, '|s1_hold', '|s1_paid')
 WHERE `id` = ? AND (`conv` = '0' OR `conv` IS NULL)
   AND `fraud_flags` LIKE '%|s1_hold%'
```
Only the run whose `rowCount() === 1` proceeds; anyone else `continue`s. Then `processConversion(...)` fires (`po` is not persisted, so it always fires with `po = 0`, `:311`) and `UPDATE … SET conv_date = IF(conv <=> ?, conv_date, NOW()), conv = ? WHERE id = ? AND (conv = '0' OR conv IS NULL)` records the result (canonical: `:315-316`). This persist uses the IF-guard, not plain `NOW()`, because it can rewrite the same value — `processConversion` can return `0` over a row already at `'0'` — and `conv_date` is assigned **before** `conv` so the NULL-safe compare (`<=>`) sees the OLD value (SET evaluates left-to-right; §14.1).

Why claim-before-postback makes partial slices and repeated runs safe:
- The claim removes the row from the SELECT predicate (`LIKE '%|s1_hold%'` no longer matches), so no later slice, concurrent run or retry can re-offer it.
- The markers are substring-safe: `'|s1_hold'` is not contained in `'|s1_paid'`, `'|s1_forfeit'` or `'|s1_refused'`.
- The deadline check at `:214` sits *before* the claim, so a budget stop always cuts between rows, never between claim and postback.
- The trade is deliberate: **at-most-once**, not at-least-once. A process death between claim and postback leaves one row at `'|s1_paid'` with `conv = 0` — visible for audit, rather than risking a double postback.

> **Defect** `$HAS_FRAUD_COLUMNS = false` (`config.php:36`, live `true`) would drop `fraud_flags` from the install INSERT entirely (`ninja-adb21.php:1806-1815`). The `'|s1_hold'` marker would never be written, this sweep would never find the row, and — with the global hold on for every network (`$STAGE1_DELAY_CONV_AN = ['*']`) — **every payout would be silently lost**, with no error anywhere.

### 9.4 The mergeConvV4 branch inside Sweep 1 — **DORMANT**

Gate (`:255-256`): `function_exists('an_in_behavior_scope') && an_in_behavior_scope($held['an']) && function_exists('mergeConvV4')`. `an_in_behavior_scope` (`behavior_lib.php:223-227`) requires `in_array($an, $BEHAVIOR_AN_ALLOWLIST, true)`, and the allowlist is `[]` (`config.php:46`). **This block never executes today.** It is the only Stage-2 refusal path in the money sweep — with it dark, no held install can be refused; every held row either pays or forfeits on age.

When it does run, in order:

1. **Decision gate** (`:257-264`): if `decision` is `''` or `'pending'`, the row `held++` and stays — never pay before the verdict.
2. **Feature reconstruction from the `behavior_flags` string** (`:265-272`):

| Feature | Derivation |
|---|---|
| `auth_session` | `strpos($bfM, 'auth_login') !== false` |
| `popup_interaction` | `strpos($bfM, 'popup_interaction') !== false` |
| `no_evidence` | `strpos($bfM, 'no_evidence') !== false` |
| `n_syncs` | `no_evidence ? 0 : 1` — *back-up only; the `no_evidence` flag is authoritative* |

3. **Tier** (`:273`): `deriveTierFromRow($held)` (`classifier_lib.php:160-170`) — `'tier3'`/`'tier2'`/`'tier1'` substring of `fraud_flags`, else `is_vm === 1` → 3, else `level !== '0'` → 3, else 1. *This is the only function still live in the otherwise dormant `classifier_lib.php`.*
4. **Merge** (`:274`): `mergeConvV4($tierM, $decM, $fM)` (`classifier_v4.php:158-184`), which maps `deferred`/`pending`/`''` → `unknown` first.

Three outcomes:

| Outcome | Condition (`classifier_v4.php`) | What Sweep 1 does |
|---|---|---|
| stay-held-because-pending | `decision` not terminal (`:257-264`) | `held++`, `continue` — retried on a later slice |
| refuse | `mvM['pay']` empty: `decision === 'bot'` at any tier (`:164-166`), or tier 3 without `auth_session`/`popup_interaction` proof (`:169-175`) | atomic refuse UPDATE (below), `conv = '4'` + `conv_date = NOW()`, `refused++` |
| pay | tier 1/2 `human` or `unknown` (ghost included, `:178-183`), or tier 3 `human` with proof | falls through to the `level === '0'` check and the atomic claim of 9.3 |

The refuse UPDATE (canonical: `:277-283`) mirrors the pay claim's idempotency and appends the merge reason:

```sql
UPDATE `ninja25`
   SET `fraud_flags` = CONCAT(REPLACE(`fraud_flags`, '|s1_hold', '|s1_refused'), ?),
       `conv` = '4',
       `conv_date` = NOW()
 WHERE `id` = ? AND (`conv` = '0' OR `conv` IS NULL)
   AND `fraud_flags` LIKE '%|s1_hold%'
```
with the bound parameter `'|' . ($mvM['reason'] ?? 'merge_refused')` — reasons are `t{1,2,3}_bot_refused`, `t3_human_no_proof_refused`, `t3_{decision}_refused` (`classifier_v4.php:165-174`). `refused++` only when `rowCount() === 1`.

> **Note** Promoting a network to `$BEHAVIOR_AN_ALLOWLIST` **must** be paired with `$STAGE1_DELAY_CONV_SEC_AN[$an] = 10800` in `config.php`. With the default 3600s delay the row reaches this checkpoint at 1h, but the decision sweep only touches rows older than 3h (`:415`), so `decision` is still `pending` and every row takes the stay-held branch at `:262` — payouts would drift to whichever later slice happens to see a terminal verdict, and rows would age toward forfeit for no reason. `BEHAVIOR_WINDOW_SEC` is 10800 (`behavior_lib.php:15`); the 3h delay exists to line the checkpoint up with it.

> **Note** The in-code comment at `:260-261` ("the merge fires on the next cron pass (~15 min later)") is stale: there is no cron on the live host. The retry happens on the next traffic-driven inline slice.

### 9.5 The conv-hygiene sweep

One bulk UPDATE, run on every slice that survives the money-first early return (`:377-406`). *An attributed install may never rest at `conv = 0`: `0` means "unattributed" or "in-flight" only.*

```sql
UPDATE `ninja25`
   SET `conv` = '4',
       `conv_date` = NOW(),
       `fraud_flags` = CONCAT(COALESCE(`fraud_flags`, ''), '|conv4_expired_unpaid')
 WHERE `an` IS NOT NULL AND `cid` IS NOT NULL
   AND (`conv` = '0' OR `conv` IS NULL)
   AND `created_date` >= '2026-07-13 09:00:00'
   AND `created_date` <= NOW() - INTERVAL 24 HOUR
   AND (COALESCE(`fraud_flags`, '') NOT LIKE '%|s1_hold%'
        OR `created_date` <= NOW() - INTERVAL 259200 SECOND)
 LIMIT 5000
```
(canonical: `:382-396`; the cutoff is bound as a parameter from `$CONV4_HYGIENE_FROM`, `:375`; the interval is `{$s1PayWindowSec}`; the `LIMIT 5000` is appended only when `$fpiRowLimit > 0`, i.e. inline. Plain `NOW()` on `conv_date` — the `conv = '0' OR IS NULL` guard makes every matched row a real transition, §14.1.)

| Element | Value / rule | Why |
|---|---|---|
| Forward-only cutoff | `$CONV4_HYGIENE_FROM = '2026-07-13 09:00:00'` (`:376`) | User decision: **no backfill**. Rows created before the rollout keep their historical `conv` forever |
| Age | `created_date <= NOW() - INTERVAL 24 HOUR` | The postback window is gone; the row is terminal not-paid |
| Attribution | `an IS NOT NULL AND cid IS NOT NULL` | Unattributed rows legitimately rest at `conv = 0` |
| s1-hold exclusion | `NOT LIKE '%|s1_hold%'` **OR** older than 72h | A held row inside the pay window is **in-flight** and belongs to Sweep 1, which still pays it at the first catch-up run. Without this clause one failed Sweep-1 pass would let this bulk UPDATE forfeit rescuable rows. `COALESCE` keeps NULL-`fraud_flags` rows closable |
| Flag gating | **none** | Deliberate: this is bookkeeping on money that was already not paid, not enforcement — so unlike the expiry sweep of 9.7 it runs in full shadow |

Rows closed here are gate failures the hold never touched (dirty `level` digits, missing sub-values), rows whose `level` flipped while held, and crash leftovers (`'|s1_paid'` claimed but the postback never fired). `rowCount()` is echoed when non-zero (canonical: `:397-400`).

### 9.6 The decision sweep

```sql
SELECT *, INET6_NTOA(`ip`) AS `ip_str`
  FROM `ninja25`
 WHERE (`decision` = 'pending' OR `decision` IS NULL)
   AND `created_date` < NOW() - INTERVAL 3 HOUR
   AND `created_date` > NOW() - INTERVAL 24 HOUR
 ORDER BY `created_date` ASC LIMIT 150
```
(`:412-419`.) **There is no `conv` filter** — `decision` is a behavioural fact resolved independently of payment, so paid installs that went silent terminalize too. No backlog breadcrumb is needed here: the 3-24h window re-offers unfinished rows to every run.

Per-row flag re-derivation (`:435-437`, `:450-453`): the three config flags are snapshotted once into `$cfgEnforce` / `$cfgDelayed` / `$cfgDefer`, then re-derived per row as `$cfg… && an_in_behavior_scope($row['an'])`. With the allowlist empty, `$inScope` is always false, so `$BEHAVIOR_ENFORCE`, `$DELAYED_CONVERSION` and `$DEFER_CONVERT` are **false for every row** regardless of config.

**Fast-uninstall shortcut** (`:467-478`): if `!empty($row['disabled'])`, the row is terminalized as a bot with no classification at all.

```sql
UPDATE `ninja25`
   SET `decision` = 'bot',
       `behavior_flags` = ?,   -- (existing behavior_flags) . '|finalize_fast_uninstall_bot'
       `decided_date` = CASE WHEN `decided_date` IS NULL THEN NOW() ELSE `decided_date` END
 WHERE `id` = ?
```

What it does **not** do: it does not read `sync_log`, does not call `extractBehaviorFeatures`/`evaluateBehavior`, and writes **no** `behavior_score`, **no** `behavior_hash`, **no** `fraud_flags`, **no** `flagged`, and **no** `conv` — `conv` stays exactly where it was, and `processConversion` is never reached. Money is untouched by this branch.

**Classifier fallthrough** (`:481-541`): fetch `sync_log` ordered by `server_ts ASC`, `extractBehaviorFeatures($row, $syncRows, $funnel, $pdo, $table)`, pick `$funnel['behavior_thresholds'][$an]` if present (accepted for signature compatibility but **not consumed** — `behavior_lib.php:421-432`), then `evaluateBehavior` → `classifyBehaviorV4`, with `unknown` mapped to `deferred`.

| Verdict | Line | Write shape |
|---|---|---|
| `human` | `:498-501` | Delegates to `runBehaviorLayer($pdo, $id, $ipStr, $row, whitelistedDom, blockedDom)`. It re-reads the row, re-extracts features and writes `behavior_score`, `behavior_flags`, `behavior_hash`, `decision`, `decided_date` itself (`behavior_lib.php:514-526`) — the sweep's own `$score`/`$flags` are discarded. Its payout and blocking side-effects are gated on `BEHAVIOR_ENFORCE`/`DELAYED_CONVERSION` and are dark |
| `bot` | `:502-511` | `UPDATE … SET behavior_score = ?, behavior_flags = ?, behavior_hash = ?, decision = 'bot', decided_date = CASE WHEN decided_date IS NULL THEN NOW() ELSE decided_date END WHERE id = ?` — `conv` stays 0 by policy |
| `deferred` | `:512-541` | Same five columns with `decision = 'deferred'`. Then, **only** if `$BEHAVIOR_ENFORCE && $DEFER_CONVERT && (string)$row['level'] === '0'`, `processConversion($an, $cid, $sid, 0)` and `UPDATE … SET conv_date = IF(conv <=> ?, conv_date, NOW()), conv = ? WHERE id = ? AND (conv NOT IN ('1','3') OR conv IS NULL)` (canonical: `:538-539`). The `OR conv IS NULL` is the 2026-09-02 three-valued-logic fix: `NULL NOT IN ('1','3')` evaluates to NULL, not true, so before it a NULL-`conv` row would fire the postback yet never record the payout. The IF-guard (not plain `NOW()`) because this guard admits same-value rewrites — `0` over `'0'`, `2` over `'2'` — which must not move `conv_date` (§14.1). **NOT ACTIVE TODAY** — both flags are false and `$inScope` is false |

The `level === '0'` test at `:524` is the same absolute rule as everywhere else: **a `level !== "0"` install is never paid, at any enforcement site.**

> **Note** The in-code comment at `:463-466` claims 0-sync rows are decided "tier-aware (Tier 1 no-sync -> unknown/deferred; Tier 2/3 no-sync -> bot)". **v4 made this untrue.** `evaluateBehavior` calls `classifyBehaviorV4`, which is tier-blind (`behavior_lib.php:421-432`); a 0-sync install scores 0 → `unknown` → `deferred` in **every** tier. The tier-aware behaviour described belongs to `classifyBehavior` in the dormant `classifier_lib.php`.

### 9.7 The straggler/expiry sweep — **NOT ACTIVE TODAY**

`$pureShadow = !$cfgEnforce && !$cfgDelayed && !$cfgDefer` (`:563`) is **true** live, and the whole block sits inside `if (!$pureShadow)` (`:564`). It never runs.

```sql
UPDATE `ninja25`
   SET `conv` = '4',
       `conv_date` = NOW(),
       `decision` = 'deferred',
       `behavior_flags` = CONCAT(COALESCE(`behavior_flags`, ''), BINARY '|finalize_expired_unpaid'),
       `decided_date` = CASE WHEN `decided_date` IS NULL THEN NOW() ELSE `decided_date` END
 WHERE (`decision` = 'pending' OR `decision` IS NULL)
   AND (`conv` = '0' OR `conv` IS NULL)
   AND `created_date` <= NOW() - INTERVAL 24 HOUR
```
(canonical: `:568-576`; `$stats['expired'] = $stmtExpire->rowCount()` at `:579`, so the stats line always reports `Expired: 0` today. Plain `NOW()` on `conv_date` — the `conv = '0' OR IS NULL` guard guarantees a real transition, §14.1.)

*The stage gate is intentional: in pure shadow `conv` must stay frozen at its install value so the observation data says "decide nothing".* The cost is precise, and the two axes diverge:

| Axis | Backstop past 24h | Status |
|---|---|---|
| `conv` (money) | conv-hygiene sweep, 9.5 — ungated, runs every slice | **Closed.** Attributed rows never rest at `conv = 0` past 24h (forward of the 2026-07-13 cutoff) |
| `decision` (behaviour) | this sweep only | **Open.** Nothing terminalizes a row once it ages past the 3-24h window |

Consequence: **a backlog of `decision = 'pending'`/NULL rows older than 24h accumulates permanently.** Any claim that the system guarantees "zero non-terminal installs past 24h" is false for the current configuration. Note the asymmetry carefully — the hygiene sweep closes those same rows' `conv` to `4`, but it does **not** write `decision`, so a >24h row can sit at `conv = 4` and `decision = 'pending'` forever.

### 9.8 `farewell.php` — the uninstall checkpoint

The uninstall page the browser opens when the extension is removed. It is a required part of the money path.

| Aspect | Detail |
|---|---|
| Trigger | GET with `?uid=…`, guarded by `isset($_GET['uid']) && $_GET['uid'] != 'undefined'` (`farewell.php:71`); `+` restored from spaces (`:72`) |
| Table | hardcoded `'ninja25'` (`:15`) — matches the live table (`ninja-adb21.php:93`) |
| Path A — fingerprint | `strlen($uid) === 64 && ctype_xdigit($uid)` → `UPDATE ninja25 SET disabled = ? WHERE fingerprint = ?` with `date('Y-m-d')` (`:75-79`). No `LIMIT`: every row sharing that fingerprint is marked |
| Path B — legacy encrypted uid | `aes-256-gcm`, key `hash('sha256', "CoinUp4LifeGeeks", true)`, IV = `openssl_cipher_iv_length` (12), tag 16 bytes, base64 (`:51-67`); `parse_str` → `id` → `UPDATE ninja25 SET disabled = ? WHERE id = ?` (`:82-90`) |
| Writes | **`disabled` = `date('Y-m-d')` and nothing else.** No `fraud_flags`, no `behavior_flags`, no `conv`, no `decision`, no `flagged` |
| Always | redirect `Location: https://forms.gle/uppnaB6poScDHUmH9` — on DB connect failure (`:31`), on any `Throwable` (`:94-97`), and on success (`:102`). `display_errors` is off (`:3`) so an uninstalling user never sees a fatal |
| Failure mode | If the PDO connect fails, the redirect fires and **the `disabled` write is lost** (`:28-33`) |

Two downstream effects make this row-level date load-bearing:

1. **It is what makes `fast_uninstall` true.** `extractBehaviorFeatures` sets `$features['fast_uninstall'] = !empty($install['disabled'])` (`behavior_lib.php:414`), worth `+30` in the v4 score (`classifier_v4.php:21`, `:69`). It is also the sole input to the decision sweep's fast-uninstall → bot shortcut (`finalize_pending_installs.php:467`).
2. **It is what causes an s1 forfeit.** `!empty($held['disabled'])` is the first half of the forfeit test at `:229` — an install that uninstalls before its 1h checkpoint is marked `'|s1_forfeit'` with `conv = '4'` and is never paid. It is also excluded from Sweep-1 payment by the merge-independent path, and it is the `disabled = DATE(created_date)` predicate that `sanitize_fast_uninstalls.php` keys on.

> **Note** The comment at `generate_fraud_report.php:302` refers to "farewell.php's `uninstall_paid_fast_uninstall` tag". No such tag exists — `farewell.php` writes no flags at all. That report derives the condition itself from `disabled` plus the active span.

### 9.9 `sanitize_fast_uninstalls.php` — manual one-shot

Not wired to any scheduler: nothing in the codebase references this file. It runs only when a human requests it, by browser or CLI (`sanitize_fast_uninstalls.php:32-38` merges `$_GET` with `argv` `k=v` pairs).

| Aspect | Detail |
|---|---|
| Modes | `?run=dry` → categorized report, no writes; `?run=live` → apply. Any other value prints usage and `return`s (`:39-45`) |
| Guard | `flock(LOCK_EX \| LOCK_NB)` on `sanitize_fast_uninstalls.lock` (`:48-52`); own PDO with `ATTR_TIMEOUT => 10` (`:56-67`); `$table = $GLOBALS['table'] ?? 'ninja25'` (`:68`) |
| Scope (`:76-81`) | `(conv = '0' OR conv IS NULL)` **AND** `an IS NOT NULL AND an <> ''` **AND** `disabled IS NOT NULL AND disabled <> ''` **AND** `disabled = DATE(created_date)` **AND** `COALESCE(fraud_flags,'') NOT LIKE '%\|conv4_fast_uninstall%'` **AND** `COALESCE(fraud_flags,'') NOT LIKE '%\|s1_hold%'` |
| Write (`:114-118`) | `UPDATE … SET conv = '4', fraud_flags = CONCAT(COALESCE(fraud_flags,''), '\|conv4_fast_uninstall')` over that same scope. `decision` columns are untouched; **no postback ever fires** |
| Idempotence | The `'\|conv4_fast_uninstall'` marker is excluded from the scope, so re-runs touch nothing twice |
| `'\|s1_hold'` rows | Excluded — they are finalize's jurisdiction (pay or `'\|s1_forfeit'`). The dry run counts them separately (`:85-92`) |
| Boundary | `disabled` is a plain `Y-m-d` from `farewell.php`, compared to `DATE(created_date)`. An install at 23:58 uninstalled at 00:05 crosses the DB-day line and is deliberately **not** matched — it is left to the >24h hygiene sweep (`:73-75`) |
| Reported before any write | total in scope, `MIN(created_date)`, overlap with `created_date >= '2026-07-14 20:00:00'` (the rescue window), and the s1_hold skip count (`:84-104`) |

*Its real added value over the hygiene sweep of 9.5 is the historical backfill before the `2026-07-13 09:00:00` forward-only cutoff, plus same-day rows still inside their first 24h.* The header carries an order caveat: run it **after** settling any rescue of uninstalled outage victims, because rows marked `'|conv4_fast_uninstall'` are not treated as rescuable.

### 9.10 Error handling and the stats lines

Each sweep has its own `try`/`catch(\Throwable)`, so one failure never aborts the others; every catch writes `error_log` plus a row into the `errors` table via `errorLog($pdo, $ip, $type, $value)` (defined guarded at `:130-139`, `substr(..., 0, 250)`).

| Scope | Line | `error_log` prefix | `errors.type` | IP |
|---|---|---|---|---|
| Sweep 1, per row | `:319-325` | `Error paying held Stage-1 conversion ID {id}:` | `s1-delay-pay-exception` | `0.0.0.0` |
| Sweep 1, whole | `:342-347` | `CRITICAL ERROR in STAGE 1 - 1 HOUR DELAY sweep:` | `s1-delay-sweep-error` | `0.0.0.0` |
| Conv hygiene | `:401-405` | `Error in conv-hygiene sweep:` | `conv4-hygiene-error` | `0.0.0.0` |
| Decision sweep, per row | `:542-549` | `Error finalizing pending install ID {id}:` | `finalize-pending-exception` | the row's `ip_str` |
| Straggler/expiry | `:578-583` | `Error expiring stragglers in finalize_pending_installs.php:` | `finalize-expire-error` | `0.0.0.0` (**dormant**) |
| Decision sweep, whole | `:591-595` | `CRITICAL ERROR in finalize_pending_installs.php sweep:` | `finalize-pending-cron-error` | `0.0.0.0` |
| Inline wrapper | `ninja-adb21.php:3763-3774` | `inline {script}: uncaught` | — | — |

Per-row catches increment `$s1Stats['errors']` / `$stats['errors']` and continue with the next row. `finally` (`:596-600`) nulls `$pdo` when `$isStandalone` — which includes inline runs.

Two stats lines exist, both emitted only under `$isStandalone` (true inline as well, where `ninja_run_inline` captures and `error_log`s them):

- **Budget-slice early return** (`:356-358`), when the money sweep spent the slice:
  `S1-delay sweep (budget slice, backlog REMAINS|clear). Due: N (Paid: N, Refused: N, Forfeited: N, Held: N, Errors: N)`
- **Full pass** (`:588-589`):
  `S1-delay sweep. Due: N (Paid: N, Refused: N, Forfeited: N, Held: N, Errors: N)`
  `Sweep completed. Total processed: N (Human: N, Bot: N, Deferred: N, Skipped: N, Expired: N, Errors: N)`

plus, when non-zero, `Conv-hygiene sweep: N attributed rows closed conv=4 (expired unpaid).` (`:399`).

> **Note** Two counters are structurally always zero in the current configuration: `Skipped` is initialized at `:428` and never incremented anywhere in the file, and `Expired` is only ever set inside the dormant `if (!$pureShadow)` block (`:577`). `Refused` is likewise always zero while `$BEHAVIOR_AN_ALLOWLIST` is empty. The useful live signals are `Due`, `Paid`, `Forfeited`, `Held`, and the two stamp files: a fresh `last_finalize_run.txt` means runs are starting; an **absent** `finalize_backlog.txt` (and a fresh `last_finalize_success.txt`) means they are keeping up.

---

## 10. The Stage-2 merge — `mergeConvV4()` (designed, not active)

### 10.1 Status banner

> **NOT ACTIVE TODAY.** `mergeConvV4()` is a pure function that is *loaded* on every request but never reaches a paying or refusing branch, because its single call site is gated on an empty allowlist. **No install can be REFUSED by this system today.**

| Fact | Value | Reference |
|---|---|---|
| Function definition | `mergeConvV4(int $tier, string $decision, array $f): array` | `classifier_v4.php:158` |
| Loaded into the live endpoint? | Yes — `behavior_lib.php:11` does `require_once __DIR__ . '/classifier_v4.php';`, and `ninja-adb21.php:78` requires `behavior_lib.php` | `behavior_lib.php:11` |
| Called from `ninja-adb21.php`? | **No.** Zero occurrences of `mergeConvV4` in the endpoint | `grep -n mergeConvV4 ninja-adb21.php` → no hits |
| Only production call site | Inside the `'|s1_hold'` money sweep of the finalizer | `finalize_pending_installs.php:274` |
| Gate on that call site | `an_in_behavior_scope($held['an']) && function_exists('mergeConvV4')` | `finalize_pending_installs.php:255-256` |
| `an_in_behavior_scope()` | `return !empty($an) && in_array($an, $list, true);` — exact match, **no `'*'` support** | `behavior_lib.php:223-227` |
| `$BEHAVIOR_AN_ALLOWLIST` | `[]` — emptied 2026-07-16, still `[]` on purpose | `config.php:46` |

Because the list is empty, `an_in_behavior_scope()` returns `false` for every `an` (and for `an=NULL`), the whole block at `finalize_pending_installs.php:255-289` is skipped, and held rows fall straight through to the Stage-1 rule: level check (`finalize_pending_installs.php:294`) → atomic `'|s1_hold'` → `'|s1_paid'` claim (`finalize_pending_installs.php:300-304`) → `processConversion()` (`finalize_pending_installs.php:310-312`).

> **Note** — The other call site, `replay_v4.php:140`, is a read-only backtest that never touches the DB (`replay_v4.php:6`). The `classifier_v4.php` self-test at `classifier_v4.php:207-223` exercises the twelve merge cases in CLI only.

> **Note — this is NOT the Stage-1 delay.** The live `$STAGE1_DELAY_CONV_AN = ['*']` / `$STAGE1_DELAY_CONV_SEC = 3600` hold (`config.php:68-69`) is **timing only**: same gate, same recipients, postback fired roughly an hour late. `$BEHAVIOR_AN_ALLOWLIST` is the separate, currently-empty **refusal** switch. The two lists are deliberately independent — `s1_delay_in_scope()` (`behavior_lib.php:237-241`) versus `an_in_behavior_scope()` (`behavior_lib.php:223-227`). Readers confuse them constantly; a network being held does **not** mean it can be refused.

> **Defect** — `$HAS_FRAUD_COLUMNS` (`config.php:36`) is the master switch for the whole mechanism. If it were ever `false`, `insertUserRecord()` would drop `fraud_flags` from the INSERT column list (`ninja-adb21.php:1806-1816`), the `'|s1_hold'` marker written at `ninja-adb21.php:3414` would never be persisted, the sweep's `WHERE fraud_flags LIKE '%|s1_hold%'` (`finalize_pending_installs.php:200`) would match nothing, and **every payout would be silently lost** while the global hold is on. The merge branch lives *inside* that same sweep, so a `false` flag also makes Stage 2 unreachable even with a populated allowlist.

### 10.2 The universal-hold principle (revised 2026-07-12)

Under the designed Stage 2, **nobody pays at install** — every attributed install settles once, at the checkpoint (`classifier_v4.php:135-139`).

> *Forward data drove the revision: 419 attributed Tier-1 bots (~4.8% of payouts) were paid at minute 0 and were provably gone by 3h, and 100% of behavioral bots live in Tier 1 — clean infrastructure is exactly where the fraud sits, so "Tier 1 pays at install" was Stage-1 thinking.* (`classifier_v4.php:135-139`)

Consequence: **Tier 1 and Tier 2 are identical at the money layer.** The code makes this literal — the T1/T2 branch at `classifier_v4.php:177-183` never reads `$tier` except to build the reason string. The tier's entire remaining money role reduces to *"hard evidence ⇒ proof required"*, which is the Tier-3 branch at `classifier_v4.php:169-175`.

Money settles **once** at the checkpoint and never moves after it. Post-checkpoint syncs and visits are analysis data only (dashboard Panel F8).

### 10.3 The money principle

> **Refuse only on POSITIVE evidence. Absence of evidence never refuses.** (`classifier_v4.php:141-148`)

There are exactly two positive-evidence refusals in the whole function:

1. a behavioral `bot` verdict, in any tier (`classifier_v4.php:164-166`);
2. Tier-3 hard signals *without* human proof (`classifier_v4.php:169-175`).

Everything else pays. This is the money-layer mirror of the classifier's own doctrine that absence of evidence yields `unknown`, not `bot` — so all Tier-1/Tier-2 unknowns are paid, ghosts included.

Input normalisation happens first: `'deferred'`, `'pending'` and `''` are all folded to `'unknown'` (`classifier_v4.php:159`), and any `$tier` outside 1–3 is clamped to `1` (`classifier_v4.php:160`) — i.e. the tier clamp **fails toward paying**.

### 10.4 Mapping table — tier × decision → pay / refuse

Proof, for Tier 3 only, is `!empty($f['auth_session']) || !empty($f['popup_interaction'])` (`classifier_v4.php:170`).

| Tier | `decision` | Proof | `pay` | `outcome` | `reason` |
|---|---|---|---|---|---|
| 1 | `bot` | — | `false` | `refuse` | `t1_bot_refused` |
| 2 | `bot` | — | `false` | `refuse` | `t2_bot_refused` |
| 3 | `bot` | — | `false` | `refuse` | `t3_bot_refused` |
| 1 | `human` | — | `true` | `pay` | `t1_human_paid` |
| 2 | `human` | — | `true` | `pay` | `t2_human_paid` |
| 1 | `unknown` | — | `true` | `pay` | `t1_unknown_paid` (synced) / `t1_ghost_paid` (never synced) |
| 2 | `unknown` | — | `true` | `pay` | `t2_unknown_paid` (synced) / `t2_ghost_paid` (never synced) |
| 3 | `human` | yes | `true` | `pay` | `t3_human_with_proof_paid` |
| 3 | `human` | no | `false` | `refuse` | `t3_human_no_proof_refused` |
| 3 | `unknown` | — | `false` | `refuse` | `t3_unknown_refused` |

Note the branch order: the `bot` test at `classifier_v4.php:164` runs **before** the Tier-3 test at `classifier_v4.php:169`, so a Tier-3 bot is reported as `t3_bot_refused`, not via the Tier-3 fallthrough at `classifier_v4.php:174`, which would build the identical string.

The ghost subcategory is computed only on the T1/T2 unknown path:

```php
$isGhost = !empty($f['no_evidence']) || (int)($f['n_syncs'] ?? 0) === 0;
```
(`classifier_v4.php:182`)

> *The Tier-3 proof requirement is v3's Tier-3 human hard gate, relocated out of the classifier and into the money layer: `decision` stops lying about behaviour, and money stays protected.* (`classifier_v4.php:168`)

### 10.5 Reason strings → conv value → `fraud_flags` marker

`mergeConvV4()` itself moves no money; it returns `['pay' => bool, 'outcome' => 'pay'|'refuse', 'reason' => string]` (`classifier_v4.php:154`). The caller at `finalize_pending_installs.php:274-289` is what would translate that into `conv` and markers.

| `reason` | `pay` | `conv` written | `fraud_flags` transition | Reason string persisted? |
|---|---|---|---|---|
| `t1_bot_refused` | `false` | `'4'` | `'|s1_hold'` → `'|s1_refused'` + `'|t1_bot_refused'` | yes |
| `t2_bot_refused` | `false` | `'4'` | `'|s1_hold'` → `'|s1_refused'` + `'|t2_bot_refused'` | yes |
| `t3_bot_refused` | `false` | `'4'` | `'|s1_hold'` → `'|s1_refused'` + `'|t3_bot_refused'` | yes |
| `t3_human_no_proof_refused` | `false` | `'4'` | `'|s1_hold'` → `'|s1_refused'` + `'|t3_human_no_proof_refused'` | yes |
| `t3_unknown_refused` | `false` | `'4'` | `'|s1_hold'` → `'|s1_refused'` + `'|t3_unknown_refused'` | yes |
| `t3_human_with_proof_paid` | `true` | `1`/`3` from `processConversion` | `'|s1_hold'` → `'|s1_paid'` | **no** |
| `t1_human_paid` / `t2_human_paid` | `true` | `1`/`3` from `processConversion` | `'|s1_hold'` → `'|s1_paid'` | **no** |
| `t1_unknown_paid` / `t2_unknown_paid` | `true` | `1`/`3` from `processConversion` | `'|s1_hold'` → `'|s1_paid'` | **no** |
| `t1_ghost_paid` / `t2_ghost_paid` | `true` | `1`/`3` from `processConversion` | `'|s1_hold'` → `'|s1_paid'` | **no** |

Mechanics of the refuse path (`finalize_pending_installs.php:278-285`):

```sql
UPDATE `{$table}`
   SET `fraud_flags` = CONCAT(REPLACE(`fraud_flags`, '|s1_hold', '|s1_refused'), ?),
       `conv` = '4'
 WHERE `id` = ? AND (`conv` = '0' OR `conv` IS NULL)
   AND `fraud_flags` LIKE '%|s1_hold%'
```

The bound parameter is `'|' . ($mvM['reason'] ?? 'merge_refused')` (`finalize_pending_installs.php:284`). The `rowCount() === 1` test is the idempotency lock, mirroring the pay claim.

> **Note** — On a `pay` verdict the merge simply falls through to the ordinary Stage-1 path (`finalize_pending_installs.php:288`), so the *pay* reason strings are **never written to the database**. `t1_ghost_paid` versus `t1_unknown_paid` is therefore only observable via `replay_v4.php` or by re-deriving `no_evidence` from `behavior_flags`, which is what the dashboard does (`dashboard_detection.php:161`: `$GHOSTF="(COALESCE(\`behavior_flags\`,'') LIKE '%no_evidence%')"`). If the ghost/unsure split ever needs to be auditable per-row on the money path, the pay branch must be taught to stamp its reason too.

Feature reconstruction on the live call site is substring matching against `behavior_flags`, not a fresh feature extraction (`finalize_pending_installs.php:265-272`):

| Merge feature | Derived from |
|---|---|
| `auth_session` | `strpos($bfM, 'auth_login') !== false` |
| `popup_interaction` | `strpos($bfM, 'popup_interaction') !== false` |
| `no_evidence` | `strpos($bfM, 'no_evidence') !== false` |
| `n_syncs` | `1`, or `0` when `no_evidence` is present — a backstop only; `no_evidence` is authoritative |

The tier comes from `deriveTierFromRow($held)` (`finalize_pending_installs.php:273`), which prefers the stored `fraud_flags` prefix `tier3`/`tier2`/`tier1` and otherwise falls back to `is_vm === 1` or `level !== '0'` ⇒ Tier 3, else Tier 1 (`classifier_lib.php:160-170`). This is the **only** function of `classifier_lib.php` still reachable; the rest of that file is **DORMANT**.

### 10.6 Ghost policy — RESOLVED 2026-07-12: ghosts are PAID

A ghost is an install that reached the checkpoint having never synced (`no_evidence` flag, emitted unconditionally at `classifier_v4.php:101` whenever `n_syncs === 0`).

**Policy: pay them.** Refusal requires positive evidence and a ghost has none in either direction; the split `ghost` (never synced) vs `unsure` (synced, not enough evidence) is an **analytics subcategory only**, carried in `reason` and rendered in dashboard Panels E and F8. It never changes the money outcome (`classifier_v4.php:141-148`, `classifier_v4.php:181-183`, `dashboard_detection.php:155-171`).

> *Ghost-unknowns survive at 1.3% (n=1,187). The cost is accepted and monitored via Panel E's "PAY · ghost" KPI and Panel F8 wake-rate/wake-latency.* (from the superseded V4 spec)

The flip is one line. To refuse ghosts, change `classifier_v4.php:183` from:

```php
return ['pay' => true, 'outcome' => 'pay', 'reason' => 't' . $tier . ($isGhost ? '_ghost_paid' : '_unknown_paid')];
```

to a branch that returns `['pay' => false, 'outcome' => 'refuse', 'reason' => 't'.$tier.'_ghost_refused']` when `$isGhost`, leaving the non-ghost return untouched. Nothing downstream needs to change — the caller already handles `pay === false` generically (`finalize_pending_installs.php:275-287`). The self-test expectations at `classifier_v4.php:214` and `classifier_v4.php:216` would need updating in the same commit.

### 10.7 The timing mismatch that must be fixed before enabling

> **This is the single hard blocker on the promotion checklist. Do not add a network to `$BEHAVIOR_AN_ALLOWLIST` before fixing it.**

The merge is designed as a **3h checkpoint** (`classifier_v4.php:132`, `BEHAVIOR_WINDOW_SEC = 10800` at `behavior_lib.php:15`). The live hold is **1h** and uniform:

| Setting | Live value | Reference |
|---|---|---|
| `$STAGE1_DELAY_CONV_AN` | `['*']` | `config.php:68` |
| `$STAGE1_DELAY_CONV_SEC` | `3600` | `config.php:69` |
| `$STAGE1_DELAY_CONV_SEC_AN` | `[]` — **no per-network override** (was `['gn' => 10800]` during the 07-12 → 07-16 rollout) | `config.php:70` |
| `s1_delay_seconds($an)` | returns the per-network map entry if present, else `$STAGE1_DELAY_CONV_SEC` | `behavior_lib.php:248-253` |

What goes wrong, step by step, if a network is allowlisted while its delay stays at 3600:

1. At `ageSec >= 3600` the row passes the per-row due test at `finalize_pending_installs.php:242-246` and enters the merge branch.
2. The merge branch first demands a **terminal** verdict: `if ($decM === '' || $decM === 'pending') { $s1Stats['held']++; continue; }` (`finalize_pending_installs.php:257-264`). Money is never paid before the verdict — correct, but at 1h most verdicts are not terminal yet.
3. `runBehaviorLayer()` only terminalizes `human` and `bot` early (`behavior_lib.php:501-502`); an undecided install stays `'pending'` until it is past `BEHAVIOR_WINDOW_SEC` (`behavior_lib.php:505-506`).
4. The finalizer's decision sweep — the backstop that force-resolves `pending`/NULL to `deferred` — selects only rows `created_date < NOW() - INTERVAL 3 HOUR AND created_date > NOW() - INTERVAL 24 HOUR` (`finalize_pending_installs.php:412-416`). Nothing at 1h is in that window.
5. So every uncorroborated install is **re-held** at 1h, 1h15, 1h30… until it crosses 3h, is terminalized to `deferred` by the decision sweep, and only then merges and pays on a subsequent pass — **at roughly 3h15**, not 1h.

Net effect: the 1h hold silently becomes a ~3h15 hold for exactly the installs the merge exists to judge, while `human`/`bot` rows (which terminalize at any age) settle at 1h. That is a two-population payout latency, a per-network attribution-window risk, and a change nobody signed off on.

**Required config change before enabling Stage 2 for a network `X`:** restore its per-network override so its checkpoint matches the design.

```php
$STAGE1_DELAY_CONV_SEC_AN = ['X' => 10800];   // config.php:70 — 3h, aligned with BEHAVIOR_WINDOW_SEC
```

Two consequences to expect from that edit, both already handled by the code:

- The sweep's SELECT lower bound uses the **shortest** configured delay (`$s1MinDelay`, `finalize_pending_installs.php:176-179`), so it stays at 3600 for the still-Stage-1 networks; each row is then re-checked against its own network's deadline at `finalize_pending_installs.php:242`. No other network's timing moves.
- Confirm network `X`'s postback attribution window exceeds ~3.5h before flipping. The pay window `$S1_HOLD_PAY_WINDOW_SEC = 259200` (72h, `config.php:89`) keeps rows *payable* far longer, but a network may simply refuse to attribute a postback fired more than 24h after the click.

### 10.8 Promotion checklist — putting ONE network into Stage 2

Ordered. Steps 1–3 are the prerequisites; step 4 is the flip.

| # | Step | File / value |
|---|---|---|
| 1 | Verify `$HAS_FRAUD_COLUMNS === true` — without it no `'|s1_hold'` marker exists and the sweep (and therefore the merge) never sees the row | `config.php:36` |
| 2 | Confirm the network's postback attribution window is > ~3.5h; confirm it carries no real payout param that would need persisting (`$po` is hardcoded `0` at install, `ninja-adb21.php:2694`, and only ev/mg/m1/m2 postbacks carry a payout param at all) | operational |
| 3 | **Fix the timing first** (§10.7): set `$STAGE1_DELAY_CONV_SEC_AN = ['X' => 10800];` and deploy. Watch one full cycle: rows for `X` must now pay at ~3h, still under Stage-1 rules, before any refusal logic exists | `config.php:70` |
| 4 | Add the network to the refusal switch: `$BEHAVIOR_AN_ALLOWLIST = ['X'];` — exact string, no `'*'`, `an_in_behavior_scope()` is `in_array(..., true)` | `config.php:46`, `behavior_lib.php:226` |
| 5 | Do **not** touch `$BEHAVIOR_ENFORCE`, `$DEFER_CONVERT` or `$DELAYED_CONVERSION`. The merge branch is deliberately **not** gated on them (`finalize_pending_installs.php:253-254`); the allowlist is the single Stage-2 switch. Those three flags remain `false` (`config.php:39-41`) | `config.php:39-41` |
| 6 | Monitor: count of `'|s1_refused'` markers and `conv='4'` rows for `X`; the merge reason suffix on each refused row; `$s1Stats['refused']` in the finalizer's stats line (`finalize_pending_installs.php:285`); and the "would-refuse-that-survived" figure on dashboard Panel B, filtered to `created_date >= 2026-07-10` (the Phase-2 tier-fix cohort cut) | dashboard |

**Rollback — one line, immediate:**

```php
$BEHAVIOR_AN_ALLOWLIST = [];   // config.php:46
```

Removing the network stops all future refusals on the next inbound request. Rows already holding `'|s1_hold'` keep draining normally, because the money sweep selects by the **marker**, not by any allowlist (`finalize_pending_installs.php:200`). Rows already stamped `'|s1_refused'` with `conv='4'` are terminal and are **not** reopened — that money is gone; rollback is forward-only. If the 3h override from step 3 should also be reverted, separately restore `$STAGE1_DELAY_CONV_SEC_AN = [];` (`config.php:70`).

> **Note** — On this host the finalizer has **no cron**. It runs in-process, post-response-flush, via `ninja_run_inline('finalize_pending_installs.php', ...)` (`ninja-adb21.php:3810`), with inbound traffic as the scheduler (`ninja-adb21.php:3794-3806`). Any code comment or prior doc describing "the 15-min cron" as the live scheduler — including `finalize_pending_installs.php:20-23` and `finalize_pending_installs.php:260-261` — is stale. Plan promotion monitoring around traffic volume, not around a clock.

### 10.9 V4-3 design direction (planned)

**NOT IMPLEMENTED.** Agreed 2026-07-10, to land with the V4-3 wiring.

The merge as written takes a single fused `$tier` plus a behavioral `$decision`. The planned replacement takes **three independent verdicts**, fused **once** at the money merge by a severity rule, with `tier` demoted to a *derived output* rather than an input:

| Layer | Verdict domain |
|---|---|
| Hardware | `physical` / `suspicious` / `vm` |
| Network | `clean` / `suspicious` / `hard` |
| Behavior | `human` / `unknown` / `bot` |

Design constraints carried forward: verdicts never consume verdicts (the defect that made the v3 nightly ASN/device blacklists self-confirming); each layer calibrates on outcomes only; identity signals such as device collisions are parked as **weak network evidence**; and `extid` remains a standalone integrity gate outside the fusion.

Also still open on the roadmap: **V4-4**, a sync-accumulation anti-spoof human signal (organic evidence that demonstrably grew across the observation window — the one thing the current corroboration gate does not stop), and **V4-5**, backfilling the >24h `pending` backlog on the decision axis.

---

## 11. Runtime envelope and scheduling

Everything in this section describes the block that begins at `ninja-adb21.php:3647` (the "RESPONSE-FIRST, TRAFFIC-DRIVEN MAINTENANCE" header) and runs to EOF at line 3897, plus the boot-time loads at `ninja-adb21.php:100-233`.

> The live host (Infomaniak) offers **no cron at all**. Inbound traffic is the only scheduler that exists today. Any statement that "the 15-minute cron" drives detection or payouts is wrong for the live deployment — the cron branch is code that is kept working for hosts that have one, and is **NOT ACTIVE TODAY**.

### 11.1 The response-first contract

The JSON response is shipped before any maintenance work starts. Order at `ninja-adb21.php:3679-3696`:

| Step | Line | Effect |
|---|---|---|
| `ignore_user_abort(true)` | `ninja-adb21.php:3679` | a client that disconnects at or before the flush cannot kill the maintenance below |
| `echo json_encode($jsonResponse)` | `ninja-adb21.php:3680` | the only response write on the success path |
| `$ninjaDetached = false` | `ninja-adb21.php:3681` | pessimistic default |
| `fastcgi_finish_request()` | `ninja-adb21.php:3682-3685` | PHP-FPM (this is the Infomaniak path). `$ninjaDetached = (@fastcgi_finish_request() !== false)` — a `false` return leaves the worker *attached* |
| `litespeed_finish_request()` | `ninja-adb21.php:3686-3689` | LiteSpeed/lsapi equivalent; sets `$ninjaDetached = true` unconditionally |
| `while (ob_get_level() > 0) { @ob_end_flush(); } @flush();` | `ninja-adb21.php:3690-3696` | mod_php/CGI fallback — no detach primitive exists, so the client may still hold the connection |

`$ninjaDetached` is the single switch that decides the maintenance budget and which jobs may run inline at all:

| `$ninjaDetached` | finalize slice budget | funnel inline fallback | rules/cosmetic inline |
|---|---|---|---|
| `true` | 12.0 s (`ninja-adb21.php:3810`) | allowed (`ninja-adb21.php:3831`) | allowed (`ninja-adb21.php:3855`) |
| `false` | 2.0 s (`ninja-adb21.php:3810`) | **skipped** — `bg_run` only | `bg_run` only, error_log on failure |

> *Why it matters: all detection maintenance — the money sweep, the decision sweep, the fraud audit, the funnel rebuild — executes strictly AFTER the response bytes are gone. A user request never waits on detection, and detection never gets to change the response it just sent.*

### 11.2 `ninja_run_inline()` — in-process maintenance

Defined at `ninja-adb21.php:3739-3775` under a `function_exists` guard. Three deliberate properties, each load-bearing:

**1. Function-scope `require` (variable isolation).** The included file is pulled in at `ninja-adb21.php:3765` inside the function body, so every top-level variable of the included script (`$table`, `$stmt`, `$sql`, `$row`, `$stats`, …) lives and dies in that call frame and cannot collide with request state. *Breaks if:* the `require` is hoisted to file scope — `finalize_pending_installs.php` would then stomp `$table`, `$pdo`, `$stmt` and the request's own locals.

**2. Config re-required, NOT `_once`.** `ninja-adb21.php:3755-3759` declares the shared config names `global` and then re-runs `require __DIR__ . '/config.php'`:

```
global $ninja_db_config, $aes_key, $proxycheck_key, $token_7thsense, $HAS_FRAUD_COLUMNS,
       $BEHAVIOR_ENFORCE, $DEFER_CONVERT, $DELAYED_CONVERSION, $BEHAVIOR_AN_ALLOWLIST,
       $STAGE1_DELAY_CONV_AN, $STAGE1_DELAY_CONV_SEC, $STAGE1_DELAY_CONV_SEC_AN,
       $S1_HOLD_PAY_WINDOW_SEC;
require __DIR__ . '/config.php';
```

This is not redundancy. The request path **mutates the enforcement flags per network** before reaching this point: `createUser()` sets `$DELAYED_CONVERSION = false` for any network failing `an_in_behavior_scope()` (`ninja-adb21.php:2683-2687`), and the fingerprint-match branch sets both `$DELAYED_CONVERSION = false` and `$BEHAVIOR_ENFORCE = false` (`ninja-adb21.php:3456-3459`). Because those are `global`, the true globals carry per-request pollution. Re-requiring config resets them to config truth for every helper that reads them via `global` — `an_in_behavior_scope()` (`behavior_lib.php:223`), `s1_delay_in_scope()` (`behavior_lib.php:237`), `s1_delay_seconds()` (`behavior_lib.php:248`), `runBehaviorLayer()` (`behavior_lib.php:442`). *Breaks if:* changed to `require_once` — the include is a no-op, the polluted per-network `false` values leak into the sweep, and `finalize_pending_installs.php:435-437` snapshots them as `$cfgEnforce/$cfgDelayed/$cfgDefer` for the whole run. Today all three are `false` in config anyway, so the observable damage is nil; the moment any flag is turned on it becomes a correctness bug that fires only for requests that happened to touch a non-allowlisted network.

**3. `$pdo` is deliberately omitted from the `global` list.** The parent has already set `$pdo = null` at `ninja-adb21.php:3639`. Inside `ninja_run_inline()` `$pdo` is simply unset, so `finalize_pending_installs.php:65` (`if (!isset($pdo))`) takes its **standalone branch**: `$isStandalone = true`, its own `flock(LOCK_EX|LOCK_NB)` single-flight on `finalize_pending_installs.lock` (`finalize_pending_installs.php:98`), its own guarded PDO connect with `ATTR_TIMEOUT => 10` (`finalize_pending_installs.php:110-119`), and its own `@touch(last_finalize_run.txt)` on connect success (`finalize_pending_installs.php:126`). *Breaks if:* `$pdo` is added to the `global` list — the include takes the embedded branch: **no flock (concurrent full sweeps), no run-stamp touch (the trigger re-fires on every single request forever), and no `$pdo = null` in the `finally`** (`finalize_pending_installs.php:596-599`).

Budget and output handling: `$budgetSec` becomes `$GLOBALS['NINJA_FPI_DEADLINE'] = microtime(true) + max(1.0, (float)$budgetSec)` (`ninja-adb21.php:3760-3762`), unset again at `:3770`. Output is captured with `ob_start()`/`ob_get_clean()` and pushed to `error_log` truncated to 400 chars with newlines flattened to `" | "` (`ninja-adb21.php:3763-3773`) — post-flush, stdout goes nowhere, so this is the only trace an inline run leaves. A `\Throwable` from the include is caught and logged as `inline {$script}: uncaught …` (`ninja-adb21.php:3766-3768`).

### 11.3 `bg_run()` — the verified self-call

Defined at `ninja-adb21.php:3697-3737`, also `function_exists`-guarded. Constants: `BG_HOST = 'ninja-block.com'`, `BG_PREFIX = '/'` (`ninja-adb21.php:3701-3702`) — the only two product-specific values in the function.

| Stage | Line | Value / behaviour |
|---|---|---|
| Windows branch | `ninja-adb21.php:3714-3717` | `popen("start /B php …")`, returns `true` unconditionally |
| Connect | `ninja-adb21.php:3718` | `fsockopen('ssl://' . BG_HOST, 443, …, 1)` — **1 s connect timeout**; failure → `error_log` + `false` |
| Read timeout | `ninja-adb21.php:3723` | `stream_set_timeout($fp, 2)` — **2 s status-line timeout** |
| Request | `ninja-adb21.php:3724-3725` | `GET /{$script} HTTP/1.1`, `Host:`, `User-Agent: ninja-bg-run`, `Connection: Close` |
| Silence (no bytes in 2 s) | `ninja-adb21.php:3728-3730` | returns **`true`** — PHP buffers to the end, so an accepted-and-grinding script sends nothing early |
| Non-2xx status | `ninja-adb21.php:3731-3734` | `!preg_match('#^HTTP/\S+\s+2\d\d#', $status)` → `error_log` + `false` (CF block page, 404 after a move, 30x) |
| Fast 2xx | `ninja-adb21.php:3735` | `true` (e.g. the target's recent-run guard skipped and echoed) |

Job allocation today:

| Job | Uses `bg_run`? |
|---|---|
| `finalize_pending_installs.php` | **No — moved fully inline.** `last_finalize_spawn.txt` is vestigial and may still exist on the server harmlessly (`ninja-adb21.php:3804-3805`) |
| `generate_fraud_report.php` | **Yes, and only `bg_run`** — no inline fallback, because it defines unguarded function names (`normalizeUaForDeviceHash` etc.) that would fatal on a second definition inside a request (`ninja-adb21.php:3781-3783`) |
| `generate_funnel_cache.php` | `bg_run` first, inline fallback when `$ninjaDetached` (`ninja-adb21.php:3831-3833`) |
| `generate_compiled_rules.php` | inline first when detached, `bg_run` otherwise (`ninja-adb21.php:3855-3866`) |
| `generate_cosmetic_rules.php` | inline first when detached, `bg_run` otherwise (`ninja-adb21.php:3886-3897`) |

The early socket close is safe because every target calls `ignore_user_abort(true)` (`finalize_pending_installs.php:34`, `generate_fraud_report.php:10`, `generate_funnel_cache.php:22`).

### 11.4 The post-flush triggers

The source numbers the post-flush blocks 1–6; block 1 is the response flush itself (§11.1). Five maintenance triggers follow, in this fixed order — money first.

| # | Job | Line | Fires when | Success marker (written by the script) | Attempt stamp (written by the trigger) | Failure mode | Class |
|---|---|---|---|---|---|---|---|
| 2 | `generate_fraud_report.php` | `ninja-adb21.php:3782-3792` | `last_audit_run.txt` missing or older than **21600 s (6 h)** AND `last_audit_spawn.txt` missing or older than **3600 s (1 h)** | `last_audit_run.txt`, `@touch`ed at `generate_fraud_report.php:349` after the report is written | `last_audit_spawn.txt`, `@touch`ed at `ninja-adb21.php:3788` before the attempt | `bg_run` false → `error_log("audit trigger: self-call failed — fraud report NOT running; last_audit_run.txt stays stale until the path is fixed")`. No inline fallback: blacklists and `calibrated_risk_threshold` freeze at their last successful build | **DETECTION-CRITICAL** |
| 3 | `finalize_pending_installs.php` | `ninja-adb21.php:3806-3811` | `last_finalize_run.txt` age **> 1200 s**, OR `finalize_backlog.txt` exists AND age **> 60 s** | `last_finalize_success.txt`, `@touch`ed only on a full drain (`finalize_pending_installs.php:340`) | `last_finalize_run.txt` — touched on run *start* after the guarded connect (`finalize_pending_installs.php:126`); the trigger touches nothing | Runs inline with a 12 s / 2 s budget. Budget hit → `finalize_backlog.txt` touched (`finalize_pending_installs.php:334`) and the trigger re-fires at 60 s. If traffic stops, nothing runs and **held payouts age toward the 72 h forfeit** | **DETECTION-CRITICAL (money path)** |
| 4 | `generate_funnel_cache.php` | `ninja-adb21.php:3813-3834` | (`time() >= 01:00 GMT+4 boundary + 900 s` AND `last_funnel_run.txt` older than that boundary) **OR** `funnel_cache.json` missing — AND `last_funnel_spawn.txt` missing or older than **1800 s** | `last_funnel_run.txt`, `@touch`ed at `generate_funnel_cache.php:464` **after** the atomic `rename()` of the cache | `last_funnel_spawn.txt`, `@touch`ed at `ninja-adb21.php:3830` | `bg_run` false + detached → inline (no slice budget; holds one detached worker for the whole build). `bg_run` false + not detached → **silently nothing**, retried at the 1800 s cadence | **DETECTION-CRITICAL** |
| 5 | `generate_compiled_rules.php` | `ninja-adb21.php:3836-3867` | `compiled_rules_cache.json` missing, OR `compiled_rules_version.txt` missing, OR its age **> 86400 s (24 h)** — AND `last_rules_regen_attempt.txt` age **> 1800 s** | `compiled_rules_version.txt` | `last_rules_regen_attempt.txt` (`ninja-adb21.php:3854`) | Detached: `flock(LOCK_EX\|LOCK_NB)` on `compiled_rules_cache.json.lock`, then inline; lock held → silently skipped this pass. Not detached and `bg_run` false → `error_log`; the DNR blocklist freezes on its last successful build | asset-only |
| 6 | `generate_cosmetic_rules.php` | `ninja-adb21.php:3869-3897` | `cosmetic_css_cache.css` missing, OR `cosmetic_specific_cache.json` missing, OR `max()` of the two version-file ages **> 86400 s** — AND `last_cosmetic_regen_attempt.txt` age **> 1800 s** | `cosmetic_css_version.txt` **and** `cosmetic_specific_version.txt` (both, via `max()` — a partial run keeps retrying until both halves are fresh) | `last_cosmetic_regen_attempt.txt` (`ninja-adb21.php:3885`) | Same shape as #5, on `cosmetic_specific_cache.json.lock` | asset-only |

> **Note** — Jobs 5 and 6 are asset-only: a stale compiled-rules or cosmetic cache degrades ad blocking, not fraud detection. Jobs 2, 3 and 4 are the detection-critical set, and only job 3 moves money.

### 11.5 SUCCESS markers vs ATTEMPT stamps

Two files per job, never one, and they answer different questions:

- **SUCCESS marker** — written by the *script itself*, at the *end*, only after the work actually landed. It answers "did it work?". Examples: `last_audit_run.txt` (`generate_fraud_report.php:349`), `last_funnel_run.txt` written after the `rename()` (`generate_funnel_cache.php:464`), `compiled_rules_version.txt`, `cosmetic_css_version.txt` + `cosmetic_specific_version.txt`, `last_finalize_success.txt` (`finalize_pending_installs.php:340`).
- **ATTEMPT stamp** — pre-touched by the *trigger*, before the attempt. It answers only "how recently did we try?" and exists purely to rate-limit retries: `last_audit_spawn.txt` (1/h), `last_funnel_spawn.txt` (1/1800 s), `last_rules_regen_attempt.txt` (1/1800 s), `last_cosmetic_regen_attempt.txt` (1/1800 s).

The incidents that produced the doctrine:

- **2026-07-15** — Hostinger maintenance stopped the cron silently while the request path still regenerated the funnel cache inline; DB saturation (`max_user_connections`, error 1203) turned requests into fatal 5xx, and the fleet retry-stormed. Fixes: the degraded 200 (§11.7), a per-request PDO `ATTR_TIMEOUT => 5`, and removal of funnel regeneration from the request path entirely (`ninja-adb21.php:225-233`).
- **2026-07-16** — the first lazy-trigger design pre-touched the *run* stamp from the endpoint; `last_audit_run.txt` therefore looked fresh for five days (2026-07-16 → 07-21) while `generate_fraud_report.php` never ran once, because the fire-and-forget `bg_run` closed the socket blind and could not tell a CF block page from success.
- **2026-07-23** — the old inline guard pre-touched `compiled_rules_cache.json` *before* generating; when the GitHub fetches began timing out daily, the fresh mtime satisfied the 24 h TTL and the cache froze on its 07-16 build for a week, shipping an upstream rule that blocked x.com's app bundle (`ninja-adb21.php:3836-3846`).

> **Note** — The rule: **never read a spawn/attempt stamp as health.** A fresh `last_audit_spawn.txt` means "we tried an hour ago", nothing more. Health is the SUCCESS marker's age, and for finalize it is `last_finalize_success.txt` plus the *absence* of `finalize_backlog.txt`.

### 11.6 Boot-time cache loads

Both loads happen before the DB connect, are pure file reads, and never regenerate anything on the request path.

| Cache | Line | Keys read | Default when missing / unparseable | Detection consequence |
|---|---|---|---|---|
| `fraud_cache.json` | `ninja-adb21.php:214-223` | `devices` → `$GLOBALS['blacklisted_devices']`; `asns` → `$GLOBALS['blacklisted_asns']`; `calibrated_risk_threshold` → `$GLOBALS['calibrated_risk_threshold']` | `[]`, `[]`, and **33** (the seeds at `ninja-adb21.php:103-105`; the per-key `?? 33` at `:221` repeats it) | Empty blacklists mean `$isDeviceBlacklisted` / `$isAsnBlacklisted` are always false at `ninja-adb21.php:3084-3091`, so **no install can enter Tier 3 via blacklist**. The threshold falls back to 33 at both read sites — the level-7 proxy/risk digit (`ninja-adb21.php:1660-1665`) and the Tier-2 `$isProxyFlagged` test (`ninja-adb21.php:3159-3161`) |
| `funnel_cache.json` | `ninja-adb21.php:225-233` | whole document → `$GLOBALS['funnel']` | `[]` (and `[]` again if `json_decode` does not return an array) | The funnel feeds `extractBehaviorFeatures()` and the per-network `behavior_thresholds` lookup (`behavior_lib.php:479-483`); with `[]` the classifier falls back to its built-in thresholds. Shadow-only today, so no user-visible impact |

`funnel_cache.json` has been **READ-ONLY on the request path since 2026-07-17** (`ninja-adb21.php:225-230`). Requests serve the last cache even when stale; regeneration is trigger #4 only. The 2026-07-15 stampede is impossible by construction now, not merely lock-limited.

### 11.7 DB-DOWN degraded mode and the exception safety net

Two entry points, one response.

1. **Connect failure.** `new PDO(...)` at `ninja-adb21.php:306` with `ATTR_TIMEOUT => 5`; the `catch (PDOException $e)` at `ninja-adb21.php:309-312` calls `db_down_response()`.
2. **Anything else uncaught.** `set_exception_handler` at `ninja-adb21.php:200-205` logs `'[ninja-adb21 uncaught] ' . message . ' @ ' . file . ':' . line` via `@error_log` and then calls `db_down_response()`. This covers a connection that succeeds and then has its grants pulled (MySQL 1142 / SQLSTATE 42000) mid-request. File-based logging only — `errorLog()` INSERTs and would itself re-throw.

`db_down_response()` (`ninja-adb21.php:117-190`) always sends `ninja_fp` (echoed back, `null` for a new install), `ninja_fp_version = '3.0'`, `ninja_data_version = '2.0'`, `ninja_sync = 10800000 + mt_rand(0, 3600000)` — a **jittered 3–4 h backoff** (`ninja-adb21.php:127`) so a recovering fleet does not re-sync as one synchronized spike — plus the CDN asset versions/URLs read from `cosmetic_css_version.txt`, `cosmetic_specific_version.txt`, `compiled_rules_version.txt`. Only when `$incomingFp` is empty does it also send `defaultWhiteList` and the two baseline rules (id 8002 leboncoin `allowAllRequests`, id 2002 default block list) — an existing client's dynamic range [30..10999] is left untouched. Then `exit`.

Trace it to the end and what does **not** happen is the whole point:

- **No row is created.** `createUser()` / `insertUserRecord()` are never reached; the install does not exist in `ninja25`.
- **No payout.** `processConversion()` is never called; no `'|s1_hold'` marker is written, so the S1 sweep has nothing to find later either — this install is invisible to finalize forever.
- **No decision.** `runBehaviorLayer()` never runs; `decision`, `behavior_score`, `behavior_flags`, `behavior_hash`, `decided_date` are never written.
- **No `sync_log` row.** `logSyncArrival()` (`ninja-adb21.php:2516`) is never reached, so the sync that hit during the outage leaves no behavioral evidence.
- **No maintenance.** `exit` fires at `ninja-adb21.php:189` — long before the post-flush block at `:3647`. A DB outage therefore also stops the scheduler, because the scheduler *is* the traffic that the outage is failing.

> **Note** — The safety net turns genuine code bugs into a degraded 200 as well. The `[ninja-adb21 uncaught]` line in the PHP error log is the only trail; grep for it before concluding the DB was down.

### 11.8 Request paths that exit before the post-flush block

"Traffic is the scheduler" holds only for traffic that reaches `ninja-adb21.php:3647`. These paths do not:

| Path | Line | Exit |
|---|---|---|
| CORS preflight | `ninja-adb21.php:56-59` | `exit(0)` on `REQUEST_METHOD === 'OPTIONS'` — headers only |
| DB connect failure | `ninja-adb21.php:309-312` → `:189` | `db_down_response()` then `exit` |
| Any uncaught throwable | `ninja-adb21.php:200-205` → `:189` | same |
| Unreadable request body | `ninja-adb21.php:335-341` | `errorLog(... 'jsonData', "false")` then `exit` |
| Empty request body | `ninja-adb21.php:342-348` | `errorLog(... 'jsonData', "empty")` then `exit` |
| cid/sid duplicate hit, user row gone | `ninja-adb21.php:2905-2906` | `errorLog(... 'dupCIDSID-userNotFound', $cid)` then `exit` |
| IP duplicate hit, user row gone | `ninja-adb21.php:3056-3057` | `errorLog(... 'dupIP-userNotFound', $dupIP['existingId'])` then `exit` |

> *Consequence: a traffic mix dominated by preflights or malformed bodies can look busy in the access log while scheduling nothing. Only a request that completes `processUserLogic` and reaches the flush advances the maintenance clock.*

### 11.9 Observability — and which files lie

Files to check, in order of trustworthiness:

| File | Written by | Read as |
|---|---|---|
| `last_finalize_success.txt` | `finalize_pending_installs.php:340`, only on a full drain | **The one finalize stamp that cannot lie.** Age > ~1 h with traffic flowing = draining is not keeping up |
| `finalize_backlog.txt` | touched `finalize_pending_installs.php:334`; unlinked `:339` | **ABSENT = keeping up.** Present = the last slice stopped with work left |
| `last_audit_run.txt` | `generate_fraud_report.php:349` | Age > 6 h = the fraud audit is not completing; blacklists and `calibrated_risk_threshold` are frozen |
| `last_funnel_run.txt` | `generate_funnel_cache.php:464`, after the `rename()` | Older than the last 01:00 GMT+4 boundary = the funnel rebuild is failing |
| `compiled_rules_version.txt`, `cosmetic_css_version.txt`, `cosmetic_specific_version.txt` | their generators | Asset freshness only |
| `last_finalize_run.txt` | `finalize_pending_installs.php:126`, on run *start* | Says "runs are starting and reaching a working DB" — **not** that any row was processed |

PHP error-log lines worth grepping:

- `inline finalize_pending_installs.php: S1-delay sweep. Due: X (Paid: Y, Refused: Z, Forfeited: …, Held: …, Errors: …)` — the stats line every inline run emits via `ninja-adb21.php:3772`.
- `inline {$script}: uncaught …` (`ninja-adb21.php:3767`).
- `bg_run: self-call connect failed for {$script}: …` / `bg_run: self-call for {$script} answered '…' — NOT executed` (`ninja-adb21.php:3720`, `:3732`).
- `audit trigger: self-call failed …`, `rules regen trigger: …`, `cosmetic regen trigger: …`.
- `[ninja-adb21 uncaught] … @ file:line` — the degraded-200 trail (§11.7).
- `finalize_pending_installs.php: previous run still in progress, skipping` / `another run started <300s ago, skipping duplicate` — normal single-flighting, not an error.

**Files that lie.** `last_audit_spawn.txt`, `last_funnel_spawn.txt`, `last_rules_regen_attempt.txt`, `last_cosmetic_regen_attempt.txt` are attempt records only — a fresh one proves an attempt was *made*, never that it succeeded. `last_finalize_run.txt` is fresh whenever a run merely started. `last_finalize_spawn.txt` is **vestigial**: nothing writes it since finalize moved inline; if it exists on the server it is a 2026-07-16 leftover and means nothing. Reading any of these as health is exactly the mistake that hid a five-day payout outage.

## 12. Invariants — claimed vs actually enforced

| Invariant | Stated where | Actually holds? | How it breaks |
|---|---|---|---|
| The decision (`decision`, `behavior_score`, `behavior_flags`) never reads `conv` | the V4 "two-axes principle"; `finalize_pending_installs.php:4-7`; `ninja-adb21.php:3192-3199` | **Yes, today.** `classifier_v4.php` contains no read of `conv` at all; `extractBehaviorFeatures()` / `evaluateBehavior()` never take it; the sweep query at `finalize_pending_installs.php:415-419` deliberately carries **no `conv` filter**, so paid installs terminalize too | Two latent couplings that only activate in Stage 2. (1) `logSyncArrival()`: `$skipHardBlocked = (!$pureShadow && (int)$existingUser['conv'] === 2)` at `ninja-adb21.php:2526` — once any behavior flag is on, a `conv=2` install stops producing `sync_log` rows, which starves the classifier's own input. That is `conv` feeding back into the decision through the data, not the code. (2) `runBehaviorLayer()` reads `conv` for its money side-effects — `$payable = ((int)$install['conv'] === 0)` at `behavior_lib.php:537` and `$alreadyPaid = in_array((int)$install['conv'], [1,3], true)` at `behavior_lib.php:568`, the latter switching the bot branch between "block" and "flag only". Both sit inside `if (!empty($BEHAVIOR_ENFORCE) \|\| !empty($DELAYED_CONVERSION))` / `if (!empty($BEHAVIOR_ENFORCE))`, so both are **NOT ACTIVE TODAY** |
| Zero non-terminal installs past 24 h | `finalize_pending_installs.php:552-558` ("HARD GUARANTEE") | **DOES NOT HOLD.** | The straggler/expiry `UPDATE` is wrapped in `if (!$pureShadow)` at `finalize_pending_installs.php:563-564`, where `$pureShadow = !$cfgEnforce && !$cfgDelayed && !$cfgDefer` and all three config flags are `false` — so the block **never runs today**. Compounding it, the decision sweep only selects rows aged `> 3 HOUR AND < 24 HOUR` (`finalize_pending_installs.php:415-416`): a row that is not terminalized inside that window falls out of every sweep and rests at `decision = 'pending'` or `NULL` forever. Partially offset by `runBehaviorLayer()` terminalizing human/bot at **any** age (`behavior_lib.php:500-507`), but only for installs that keep syncing |
| Money settles once at the 3 h checkpoint and never moves after | the V4-2 design narrative | **DOES NOT HOLD — twice over.** | (a) The live hold is **1 h, not 3 h**: `$STAGE1_DELAY_CONV_SEC = 3600` with `$STAGE1_DELAY_CONV_SEC_AN = []` (no override), read by `s1_delay_seconds()` at `behavior_lib.php:248-253` and applied at `finalize_pending_installs.php:242-246`. `BEHAVIOR_WINDOW_SEC = 10800` (3 h) is the *observation* window for classification, a different clock. (b) The pay window is **`$S1_HOLD_PAY_WINDOW_SEC = 259200` (72 h)** — a held row stays payable for three days and is paid at the first catch-up run (`finalize_pending_installs.php:187`, `:225-239`). After any outage, payments landing 24–72 h after install are routine, not exceptional. Note the standing caveat in config: networks may refuse to attribute a postback fired > 24 h after click. **The 1 h hold is the STAGE-1 DELAY — timing only.** Same gate, same recipients, postback ~1 h late. It is NOT Stage 2; Stage 2 is `$BEHAVIOR_AN_ALLOWLIST`, which is `[]`, so **no install can be REFUSED today** |
| An attributed install never rests at `conv=0` past 24 h | `finalize_pending_installs.php:363-374` (CONV HYGIENE SWEEP) | **Yes.** The one terminal-state sweep that is *not* flag-gated — "bookkeeping on money that was already not paid, not enforcement" | Bulk `UPDATE … SET conv='4', conv_date=NOW(), fraud_flags = CONCAT(COALESCE(fraud_flags,''), '\|conv4_expired_unpaid') WHERE an IS NOT NULL AND cid IS NOT NULL AND (conv='0' OR conv IS NULL) AND created_date >= '2026-07-13 09:00:00' AND created_date <= NOW() - INTERVAL 24 HOUR` (canonical: `finalize_pending_installs.php:382-396`). Two bounded exceptions, both deliberate: it is **FORWARD-ONLY** — rows created before `$CONV4_HYGIENE_FROM = '2026-07-13 09:00:00'` keep their historical `conv` forever (`finalize_pending_installs.php:375`); and held rows still inside the 72 h pay window are excluded as in-flight, via `AND (COALESCE(fraud_flags,'') NOT LIKE '%\|s1_hold%' OR created_date <= NOW() - INTERVAL 259200 SECOND)` (`finalize_pending_installs.php:390-391`) — so a held row may legitimately sit at `conv=0` for up to 72 h. In inline mode the statement carries `LIMIT 5000` (`finalize_pending_installs.php:395`), so a large backlog closes over several slices |
| A payout fires at most once per install | `finalize_pending_installs.php:163-169`; `behavior_lib.php:532-536` | **Yes.** | The guarantee is a single atomic claim, `finalize_pending_installs.php:299-307`: `UPDATE … SET fraud_flags = REPLACE(fraud_flags, '\|s1_hold', '\|s1_paid') WHERE id = ? AND (conv='0' OR conv IS NULL) AND fraud_flags LIKE '%\|s1_hold%'`, followed by `if ($stmtClaim->rowCount() !== 1) { continue; }`. Exactly one concurrent run can flip the marker, so exactly one proceeds to `processConversion()` (`:297-299`). The markers are substring-safe by construction — `'\|s1_hold'` is not contained in `'\|s1_paid'`, `'\|s1_forfeit'` or `'\|s1_refused'`. This is deliberately **at-most-once, not exactly-once**: if the process dies between the claim and the postback the row stays `'\|s1_paid'` with `conv=0`, visible for audit, and is later closed `conv=4` by the hygiene sweep rather than risking a double postback. The forfeit and refuse paths use the same claim shape (canonical: `:229-235`, `:277-283` — both also stamping `conv_date = NOW()` since 2026-09-02). Three further layers make overlap unlikely before it has to be safe: the trigger's stamp check (`ninja-adb21.php:3810`), the `LOCK_EX\|LOCK_NB` flock (`finalize_pending_installs.php:97-101`), and the 300 s recent-run guard for cron/manual runs (`:91-96`) |
| Once paid, never un-paid | `behavior_lib.php:532-536`, `:576-584` | **Yes.** | Every `conv`-mutating statement in the codebase is guarded to the unpaid state. `runBehaviorLayer()`'s human branch pays only when `$payable = ((int)$install['conv'] === 0)` (`behavior_lib.php:537`) — which simultaneously blocks re-paying `conv` 1/3, un-blocking a Tier-3 `conv=2`, and re-converting an expired `conv=4`. Its bot branch splits on `$alreadyPaid` (`behavior_lib.php:567-595`): a **paid** install has `conv` left untouched and receives only `flagged = CURDATE()`, `device_hash`, and a `'\|behavior_bot_post_pay:<flags>'` marker appended to `fraud_flags` — this is the **chargeback path**, evidence for a post-hoc claim rather than a reversal; only an unpaid install goes `conv 0 → '4'` (2026-09-03 convention). The finalize sweeps all carry `AND (conv='0' OR conv IS NULL)`, as does `sanitize_fast_uninstalls.php:76`. **Since 2026-09-02 the pay UPDATE itself also carries `AND (conv = '0' OR conv IS NULL)`** (canonical: `behavior_lib.php:556`) — before the fix it was `WHERE id = ?` only, so a terminal `conv` (2/4) set concurrently during `processConversion`'s network wait could be silently overwritten; and the bot block writes **quoted** `'4'` (2026-09-03 convention; canonical: `behavior_lib.php:594-598` — the pre-fix unquoted `2` would have stored `'1'` = PAID via the enum-index trap, §14.1). `farewell.php` writes only `disabled` (`farewell.php:77`, `:87`) and never touches `conv`. Both bot branches are inside `if (!empty($BEHAVIOR_ENFORCE))` — **NOT ACTIVE TODAY** |
| Organic (`an = NULL`) installs are never paid | `config.php` `$BEHAVIOR_AN_ALLOWLIST` comment; `behavior_lib.php:221-222`; `ninja-adb21.php:1708-1710` | **Yes — verified at four independent layers.** | (1) `processConversion()` returns `0` immediately on `if (empty($an))` (`ninja-adb21.php:1708-1711`), and again on `if (empty($cid))` (`:1714-1716`) — centralized so *every* caller is protected. (2) The install gate requires `!is_null($an) && !is_null($cid) && $level === "0" && $conv !== 2` (`ninja-adb21.php:3320`). (3) `s1_delay_in_scope()` returns false on `empty($an)` (`behavior_lib.php:240-241`), so an organic install never gets a `'\|s1_hold'` marker and is invisible to the S1 sweep, whose SELECT keys on `fraud_flags LIKE '%\|s1_hold%'` (`finalize_pending_installs.php:200`). (4) `an_in_behavior_scope()` also returns false on `empty($an)` (`behavior_lib.php:226`), so organic is Stage 1 everywhere. The hygiene sweep additionally requires `an IS NOT NULL AND cid IS NOT NULL` (`finalize_pending_installs.php:387`) — organic rows are simply left alone at `conv=0` |
| `level !== "0"` is never paid | `ninja-adb21.php:3320`; `finalize_pending_installs.php:291-296`; `behavior_lib.php:541-542` | **Yes — the strongest invariant in the system.** Absolute, at every enforcement site | Enforced independently at all three money sites, each restating the Stop Ads / Wonder parity rule: install gate `$level === "0"` (`ninja-adb21.php:3320`); S1 held-payout sweep `if ((string)($held['level'] ?? '') !== '0') { $s1Stats['held']++; continue; }` (`finalize_pending_installs.php:294-297`) — the row is left **held**, not paid, and ages out to `'\|s1_forfeit'` at the 72 h pay window; behavior layer `$levelClean = ((string)($install['level'] ?? '') === '0')` ANDed into the pay condition (`behavior_lib.php:541-542`); and the deferred branch `if (!empty($BEHAVIOR_ENFORCE) && !empty($DEFER_CONVERT) && $levelClean)` (`finalize_pending_installs.php:523-525`). Note the comparison is a **string** compare against `"0"` at every site — `level` accumulates digits (`$level .= '7'` etc., `ninja-adb21.php:1663`), so any single risk digit permanently disqualifies the install |
| `conv_date` always dates the CURRENT `conv` value | `migrate_conv_date_updated_at.sql`; §14.1 rule | **Yes — canonical since 2026-09-02.** | At INSERT, no code writes it: the column `DEFAULT` fires in the same statement as `created_date`'s and `NOW()` is per-statement, so `conv_date == created_date` exactly (`ext-server.php:2240` is the only user-table INSERT). On UPDATE, every `conv` writer maintains it: plain `conv_date = NOW()` where the `WHERE` guard (`conv='0' OR conv IS NULL`) guarantees a real transition (canonical: `finalize_pending_installs.php:232/280/384/570`), and `SET conv_date = IF(conv <=> ?, conv_date, NOW()), conv = ?` — `conv_date` assigned **before** `conv`, since SET evaluates left-to-right — at the four sites that can rewrite the same value (canonical: `finalize_pending_installs.php:315/538`, `behavior_lib.php:556/594`). A same-value rewrite never moves the stamp, so the column stays truthful for forensics: *which code path set this `conv`, and when*. Pre-migration rows are backfilled to `created_date` (best available approximation). Holds only if every **future** `conv` write follows the same pattern — that is the standing rule, §14.1 |

> **Defect** — `$HAS_FRAUD_COLUMNS = false` silently destroys the entire Stage-1 delay while the global hold is on. `$GLOBALS['has_fraud_columns']` (`ninja-adb21.php:101`) gates whether `device_hash`, `fraud_score` and `fraud_flags` are appended to the INSERT column list at `ninja-adb21.php:1806-1816`. With it false, the `'|s1_hold'` marker built at `ninja-adb21.php:3414` is never written to the row. Since the S1 sweep selects **exclusively** by `fraud_flags LIKE '%|s1_hold%'` (`finalize_pending_installs.php:200`), the row is never found, never paid, and — because it is attributed and sits at `conv=0` — is quietly closed `conv=4` with `'|conv4_expired_unpaid'` by the hygiene sweep 24 h later. Every payout is lost, with no error logged anywhere. The flag is `true` in the live config (`config.php:36`); it must never be flipped while `$STAGE1_DELAY_CONV_AN` is non-empty.

> **Defect** — `proxycheck` `'lookup_failed'` fails OPEN. `evaluateRisk()` returns level `'0'` and `conv = 0` on a lookup outage, so the install passes `$level === "0"` at `ninja-adb21.php:3320` and is paid. A proxycheck outage therefore makes every install of that period look clean and payable, and the invariant table above holds vacuously for those rows — `level` is `"0"` because nothing was checked, not because the install is clean.

> **Defect** — `evaluateVirtualMachine()`'s `catch` returns `'detection_error:…'` with `is_vm = 0`, rather than a `'physical:'` prefix. An evaluator exception therefore lands the install in `tier1:high_trust` — the VM layer fails **open**, not closed.

---

## 13. Configuration reference

Every knob that gates detection or money lives in one of five places: `config.php` (deployment-editable), `define()` constants in the code, plain file-scope globals set at boot in `ninja-adb21.php`, the row itself, and the stamp/lock files on disk. This section enumerates all of them.

> **Note** — The live host is Infomaniak, which has **no cron**. Several comments inside `config.php` and `finalize_pending_installs.php` still describe "the 15-min cron" as the scheduler (`config.php:69`, `config.php:81-83`, `finalize_pending_installs.php:20-28`). Those comments are **stale**. Today `finalize_pending_installs.php` is driven in-process, post-flush, by inbound traffic (`ninja-adb21.php:3792-3795` → `ninja_run_inline`). Read the code, not the comment.

### 13.1 config.php

File: `/backend/infomaniak/config.php`, 90 lines. Loaded by `ninja-adb21.php:76`, `behavior_lib.php` consumers, `finalize_pending_installs.php:37`, `farewell.php:11`, `rerun-vm.php:3`, and every analysis tool. Re-required (not `_once`) inside `ninja_run_inline()` at `ninja-adb21.php:3759` so an inline finalize slice reads authoritative config values rather than request-mutated globals.

In file order:

| Variable | Live value | What it does | Change it when | Rollback |
|---|---|---|---|---|
| `NINJA_PDO_INIT_CMD` (const) | `Pdo\Mysql::ATTR_INIT_COMMAND` on PHP ≥ 8.4, else `PDO::MYSQL_ATTR_INIT_COMMAND` | Picks the non-deprecated PDO init-command key at runtime. Every PDO connection site MUST pass it (`SET NAMES utf8mb4 COLLATE utf8mb4_unicode_ci`) or joins hit MySQL error 1270. `config.php:8-10` | Never by hand — it is version-adaptive | n/a (guarded by `!defined`) |
| `$ninja_db_config` (commented block) | *(inactive)* Hostinger credentials, `host=localhost`, `dbname=u382419048_ninjablock25` | Dead block kept for a Hostinger rollback. `config.php:11-19` | You move back to Hostinger | Swap the comment markers with the block below |
| `$ninja_db_config` | `host=7k10c.myd.infomaniak.com`, `dbname=7k10c_ninjablock`, `user=7k10c_ninja`, `pass=`*(secret)*, `charset=utf8mb4` | The single DB handle source for the endpoint, finalize, farewell, and all tools. `config.php:20-26` | Host migration only | Re-comment and un-comment the Hostinger block |
| `$aes_key` | *(secret)* | Legacy uninstall-UID decryption key. `farewell.php` SHA-256s it to a 32-byte key for `aes-256-gcm` (`farewell.php:54`). **It is duplicated as a bare string literal inside `farewell.php:54`, not read from `$aes_key`** — changing `config.php` alone silently breaks legacy-UID uninstall attribution. `config.php:28` | Never, unless you also edit `farewell.php:54` in the same deploy | Restore both sites together |
| `$proxycheck_key` | *(secret)* | API key for proxycheck.io, the source of `provider`/`organisation`/`hostname`/`proxy`/`type`/`risk`/`asn` that feed `evaluateRisk()` and the tier layer. `config.php:29` | Key rotation / quota upgrade | Restore the previous key |
| `$token_7thsense` | *(secret)* JWT | Bearer token for `https://api.7thsense.media/api/v1/payouts/{sid}/conversion` — the payout-authorization call at the head of `processConversion()` (`behavior_lib.php:793-817`). Expiry `exp` in the payload sits in 2059. `config.php:30` | Token rotation | Restore the previous token |
| `$HAS_FRAUD_COLUMNS` | `true` | Schema capability flag (2026-07-17). Replaces the old per-request `SHOW COLUMNS` probe. Copied to `$GLOBALS['has_fraud_columns']` at `ninja-adb21.php:101`; consumed at `ninja-adb21.php:1806` (append `device_hash`/`fraud_score`/`fraud_flags` to the INSERT) and `ninja-adb21.php:3084` (device-collision lookup). `config.php:36` | Only when deploying against a non-migrated DB | Set back to `true` |
| `$BEHAVIOR_ENFORCE` | `false` | Master enforcement switch. `true` lets `runBehaviorLayer` block bots (conv 0→'4') and lets the tier layer gate conv at install. `config.php:39` | Stage 2 go-live | `false` |
| `$DEFER_CONVERT` | `false` | Pays `decision='deferred'` installs at finalization (`finalize_pending_installs.php:525`). Only reachable when `$BEHAVIOR_ENFORCE` is also `true`. `config.php:40` | Stage 2 go-live | `false` |
| `$DELAYED_CONVERSION` | `false` | Moves payment from install to the behavioral checkpoint and activates Stage-2 tier gating. `config.php:41` | Stage 2 go-live | `false` |
| `$BEHAVIOR_AN_ALLOWLIST` | `[]` | **The REFUSAL switch.** Exact-match list of ad networks whose held installs go through `mergeConvV4` at the checkpoint and can be **REFUSED** (`finalize_pending_installs.php:255-289`). Empty ⇒ `an_in_behavior_scope()` returns `false` for every network ⇒ **STAGE 2 IS OFF; no install can be REFUSED today.** Was `['gn']` 2026-07-13→07-16. `config.php:46` | You want refusals for a specific network | `[]` |
| `$STAGE1_DELAY_CONV_AN` | `['*']` | **The TIMING switch.** Networks whose Stage-1 payout is HELD at install and fired ~1h later. Supports the `'*'` wildcard (`behavior_lib.php:240`). Same gate, same recipient, later postback. `config.php:68` | You want pay-at-install back | `[]` (see the commented rollback block, `config.php:77-79`) |
| `$STAGE1_DELAY_CONV_SEC` | `3600` | Default hold duration in seconds for held networks. `config.php:69` | Change the uniform hold length | `3600`, or `0` for immediate drain |
| `$STAGE1_DELAY_CONV_SEC_AN` | `[]` | Per-network override map `an => seconds` (`behavior_lib.php:251`). Empty ⇒ every network uses `$STAGE1_DELAY_CONV_SEC`. Was `['gn' => 10800]` during the 07-12→07-16 rollout. `config.php:70` | One network needs a different deadline | `[]` |
| `$S1_HOLD_PAY_WINDOW_SEC` | `259200` (72 h) | How long a `\|s1_hold` row stays PAYABLE after install. Past it → `\|s1_forfeit` + `conv=4` (`finalize_pending_installs.php:229-239`). Also excludes in-flight held rows from the conv-hygiene bulk close (`finalize_pending_installs.php:391-392`). `config.php:89` | Outage rescue needs a longer catch-up window | `259200`; was 24 h, then 48 h |

> *Rationale for the 72 h window: rows whose deadline passed during the 2026-07-15 maintenance outage are not the users' fault; the sweep pays them at the first catch-up run. Ad networks may not attribute a postback fired >24 h after the click, so the tail of that window is best-effort.*

> **Defect** — `$HAS_FRAUD_COLUMNS = false` would drop `fraud_flags` from the INSERT column list (`ninja-adb21.php:1806-1815`). The `'|s1_hold'` marker at `ninja-adb21.php:3414` would then never be written, the finalize sweep's `WHERE fraud_flags LIKE '%|s1_hold%'` (`finalize_pending_installs.php:200`) would never find the row, and — because the install path skips `processConversion` whenever `s1_delay_in_scope($an)` is true (`ninja-adb21.php:3330-3333`) — **every payout would be silently lost** for as long as the global hold is on. Nothing logs this. Do not set the flag to `false` while `$STAGE1_DELAY_CONV_AN` is non-empty.

### 13.2 The two switches readers confuse

`$BEHAVIOR_AN_ALLOWLIST` and `$STAGE1_DELAY_CONV_AN` are both "lists of ad networks" in the same file, ten lines apart. They do completely different things and are deliberately independent (`behavior_lib.php:232-236`).

| | `$BEHAVIOR_AN_ALLOWLIST` | `$STAGE1_DELAY_CONV_AN` |
|---|---|---|
| Nickname | **The REFUSAL switch** (Stage 2) | **The TIMING switch** (Stage-1 delay) |
| Live value | `[]` | `['*']` |
| Read by | `an_in_behavior_scope()`, `behavior_lib.php:223-227` | `s1_delay_in_scope()`, `behavior_lib.php:237-241` |
| Matching | **Exact match only** — `in_array($an, $list, true)`. No `'*'` support | `'*'` wildcard OR exact match |
| Consumed at | `finalize_pending_installs.php:255` (the `mergeConvV4` branch inside the `\|s1_hold` sweep); `finalize_pending_installs.php:450-453` (per-row re-derivation of the three behavior flags) | `ninja-adb21.php:3330` (hold instead of paying at install) |
| What it changes | **WHO gets paid** — can turn a payout into a refusal (`conv=4`, `\|s1_refused`) | **WHEN the postback fires** — never who |
| Effect on the gate | none; the Stop-Ads gate still runs first | none; byte-identical gate, `$po` is hardcoded `0` at both sites |
| Effect on shadow data | none — `decision` is conv-independent | none |
| Rollback semantics | emptying it stops new refusals immediately | emptying it makes new installs pay at install; already-held rows still drain, because the sweep selects by the `\|s1_hold` marker, not by this list |

Truth table of the four combinations:

| `$BEHAVIOR_AN_ALLOWLIST` | `$STAGE1_DELAY_CONV_AN` | Result |
|---|---|---|
| `[]` | `[]` | Pure Stage 1, pay at install. `processConversion` fires inside `createUser`. No `\|s1_hold` markers exist. This is the 2026-07-16 state. |
| `[]` | `['*']` | **LIVE TODAY.** Pay at install is deferred by 3600 s for every attributed network; the same installs get the same money, ~1 h late. Nothing can be refused. Markers: `\|s1_hold` → `\|s1_paid` or `\|s1_forfeit`. |
| `['gn']` | `[]` | Incoherent — the merge branch lives *inside* the `\|s1_hold` sweep (`finalize_pending_installs.php:255`), so with nothing held it never executes. The allowlist has no effect on payouts; it only narrows the per-row flag re-derivation at `finalize_pending_installs.php:450`. Do not deploy this shape expecting refusals. |
| `['gn']` | `['*']` (or `['gn']`) | Full V4-3 for `gn`: held rows reaching their deadline run `mergeConvV4(deriveTierFromRow, decision, proof)`. Bots and Tier-3-without-proof become `\|s1_refused` + `conv=4`; everyone else falls through to the level check and pays. Rows whose `decision` is still `pending` stay held for a later pass (`finalize_pending_installs.php:258-264`). |

> **Note** — Neither switch is `$DELAYED_CONVERSION`. That flag (`config.php:41`) activates Stage-2 tier gating at install (`ninja-adb21.php:3295-3309`) and is `false`. The Stage-1 delay is independent of all three behavior flags.

### 13.3 Code constants

| Constant | Live value | Defined at | Read at | Purpose |
|---|---|---|---|---|
| `BEHAVIOR_COLLECT_SYNC_MS` | `600000` (10 min) | `ninja-adb21.php:87` | `ninja-adb21.php:2532`, `ninja-adb21.php:3351` | Re-sync interval pushed to the extension while an install is inside the observation window. `10 min × BEHAVIOR_SYNC_CAP(20) = 200 min`, covering the 3 h window. |
| `BEHAVIOR_WINDOW_SEC` | `10800` (3 h) | `ninja-adb21.php:88`; re-guarded in `behavior_lib.php:15` so the standalone finalize run also has it | `ninja-adb21.php:2529`, `behavior_lib.php:499` | Observation window. Past it, `runBehaviorLayer` forces an undecided install to terminal `deferred`. |
| `BEHAVIOR_SYNC_CAP` | `20` | `ninja-adb21.php:89` | `ninja-adb21.php:2521` | Max `sync_log` rows written per install (compared against the `updates` counter). |
| `GENERICS` | `['google.com','youtube.com','chromewebstore.google.com','bing.com']` | `behavior_lib.php:18-24` | `is_generic()`, `behavior_lib.php:68` | Domains excluded from the organic core. |
| `AUTH_MARKERS` | `['web.whatsapp.com','mail.google.com','docs.google.com','drive.google.com','accounts.google.com','outlook.*','instagram.com','facebook.com','x.com','reddit.com','tiktok.com','linkedin.com','chatgpt.com','canva.com','discord.com','netflix.com']` | `behavior_lib.php:133-150` | `matches_auth_marker()`, `behavior_lib.php:154`; used for `auth_session` and `instdom_has_login` | Logged-in-service evidence. `'outlook.*'` is special-cased to prefix matching (`behavior_lib.php:157-160`); every other entry matches exact or as a dot-suffix. |
| `BG_HOST` | `'ninja-block.com'` | `ninja-adb21.php:3701` | `bg_run()`, `ninja-adb21.php:3718-3724` | Host for the verified HTTPS self-call. |
| `BG_PREFIX` | `'/'` | `ninja-adb21.php:3702` | same | Path prefix for the self-call. |
| `NINJA_PDO_INIT_CMD` | see §13.1 | `config.php:8-10` | every PDO connect site | — |
| `$CONV4_HYGIENE_FROM` | `'2026-07-13 09:00:00'` | `finalize_pending_installs.php:376` | `finalize_pending_installs.php:389,396` | Forward-only cutoff. Rows created before this timestamp keep their historical `conv` values forever; the hygiene sweep never backfills. |

Finalize row limits and deadlines:

| Knob | Live value | Where | Applies to |
|---|---|---|---|
| `$fpiRowLimit` | `400` in inline mode, `0` (unlimited) standalone | `finalize_pending_installs.php:58` | `ORDER BY created_date ASC LIMIT 400` on the money sweep (`:206`) |
| Inline slice budget | `12.0` s when the worker detached (`fastcgi_finish_request` succeeded), `2.0` s otherwise | `ninja-adb21.php:3810` | `$GLOBALS['NINJA_FPI_DEADLINE']` (`:3765`), checked per-row at `finalize_pending_installs.php:214` and `:442` |
| Recent-run guard | `300` s | `finalize_pending_installs.php:93` | standalone/cron runs only; inline mode bypasses it |
| S1 SELECT lower bound | `$s1MinDelay` = `3600` s (min over `$STAGE1_DELAY_CONV_SEC` and `$STAGE1_DELAY_CONV_SEC_AN`) | `finalize_pending_installs.php:172-179,202` | oldest-eligible age |
| S1 SELECT floor | `$s1PayWindowSec + 86400` = `345600` s (96 h) | `finalize_pending_installs.php:191,203` | keeps just-aged-out rows fetchable so they get a proper `\|s1_forfeit` |
| Conv-hygiene age | `24 HOUR` | `finalize_pending_installs.php:390` | attributed rows still at `conv` 0/NULL |
| Conv-hygiene inline cap | `LIMIT 5000` | `finalize_pending_installs.php:395` | inline mode only |
| Decision sweep window | `created_date` between `NOW()-24 HOUR` and `NOW()-3 HOUR` | `finalize_pending_installs.php:415-416` | `decision` still `pending`/NULL |
| Decision sweep inline cap | `LIMIT 150` | `finalize_pending_installs.php:419` | inline mode only |
| Straggler expiry cutoff | `<= NOW() - INTERVAL 24 HOUR` | `finalize_pending_installs.php:574` | **NOT ACTIVE TODAY** — see below |
| `memory_limit` / `set_time_limit` | `512M` / `300` s | `finalize_pending_installs.php:30-31` | standalone runs |
| curl timeouts | `CURLOPT_TIMEOUT 10`, `CURLOPT_CONNECTTIMEOUT 5` | `behavior_lib.php:631-632` (`convPixel`), `:816-817` (`processConversion`) | a dead network endpoint cannot hang a slice |

> **Defect** — The straggler/expiry sweep at `finalize_pending_installs.php:564-584` is wrapped in `if (!$pureShadow)`, and `$pureShadow` is `true` because all three behavior flags are `false` (`finalize_pending_installs.php:563`). **It never runs today.** The frequently-repeated guarantee "zero non-terminal installs past 24 h" is therefore **not enforced**: rows can rest at `decision='pending'` indefinitely once they fall out of the 3–24 h sweep window. Only the *conv* side is closed, by the ungated hygiene sweep at `:383-396`.

Maintenance TTLs, all evaluated post-flush at the bottom of `ninja-adb21.php`:

| Job | Success marker (TTL) | Attempt stamp (rate limit) | Trigger line | Execution |
|---|---|---|---|---|
| Fraud report | `last_audit_run.txt`, `21600` s (6 h) | `last_audit_spawn.txt`, `3600` s | `ninja-adb21.php:3772-3773` | `bg_run()` only — **no inline fallback** (the script defines unguarded function names that would fatal on a second definition) |
| Finalize (money) | — (uses the *run* stamp, not a success TTL) | `last_finalize_run.txt` > `1200` s, or `finalize_backlog.txt` present and run stamp > `60` s | `ninja-adb21.php:3794-3796` | `ninja_run_inline()` — always in-process |
| Funnel cache | `last_funnel_run.txt` older than the most recent 01:00 GMT+4 boundary + 900 s, or `funnel_cache.json` missing | `last_funnel_spawn.txt`, `1800` s | `ninja-adb21.php:3811-3818` | `bg_run()` first, `ninja_run_inline()` fallback **only when detached** |
| Compiled DNR rules | `compiled_rules_version.txt`, `86400` s (24 h), or cache/marker missing | `last_rules_regen_attempt.txt`, `1800` s | `ninja-adb21.php:3835-3854` | inline under `compiled_rules_cache.json.lock` when detached, else `bg_run()` |
| Cosmetic CSS | `max(mtime)` of `cosmetic_css_version.txt` and `cosmetic_specific_version.txt` > `86400` s, or either cache missing | `last_cosmetic_regen_attempt.txt`, `1800` s | `ninja-adb21.php:3862-3883` | inline under `cosmetic_specific_cache.json.lock` when detached, else `bg_run()` |

> *Doctrine (2026-07-21/07-23): the SUCCESS marker is touched only by the job itself, at the end of a successful run; the ATTEMPT stamp only rate-limits retries. Pre-touching an attempt as if it were a success is exactly how the audit stamp looked fresh for five days while the report never ran, and how the compiled rule cache froze on a week-old build.*

### 13.4 $GLOBALS set at boot

Set unconditionally in `ninja-adb21.php` before any request logic:

| Global | Live value | Set at | Read at | Notes |
|---|---|---|---|---|
| `$GLOBALS['audit_window_days']` | `30` | `ninja-adb21.php:100` | **nowhere** | **DEAD CODE.** A directory-wide grep finds this name only at its four assignment sites (`ninja-adb21.php:100`, plus the dev/legacy copies `ninja-adb21-dev.php:100`, `ninja-adb.php:69`, `ninja-adb-dev.php:69`). No reader exists in any live file, tool, or doc. Changing it does nothing. |
| `$GLOBALS['has_fraud_columns']` | `true` (from `$HAS_FRAUD_COLUMNS`, defaulting to `true` when unset) | `ninja-adb21.php:101` | `ninja-adb21.php:1801`, `ninja-adb21.php:3084` | Gates the three fraud columns in the INSERT and the 7-day device-collision count |
| `$GLOBALS['calibrated_risk_threshold']` | `33` fallback; overwritten from `fraud_cache.json` when present | `ninja-adb21.php:102`, `:221` | `evaluateRisk()` `ninja-adb21.php:1660`, tier layer `ninja-adb21.php:3173` | Boundary between `level` digit `7` / Tier-2 `proxy_flag` and Tier-2 `minor_risk_score` |
| `$GLOBALS['blacklisted_devices']` | `[]` unless `fraud_cache.json` supplies it | `ninja-adb21.php:103`, `:219` | `ninja-adb21.php:3085-3086` → Tier 3 `blacklisted_device` | Ships **empty** since 2026-07-09; manual emergency lever only |
| `$GLOBALS['blacklisted_asns']` | `[]` unless `fraud_cache.json` supplies it | `ninja-adb21.php:104`, `:220` | `ninja-adb21.php:3089-3090` → Tier 3 `blacklisted_asn` | Ships **empty** since 2026-07-09; manual emergency lever only |
| `$GLOBALS['funnel']` | contents of `funnel_cache.json`, or `[]` | `ninja-adb21.php:233` (and `finalize_pending_installs.php:142-149` for standalone runs) | `extractBehaviorFeatures()` `behavior_lib.php:261-263`; `runBehaviorLayer` `behavior_lib.php:466` | Read-only in the request path — **never regenerated inside a web request** since 2026-07-17 |
| `$table` / `$GLOBALS['table']` | `'ninja25'` | `ninja-adb21.php:93` (also `farewell.php:15`, `rerun-vm.php:7`, `dashboard_detection.php:22`) | every SQL site | `finalize_pending_installs.php:128` falls back to `'ninja25'` and its inline comment `prod = stopads24` is **stale** — the live table is `ninja25` on every entry point |

Plain file-scope variables that gate detection and are read through `$GLOBALS`:

| Variable | Live value | Set at | Read at | Effect |
|---|---|---|---|---|
| `$storeExtId` | `'ppfa'` | `ninja-adb21.php:354` | `evaluateRisk` `:1621`, tier layer `:3145` | First 4 chars of the published Chrome extension ID |
| `$storeExtIdBing` | `''` | `ninja-adb21.php:355` | same | Reserved for the Edge/Bing build; never populated |
| `$isExtApproved` | `true` | `ninja-adb21.php:356` | `ninja-adb21.php:2626` | Whether rule injection is on |
| `$shortExtId` | `substr($extid, 0, 4)`, or `null` when the payload carries no `extid` | `ninja-adb21.php:368` | `evaluateRisk` `:1622`, tier layer `:3146` | Feeds the `extid_mismatch` test |

> **Defect** — With `$storeExtIdBing = ''` (`ninja-adb21.php:355`) and `$shortExtId = null` when the extension sends no `extid` (`ninja-adb21.php:368`), the mismatch test at `ninja-adb21.php:1621-1623` evaluates `'ppfa' !== '' && null !== 'ppfa' && null !== ''` → **true**. Every extid-less request therefore gets `level` digit `4` and `conv = 2` (`ninja-adb21.php:1624-1625`), and the same condition at `ninja-adb21.php:3145-3147` puts it in `tier3:extid_mismatch`. This is a missing-data case being scored as a positive integrity violation.

> **Defect** — `evaluateRisk()` returns `['level' => '0', 'conv' => 0]` on `lookup_failed` (`ninja-adb21.php:1615-1617`). A proxycheck outage therefore makes **every install look clean and payable**: no risk digits, no Tier-2 `proxy_flag`, no Tier-3 `hard_proxy_type`. The failure mode is silent and open.

### 13.5 File-based state

All paths are relative to `__DIR__` of `ninja-adb21.php` (the deployment root). A stamp is only meaningful in combination with its kind: **SUCCESS** stamps are touched by the job itself, at the end, after the work landed; **ATTEMPT** stamps are touched by the trigger before the work starts and prove nothing about the outcome.

| File | Kind | Written by | Read by | Absence means |
|---|---|---|---|---|
| `fraud_cache.json` | cache | `generate_fraud_report.php:277` | `ninja-adb21.php:215-222` | `blacklisted_devices` / `blacklisted_asns` stay `[]` and the risk threshold stays `33` — the live default anyway |
| `fraud_report.json` | report | `generate_fraud_report.php` (step 4) | nothing in the request path; humans / dashboard | no nightly report has been produced |
| `funnel_cache.json` | cache | `generate_funnel_cache.php:452-456` (atomic tmp + `rename`) | `ninja-adb21.php:231-233`; `finalize_pending_installs.php:142-149` | `$GLOBALS['funnel'] = []`; funnel-domain and campaign signals are simply absent (shadow-only impact). Also arms the "missing file" branch of the regen trigger (`ninja-adb21.php:3827`) |
| `last_audit_run.txt` | **SUCCESS** (6 h TTL) | `generate_fraud_report.php:349` | `ninja-adb21.php:3786` | the fraud report is due; a *stale* file is the loud signal that it is silently failing |
| `last_audit_spawn.txt` | **ATTEMPT** (1 h) | `ninja-adb21.php:3788` | `ninja-adb21.php:3787` | no spawn attempted in the last hour |
| `last_finalize_run.txt` | **ATTEMPT-ish** — touched when a run has STARTED *and* reached a working DB connection | `finalize_pending_installs.php:126` | `ninja-adb21.php:3808`; `finalize_pending_installs.php:91-93` | age = `PHP_INT_MAX` → the inline trigger fires on the very next request. A fresh file means "runs are starting", **not** "work is getting done" |
| `last_finalize_success.txt` | **SUCCESS** — touched only on a full drain with nothing left behind | `finalize_pending_installs.php:340` | nothing in code; operator health check | no sweep has ever fully drained. Age > ~1 h with traffic flowing = the drain is not keeping up |
| `finalize_backlog.txt` | breadcrumb | touched `finalize_pending_installs.php:334` when the slice budget was hit or the LIMITed SELECT came back full; `unlink`ed `:339` on a full drain | `ninja-adb21.php:3809` | the last sweep drained to empty — the trigger falls back to its 1200 s cadence. Present ⇒ re-fire every 60 s |
| `finalize_pending_installs.lock` | `flock` (`LOCK_EX\|LOCK_NB`) | `finalize_pending_installs.php:97-101` | itself | no run has ever taken the lock. Single-flights cron, web, and inline attempts alike |
| `last_funnel_run.txt` | **SUCCESS** | `generate_funnel_cache.php:464`, after the cache write | `ninja-adb21.php:3826` | the 01:00 GMT+4 boundary check treats the funnel as due |
| `last_funnel_spawn.txt` | **ATTEMPT** (1800 s) | `ninja-adb21.php:3830` | `ninja-adb21.php:3829` | no funnel spawn attempted recently |
| `last_finalize_spawn.txt` | **legacy** | nothing — `bg_run` was dropped for finalize (`ninja-adb21.php:3804-3805`) | nothing | harmless; the file may linger on the server |
| `compiled_rules_version.txt` | **SUCCESS** (24 h TTL) | `generate_compiled_rules.php:259` | `ninja-adb21.php:135,691,709,3847,3851` | rules regen is due; also emitted as `ninja_rules_version` in the degraded DB-down response |
| `compiled_rules_cache.json` | cache | `generate_compiled_rules.php` | served via `compiled_rules.php` | regen is due (`ninja-adb21.php:3849`) |
| `compiled_rules_cache.json.lock` | `flock` | `ninja-adb21.php:3856-3863` | itself | no inline rules regen in flight |
| `last_rules_regen_attempt.txt` | **ATTEMPT** (1800 s) | `ninja-adb21.php:3854` | `ninja-adb21.php:3852` | no regen attempt in the last 30 min |
| `cosmetic_css_version.txt` | **SUCCESS** | `generate_cosmetic_rules.php:60` | `ninja-adb21.php:133,698,3873,3877` | generic-CSS half is due |
| `cosmetic_specific_version.txt` | **SUCCESS** | `generate_cosmetic_rules.php:148` | `ninja-adb21.php:134,703,3874,3878` | specific half is due |
| `cosmetic_css_cache.css`, `cosmetic_specific_cache.json` | caches | `generate_cosmetic_rules.php` | served from the CDN; presence checked `ninja-adb21.php:3880-3882` | regen is due |
| `cosmetic_specific_cache.json.lock` | `flock` | `ninja-adb21.php:3887-3894` | itself | no inline cosmetic regen in flight |
| `last_cosmetic_regen_attempt.txt` | **ATTEMPT** (1800 s) | `ninja-adb21.php:3885` | `ninja-adb21.php:3883` | no cosmetic regen attempt recently |

> **Note** — The only two files worth checking to answer "is the money path healthy?" are `last_finalize_success.txt` (should be fresh) and `finalize_backlog.txt` (should be absent). `last_finalize_run.txt` proves only that runs *start*.

### 13.6 config-hostinger.php

`config-hostinger.php` is a 76-line alternate deployment config. A line-by-line comparison against `config.php` shows exactly two substantive differences:

| Aspect | `config.php` (live) | `config-hostinger.php` |
|---|---|---|
| `$ninja_db_config` | Infomaniak (`7k10c.myd.infomaniak.com` / `7k10c_ninjablock`) active; the Hostinger block is commented out at `:11-19` | Hostinger (`localhost` / `u382419048_ninjablock25`) active; **no Infomaniak block at all** |
| `$HAS_FRAUD_COLUMNS` | `true`, declared at `config.php:36` | **the variable is absent entirely** |
| `NINJA_PDO_INIT_CMD`, `$aes_key`, `$proxycheck_key`, `$token_7thsense`, the three behavior flags, `$BEHAVIOR_AN_ALLOWLIST`, all three `$STAGE1_DELAY_*`, `$S1_HOLD_PAY_WINDOW_SEC` | see §13.1 | **identical values, identical comments** |

Correcting a common expectation: deploying `config-hostinger.php` **does not set `$HAS_FRAUD_COLUMNS` to `false`**. The endpoint reads it defensively —

```php
$GLOBALS['has_fraud_columns'] = (isset($HAS_FRAUD_COLUMNS) ? (bool)$HAS_FRAUD_COLUMNS : true);
```
`ninja-adb21.php:101`

— so an undefined variable falls back to **`true`**, and the fraud columns stay in the INSERT. The real hazard is the mirror image of the usual warning: if the Hostinger database is *not* migrated, that `true` makes `insertUserRecord()` build an INSERT naming `device_hash`, `fraud_score`, and `fraud_flags` against a table that lacks them, and every install INSERT throws. The uncaught-exception handler at `ninja-adb21.php:200-205` would convert each of those into a degraded HTTP 200 with a jittered back-off, so the failure would show up as an `[ninja-adb21 uncaught]` line in the PHP error log and zero new rows — not as an outage.

> **Note** — `ninja_run_inline()` declares `$HAS_FRAUD_COLUMNS` in its `global` list (`ninja-adb21.php:3741`) before re-requiring the config. Under `config-hostinger.php` that global is simply never assigned; `finalize_pending_installs.php` does not read it, so inline finalize is unaffected either way.

### 13.7 Security notes

Factual, current state of the non-endpoint files in the deployment root:

| File | Guard | Capability |
|---|---|---|
| `dashboard_detection.php` | `define('DASH_KEY', 'testonly')`, `hash_equals` against `?key=` (`:13-14`) | read-only analytics over the production table |
| `replay_v4.php` | `const REPLAY_TOKEN = 'testonly'`, `hash_equals` against `?token=` (`:21`, `:34-37`) | read-only classifier replay |
| `analyze_pa_quality.php` | `define('ACCESS_KEY', 'testonly')`, `hash_equals` against `?key=`; CLI runs exempt (`:67-75`) | read-only heavy cohort queries against production |
| `export_pa_flagged.php` | `define('ACCESS_KEY', 'testonly')`, `hash_equals` against `?key=`; CLI runs exempt (`:67-74`) | read-only CSV/JSON export of user-cohort rows |
| `rerun-vm.php` | **none** — the file contains no `$_GET` check and no token constant | **not read-only**: `UPDATE {$table} SET is_vm = :is_vm, vm_type = :vm_type, diagnostic_date = NOW() WHERE id = :id` (`rerun-vm.php:301`), executed on any HTTP hit. It also runs with `ini_set('display_errors', 1)` (`:10`) and `die("DB Connection Failed: " . $e->getMessage())` (`:24`), which leaks connection details on failure. |
| `sanitize_fast_uninstalls.php` | no token; requires `?run=dry` or `?run=live` and refuses otherwise (`:40-45`) | `?run=live` writes `conv=4` + `\|conv4_fast_uninstall` |
| `rescue_conv_20260715.php` | one-shot backfill script | writes `conv` and `\|rescue0715`; fires real postbacks |

Four analysis tools ship with the same literal token, `'testonly'`, which each file's own comment flags as a placeholder to change before deploying (`replay_v4.php:12`, `analyze_pa_quality.php:66`). `rerun-vm.php` has no guard at all and mutates detection columns on the live table.

---

## 14. State vocabulary

Four columns carry the system's state: `conv` (money), `decision` (behavioral fact), `fraud_flags` (append-only markers, stable), `behavior_flags` (classifier tokens, overwritten on every evaluation). They are deliberately independent axes — a paid install can be a bot, and a refused install can be human.

### 14.1 `conv` — the payout state machine

| Value | Meaning | Written by | Reference |
|---|---|---|---|
| `0` | **Unattributed, or in-flight.** No ad network, no `cid`, or a held `\|s1_hold` row whose deadline has not passed | install path default (`$conv = 0`) | `ninja-adb21.php:3312`; `processConversion` early returns `behavior_lib.php:796-804` |
| `1` | **Paid.** `processConversion` got a payout-API green light and `sendConversion` received HTTP 200 from the network postback | `processConversion` return | `behavior_lib.php:834-836` |
| `2` | **Refused, or postback failed.** *Dual meaning, pre-existing.* (a) the install-time risk gate condemned the row before any postback: `evaluateRisk` set `conv = 2` for `level` digits 3/4/7/8/9 and `createUser` copied it; (b) `processConversion` ran but the network postback did **not** return 200 | (a) `ninja-adb21.php:3313-3315`; (b) `behavior_lib.php:837-838` | (c) *retired 2026-09-03, builds 25+26:* the dormant Stage-2 hard block and the dormant bot block now write `'4'` under the conv-attribution convention below (`ninja-adb21.php:3302`; unpaid-bot branch ninja `behavior_lib.php:595-598`, Ghost `:594-597` — synced 2026-09-03, uploads pending). **All DORMANT** |
| `3` | **Paid, alternate payout code.** The 7thsense payouts API returned `3`; `processConversion` short-circuits and returns it verbatim without calling `sendConversion` | `behavior_lib.php:841-842` | treated as paid everywhere (`conv NOT IN ('1','3')` guards) |
| `4` | **Terminal, never paid.** A row that closed without ever reaching a successful `processConversion` | forfeit (canonical: `finalize_pending_installs.php:231`); hygiene (`:383`); refuse (`:279`, **DORMANT**); expiry (`:569`, **NOT ACTIVE TODAY**); `sanitize_fast_uninstalls.php:115` (manual); builds 25+26 dormant fraud-closures per the attribution convention: Stage-2 tier3 (`ninja-adb21.php:3302`), bot block (ninja `behavior_lib.php:595-598`, Ghost `:594-597`), early_fraud enforcement (`ninja-adb21.php:3344`, **COMMENTED**) | — |

**The conv-attribution convention (2026-09-03, builds 25+26 — synced same day; uploads pending).** `conv = '2'` is reserved for `evaluateRisk` (level) blocks alone; every fraud-detection, behavior, forfeit or expiry closure writes `conv = '4'`, with the `fraud_flags` tokens carrying which mechanism closed the row (`|early_fraud:` = blocked at the door — the only conv=4 with no closure token; `|s1_forfeit` = held then lost; `|conv4_expired_unpaid` = hygiene; `|conv4_fast_uninstall` = sanitize; `|behavior_bot:` = behavior layer). Reading a row therefore takes two steps: `conv` says *which family* closed it, the tokens say *exactly why*.

**The forward-only rule (2026-07-13).** An *attributed* install may never REST at `conv=0`. `0` means "unattributed" or "in-flight" and nothing else. Any row with `an IS NOT NULL AND cid IS NOT NULL` still at `conv` 0/NULL past 24 h is closed as `conv=4` with `|conv4_expired_unpaid` (`finalize_pending_installs.php:383-396`). The rule is **forward-only by explicit decision**: the sweep carries `AND created_date >= '2026-07-13 09:00:00'` (`finalize_pending_installs.php:376,389`), so rows older than that cutoff keep their historical `conv` values untouched, forever. No backfill was ever run for them.

> *Why forward-only: the pre-cutoff population mixes several retired gate designs; rewriting their conv would destroy the ability to compare cohorts across eras.*

Every write is guarded so money moves at most once: the pay claim uses `WHERE ... AND (conv = '0' OR conv IS NULL) AND fraud_flags LIKE '%|s1_hold%'` and proceeds only on `rowCount() === 1` (`finalize_pending_installs.php:300-307`); the deferred-convert path uses `WHERE (conv NOT IN ('1','3') OR conv IS NULL)` (canonical: `finalize_pending_installs.php:538` — the `OR conv IS NULL` was added 2026-09-02 because `NULL NOT IN ('1','3')` is NULL, not true, so a NULL-`conv` row fired the postback yet never recorded it); `runBehaviorLayer` pays only `conv === 0` (`behavior_lib.php:537`) **and** its pay UPDATE re-checks the unpaid state in SQL, `AND (conv = '0' OR conv IS NULL)` (canonical: `behavior_lib.php:556`, the 2026-09-02 race guard — parity with the S1 payout write).

**The two conv-write rules — canonical since 2026-09-02, binding on every future write:**

1. **Every `conv` write maintains `conv_date`.** At INSERT nothing writes it — the column `DEFAULT` fires in the same statement as `created_date`'s (`NOW()` is per-statement), so `conv_date == created_date` exactly, covering pay-at-install and delayed conversion alike with no config gating (§6.6, §14.6). On UPDATE: plain `conv_date = NOW()` where the `WHERE` guard guarantees a real transition (canonical: `finalize_pending_installs.php:232/280/384/570`); `SET conv_date = IF(conv <=> ?, conv_date, NOW()), conv = ?` where a same-value rewrite is possible (canonical: `finalize_pending_installs.php:315/538`, `behavior_lib.php:556/594`) — NULL-safe compare, and `conv_date` assigned **before** `conv`, because UPDATE SET evaluates left-to-right. A same-value rewrite never moves `conv_date`: the column always answers *which code path set this `conv`, and when*.
2. **SQL `conv` literals are always quoted strings.** `conv` is `enum('0','1','2','3','4')`, and MariaDB/MySQL reads an unquoted number assigned or compared to an ENUM as the enum **INDEX** — index 2 = value `'1'` (PAID), `IN (1,3)` matches `'0'`,`'2'`. This trap shipped real damage (the pre-fix bot block and eight fraud-report comparisons — §4.5, §8.8); bound prepared-statement parameters are safe, bare numeric literals never are.

> **Note** — `level !== "0"` ⇒ the install is **NEVER** paid. This holds at every enforcement site without exception: the install gate (`ninja-adb21.php:3320`), the held-row sweep (`finalize_pending_installs.php:294-297`, where the row stays held and ages out to forfeit rather than paying), the deferred-convert branch (`finalize_pending_installs.php:524-525`), and `runBehaviorLayer` (`behavior_lib.php:541-542`). It is absolute.

### 14.2 `decision` — the behavioral fact

Four values, one enum, no schema change since v3.

| Value | Meaning | Terminal? | Written by |
|---|---|---|---|
| `pending` (or `NULL`) | Still inside the 3 h observation window, still collecting | no | `runBehaviorLayer` `behavior_lib.php:508`; the **DDL default is `'pending'`** (`DB_SCHEMA.md` §9.2/§9.3), so an install — which never writes `decision` — stores `'pending'`; legacy rows may hold `NULL` and read-side guards treat the two identically |
| `human` | Classifier scored ≤ `human_line` (`-15`) **and** cleared the corroboration gate (≥ 2 independent human signals) | yes | `runBehaviorLayer` `behavior_lib.php:502`; finalize `:498-501` (which delegates to `runBehaviorLayer`) |
| `bot` | Classifier scored ≥ `bot_line` (`+30`), or the install was found uninstalled while still undecided | yes | `runBehaviorLayer` `behavior_lib.php:502`; finalize fast-uninstall branch `:470-475`; finalize behavioral-bot branch `:505-510` |
| `deferred` | **Terminal "unknown".** The classifier returned `unknown` and the row is past the 3 h window (or was already terminal-deferred) | yes | `runBehaviorLayer` `behavior_lib.php:504,506`; finalize `:515-520`; expiry sweep `:569` (**NOT ACTIVE TODAY**) |

The `'deferred' = terminal-unknown` convention: `classifyBehaviorV4` returns `human | bot | unknown` (`classifier_v4.php:126`); `evaluateBehavior` maps `unknown → deferred` at `behavior_lib.php:431` purely so every existing caller keeps its vocabulary. This reuses the pre-existing enum value and required **zero schema change**. The distinction between transient and terminal unknown is carried entirely by age:

```php
$pastWindow = $ageSec > BEHAVIOR_WINDOW_SEC;   // behavior_lib.php:499
if ($decision === 'human' || $decision === 'bot') $dbDecision = $decision;      // terminal
elseif ($install['decision'] === 'deferred')      $dbDecision = 'deferred';     // stays terminal
elseif ($pastWindow)                              $dbDecision = 'deferred';     // forced resolution
else                                              $dbDecision = 'pending';      // keep collecting
```
`behavior_lib.php:501-509`

Two consequences worth stating plainly:

- **`runBehaviorLayer` terminalizes `human`/`bot` at ANY age**, not only past 3 h. The `$pastWindow` test applies exclusively to the unknown case. A first sync at minute 4 that scores ≥ 30 writes a terminal `bot` immediately.
- A terminal verdict stamps `decided_date` (`behavior_lib.php:520-523`), and `logSyncArrival` refuses to write further `sync_log` rows once `decided_date` is set (`ninja-adb21.php:2519`, condition at `:2514`). **Terminalizing an install ends its behavioral data collection.** An early bot verdict is self-sealing: no later sync can contradict it, because no later sync is recorded.

`runBehaviorLayer` also returns immediately for an already-`human`/`bot` row (`behavior_lib.php:456-458`), so terminal `human`/`bot` never reopens. Terminal `deferred` never reopens either (`behavior_lib.php:503-504`).

### 14.3 `fraud_flags` — the complete marker vocabulary

`fraud_flags` is a single `|`-joined string. The **first token** is always the install-time tier label; everything after it is an appended marker — the lifecycle markers below plus, since 2026-09-03, the install-time `|early_fraud:` detection marker written inside the INSERT itself. Markers are substring-safe by construction: `'|s1_hold'` is not a substring of `'|s1_paid'` or `'|s1_forfeit'`, which is what makes the `REPLACE()`-based atomic claims correct.

**Tier labels (position 1, written once at INSERT):**

| Exact string | Written by | Read by | Transitions to |
|---|---|---|---|
| `tier1:high_trust` | `ninja-adb21.php:3202` | `deriveTierFromRow` `classifier_lib.php:164`; `dashboard_detection.php:72` | nothing — immutable |
| `tier2:` + `\|`-joined subset of `borderline_vm:<vm_type>`, `vpn`, `proxy_flag:<risk>`, `banned_provider`, `linux_ua`, `minor_risk_score:<risk>`, `device_collisions:<n>` | `ninja-adb21.php:3192-3199` | `deriveTierFromRow` `classifier_lib.php:163`; `dashboard_detection.php:72,148` | nothing — immutable |
| `tier3:` + `\|`-joined subset of `hard_vm:<vm_type>`, `hard_proxy_type:<type>`, `extid_mismatch`, `blacklisted_device`, `blacklisted_asn` | `ninja-adb21.php:3163-3169` | `deriveTierFromRow` `classifier_lib.php:162`; `dashboard_detection.php:44,72` | nothing — immutable |

> **Note** — Tier reasons are themselves `|`-joined, and `borderline_vm:<vm_type>` embeds a `vm_type` that may contain its own `|` (e.g. `physical:old_laptop|reducer_touch`, `ninja-adb21.php:1068`). Splitting `fraud_flags` on `|` therefore does **not** yield a clean marker list. Every live consumer uses `LIKE '%marker%'` substring matching instead.

**Lifecycle markers (appended after the tier label):**

| Exact string | Written by | Read by | Transitions to |
|---|---|---|---|
| `\|s1_hold` | `ninja-adb21.php:3414` (appended at INSERT when `$s1DelayHold`, set at `:3331`) | `finalize_pending_installs.php:200,234,283,303`; hygiene exclusion `:391`; `sanitize_fast_uninstalls.php:81,91`; `_index_diagnostics.php:82` | `\|s1_paid`, `\|s1_forfeit`, or `\|s1_refused` |
| `\|s1_paid` | `finalize_pending_installs.php:301` — `REPLACE(fraud_flags, '|s1_hold', '|s1_paid')`, the atomic claim; the row proceeds to `processConversion` only on `rowCount() === 1` | audit / dashboard | terminal. If the process dies between claim and postback, the row rests at `\|s1_paid` with `conv=0` — at-most-once is preferred to at-least-once |
| `\|s1_forfeit` | `finalize_pending_installs.php:231` — `REPLACE('|s1_hold','|s1_forfeit')` plus `conv='4'` + `conv_date=NOW()` (canonical 2026-09-02). Fired when `disabled` is set (uninstalled) **or** age > `$S1_HOLD_PAY_WINDOW_SEC` (72 h) | audit / dashboard; `rescue_conv_20260715.php:128` treats it as rescuable | terminal |
| `\|s1_refused` | `finalize_pending_installs.php:279` — `CONCAT(REPLACE('|s1_hold','|s1_refused'), ?)` plus `conv='4'` + `conv_date=NOW()` (canonical 2026-09-02). **DORMANT**: the enclosing branch requires `an_in_behavior_scope()` (`:255`), which is `false` for every network while `$BEHAVIOR_AN_ALLOWLIST = []` | audit / dashboard | terminal |
| `\|t1_bot_refused`, `\|t2_bot_refused`, `\|t3_bot_refused` | appended by `finalize_pending_installs.php:284` from `mergeConvV4` `classifier_v4.php:165`. **DORMANT** | audit | terminal (accompanies `\|s1_refused`) |
| `\|t3_human_no_proof_refused` | `classifier_v4.php:174` via `finalize_pending_installs.php:284`. **DORMANT** | audit | terminal |
| `\|t3_unknown_refused` (generic form `t3_<decision>_refused`) | `classifier_v4.php:174`. **DORMANT** | audit | terminal |
| `\|merge_refused` | `finalize_pending_installs.php:284` fallback when `mergeConvV4` returns no `reason`. **DORMANT** — unreachable in practice, every `mergeConvV4` return path sets one | audit | terminal |
| `\|early_fraud:<reasons>` | shadow detection block inside the INSERT (`ninja-adb21.php:3269-3277`) — reasons `tzdc`/`fpvote`/`ipstack`, `+`-joined. **Immutable**: nothing rewrites it; consumers match `LIKE '%early_fraud%'`. Enforcement pre-wired but COMMENTED (grep `EARLY_FRAUD_FLIP`) | audit; future T0 enforcement gate | permanent record (SHADOW — not terminal by itself) |
| `\|conv4_expired_unpaid` | `finalize_pending_installs.php:386` — bulk `CONCAT(COALESCE(fraud_flags,''), '|conv4_expired_unpaid')` + `conv='4'` + `conv_date=NOW()` (canonical 2026-09-02) for attributed rows still at conv 0/NULL past 24 h, created ≥ `$CONV4_HYGIENE_FROM`, and not an in-flight `\|s1_hold` | audit; `rescue_conv_20260715.php:128` | terminal |
| `\|behavior_bot:<flags>` | `behavior_lib.php:573-576` — comma-joined classifier flags, appended for an unpaid behavioral bot along with `conv='4'` + IF-guarded `conv_date` (**quoted**, compare `'4'` — enum rule + 2026-09-03 convention, §14.1; ninja SQL `behavior_lib.php:595-598`, Ghost `:594-597` — synced 2026-09-03, uploads pending), `flagged=CURDATE()`, `device_hash`. **DORMANT** (requires `$BEHAVIOR_ENFORCE`) | chargeback report | terminal |
| `\|behavior_bot_post_pay:<flags>` | `behavior_lib.php:573` — same, but for an install already at `conv` 1 or 3. **`conv` is never touched**; only evidence is recorded. **DORMANT** | post-hoc chargeback report | terminal |
| `\|conv4_fast_uninstall` | `sanitize_fast_uninstalls.php:116` — manual one-shot for attributed, never-paid rows uninstalled the same day they were created. Idempotent: the marker is excluded from its own scope (`:80`) | `rescue_conv_20260715.php` treats it as **not** rescuable | terminal |
| `\|rescue0715` | `rescue_conv_20260715.php:148` — manual one-shot atomic claim marker for the 2026-07-15 outage backfill | `rescue_conv_20260715.php:97,150` (re-run guard) | terminal |

**Legacy markers present in historical rows only — no live writer:**

| Exact string | Origin | Status |
|---|---|---|
| `\|timeout_tier2` | the previous endpoint generation, `ninja-adb.php:2558` / `ninja-adb-dev.php:2541`, when a Tier-2 install expired at `age > 3h` or `updates >= 6` | **`ninja-adb21.php` never writes it.** The legacy Tier-2 sync deferral is explicitly retired (`ninja-adb21.php:2573-2576`). Rows carrying it predate the current endpoint |
| `\|audit_proxy_type`, `\|audit_device_clone_cluster` (and other `audit_*` reasons) | the removed Weight-of-Evidence batch-scoring stage of `generate_fraud_report.php` | **No writer exists in the current tree.** The WOE model and per-row `fraud_score` were removed 2026-07-13 (`generate_fraud_report.php:180-181,248-249`) |

### 14.4 `behavior_flags` — the classifier's token vocabulary

Separator: a single `|`. Written as `implode('|', $flags)` (`behavior_lib.php:511`, `finalize_pending_installs.php:504,514`). **Overwritten in full on every evaluation** — never append lifecycle state here; that is what `fraud_flags` is for (`ninja-adb21.php:3407-3409`).

Tokens emitted by `classifyBehaviorV4`, with the score contribution that accompanies each:

| Token | Weight | Emitted when | Line |
|---|---|---|---|
| `farm_collision` | `+50` | another install shares the same `behavior_hash` within ±30 min | `classifier_v4.php:68` |
| `fast_uninstall` | `+30` | `disabled` is set | `classifier_v4.php:69` |
| `zero_organic` | `+20` | `n_syncs > 0` **and** `organic_core_size === 0` | `classifier_v4.php:71` |
| `metronome` | `+25` | `n_syncs >= 6` **and** `interval_cv < 0.15` **and** `zero_activity_ratio > 0.8` | `classifier_v4.php:73-78` |
| `funnel_only` | `+20` | every install-tab domain is a campaign funnel domain | `classifier_v4.php:80` |
| `auth_login` | `-40` | an organic-core domain matches `AUTH_MARKERS` | `classifier_v4.php:83` |
| `popup_interaction` | `-25` | `whitelistedDom` or `blockedDom` is genuinely non-empty (not `''`, `a:0:{}`, `N;`, `null`, `[]`) | `classifier_v4.php:84`; feature at `behavior_lib.php:307-311` |
| `gov_edu_corp` | `-35` | an organic-core domain matches `/\.(go\.id\|ac\.id\|gov\|edu)$\|instructure\.com\|okta\.com\|sharepoint/i` | `classifier_v4.php:85`; feature at `behavior_lib.php:297` |
| `organic_domains:<n>` | `max(-40, -5 × n)` | `organic_core_size > 0`; `<n>` is the literal count | `classifier_v4.php:87-89` |
| `organic_revisit` | `-20` | `has_revisit` **and** `organic_core_size >= 2` | `classifier_v4.php:90-93` |
| `no_evidence` | `0` | `n_syncs === 0`, emitted **regardless of score** | `classifier_v4.php:101` |
| `human_gate_uncorroborated:<n>` | `0` (downgrades the verdict) | score reached `human` but fewer than `human_min_signals` (`2`) independent human signals were present; `<n>` is the count | `classifier_v4.php:120-123` |

Two tokens are written outside the classifier, by finalize:

| Token | Written by | Note |
|---|---|---|
| `finalize_fast_uninstall_bot` | `finalize_pending_installs.php:468-469` — **appended** to the existing `behavior_flags` (the one place that appends rather than replaces) | Uninstalled + still undecided ⇒ terminal bot, no classification run |
| `finalize_expired_unpaid` | `finalize_pending_installs.php:570` — `CONCAT(COALESCE(behavior_flags,''), BINARY '\|finalize_expired_unpaid')` | **NOT ACTIVE TODAY** — inside the `if (!$pureShadow)` block |

Decision thresholds: `bot_line = 30`, `human_line = -15`, `human_min_signals = 2`, `human_breadth_min = 3` (`classifier_v4.php:35-42`). Score is `round($S, 2)` into `behavior_score`.

### 14.5 Lifecycle of a held install

The live path today — global 1 h Stage-1 delay, Stage 2 off:

```
                      INSTALL (createUser)
                             |
             evaluateRisk -> level, conv-gate
             evaluateVirtualMachine -> is_vm, vm_type
             tier layer -> fraud_flags = "tierN:<reasons>"
                             |
              an? cid? level==="0"? conv!==2 ?
                   |                        \
                  yes                        no
                   |                          \
        s1_delay_in_scope($an)              conv stays 0 or 2
        ['*'] -> TRUE                       (never held, never paid)
                   |
       INSERT: conv=0, fraud_flags="tierN:...|s1_hold"
       decision='pending' (DDL default; column omitted), decided_date=NULL, ninja_sync=600000
                             |
             .... extension syncs every 10 min ....
             logSyncArrival writes sync_log (cap 20, <=3h,
             stops the moment decided_date is set)
             runBehaviorLayer scores every sync:
               human/bot at ANY age -> TERMINAL + decided_date
               unknown & <3h        -> decision='pending'
               unknown & >3h        -> decision='deferred'
             ** no money moves here: all behavior flags false **
                             |
        ~1h later, an inbound request runs finalize INLINE
        (post-flush, 12s budget; no cron on this host)
                             |
        SELECT ... WHERE fraud_flags LIKE '%|s1_hold%'
                   AND conv IN (0,NULL)
                   AND age > 3600s AND age < 345600s
                             |
        +--------------------+---------------------------+
        |                    |                           |
   disabled set          age > 72h                  still installed
   (farewell.php)     (pay window gone)              age >= 3600s
        |                    |                           |
        +---------+----------+                           |
                  |                              level still "0" ?
      |s1_hold -> |s1_forfeit                      /            \
             conv = 4                            no             yes
           TERMINAL, never paid                   |               |
                                            stays held      atomic claim
                                            (ages out       |s1_hold ->
                                             to forfeit)     |s1_paid
                                                                 |
                                                    processConversion(an,cid,sid,0)
                                                                 |
                                                 conv = 1 (200) | 2 (non-200) | 3
                                                             TERMINAL

  [ DORMANT branch, between "age >= 3600s" and the level check ]
      if an_in_behavior_scope($an)  -- FALSE today, allowlist is []
          decision still pending? -> stay held, retry next slice
          mergeConvV4(tier, decision, proof)
              refuse -> |s1_hold -> |s1_refused + |tN_<reason>, conv=4
              pay    -> fall through to the level check above

  [ UNGATED, runs every full slice ]
      conv-hygiene: attributed rows at conv 0/NULL past 24h,
      created >= 2026-07-13 09:00:00, not an in-flight |s1_hold
          -> conv=4, append |conv4_expired_unpaid

  [ NOT ACTIVE TODAY -- inside if(!$pureShadow) ]
      straggler expiry: decision pending/NULL + conv 0/NULL past 24h
          -> conv=4, decision='deferred', |finalize_expired_unpaid
```

> **Canonical (2026-09-02):** every `conv` transition in the diagram also stamps `conv_date` — the INSERT via column DEFAULT (== `created_date`), every later write per the §14.1 rule. A same-value rewrite never moves it, so `conv_date` always dates the arrow that actually fired.

### 14.6 Other detection-relevant columns

| Column | Written by | When |
|---|---|---|
| `is_vm` | `createUser` INSERT from `$vmResult['is_vm']` (`ninja-adb21.php:3394`); re-diagnosis UPDATEs at `ninja-adb21.php:2726`, `:2843`, `:2991`, `:3524`, and inside the fingerprint-migration UPDATEs `:2801-2808`, `:2949-2956`, `:3479-3487`; `rerun-vm.php:301` (unguarded batch tool) | at install; on a later sync when `diagnostic_date` is empty (or, on the dup-CID/dup-IP paths, older than `2026-05-25 00:00:00`) |
| `vm_type` | same sites as `is_vm` | same. Values are `physical`, `physical:<reason>[\|<reason>…]` (Tier-2 borderline), or a hard VM tag |
| `device_hash` | `createUser` INSERT (`ninja-adb21.php:3404`), computed by `generateDeviceHash`; `runBehaviorLayer` bot branch (`behavior_lib.php:582,597`, **DORMANT**); `generate_fraud_report.php:114` backfills NULL rows | at install; nightly backfill. v2 = 13 components joined with `\|\|\|`, SHA-256 (`behavior_lib.php:200-216`) — the definition is duplicated in three files and must stay identical |
| `canvas_hash` | `createUser` INSERT only (`ninja-adb21.php:3391`) | at install; never updated |
| `behavior_score` | `runBehaviorLayer` (`behavior_lib.php:515-526`); finalize bot branch (`:506`) and deferred branch (`:516`) | on every sync while undecided, and at finalization. Overwritten each time |
| `behavior_flags` | same three sites, plus the append at `finalize_pending_installs.php:469` and the dormant expiry `CONCAT` at `:570` | overwritten each evaluation — see §14.4 |
| `behavior_hash` | same three sites; value is `sha256(implode('\|', $organicCore))`, or `NULL` when `organic_core_size < 3` (`behavior_lib.php:315-319`) | on every evaluation. Feeds the `farm_collision` self-join (`behavior_lib.php:355-361`) |
| `decided_date` | set to `NOW()` **only if currently NULL** at every terminal write: `behavior_lib.php:520-523`, `finalize_pending_installs.php:473,508,518,571` | first terminal verdict. **Its presence stops `logSyncArrival` from writing any further `sync_log` rows** (`ninja-adb21.php:2519,2514`) |
| `diagnostic_date` | `NOW()` in the INSERT (`ninja-adb21.php:1775,1784`) and in every VM re-diagnosis UPDATE (`:2726`, `:2808`, `:2843`, `:2956`, `:2991`, `:3487`, `:3524`); `rerun-vm.php:301` | at install and at each VM re-evaluation. Gates whether a re-diagnosis runs at all |
| `disabled` | `farewell.php:78` (by `fingerprint`) or `farewell.php:85` (legacy encrypted UID, by `id`) — value is `date('Y-m-d')` | when the user uninstalls and the farewell page is reached. Read as the `fast_uninstall` feature (`behavior_lib.php:414`) and as the forfeit trigger (`finalize_pending_installs.php:229`) |
| `flagged` | `runBehaviorLayer` bot branch, `CURDATE()` (`behavior_lib.php:582,597`) — **DORMANT**. `updateFlaggedStatus()` exists at `ninja-adb21.php:1839-1843` but **has no callers anywhere in the tree** | never, in the live configuration |
| `updates` | `updateUserRecord` (`ninja-adb21.php:2482`), from a counter read out of `processUserLogic` (`:2663`) and incremented at `:2743`, `:2863`, `:3011`, `:3541` | once per sync. Compared against `BEHAVIOR_SYNC_CAP` (20) to cap `sync_log` growth (`ninja-adb21.php:2521`) |
| `duplicate` | `createUser` INSERT (`ninja-adb21.php:3380`); incremented and UPDATEd on the duplicate-fingerprint / duplicate-CID-SID / duplicate-IP re-entry paths (`ninja-adb21.php:2744` + `:2755`, `:2874`, `:3025`) | when an existing row is matched by fingerprint, `cid`+`sid`, or IP-and-network heuristics instead of creating a new install |
| `fraud_score` | `createUser` INSERT, **hardcoded `null`** (`ninja-adb21.php:3405`), passed through `insertUserRecord` (`:1814`) | **ALWAYS NULL for every row created since 2026-07-13.** The Weight-of-Evidence scoring loop that used to populate it was removed from `generate_fraud_report.php` (`:180-181`, `:248-249`) because nothing consumed it for any decision. Pre-cutoff rows keep their frozen historical values. No writer exists in the current tree |
| `fraud_checked` (`datetime`) | **no writer in the current tree** | **Inert — frozen at its last value.** It paired with `fraud_score` as the timestamp of the last Weight-of-Evidence pass; when that scoring loop was removed (2026-07-13, the same change that froze `fraud_score`) its writer went with it. Retained in the schema (`DB_SCHEMA.md` §9) so pre-cutoff rows keep their historical stamp; the detection changelog records both columns as inert. Do **not** wire new logic onto it |
| `conv_date` (`datetime`, sits directly after `conv` — **canonical 2026-09-02**, `migrate_conv_date_updated_at.sql`) | Nothing at INSERT — the column `DEFAULT current_timestamp()` fires in `insertUserRecord`'s statement (`ext-server.php:2240`), so `conv_date == created_date` exactly. On UPDATE, **every** `conv` writer maintains it per the §14.1 rule: plain `NOW()` at the real-transition-guarded sites (canonical: `finalize_pending_installs.php:232/280/384/570`), `IF(conv <=> ?, conv_date, NOW())` assigned before `conv` at the four same-value-capable sites (canonical: `finalize_pending_installs.php:315/538`, `behavior_lib.php:556/594`) | at install (== `created_date`), then whenever `conv` genuinely changes value — a same-value rewrite never moves it. Pre-migration rows backfilled to `created_date`. **No index — deliberate**: never filtered by live queries |
| `updated_at` (`datetime`, the LAST column — **canonical 2026-09-02**, same migration) | **No code writes it, ever** — maintained entirely by the DB (`DEFAULT current_timestamp() ON UPDATE current_timestamp()`). The migration backfill's explicit assignment (to `created_date`) overrides the ON UPDATE clause, by design | on any UPDATE that changes at least one column value, in any file, present or future; a no-change UPDATE does **not** bump it. **Distinct from the heartbeat `updated`** — that column has no auto-update clause, is code-written on sync and feeds the 36h-survival metrics, untouched by this change. **No index — deliberate**: rewritten on every row touch, the same "harmful index" category as the dropped `updated` index |

---

## 15. Operations — deploy, promote, roll back, diagnose

> **Actor (`00_INDEX.md` §7): every operation in §15 is `[OWNER — manual]`.** Deploying,
> uploading `config.php`, running `php -l` / the classifier self-test on the host, promoting a
> network to Stage 2, and rolling back are owner/operator actions on the live fraud backend. The
> agent never performs them — it prepares a change and reasons about it offline, then hands the
> owner the exact ordered steps.

### 15.1 Deployment rules

| Rule | Why |
|---|---|
| `ninja-adb21.php` and `finalize_pending_installs.php` ship **as a pair** | the endpoint runs finalize in-process via `ninja_run_inline()`; a version skew between them breaks the money sweep silently |
| Back up before every upload, into `_backups/` | naming convention: `<file>.before-<change-slug>.bak`, or a dated directory `_backups/<slug>_<YYYY-MM-DD>/` for multi-file changes |
| `config.php` is **host-specific** — never copy it between hosts | see `config-hostinger.php` and §13.6 |
| Run `php -l` on every changed file before upload | the endpoint has no staging environment |
| Run `php classifier_v4.php` after touching the classifier | it carries a self-test (25 cases) that runs under CLI |

`$HAS_FRAUD_COLUMNS = true` records that the fraud/behaviour columns are already present on the
live database. The canonical schema additionally carries `conv_date` / `updated_at`
(2026-09-02, `migrate_conv_date_updated_at.sql`); porting a build onto it has a **hard deploy
order**: run the `ALTER` **before** uploading the PHP — the canonical
`finalize_pending_installs.php` / `behavior_lib.php` reference `conv_date` in their `conv`
UPDATEs and would throw "Unknown column" against an unmigrated DB. The migration is **RUN
ONCE ONLY**: its backfill (both columns to `created_date`) is not idempotent — re-running it
after the app has started stamping real transitions would clobber every legitimate
`conv_date`/`updated_at`.

### 15.2 Changing payout timing (Stage 1)

This is the safe, reversible knob. It changes **when** money moves, never **who** gets it.

To change the hold duration, edit `$STAGE1_DELAY_CONV_SEC` in `config.php`. To roll back to
pay-at-install, `config.php` carries a ready-made block: comment out the three live lines and
uncomment the three rollback lines directly beneath them (keep only one version live — they
redeclare the same variables).

Rolling back does **not** strand already-held rows: the finalize sweep selects by the `|s1_hold`
marker, not by the config list, so the existing backlog still drains. Setting `_SEC = 0` removes
the 1-hour floor so that drain is immediate.

> **Standing prerequisite while the hold is on.** Finalization must keep running. With a global
> hold it carries **100% of payout traffic** — if it stalls, held rows pile up unpaid and forfeit
> at `$S1_HOLD_PAY_WINDOW_SEC` (72 h). See §15.5.

### 15.3 Promoting one network to Stage 2

This is the money-moving change. It makes refusals possible for that network and nothing else.

1. Confirm the network's postback attribution window is longer than ~3.5 h. If it is not, stop —
   the 3-hour checkpoint will land outside it.
2. Set `$STAGE1_DELAY_CONV_SEC_AN = ['<an>' => 10800]`. **This step is mandatory and is the one
   that gets forgotten** — without it the row arrives at the 1-hour checkpoint still `pending`,
   is re-held, and pays at ~3 h 15 anyway, defeating the point (§10.7).
3. Add the network to `$BEHAVIOR_AN_ALLOWLIST` — exact string, no wildcards.
4. Leave `$BEHAVIOR_ENFORCE` / `$DEFER_CONVERT` / `$DELAYED_CONVERSION` at `false`. The V4-3 merge
   is gated on the allowlist alone, deliberately, to sidestep the flag-scoping asymmetry audited
   on 2026-07-12.
5. Upload `config.php` only. No code change is required.
6. Watch: `|s1_refused` reasons in `fraud_flags`, and the refused cohort's survival rate. A healthy
   canary refuses at roughly its non-uninstall bot rate.

**Rollback:** remove the network from `$BEHAVIOR_AN_ALLOWLIST`. Immediate, config-only. Rows
already refused keep their `conv = 4`; rows in flight fall back to the Stage-1 pay path.

### 15.4 Emergency levers

| Situation | Lever | Effect |
|---|---|---|
| A device or ASN must be blocked now | write it into `fraud_cache.json` (`devices` / `asns`) | immediate Tier 3 on the next request; the nightly generator keeps the lists empty otherwise, so a manual entry survives only until the next successful audit run |
| Payouts must stop immediately | `$STAGE1_DELAY_CONV_SEC` to a very large value | rows hold instead of paying; reversible, but they forfeit at 72 h if left |
| The endpoint must be taken off the air | `_EMERGENCY_htaccess.txt` | drops traffic at the web-server layer, before PHP |

### 15.5 Incident playbook

| Symptom | First check | Likely cause | Action |
|---|---|---|---|
| No postbacks firing | `last_finalize_run.txt` freshness | finalize is not running — on a cron-less host this means inbound traffic is not reaching the post-flush block | hit the endpoint manually; check `finalize_backlog.txt` (present = slices are stopping with work left) |
| Held rows aging toward 72 h | same | as above | repeated manual runs are safe — the atomic claim (§9.3) makes any number of partial slices idempotent |
| Every install suddenly looks clean | proxycheck reachability | `lookup_failed` **fails open** — `level` becomes `'0'` and everything is payable (§3.4) | treat as a money incident, not a detection one |
| Two behaviour signals stop firing | `funnel_cache.json` exists? | a missing funnel cache silently disables funnel-derived signals (§7.4) | the request path never regenerates it; the post-flush trigger does |
| HTTP 500s / retry storm | PHP error log for `[ninja-adb21 uncaught]` | a mid-request DB failure | the `set_exception_handler` should be converting these to a degraded 200 — if you see 500s, that handler is not installed |
| `decision` backlog past 24 h growing | expected | the straggler sweep is gated off in shadow (§9.7) | this is known and tracked as V4-5; the `conv` axis is still closed by the hygiene sweep |

> **Never read a spawn stamp as health.** `last_*_spawn.txt` records an *attempt*. During the
> 2026-07-16 → 07-21 outage the spawn stamp stayed fresh for five days while zero runs completed.
> Only the SUCCESS markers in §11.4 prove work happened.

### 15.6 What to check after any detection change

1. `php -l` clean on every changed file, and `php classifier_v4.php` self-test passing.
2. A live request returns HTTP 200 with a `ninja_sync` value and no PHP notice in the log.
3. New rows appear with a populated `fraud_flags` (tier label present) and `decision = 'pending'`.
4. `last_finalize_run.txt` advances within ~20 minutes of live traffic.
5. Postbacks resume — a held row transitions `|s1_hold` → `|s1_paid` and lands `conv = 1`.

---

## 16. Validation protocol and roadmap

> **Actor (`00_INDEX.md` §7): the live-data validation in §16 is `[OWNER — manual]`.** Running
> the server-side replay/backtest on the host, judging cohorts on the dashboard, and the
> enforcement gate are owner/operator actions on live data. The agent never runs them; it
> prepares the change and interprets the results the owner provides.

### 16.1 How a change to the classifier gets validated

1. **Replay** — `replay_v4.php` (read-only) re-labels the last N days under the candidate model
   and reports the v3→v4 transition matrix, ground truth per bucket (uninstall %, 24 h retention,
   36 h survival, paid %), a per-flag audit and a verdict banner.
   **Known replay biases, both optimistic on humans:**
   - `visit` is cumulative, so organic evidence includes post-3h browsing.
   - `sync_log` was truncated by early terminalization, so sync-derived signals are under-observed.
     This bias is *not* historical only — it still applies (§8.4).
2. **Forward** — judge the post-change cohort on the dashboard, same panels.
   > **Cohort cut (2026-07-10).** Tier labels stamped before the Phase-2 tier fix are polluted
   > (blacklist era; ~70% of installs mislabeled Tier 3, mostly residential-ISP humans). Any
   > tier-segmented analysis must filter `created_date >= 2026-07-10`. The behaviour-only decision
   > axis is unaffected and may use data from 2026-07-09.
3. **Enforcement gate** — before any network is promoted to Stage 2: verdict validated forward,
   "would-refuse-that-survived" acceptably low, and the invariant panel green — measured on the
   post-Phase-2 cohort only.

*Do not tune thresholds on replay.* Forward data hints the mortality cliff sits nearer 0 than
+30. The design goal is to move a line, not to add ten rules.

### 16.2 Roadmap

| Step | What | Status |
|---|---|---|
| V4-1 | `classifier_v4.php` + `replay_v4.php`, replay validated | done |
| V4-2 | behaviour-only classifier live in shadow; conv filter dropped from the resolution sweep | **done 2026-07-09 — this is what runs today** |
| V4-2b | ≥2 corroboration gate | done 2026-07-09 |
| T-fix 0/1/2 | tier correction: blacklist suspension · VM de-fang · tier demotions | done 2026-07-09/10 |
| V4-3 | `mergeConvV4` wired into the hold sweep, gated on the allowlist | **code shipped 2026-07-13; DORMANT since 2026-07-16** when the allowlist was emptied. §10 |
| V4-4 | sync-accumulation anti-spoof human signal | pending — needs uncut sync histories, which §8.4 currently prevents |
| V4-5 | backfill / backstop the >24 h `pending` backlog | pending — and now load-bearing, since the straggler sweep is gated off (§9.7, §12) |

### 16.3 Known open items

- **§8.4 is a prerequisite for V4-4.** The sync-accumulation signal needs evidence that grew across
  the window; early terminalization stops collection for exactly the confident rows.
- **The 1 h / 3 h timing mismatch** must be resolved per network before Stage 2 (§10.7).
- **Per-campaign `behavior_thresholds`** remain deliberately disconnected (§7.12). Reconnect only
  if forward data demands it.
- **Watch `calibrated_risk_threshold` on the first fraud-report run after the 2026-09-02
  quoted-enum fix.** Every conv-derived stat in `fraud_report.json` step-changes at that run
  (the numbers were wrong BEFORE the fix, not after), and the threshold — the only fraud-report
  output the live request path consumes — can move with the corrected cohorts (§3.5, §4.5). The
  `[25, 50]` clamp and `33` fallback bound it.
- **Dead code** flagged in this document — the old-Chrome branch (§2.6), `fraud_score` (§14.6),
  `audit_window_days` (§13.4) — is documented rather than removed, so that removal can be a
  deliberate, separately-verified change.

---

## 17. What changed from V4

`DETECTION_SPEC_V4.md` remains on disk for history. It is a sound design rationale and its §0–§2
arguments are carried forward here unchanged. What it was not, and what this document is, is a
reference manual. Three kinds of change were made.

### 17.1 Corrections — statements V4 made that the code contradicts

| V4 said | Reality | Now in |
|---|---|---|
| §4 "Nothing terminalizes early" | `human`/`bot` are terminal at **any** age | §8.3 |
| §4 "zero non-terminal installs past 24 h" | the straggler sweep is gated off in shadow — **not enforced** | §9.7, §12 |
| §3 "Money settles ONCE at the 3 h checkpoint and never moves after it" | settles at **1 h**, and the 72 h pay window makes late payment routine after an outage | §6, §9.3, §12 |
| §3 "the 15-min cron (3–24 h backstop)" | the live host has no cron; **inbound traffic** is the scheduler | §11.1 |
| §6 "V4-3 ✅ LIVE for gn" | the allowlist was emptied 2026-07-16; the merge is unreachable | §10.1 |
| §1 hardware "at install" | also re-evaluated retroactively on sync, without recomputing the tier | §2.7 |
| §2 `organic_revisit` weight, unconditional | requires `organic_core_size >= 2` | §7.8 |
| §2 `fast_uninstall` "uninstalls in-window" | has **no** time bound | §7.10 |
| §1.1 Tier 2 "behaviour decides" | four Tier-2 network signals hard-block the payout via `level` | §3.6, §5.5 |

### 17.2 Additions — subsystems V4 never documented

`evaluateVirtualMachine()` in full (§2) · the `level` string and the money invariant that hangs
off it (§3) · identity and duplicate handling (§4) · the install-time decision path (§6) ·
feature extraction (§7.2–7.7) · collection and terminalization (§8) · the finalization sweeps and
the uninstall checkpoint (§9) · the runtime envelope (§11) · a configuration reference (§13) · the
complete state vocabulary (§14) · an operations runbook (§15).

### 17.3 Known defects, now documented rather than hidden

Six live-code defects are recorded in `> **Defect**` blockquotes: the dead old-Chrome branch
(§2.6), the fail-open evaluator exception (§2.9), the unconditional `ninja_sync` overwrite (§6.3),
the `$HAS_FRAUD_COLUMNS` payout trap (§6.6), the proxycheck fail-open (§3.4), and the
terminalization/collection interaction (§8.4). None are fixed by this document — it records them
so a fix can be a deliberate change with its own verification.

---

## 18. Per-extension fraud implementation

This document describes the **V5 stack** as it runs in the reference backend
(Ninja, ported to Ghost). Not every build carries the whole stack — the older V1
builds have only the network layer. The client's role is minimal and server-side in
every build: **V2** clients ship **raw hardware signals** while unidentified or
migrating and nothing more; **V1** clients collect and ship **no hardware** at all.
Every verdict is server-side (`EXTENSION_COMPOSITION.md` §7.2).

**Legend:** ✅ present · ⚠️ partial/legacy · ❌ absent · ⬜ to copy.

| Layer | § | 12 Pro | 21 North | 22 Hunter | 23 Wonder | 25 Ninja | 26 Ghost | 27 |
| :-- | :-- | :-- | :-- | :-- | :-- | :-- | :-- | :-- |
| Hardware / VM (`evaluateVirtualMachine`) | 2 | — | ❌ | ❌ | ⚠️ legacy `devVM` (stale) | ✅ | ✅ | ⬜ copy |
| Network risk + `level` digits | 3 | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ⬜ copy |
| Device hash / fingerprint / dupes | 4 | ⚠️ IP/CID + `duplicate` only | ⚠️ IP+`an`+instdom only | ⚠️ same | ⚠️ same | ✅ device_hash | ✅ device_hash **+ device_sig** | ⬜ copy |
| Tier assembly at install | 5 | — | ❌ | ❌ | ❌ | ✅ | ✅ | ⬜ copy |
| Behaviour layer (v4 classifier) | 7 | — | ❌ | ❌ | ❌ | ✅ shadow | ✅ shadow | ⬜ copy |
| Finalization sweeps | 9 | — | ❌ | ❌ | ❌ | ✅ | ✅ | ⬜ copy |
| Enforcement live? | 0 | n/a | n/a | n/a | n/a | **shadow** (Stage-1 1 h hold) | shadow | n/a |
| `conv_date` / `updated_at` audit columns | 14 | — | ❌ | ❌ | ❌ | ❌ | ✅ | ⬜ copy |
| Quoted-enum `conv` SQL + NULL/race guards (2026-09-02) | 14 | n/a | n/a | n/a | n/a | ✅ local (2026-09-03 sync, upload pending) | ✅ | ⬜ copy |

**Reading the fleet.** **12 (`Gen 0` legacy) also runs the network layer only** —
proxycheck risk → `level`, and IP/CID + a `duplicate` counter — with **no** hardware
VM, tiers, or behaviour layer (it collects no hardware and has no fraud columns). Its
`—` cells are **permanent by lineage**, exactly like the V1 builds' `❌` cells:
never "close" them (that would require adding hardware collection + a `fingerprint`
column, i.e. re-birthing it as a V2 build). The V1 builds run the same **network layer
only**: proxycheck risk, the `level` digit string, and IP/attribution duplicate checks —
no tiers, no behaviour layer. **North and Hunter collect no hardware; Wonder carries a
*stale legacy* `devVM` VM-check** (⚠️ in the table above) — a pre-V5 artifact, **not** the
final Ninja/Ghost hardware stack and **never a copy source**; the V5 fraud stack still
treats all three V1 builds as network-only. Ninja is the full V5 reference this document
describes. Ghost
is a parity port that additionally splits **`device_sig`** (stable, identity —
`USER_CREATION_AND_UPDATE.md` §3.2) from **`device_hash`** (volatile, fraud) — the
two must never be unified. **Since 2026-09-02 Ghost is also the canonical copy source for the
conv pipeline**: it alone carries the `conv_date`/`updated_at` audit columns
(`migrate_conv_date_updated_at.sql`) and the quoted-enum + NULL + race fixes (header note;
§14.1). The full fix set was ported into Ninja's local mirror on 2026-09-03 (afternoon) —
quoted comparisons in `generate_fraud_report.php`, the NULL guard on the deferred payout,
the race guard on the pay UPDATE, IF-guarded `conv_date` maintenance — but **nothing is
uploaded yet**: until the upload, the live host still runs the pre-fix set, so **its live
`fraud_report` stats are still computed with the wrong enum-index semantics** the fix removed
(`DETECTION_CHANGELOG.md` 2026-09-02, 2026-09-03). The `n/a` cells reflect that no other build carries the fraud pipeline; 21/22/23 were
verified to bind `conv` only as prepared-statement parameters, which the enum trap cannot
reach. 27's backend is copied from Ghost/Ninja, so it inherits
the whole stack (⬜) — copy Ghost, not Ninja.

**These are permanent properties, not gaps.** A V1 build is Gen 1 for life; its
`❌` rows must **never** be "fixed" by adding fraud, because that would require
changing the identity — which orphans every existing user
(`BACKEND_ARCHITECTURE.md` §8.3). A product that needs fraud is built new on V2, not
migrated from a V1.

---

## 19. Conformance checklist

> **Almost entirely V2.** This document specifies the **V2 fraud stack**; a **V1**
> build carries **only the network-risk sub-layer** of §3 — proxycheck risk, the
> `level` digit string (including the desktop-Linux flag), and the four-condition
> conversion gate — and has **no** tiers, VM evaluation, device hashing, behaviour
> layer, or finalization (§18). So on a **V1** build run only these items, which bind
> both generations: `level != "0"` is never paid; **risk lookup fails open**
> (`level = "0"`); organics (`an` empty) never pay; and any `linux` User-Agent is
> flagged (`level` digit `9`, `conv = 2`). **Every other item below is V2-only** —
> and never "fix" a V1 build's absent fraud rows by adding them, because that changes
> the identity and orphans every existing user (§18, `BACKEND_ARCHITECTURE.md` §8.3).

The fraud system (V2) is conformant when:

- [ ] The two axes never fuse: `decision` (who is this user — behaviour) and `conv` (what we did — money) are separate columns; **`conv` may read `decision`; `decision` never reads `conv`** — §1.1.
- [ ] The three layers stay independent: hardware→tier, network→tier+`level`, behaviour→`decision`; the behaviour score **never** ingests the tier prior or `is_vm` (the v3 defect) — §1.2, §7.
- [ ] `level != "0"` is never paid, at any layer, ever — §3.3.
- [ ] Risk lookup **fails open** (`level = "0"`); identity/token mint **fails closed** (no row) — §3.4, and `USER_CREATION_AND_UPDATE.md` §9.
- [ ] Money settles **once**, at the merge; a paid conversion is never reversed (a later bot verdict is recorded as a chargeback flag, never a demotion); **organics (`an` empty) never pay** — §6, §9.
- [ ] Refusal requires **positive** evidence; absence of evidence never refuses — all unknowns (ghosts included) are paid — §9.
- [ ] **(V2-B only)** The identity signature (`device_sig`, stable components) and the fraud hash (`device_hash`, volatile components) are **separate columns from separate component sets** — §4, `USER_CREATION_AND_UPDATE.md` §3.2. *(V2-A ships `device_hash` only; skip this item for V2-A, `DB_SCHEMA.md` §8.3.)*
- [ ] ASN/device auto-blacklisting stays **suspended** unless rebuilt on outcome-based numerators with residential/mobile ASNs capped at Tier 2 — §4/§5 history.
- [ ] Every SQL literal compared to or assigned into `conv` is a **quoted string** (`'2'`, `IN ('1','3')`) — never a bare number: `conv` is an ENUM, and an unquoted number is read as the enum **index** (index 2 = value `'1'` = PAID) — §8.8, §14.1. *(A build whose only `conv` SQL uses bound prepared-statement parameters satisfies this trivially.)*
- [ ] Every `conv` write also maintains `conv_date`: plain `NOW()` where the `WHERE` guard guarantees a real transition; `conv_date = IF(conv <=> ?, conv_date, NOW())` assigned **before** `conv` where a same-value rewrite is possible; pay/deferred `WHERE` guards are NULL-safe (`OR conv IS NULL`) and re-check the unpaid state in SQL. `updated_at` is DB-maintained and never hand-written outside a migration backfill — §9, §14.1, §14.6. *(Requires the 2026-09-02 `conv_date`/`updated_at` migration.)*
- [ ] New telemetry/fraud columns are gated by a **schema-capability check** (`$HAS_FRAUD_COLUMNS`, or the legacy cached `SHOW COLUMNS` probe) so a not-yet-migrated column never enters the INSERT. **This is NOT fully order-independent:** a stale `true` flag against an un-migrated DB throws every INSERT (§13.1, §13.6) — the flag must track the migrated schema — §13; `USER_CREATION_AND_UPDATE.md` §12.
- [ ] Any User-Agent containing `linux` is flagged (risk digit `9`, `conv = 2`) — a preserved product constraint — §3.2.
- [ ] Enforcement and Stage-2 refusals stay behind explicit config flags (`$BEHAVIOR_ENFORCE`, `$BEHAVIOR_AN_ALLOWLIST`); the default is full shadow — §0, §10.8.

---

*Live-system reference for the V5 fraud stack. The client only ever ships raw
signals (`EXTENSION_COMPOSITION.md` §7.2); identity columns are specified in
`USER_CREATION_AND_UPDATE.md`; rule effects of `level`/`script` in
`RULES_AND_PRIORITIES.md`. Start from `00_INDEX.md`.*
