# CLAUDE.md — wcag-auditor

Project-specific instructions for Claude Code when working in this repo.

## What this repo is

A Claude Code **skill** (and plugin) that audits web applications for WCAG 2.2
accessibility compliance. It runs axe-core via Playwright, maps violations to
WCAG success criteria, proposes remediation diffs, and generates VPAT 2.5 INT
Accessibility Conformance Reports.

Primary consumer: **the skill audits the user's own web app**, so Claude
should assume it can read the user's source repo to locate and fix
violations.

See `PLAN.md` for the full design decisions and scope.

## Repository

- **GitHub:** https://github.com/benry-products/wcag-auditor.git
- **Clone:** `git clone https://github.com/benry-products/wcag-auditor.git`

## Standard session workflow

1. **Start** — open the project in the terminal, `git pull` just to be safe
2. **Describe** — tell Claude what you want in plain English (feature, bug, change)
3. **Let it work** — Claude writes code, runs tests, iterates. For non-trivial
   changes (3+ steps or architectural decisions) Claude will enter plan mode
   first.
4. **Review** — read the diff before doing anything else
5. **Commit** — either Claude does it (only when explicitly asked) or you run
   `git commit -m "what changed"`
6. **Push** — `git push` to GitHub
7. **Deploy** — depends on setup; often automatic on push to `main`

## Repo layout

```
.
├── PLAN.md                           # design decisions + scope
├── README.md                         # install + quick-start
├── .claude-plugin/plugin.json        # plugin manifest (for /plugin install)
├── package.json                      # deps: playwright, @axe-core/playwright, axe-core, fast-xml-parser, docx, jszip
└── skills/wcag-auditor/
    ├── SKILL.md                      # skill entry point + workflow
    ├── references/                   # authored + generated content
    │   ├── wcag-2.2-criteria.md
    │   ├── axe-rule-mapping.md       # GENERATED — re-run gen-axe-mapping.mjs
    │   ├── manual-checks.md
    │   ├── manual-checks-aaa.md
    │   ├── remediation-patterns.md
    │   ├── url-discovery.md
    │   ├── section-508-mapping.md
    │   ├── en-301-549-mapping.md
    │   └── vpat-mapping.md
    ├── scripts/
    │   ├── audit.mjs                 # single-URL audit
    │   ├── audit-site.mjs            # multi-URL audit
    │   ├── login.mjs                 # Playwright storageState capture
    │   ├── score.mjs                 # per-SC conformance classification
    │   ├── report-generate.mjs       # deterministic markdown report
    │   ├── vpat-fill.mjs             # VPAT worksheet + .docx fill-in
    │   ├── gen-axe-mapping.mjs       # regenerates axe-rule-mapping.md
    │   └── lib/
    │       ├── audit-core.mjs        # shared Playwright+axe primitives
    │       ├── url-sources.mjs       # sitemap / urls.txt / crawl
    │       └── wcag-sc-list.mjs      # canonical 86-SC registry
    └── examples/
        ├── broken-page.html          # 5 planted violations (ground-truth test)
        └── broken-page-manual.html   # removed focus ring (manual-check test)
```

## Conventions

- **Node version:** `engines.node >= 22.0.0` (current active LTS)
- **Module format:** ES modules (`.mjs`, `"type": "module"`)
- **Script I/O:** logs to stderr; JSON/data output to stdout or `--out` file
- **Exit codes (all scripts):** 0 = success, 1 = threshold crossed (a11y gate),
  2 = script error
- **Deterministic outputs:** `report-generate.mjs` must produce byte-identical
  output for identical inputs
- **Never ship ITI VPAT content:** `vpat-fill.mjs` requires user-supplied
  `.docx` template. `references/vpat-mapping.md` is **our** documentation, not
  a copy of ITI's form.

## Regenerating axe-rule-mapping.md

After upgrading axe-core:

```bash
npm run gen:axe-mapping
```

This re-derives `references/axe-rule-mapping.md` from `axe.getRules()` so it
stays in sync.

## Testing against the fixtures

The canonical regression test:

```bash
# Should detect exactly these 5 rules: image-alt, color-contrast, button-name, label, list
node skills/wcag-auditor/scripts/audit.mjs \
  --url "file://$(pwd)/skills/wcag-auditor/examples/broken-page.html" \
  --out /tmp/fixture-audit.json
```

If any of the 5 rules stops firing, the fixture or the audit pipeline
regressed.

## Known scope boundaries (v0.1.0)

Do NOT add these without discussion:
- Bearer-token / basic-auth / custom-header authentication (storageState only)
- SPA client-side route discovery beyond seeded crawl
- Framework route-manifest parsers (recipes only for now)
- Baseline / suppressions file
- Independent evaluation of Section 508 or EN 301 549 (derived via crosswalks only)

These are explicitly deferred to later versions. See `PLAN.md` → "Known scope boundaries".

## Committing

- Only commit when explicitly asked
- Never use `--no-verify` or `--amend` without explicit request
- Prefer new commits over amending
- Commit message format: conventional-ish (`feat:`, `fix:`, `docs:`, `refactor:`, `test:`)
