# WCAG Auditor — Claude Code Skill

Audit your own web application for **WCAG 2.2 Level AA** accessibility
compliance, directly from a Claude Code conversation.

- Runs [axe-core](https://github.com/dequelabs/axe-core) via Playwright against
  any URL — public or **authenticated**
- Maps violations to specific WCAG success criteria
- Claude reads your source repo and proposes **concrete Edit diffs** as fixes
- Produces a deterministic markdown **findings report**
- Optionally fills in a user-supplied ITI **VPAT 2.5 INT** template to produce a
  formal Accessibility Conformance Report (ACR)

**Primary use case:** auditing your own web app. The skill assumes Claude can
read your source to locate and fix violations.

## Install

This repo ships as **both** a Claude Code plugin and a standalone skill. Pick
whichever install path suits you:

### As a plugin (recommended)

```
/plugin install /path/to/wcag-auditor
```

### As a standalone skill

```
cp -r skills/wcag-auditor ~/.claude/skills/
```

## Requirements

- **Node.js >= 22** (current active LTS)
- **Playwright Chromium** — installed automatically with `npx playwright install chromium`

Install the runtime dependencies:

```
cd wcag-auditor
npm install
npx playwright install chromium
```

## Quick start

From any Claude Code session:

> audit https://my-app.example.com for WCAG 2.2 AA

Claude will trigger the skill, clarify scope, and run the audit. For
authenticated routes, Claude will walk you through generating an `auth.json`
session-state file via a one-time headed login.

## CLI usage (direct)

The scripts are also runnable directly:

```bash
# Single URL
npm run audit -- --url https://example.com --out ./audit.json

# Multi-page via sitemap + explicit list
npm run audit:site -- \
  --sitemap https://example.com/sitemap.xml \
  --urls ./urls.txt \
  --auth ./auth.json \
  --out ./audit/

# Generate markdown findings report
npm run report -- --aggregated ./audit/aggregated.json --out ./report.md

# Optional: fill a user-supplied ITI VPAT 2.5 INT template
npm run vpat:fill -- \
  --template ./VPAT2.5INT.docx \
  --aggregated ./audit/aggregated.json \
  --product "MyApp" --version "1.0"
```

See `skills/wcag-auditor/SKILL.md` for the full workflow and flag reference.

## What's in scope (v0.1.0)

- WCAG 2.2 Level AA by default; Level AAA opt-in via `--level AAA`
- Public + `storageState`-authenticated routes
- Sitemap + explicit URL list + bounded seeded crawl
- Findings mapped to Section 508 and EN 301 549 via published crosswalks
- Markdown findings report (deterministic)
- VPAT 2.5 INT `.docx` fill-in (user supplies template)

## What's out of scope (for v0.1.0)

- Bearer-token / basic-auth / custom-header authentication
- Framework route-manifest parsers (recipes for Next.js/Rails/etc. documented
  instead — see `references/url-discovery.md`)
- Baseline / suppressions file
- Independent evaluation of 508 or EN 301 549 (derived from WCAG findings only)

## VPAT template

This repo **does not redistribute** the ITI VPAT template. The VPAT name and
form are registered service marks of the Information Technology Industry
Council (ITI). To produce a formal VPAT 2.5 INT ACR, download the current
template directly from
[itic.org/policy/accessibility/vpat](https://www.itic.org/policy/accessibility/vpat),
then pass its path to `vpat-fill.mjs`.

## License

MIT — see `LICENSE`.
