# Code Review Files

Reviews code files against TableHoppingApp standards.

---

## Process

1. **Identify files to review** — user-attached or pasted code
2. **Delegate** to the `code-quality-reviewer` subagent with file contents
3. **Apply** full Flutter and TableHopping review standards per `CHECKLIST.md` and `CLAUDE.md`
4. **Return** structured review output

## Output Format

```
## Summary
Brief overview of the reviewed code.

## Critical
Issues that must be fixed before merge.

## Suggestions
Improvements that would enhance quality.

## Nice to Have
Minor improvements, style preferences.

## Compliance
| Check | Status |
|-------|--------|
| Design tokens (AppColors, AppTheme) | Pass/Fail |
| Layer boundaries | Pass/Fail |
| State management patterns | Pass/Fail |
| DI annotations | Pass/Fail |
| Line length (80 chars) | Pass/Fail |

## Positive Notes
What the code does well.
```

## Reference

- **Checklist:** `CHECKLIST.md` (this directory)
- **Full conventions:** `CLAUDE.md` (project root)
- **Design tokens:** `lib/core/theme/app_colors.dart`, `lib/core/theme/app_theme.dart`, `lib/core/theme/app_typography.dart`
- **Base classes:** `lib/core/data/base_repository.dart`, `lib/core/resources/base_data_source.dart`
- **Error handling:** `lib/core/errors/domain/domain_exception.dart`, `lib/presentation/blocs/common/feedback_notice.dart`
