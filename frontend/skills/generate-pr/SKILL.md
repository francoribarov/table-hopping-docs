---
name: generate-pr
description: >-
  Drafts a Spanish Pull Request description from git diff against origin/develop,
  classifies changes by TableHopping Clean Architecture layers, writes the result
  to .cursor/pr-drafts/*.md, and replies with only a short English path confirmation.
  Use when the user asks to generate a PR description, squash message, or
  mentions generate-pr / generar-pr; or when preparing a PR for this Flutter repo.
---

# Generate PR (Table Hopping)

Act as **ExpertGitAgent**: produce a PR draft suitable for squash merge. **Do not** modify `lib/**` or other application source.

**PR body language:** Spanish only (title, sections, lists), regardless of chat language.

**Output template:** Read [pr-template.md](pr-template.md) before Step 4 and follow that Markdown structure exactly.

## Execution

### Step 1: Base branch

Do not ask the user. Base is always `origin/develop`. Compare `HEAD` to it.

Announce in English: `Analyzing diff against origin/develop...`

### Step 2: Diff

From repo root, run exactly:

```bash
git diff origin/develop...HEAD --no-color --find-renames
```

Use the output as primary context. If it fails or is empty, reply in English only (e.g. `No differences found between HEAD and origin/develop, or git diff failed.`) and **stop** — do not invent a PR or write a file.

### Step 3: Analyze

Ignore trivial/generated files (`*.freezed.dart`, `*.g.dart`, `injection.config.dart`). Classify meaningful changes by layer:

| Layer | Typical paths |
|-------|----------------|
| Presentation / UI | `lib/presentation/pages/**`, `widgets/**`, `blocs/**` |
| Domain | `lib/domain/usecase/**`, `model/**`, `repository/**`, `validators/**` |
| Data | `lib/data/datasource/**`, `dto/**`, `repository/**`, `service/**`, `mapper/**` |
| Core | `lib/core/di/**`, `network/**`, `routing/**`, `theme/**`, `errors/**`, `widgets/**` |
| Refactors | No business-logic behavior change |

### Step 4: Generate content (Spanish)

Fill [pr-template.md](pr-template.md): every section that applies, omit irrelevant ones where noted.

Rules:

1. Draft must start with the first `##` from the template — no preamble in the file.
2. Max **4000** characters total; shorten if needed.
3. Tone: clear, concrete, valuable as a squash commit message.

### Step 5: Write draft file

Only if Step 2 had a non-empty diff.

1. Directory: `.cursor/pr-drafts/` (create if missing).
2. Filename: `PR-<sanitized-branch>-<YYYY-MM-DD>_<HHmm>.md`
   - Branch: `git branch --show-current`, lowercased, `/` → `-`, strip invalid filename characters (use `-`).
   - Timestamp: local date and 24h time, zero-padded (e.g. `2026-04-06_1430`).
3. File body: Step 4 output only (Spanish PR markdown).
4. If the path exists, use `-2`, `-3`, … before `.md`.

Chat reply: **one short English line** with the relative path (e.g. `Wrote .cursor/pr-drafts/PR-feature-1-foo-2026-04-06_1430.md`). **Do not** paste the full PR in chat.
