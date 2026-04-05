# WCAG Accessibility Auditor — Claude Skill

## Context

Scott wants a reusable Claude Code skill that helps web developers evaluate
**their own running web applications** against the W3C/WAI accessibility
standards. The skill should:

- Run a **real-time audit** of any URL (local dev server or production), including
  **authenticated** routes behind a login
- Cover the full **WCAG 2.2 Level AA** success-criterion set by default
  (transitively includes 2.0 and 2.1 A/AA); **AAA is opt-in** via a flag
- Report findings mapped to specific WCAG success criteria with **concrete
  remediation diffs** — Claude reads the user's source repo and proposes Edit
  tool changes, not generic snippets
- Produce a **markdown findings report** as the primary artifact (deterministic
  transformation of the audit JSON)
- Optionally **fill in a user-supplied ITI VPAT 2.5 INT `.docx` template** to
  produce a formal Accessibility Conformance Report; the skill itself does not
  redistribute ITI's template

The skill is being authored as a **standalone git repo** that installs as
**both** a Claude Code plugin and a standalone skill.

### Decisions locked in

| # | Topic | Decision |
|---|---|---|
| 1 | Audit target | User's own web app — Claude reads repo source, proposes Edit diffs as remediation |
| 2 | WCAG level | Default 2.2 AA; AAA opt-in via `--level AAA` flag |
| 3 | 508 / EN 301 549 | Derived from WCAG findings via cited crosswalk files; non-WCAG clauses marked N/A with rationale |
| 4 | Authentication | Playwright `storageState` via a `login.mjs` helper; `auth.json` gitignored; no bearer/basic-auth in v0.1.0 |
| 5 | Multi-page discovery | Layered sources unioned: `--sitemap` + `--urls` + bounded `--crawl-from --depth N` (auth-aware, same-origin) |
| 6 | Exit codes | `--fail-on <severity>` using axe impact levels; default `none` |
| 7 | Distribution | Ship as **both** plugin (`.claude-plugin/plugin.json`) and skill; README documents both install paths |
| 8 | VPAT | **Do not redistribute ITI template.** Primary artifact = markdown findings report. Optional `vpat-fill.mjs` populates a **user-supplied** ITI `.docx` |
| 9 | Viewports | Default **2 viewports** (1280×800 desktop + 375×667 mobile); `--viewport` flag repeatable to override |
| 10 | Concurrency | Default `--concurrency 1` (safe for dev servers); user opts into higher |
| 11 | `best-practice` rules | Excluded by default; `--include-best-practice` opts in. Never affect VPAT conformance. |
| 12 | Report format | Markdown report is a **deterministic** transformation of `aggregated.json`. Conversational triage with Edit diffs is a parallel flow. |
| 13 | Fixtures | Two: `broken-page.html` (5 WCAG-normative violations) + `broken-page-manual.html` (removed focus ring) |
| 14 | Node version | `engines.node >= 22.0.0` (current active LTS) |

Other fixed inputs:

- **Audit engine:** `axe-core` via `@axe-core/playwright` (Deque, industry standard)
- **Repo location:** `/Users/scottbaldwin/projects/wcag-auditor`
- **GitHub remote:** `https://github.com/benry-git/wcag-auditor.git`
- **Self-test targets:** two local fixtures + an authenticated test app

### Known numbers

- WCAG 2.2 has **86 success criteria** (2.1 had 78; 2.2 added 9, obsoleted 4.1.1 Parsing).
  This is the source-of-truth count for `references/wcag-2.2-criteria.md`.

## Deliverable: repo layout

```
~/projects/wcag-auditor/
├── README.md                         # documents both install paths (plugin + skill)
├── .gitignore                        # includes auth.json, audit-*.json, .env, ./audit/
├── package.json                      # engines.node >= 22; deps below
├── .claude-plugin/
│   └── plugin.json                   # bundles the skill; enables /plugin install
└── skills/
    └── wcag-auditor/
        ├── SKILL.md
        ├── references/
        │   ├── wcag-2.2-criteria.md          # all 86 SCs, level, URL, intent
        │   ├── axe-rule-mapping.md           # GENERATED from axe.getRules()
        │   ├── section-508-mapping.md        # 508 → WCAG crosswalk (cited)
        │   ├── en-301-549-mapping.md         # EN 301 549 → WCAG crosswalk (cited)
        │   ├── vpat-mapping.md               # how findings map into VPAT tables (our docs, not ITI's form)
        │   ├── manual-checks.md              # SCs axe cannot automate
        │   ├── manual-checks-aaa.md          # AAA-only manual checks
        │   ├── remediation-patterns.md       # fix library by rule
        │   └── url-discovery.md              # per-framework recipes
        ├── scripts/
        │   ├── login.mjs                     # headed login → auth.json
        │   ├── audit.mjs                     # single-URL audit
        │   ├── audit-site.mjs                # multi-URL, layered discovery
        │   ├── score.mjs                     # aggregate → per-SC conformance
        │   ├── report-generate.mjs           # aggregated.json → report.md (deterministic)
        │   ├── vpat-fill.mjs                 # user-supplied ITI .docx → filled ACR .docx
        │   └── gen-axe-mapping.mjs           # regenerates axe-rule-mapping.md
        ├── assets/
        │   └── report-template.md            # markdown template used by report-generate.mjs
        └── examples/
            ├── broken-page.html              # 5 planted WCAG-normative violations
            ├── broken-page-manual.html       # removed focus ring (manual-check fixture)
            ├── sample-audit.json
            ├── sample-urls.txt
            └── sample-report.md
```

