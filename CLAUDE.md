# CLAUDE.md

Guidance for AI assistants (Claude Code and others) working in this repository.

## What this repository is

**Antigravity Awesome Skills** (`sickn33/antigravity-awesome-skills`) is an
installable library of 1,500+ reusable agentic **skills** — `SKILL.md`
playbooks for AI coding assistants (Claude Code, Cursor, Codex CLI, Gemini CLI,
Antigravity, Kiro, OpenCode, GitHub Copilot, and more). It is *not* an
application: the "source" is the collection of `SKILL.md` files under `skills/`,
plus a Node/Python tooling layer that **validates** those skills and
**generates** every downstream artifact (catalog, indexes, plugin bundles,
web-app data).

It also ships an **npm installer** (`npx antigravity-awesome-skills`) that copies
skills into the directory your tool expects (default `~/.agents/skills`), and a
companion **web app** under `apps/web-app/` (a hosted discovery/search surface).

The two main jobs here:
1. **Add or improve a skill** — edit/create `skills/<slug>/SKILL.md`, then validate.
2. **Regenerate canonical artifacts** — run the sync chain so derived files match the skills.

## Repository layout

```
skills/<slug>/SKILL.md      # the actual product — 1,500+ skill playbooks
skills/<slug>/resources/    # optional deep-dive material referenced from SKILL.md
plugins/                    # GENERATED plugin bundles (full-library + ~50 specialized)
apps/web-app/               # Vite/TS web app (discovery UI); its own package.json + tests
tools/
  bin/install.js            # npm installer entry point (the `bin`)
  lib/                      # shared JS helpers (workflow-contract, skill-utils, ...)
  scripts/                  # validators, fixers, generators (Python + JS/CJS)
  scripts/tests/            # test suite (run via run-test-suite.js)
  config/                   # generated-files.json, validation-budget.json, ...
  requirements.txt          # Python deps (pyyaml)
schemas/                    # skills-index.v1.schema.json (public index JSON schema)
data/                       # GENERATED: skills_index.json, catalog.json, bundles.json, ...
scripts/                    # user-facing activate-skills.{sh,bat} + validate-*.sh helpers
docs/                       # English docs (users/, contributors/, maintainers/, ...)
docs_zh-CN/                 # Simplified-Chinese mirror of docs/
.claude-plugin/             # plugin.json + marketplace.json (GENERATED)
.agents/plugins/            # GENERATED marketplace manifest
CATALOG.md, CHANGELOG.md    # GENERATED catalog + release changelog
skills_index.json           # GENERATED root index (mirrored into data/ and the web app)
README.md                   # public landing page (partly generated stats)
```

**`plugins/`, `data/`, `CATALOG.md`, `skills_index.json`, `.claude-plugin/`, and
`.agents/plugins/` are generated output — never hand-edit them.** The canonical
list lives in `tools/config/generated-files.json` (`derivedFiles`,
`mixedFiles`, `releaseManagedFiles`). Use `node tools/scripts/generated_files.js`
to print the managed set.

## Skill file format (the core convention)

Every skill is a directory under `skills/` containing a `SKILL.md` with YAML
frontmatter. The canonical template is `docs/contributors/skill-template.md`.

```markdown
---
name: your-skill-name          # must match the directory slug
description: "One sentence, under 200 chars"
category: your-category
risk: safe                     # risk level (safe | unknown | ...); classified/validated by tooling
source: community              # community | self | official
source_repo: owner/repo        # required when adapting external material
source_type: community         # community | official | self
date_added: "YYYY-MM-DD"
tags: [tag-one, tag-two]
tools: [claude, cursor, gemini]
---
```

Conventions:
- **`name` must equal the directory name** and be unique across the roster.
- The public index schema (`schemas/skills-index.v1.schema.json`) requires
  `id`, `path`, `category`, `name`, `description`, `risk`, `source`, `date_added`.
- Skills adapted from an external repo must declare `source_repo` + `source_type`
  so README credit and reference validation pass; use `source: self` for originals.
