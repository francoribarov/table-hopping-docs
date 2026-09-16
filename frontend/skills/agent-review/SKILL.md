# Agent Review — TableHoppingApp PR Review

Runs a comprehensive multi-agent review of the current branch against `develop`. Launches specialist subagents in parallel, then synthesizes findings into a prioritized action list.

---

## Step 1: Gather Git Context

Run these commands to collect the diff:

```bash
git fetch origin develop
git log --oneline origin/develop..HEAD
git diff --stat origin/develop...HEAD
git diff origin/develop...HEAD
```

Save the outputs:
- **Commit log** — summary of what changed and why
- **Diff stat** — file list with change size
- **Full diff** — line-by-line changes

---

## Step 2: Pre-Review Configuration

### Classify Conditional Reviewers

Inspect changed file paths from diff stat:

| Reviewer | Trigger paths |
|----------|--------------|
| **perf-reviewer** | `lib/presentation/blocs/`, `*_bloc.dart`, `*_cubit.dart` |
| **data-layer-reviewer** | `lib/data/` |
| **ui-reviewer** | `lib/presentation/pages/`, `lib/presentation/widgets/`, `*_page.dart`, `*_widget.dart` |

### Ask the User

Before proceeding, ask:

1. "Should I run all conditional reviewers, or only those triggered by the changed files?"
2. "Any specific area to focus on? (architecture / data layer / UI / performance / all)"

### Always Ignore Generated Files

Never include these in diffs, file lists, or agent context (no user prompt):

- `*.freezed.dart`
- `*.g.dart`
- `*.config.dart` (DI generated)

---

## Step 3: Apply Scope Filter (if requested)

If the user selected a specific focus area, filter the diff to that scope before passing to agents. **Always** apply the generated-file exclusions above to every review (including full-diff and high-risk preloads).

---

## Step 4: Build Staged Context Bundle

For **high-risk files** (new BLoCs, new repositories, new data sources, changes to core error handling or routing), preload the full file contents. This gives agents the complete context beyond just the diff.

High-risk indicators:
- New `extends Bloc<>`, `extends Cubit<>`, `extends BaseRepository`, `extends BaseDataSource`
- Changes to `lib/core/routing/app_router.dart`
- Changes to `lib/core/di/injection.dart`
- Changes to `lib/core/errors/`

For large diffs (>500 lines), be selective — preload at most 5-8 key files.

---

## Step 5: Identify Consumer Impact

For any **changed public interfaces** (repository interfaces, use case signatures, domain models):

```bash
grep -r "FooRepository\|FooUseCase\|FooModel" lib/ --include="*.dart" -l
```

Note which files consume the changed APIs. Pass this impact list to the architecture-reviewer.

---

## Step 6: Launch Subagents in Parallel

**Always run (4 core agents):**
- `code-quality-reviewer` — correctness, quality, design token compliance
- `bug-reviewer` — runtime crashes, async defects, contract mismatches
- `architecture-reviewer` — layer boundaries, DI, routing, use case structure
- `pre-pr-validator` — binary pass/fail mechanical compliance

**Conditional agents (based on Step 2 classification):**
- `perf-reviewer` — widget rebuilds, memory leaks, BLoC efficiency
- `data-layer-reviewer` — Retrofit, DTOs, data sources, repositories
- `ui-reviewer` — design system compliance, widget structure, accessibility

**Each agent receives:**
1. Full git diff (filtered to scope; generated files stripped)
2. List of changed files (excluding generated)
3. Preloaded full file contents for high-risk files
4. Consumer impact map (for architecture-reviewer)
5. User's focus area (if specified)

Launch all selected agents in parallel and collect their outputs.

---

## Step 7: Challenge Critical Findings

After all parallel agents complete, run `finding-challenger`:

**Pass it:**
- All Critical/Blocker findings from all agents (not Suggestions or Nice to Have)
- The full file contents for each cited file
- The full diff

The challenger verifies each finding, renders CONFIRMED / DOWNGRADED / REJECTED verdicts, and returns a verdict table.

---

## Step 8: Synthesize Final Report

Run `review-synthesizer` with:
- All parallel agent outputs
- Finding-challenger verdicts
- PR metadata (branch name, commits, diff stat)

The synthesizer applies challenger verdicts, deduplicates findings, and produces the executive summary.

---

## Agent Context Template

When invoking each agent, provide:

```
## Context

**Branch:** [current branch]
**Base:** origin/develop
**Commits:** [git log output]
**Files changed:** [diff stat]
**Focus area:** [user selection or "all"]

## Diff
[filtered git diff]

## Preloaded File Contents
[full contents of high-risk files]

## Consumer Impact Map
[list of files consuming changed APIs]
```

---

## Validation Commands (run locally before or after review)

```bash
make format       # dart format --line-length=80
make analyze      # dart analyze --fatal-infos --fatal-warnings
make test         # flutter test
make pre-pr       # full: format + analyze + test + codegen check
```

---

## Review Pipeline Summary

```
[git diff] → [classify reviewers] → [ask user] → [build context]
     ↓
[parallel: code-quality + bug + architecture + pre-pr-validator]
     + [optional: perf + data-layer + ui]
     ↓
[finding-challenger] → validate all Critical findings
     ↓
[review-synthesizer] → executive summary + action list
```