## Package dependencies

```json
{
  "engines": { "node": ">=22.0.0" },
  "dependencies": {
    "playwright": "^1.x",
    "@axe-core/playwright": "^4.x",
    "fast-xml-parser": "^4.x",   // sitemap parsing
    "docx": "^9.x"               // VPAT .docx fill-in
  }
}
```

## Critical files to create

| File | Purpose |
|---|---|
| `skills/wcag-auditor/SKILL.md` | Trigger description + workflow (discover → audit → triage → remediate → report → optional VPAT) |
| `scripts/login.mjs` | Headed Playwright: user logs in, saves `storageState` to `./auth.json` |
| `scripts/audit.mjs` | Single URL: Playwright + axe at configured viewports, writes per-URL JSON |
| `scripts/audit-site.mjs` | Multi-URL: unions `--sitemap` + `--urls` + `--crawl-from`, dedupes violations |
| `scripts/score.mjs` | Aggregated JSON → per-SC supports/partial/not-supported/N-A |
| `scripts/report-generate.mjs` | Deterministic transform: `aggregated.json` → `report.md` |
| `scripts/vpat-fill.mjs` | User-supplied ITI VPAT 2.5 INT `.docx` → filled ACR `.docx` |
| `scripts/gen-axe-mapping.mjs` | Generates `axe-rule-mapping.md` from `axe.getRules()` |
| `references/wcag-2.2-criteria.md` | Single source of truth for 86 SCs |
| `references/section-508-mapping.md` | 508 Chapters 5/6 ↔ WCAG SC crosswalk (cited) |
| `references/en-301-549-mapping.md` | EN 301 549 clauses ↔ WCAG SC crosswalk (cited to Annex A) |
| `references/vpat-mapping.md` | How findings map into VPAT tables (authored by us) |
| `references/manual-checks.md` | Checklist for SCs axe can't automate (~30% of AA) |
| `references/manual-checks-aaa.md` | AAA-only manual checks |
| `references/remediation-patterns.md` | Fix library: per axe rule, 2–3 HTML/React/ARIA snippets |
| `references/url-discovery.md` | Per-framework recipes (Next.js, Nuxt, Remix, SvelteKit, Astro, Rails, Django, Flask, Laravel) |
| `.claude-plugin/plugin.json` | Manifest for Claude Code plugin install |
| `examples/broken-page.html` | 5 planted WCAG-normative violations (fixture for automated pass) |
| `examples/broken-page-manual.html` | Removed focus ring (fixture for manual-check pass) |

## Fixture contents

### `examples/broken-page.html` — 5 WCAG-normative violations

| SC | Level | Violation | Axe rule | Tag |
|---|---|---|---|---|
| 1.1.1 Non-text Content | A | `<img>` with no alt attribute | `image-alt` | wcag2a |
| 1.3.1 Info and Relationships | A | `<table>` data with no `<th>` headers | `th-has-data-cells` / `td-headers-attr` | wcag2a |
| 1.4.3 Contrast (Minimum) | AA | Text at ~2.5:1 contrast ratio | `color-contrast` | wcag2aa |
| 3.3.2 Labels or Instructions | A | `<input type="text">` with no label | `label` | wcag2a |
| 4.1.2 Name, Role, Value | A | `<button>` with no accessible name | `button-name` | wcag2a |

All five are WCAG-normative (default axe tag set catches them), cover all four WCAG principles (POUR), and hit both Level A and AA. Each maps to a single specific axe rule for unambiguous assertion.

### `examples/broken-page-manual.html` — manual-check fixture

Button with `outline: none` on `:focus` and no replacement focus indicator. Verifies SC 2.4.7 Focus Visible detection via the manual-check pass (Playwright MCP: Tab through, screenshot focused element, confirm no visible change).

## CLI contracts

### `login.mjs`
```
node scripts/login.mjs --url <login-url> --out ./auth.json
# Launches headed browser, user completes login (incl. MFA),
# user hits Enter in terminal, storageState saved to --out.
```

