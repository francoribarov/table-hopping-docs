# Code Review Files

Reviews Python/FastAPI files against TableHopping backend standards.

---

## Process

1. **Identify files to review** — user-attached or pasted code
2. **Delegate** to the `code-quality-reviewer` subagent with file contents
3. **Apply** backend review standards per `CHECKLIST.md` and `CLAUDE.md`
4. **Return** structured review output

## Output Format

```text
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
| Clean Architecture boundaries | Pass/Fail |
| Async I/O usage | Pass/Fail |
| Type hints (mypy strict) | Pass/Fail |
| Error handling/domain exceptions | Pass/Fail |
| Lint/format readiness (ruff) | Pass/Fail |

## Positive Notes
What the code does well.
```

## Reference

- **Checklist:** `CHECKLIST.md` (this directory)
- **Full conventions:** `CLAUDE.md` (project root)
- **Dependency injection:** `app/api/service_dependencies.py`, `app/api/repository_dependencies.py`
- **Error mapping:** `app/api/exception_mappers.py`, `app/core/exceptions.py`
- **Core architecture entrypoint:** `app/main.py`
