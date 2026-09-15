# Create State — `<build>` (genuine ad-blocker frontend)

> The written picture of "what this new build's frontend looks like when done." Drafted by
> `create-extension-frontend` PHASE T from `EXTENSION_COMPOSITION.md` (the baseline) + the
> owner's build choices, then **confirmed/edited by the owner**. It is the yardstick for the
> build plan (PHASE A/P) and the acceptance test at the end (§26 conformance). Copy this
> template to `<project>/CREATE_STATE.md` and fill it in. Update it as the build converges.

- **Build:** `<n - Name>`  ·  **Product name:** `<...>`
- **Naming scheme:** storage-key prefix `<...>` · rule-ID scheme `<...>` · message-type style `<...>`
- **Server posture:** `<local-only | + filter-update feed | + user-preference sync>` (§7 — never identity/monetization)
- **Drafted:** `<YYYY-MM-DD>`  ·  **Confirmed by owner:** `<yes/no + date>`

---

## 1. Build choices (the recorded picks)

- **Popup block-landing style** `[VARIANT: choose-one]`: `<A self-closing interstitial (recommended) | B on-page notification>`
- **Toggleable features:** YouTube `<ship? default OFF>` · cookie-consent `<ship? default OFF>`
- **Optional features:** element picker `<ship / omit>`
- **Sync seam (§7):** `<local-only | filter-update feed URL(s): … | preference-sync: user prefs only>`

## 2. Invariants (checked every phase)

*The load-bearing behaviors. Every change is validated against these before advancing.*

- **MV3 clean (§0.2 / §22.1):** no remote code (`eval`/`new Function`/remote script), declarative-only filtering (no blocking `webRequest`), no remotely-referenced resource (`<script src>`/`<link>`/`@import`/`url(http)`/remote img/iframe) — everything ships in the package.
- **Zero-flicker Layer 1** reads `chrome.storage.local` **directly** at `document_start` (never messages the worker) — `EXTENSION_COMPOSITION.md` §11.1.
- **Whitelisted == paused** gates **every** path (network, scriptlets, cosmetics, zero-flicker); the user can whitelist/pause **any** site — §13.4.
- **Priority ladder:** ship only the §4.2 permanent-floor (`always`) tiers; user tier above the block rules; the `script=1` injection tiers are the monetization band — **not built** — `RULES_AND_PRIORITIES.md` §4.2.
- `<other load-bearing behavior for this build>`

## 3. Definition of done — target per area

> **Seed this table from the FULL surface — the skill's §3 feature map AND every
> `EXTENSION_COMPOSITION.md` §26 item — one row per area — so nothing is silently dropped.**
> Mark `n/a` (with a reason) where an area does not ship, rather than omitting it.

*Each line is `to build → done?`; `n/a` = not shipping in this build (say why).*

| Area | § | Target | Done? |
| :-- | :-- | :-- | :-: |
| Manifest & platform baseline (MV3, module worker, capabilities, injection points, localization) | 2 | | ☐ |
| Storage model — local source of truth, sync-registered listeners, persisted state | 3 | | ☐ |
| Versioning & storage-purge — data-version purge, **preserved set**, stamp last, version pin | 5 | | ☐ |
| DNR rule system — rule-ID partitioning, range-scoped appliers, instant feedback + restart persistence | 8 | | ☐ |
| User-tier priority (above the block floor; injection band not built) | 8 / RULES §4.2 | | ☐ |
| Static rulesets shipped **enabled** + vendor self-allow at max priority | 8.4–8.5 | | ☐ |
| Blocklist & compiled bulk — version-gated | 9 | | ☐ |
| Cosmetic filtering — four-layer | 10 | | ☐ |
| Zero-flicker — two-layer, Layer-1 direct storage read, built locally | 11.1 | | ☐ |
| Scriptlets — static uBO DB, main-world + CSP fallback, quiet aborts | 12.1–12.3 | | ☐ |
| Surrogates actually shipped + web-accessible | 12.4 | | ☐ |
| Whitelist / blocklist / pause — user lists, removal tracking, shipped system whitelist | 13.1–13.3 | | ☐ |
| Whitelisted == paused (gates every path, zero-flicker included) | 13.4 | | ☐ |
| YouTube skipper `[TOGGLEABLE off]` | 14 | | ☐ |
| Cookie-consent `[TOGGLEABLE off]` | 15 | | ☐ |
| Element picker `[OPTIONAL]` | 16 | | ☐ |
| Popup-ad blocker & block-landing `[VARIANT A/B]` | 17 | | ☐ |
| Local visit stats `{count, last-time}`, cap ~50, decay-evicted | 18 | | ☐ |
| Sync cadence (action-driven) — only if a §7 seam exists | 19 | | ☐ |
| Lifecycle — install / **update must not reopen onboarding** / uninstall URL (fail-safe, non-tracking) / kill-switch | 20 | | ☐ |
| UI surfaces — popup, options, onboarding, block-landing, localized strings | 21 | | ☐ |
| MV3 compliance & privacy — no remote code/resources, CSS grammar, CSP self-host, no phishing, verbose-log gate | 22 | | ☐ |
| *(add any §26 checklist item not listed above — do not omit)* | | | ☐ |

## 4. Explicitly OUT of scope (not part of a genuine blocker — do not build)

*These are the identity/monetization layer, out of scope for this skill (§0.1 / §3 / §9).*

- Durable server identity handle + variants (`EXTENSION_COMPOSITION.md` §6)
- Install-attribution capture (`an`/`cid`/`sid`) + hardware fingerprint (§7)
- Server CSS directive as an injection/result-mask (§11.2) · protected-domain carve-out (§13.5)
- Monetization postures / enable-gate / search-redirect / injection (§24)

## 5. Known notes

- `<anything the owner knows the baseline doesn't — filter-list sources, UI intent, quirks>`