### `audit.mjs`
```
node scripts/audit.mjs
  --url <url>                              # required
  --auth <path>                            # optional storageState
  --level AA|AAA                           # default AA
  --viewport <w>x<h>                       # repeatable; default: 1280x800 AND 375x667
  --include-best-practice                  # opt in to axe best-practice rules
  --out <path>                             # default ./audit-<ts>.json
  --fail-on none|minor|moderate|serious|critical|any   # default none
```

### `audit-site.mjs`
```
node scripts/audit-site.mjs
  # URL sources (one or more required, unioned + deduped)
  --sitemap <url>
  --urls <path>
  --crawl-from <url> --depth <n>           # bounded, same-origin, auth-aware

  # Filters applied to union
  --include <glob>
  --exclude <glob>

  --auth <path>
  --level AA|AAA
  --viewport <w>x<h>                       # repeatable; default: 1280x800 AND 375x667
  --concurrency <n>                        # default 1 (safe for dev servers)
  --include-best-practice
  --out <dir>                              # per-URL JSON + aggregated.json + urls.resolved.txt
  --fail-on none|minor|moderate|serious|critical|any   # default none
```

### `report-generate.mjs`
```
node scripts/report-generate.mjs
  --aggregated <path>                      # path to aggregated.json
  --out <path>                             # default ./report.md
# Deterministic transformation — same input always produces same output.
```

### `vpat-fill.mjs` (optional step)
```
node scripts/vpat-fill.mjs
  --template <path>                        # user-supplied ITI VPAT 2.5 INT .docx
  --aggregated <path>                      # aggregated.json
  --product "<name>" --version "<v>"
  --out <path>                             # default ./ACR-<product>-<date>.docx
```

### Exit code contract (all scripts)

| Code | Meaning |
|---|---|
| `0` | Ran successfully; `--fail-on` threshold not crossed |
| `1` | Ran successfully; threshold crossed (a11y gate failed) |
| `2` | Script error (network, bad URL, expired auth, axe crash) |

`--fail-on` operates on axe `violations` only (never on `incomplete`, which are
routed to the manual-check flow). `best-practice` findings (if included via flag)
never contribute to `--fail-on` thresholds and are tagged separately in outputs.

## Skill workflow (documented in SKILL.md)

1. **Trigger** — user says "audit my site" / "check accessibility" / "run an a11y check" / "generate a VPAT."
2. **Clarify scope** — URL(s) or dev server, WCAG level (default 2.2 AA), auth required?, viewport overrides?, framework.
3. **URL discovery** — Claude reads `references/url-discovery.md`, detects the framework from the repo, runs the appropriate recipe, generates `urls.txt`, shows it for user review.
4. **Auth setup (if needed)** — Claude runs `node scripts/login.mjs`; user completes login interactively; `auth.json` saved and gitignored.
5. **Automated pass** — `node scripts/audit-site.mjs --urls urls.txt [--sitemap …] [--crawl-from …] --auth auth.json --out ./audit/`.
6. **Manual pass** — Claude reads `references/manual-checks.md` and uses Playwright MCP tools (`browser_snapshot`, `browser_press_key`, `browser_evaluate`, `browser_take_screenshot`) to verify criteria axe can't cover: keyboard traps, focus visibility, focus order, reflow, target size, captions/transcripts, cognitive/reading level.
7. **Triage & remediate (conversational, parallel to report)** — for each violation:
   - Cite the SC (e.g. "SC 1.4.3 Contrast (Minimum), Level AA")
   - Explain user impact
   - **Locate the source** by matching axe's CSS selector / HTML snippet back to files in the user's repo
   - Propose a concrete fix as an **Edit tool diff** using `references/remediation-patterns.md`
8. **Score** — `node scripts/score.mjs ./audit/aggregated.json` produces per-SC supports / partially supports / does not support / not applicable.
9. **Generate markdown report** — `node scripts/report-generate.mjs --aggregated ./audit/aggregated.json --out ./report.md`. Deterministic, reproducible, diffable between runs.
10. **Optional: fill VPAT** — if user needs a formal ACR: user downloads the ITI VPAT 2.5 INT `.docx` template from itic.org, then `node scripts/vpat-fill.mjs --template ./VPAT2.5INT.docx --aggregated ./audit/aggregated.json --product "<name>" --version "<v>"`. Populates WCAG table directly; 508 and EN 301 549 cells inherit via crosswalks; non-WCAG clauses marked N/A with rationale.
11. **Re-audit** — after fixes, rerun step 5 and diff against prior aggregated.json.

## SKILL.md frontmatter

