---
name: generate-pr
description: >-
  Drafts a Spanish Pull Request description from git diff against origin/develop,
  classifies changes by TableHopping backend Clean Architecture layers, writes
  the result to .cursor/pr-drafts/*.md, and replies with only a short English
  path confirmation. Use when the user asks to generate a PR description,
  squash message, or mentions generate-pr / generar-pr.
---

# Generate PR (Table Hopping Backend)

Act as **ExpertGitAgent**: produce a PR draft suitable for squash merge.
**Do not** modify `app/**` or other source code while generating the draft.

**PR body language:** Spanish only (title, sections, lists), regardless of chat
language.

**Output template:** Read [pr-template.md](pr-template.md) before Step 4 and
follow that Markdown structure exactly.

## Execution

### Step 1: Base branch

Do not ask the user. Base is always `origin/develop`. Compare `HEAD` to it.

Announce in English: `Analyzing diff against origin/develop...`

### Step 2: Diff

From repo root, run exactly:

```bash
git diff origin/develop...HEAD --no-color --find-renames
```

Use output as primary context. If it fails or is empty, reply in English only
(e.g. `No differences found between HEAD and origin/develop, or git diff failed.`)
and **stop** — do not invent a PR or write a file.

### Step 3: Analyze

Ignore trivial/generated files (`__pycache__`, `*.pyc`, `.pytest_cache`,
`.mypy_cache`). Classify meaningful changes by layer:

| Layer | Typical paths |
|-------|---------------|
| API | `app/api/endpoints/**`, `app/api/schemas/**`, `app/api/*dependencies.py`, `app/api/router.py` |
| Domain | `app/domain/entities/**`, `app/domain/repositories/**`, `app/domain/service_interfaces/**`, `app/domain/services/**`, `app/domain/exceptions.py` |
| Infrastructure | `app/infrastructure/persistence/**`, `app/infrastructure/security/**`, `app/infrastructure/external/**` |
| Core | `app/core/**`, `app/main.py` |
| Tests | `tests/**` |
| Database | `alembic/**` |
| Refactors | No behavioral/business-logic change |

### Step 4: Generate content (Spanish)

Fill [pr-template.md](pr-template.md): every section that applies, omit irrelevant
ones where noted.

Rules:

1. Draft must start with first `##` from template — no preamble in file.
2. Max **4000** characters total; shorten if needed.
3. Tone: clear, concrete, useful as squash commit message.

### Step 5: Write draft file

Only if Step 2 had non-empty diff.

1. Directory: `.cursor/pr-drafts/` (create if missing).
2. Filename: `PR-<sanitized-branch>-<YYYY-MM-DD>_<HHmm>.md`
   - Branch: `git branch --show-current`, lowercased, `/` -> `-`, strip invalid
     filename characters (use `-`).
   - Timestamp: local date and 24h time, zero-padded (e.g. `2026-04-06_1430`).
3. File body: Step 4 output only (Spanish PR markdown).
4. If path exists, use `-2`, `-3`, ... before `.md`.

Chat reply: **one short English line** with relative path (e.g.
`Wrote .cursor/pr-drafts/PR-feature-auth-refresh-2026-04-06_1430.md`).
**Do not** paste full PR in chat.