- Put deep detail in `skills/<slug>/resources/` and reference it from `SKILL.md`
  (progressive disclosure), rather than bloating the always-loaded body.
- If a skill includes shell / network / credential / mutation / install guidance,
  run `npm run security:docs` and review failure modes manually.

## Workflows

Requires Node (LTS) and Python 3.10. Python scripts are invoked through
`tools/scripts/run-python.js`; install Python deps with `pip install -r tools/requirements.txt`.

### Validate skills (do this before committing skill changes)

```bash
npm install
npm run validate                # validate all skills against schema/frontmatter
npm run validate:strict         # stricter validation
npm run check:warning-budget    # enforce the validation warning budget
npm run validate:references     # cross-reference / source-credit checks
npm run security:docs           # docs security content checks
```

Common fixers (each rewrites skill files; review the diff):
`npm run fix:missing-sections`, `fix:missing-metadata`,
`fix:truncated-descriptions`, `cleanup:synthetic-sections`.

### Regenerate canonical artifacts after editing skills

```bash
npm run chain        # validate + plugin-compat:sync + index + bundles:sync + sync:metadata
npm run catalog      # rebuild CATALOG.md from skills
npm run build        # chain + catalog
npm run sync:repo-state   # full local maintainer sync (chain + catalog + web assets + audits)
```

`npm run index` regenerates `skills_index.json`; `npm run readme` updates
generated README sections. Single artifacts have dedicated scripts (see
`package.json` `scripts`).

### Test

```bash
npm test                 # repo test suite (tools/scripts/tests/run-test-suite.js)
npm run test:local       # offline subset
npm run app:install && npm run app:test   # web-app vitest suite
```

### Web app

```bash
npm run app:setup        # stage generated skills data into apps/web-app/public/
npm run app:dev          # setup + Vite dev server
npm run app:build        # setup + production build
```

The web app is a self-contained Vite/TypeScript project under `apps/web-app/`
with its own `package.json`, eslint config, and vitest tests.

## CI (`.github/workflows/`)

`ci.yml` ("Skills Registry CI") runs on PRs and pushes to `main`:
- **PR jobs are source-only.** A PR that touches generated/derived files
  (`CATALOG.md`, `skills_index.json`, `data/*.json`, `plugins/`, etc.) fails the
  `pr-policy` job. PRs must also include the Quality Bar Checklist from the
  template. `main` regenerates derived files after merge and auto-commits them.
- `source-validation` runs `npm run validate`, `check:warning-budget`,
  `check:readme-credits`, conditionally `validate:references`, `npm audit`,
  `npm run test`, web-app coverage, and `security:docs`.
- `main-validation-and-sync` runs `sync:repo-state` and auto-commits canonical
  artifacts (`chore: sync repo state [ci skip]`) — only the managed file set.

Other workflows: `actionlint`, `codeql`, `dependency-review`, `repo-hygiene`,
`publish-npm`, `pages` (web-app deploy), and `skill-review` / `skill-apply-optimize*`
(automated skill review on PRs touching `SKILL.md`).

There is no traditional linter/formatter gate beyond the validators above and
`actionlint` for workflows. "Tests pass" here means `npm run validate`,
`npm run test`, and the web-app coverage suite pass.

## Conventions for an AI assistant operating here

- **Never hand-edit generated files** (see `tools/config/generated-files.json`).
  Edit `skills/` (and source docs), then regenerate via `npm run chain` / `catalog`.
- Keep PRs **source-only** — let `main` canonicalize derived artifacts. CI will
  reject derived-file changes in a PR.
- Match the skill template structure and frontmatter exactly; the slug, `name`,
  and required index fields are load-bearing.
- **Bilingual docs:** `docs/` (English) is mirrored by `docs_zh-CN/`
  (Simplified Chinese). When you change a doc that exists in both, update both.
- Bash scripts live in `scripts/` (user-facing `activate-skills.*`, validation
  helpers); maintenance/generation logic lives in `tools/scripts/`.
- Do not commit temporary report/analysis files — `.gitignore` already excludes
  `*_REPORT.md`, `*_ANALYSIS*.md`, `*_results.json`, and similar.
```