```yaml
---
name: wcag-auditor
description: Use when the user wants to audit their own web application for WCAG accessibility compliance, run an a11y check on a URL (including authenticated routes), generate a markdown accessibility findings report, fill in a VPAT / ACR, or get concrete remediation diffs for accessibility violations. Default target is WCAG 2.2 Level AA (covers 2.0/2.1 A/AA transitively); AAA is opt-in. VPAT fill-in requires a user-supplied ITI VPAT 2.5 INT template; covers WCAG + Section 508 + EN 301 549 via crosswalks.
metadata:
  author: Scott Baldwin
  version: "0.1.0"
license: MIT
---
```

## Reusable pieces found in existing skills

- **`webapp-testing` skill** — Playwright patterns, `wait_for_load_state('networkidle')` convention, server lifecycle. `audit.mjs` follows the same CLI-script pattern.
- **Playwright MCP tools** (`mcp__plugin_playwright_playwright__*`) — used directly in the manual-check pass rather than reimplementing a browser driver.
- **`web-design-guidelines` skill** — complementary (heuristic design review vs. normative WCAG compliance).

## Verification plan

1. **Automated-pass fixture test (ground truth)** — run `audit.mjs` against `examples/broken-page.html`; assert `violations[]` contains all 5 planted rules: `image-alt`, `color-contrast`, `button-name`, `label`, and (`th-has-data-cells` OR `td-headers-attr`).
2. **Manual-check fixture test** — run the manual-check pass against `examples/broken-page-manual.html`; assert Claude identifies the missing focus indicator via Playwright MCP inspection.
3. **Viewport coverage** — audit a page with a mobile-only violation (e.g. target-size failure at 375px); assert it's caught in the default 2-viewport run and missed when run with only `--viewport 1280x800`.
4. **Best-practice exclusion** — audit a page with only `best-practice` violations (e.g. missing `<main>` landmark); default run produces zero violations; `--include-best-practice` surfaces them.
5. **Authenticated self-test** — run `login.mjs` against a test SaaS app, audit an authenticated route, confirm violations are returned (proves `storageState` flow end-to-end).
6. **Layered discovery test** — run `audit-site.mjs` with `--sitemap` + `--urls` + `--crawl-from` simultaneously; assert union is deduped and `urls.resolved.txt` is correct.
7. **Report determinism** — run `report-generate.mjs` twice on the same `aggregated.json`; assert byte-identical output.
8. **VPAT fill-in** — download ITI template, run `vpat-fill.mjs` against fixture audit; spot-check (a) WCAG rows reflect findings, (b) 508 rows inherit from mapped WCAG rows, (c) EN 301 549 non-WCAG clauses marked N/A with rationale, (d) methodology populated.
9. **Re-audit diff** — apply a fix to the fixture, rerun, confirm the violation drops out.
10. **Exit code contract** — run with `--fail-on serious` against a page with only `minor` violations (expect exit 0); rerun against a page with `critical` violations (expect exit 1); kill the network mid-run (expect exit 2).
11. **Install paths (both)** — verify `/plugin install <path>` installs via plugin manifest; verify direct skill install (copy to `~/.claude/skills/`) also works; both should surface the skill in a fresh Claude session.

## Open implementation notes (defer to build phase)

- axe-core tag list:
  - AA default: `['wcag2a','wcag2aa','wcag21a','wcag21aa','wcag22aa']`
  - AAA (opt-in): add `['wcag2aaa','wcag21aaa','wcag22aaa']`
  - `best-practice` only added when `--include-best-practice` is passed
- `axe-rule-mapping.md` is generated at setup/build time from `axe.getRules()` via `gen-axe-mapping.mjs` to avoid drift with Deque's releases.
- **VPAT template licensing** — ITI's page states the VPAT name and report form are registered service marks and should not be altered without express written permission. The skill therefore **does not ship any ITI-authored content**. `vpat-fill.mjs` requires the user to download the `.docx` themselves and populates it programmatically. `references/vpat-mapping.md` is our own documentation of how findings map into the template's tables, not a copy of the template.
- Auth caveats to document in SKILL.md: `storageState` expires with the session; MFA is handled because state is captured post-login; short-lived CSRF tokens may require re-login; `auth.json` contains live credentials and must be gitignored.
- Framework route-manifest **parsers** (`--routes-from <path>`) are **deferred to v0.2.0**. v0.1.0 uses documented recipes in `references/url-discovery.md` that Claude executes against the user's repo.
- **False-positive suppressions / "partially supports" rationales** — needed for real VPATs. Deferred to v0.2.0 as a baseline/suppressions file (`--baseline prior-audit.json`).

## Known scope boundaries for v0.1.0

- Single-framework recipe execution (no automated multi-framework detection)
- No SPA client-side route discovery beyond seeded crawl
- No framework route-manifest parsers (recipes only)
- No baseline / suppressions file
- No independent 508 or EN 301 549 evaluation (derived only)
- No bearer-token / basic-auth / custom-header auth (storageState only)
- No ITI template redistribution; VPAT fill-in requires user-supplied `.docx`
