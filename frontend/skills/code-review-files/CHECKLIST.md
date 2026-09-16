# Code Review Checklist — TableHoppingApp

Quick-reference checklist. Full authority in `CLAUDE.md`.

---

## Architecture

- [ ] **Layer boundaries:** `presentation → domain ← data`, cross-cutting in `core/`
- [ ] **Domain purity:** No Flutter imports in `lib/domain/` (pure Dart only)
- [ ] **Presentation uses use cases only** — never repositories or data sources directly
- [ ] **DTOs and mapping stay in data layer** — never leak into domain or presentation
- [ ] **New code uses layer-first structure** — `core/`, `domain/`, `data/`, `presentation/`

## State Management

- [ ] **Freezed:** All BLoC states and events use `@freezed` sealed classes
- [ ] **Error handling:** `Either.fold()` (standard) or `DomainException` catch (imperative flows)
- [ ] **Catch order:** `on DomainException catch (e)` first, then `on Exception` fallback
- [ ] **FeedbackNotice:** API/submit errors use `FeedbackNotice` in state → `BlocConsumer.listener` → `FeedbackMessenger.showError` → `clearNotice()`
- [ ] **Field validation:** Inline `String? errorMessage` in state for form field errors
- [ ] **BlocProvider:** `BlocProvider(create: (_) => getIt<Bloc>())` — never `.value` for new instances

## Data & DI

- [ ] **DI annotations:** `@injectable` (per-page) or `@lazySingleton` (app-wide). No `@Named`
- [ ] **Data sources:** Extend `BaseDataSource`, all HTTP calls through `getStateOf()`
- [ ] **Repositories:** Extend `BaseRepository`, use `executeDataSource*()` / `executeVoidDataSource()` / `toEither()`
- [ ] **DTOs:** Implement `BaseDtoResponse<T>` with `toDomainModel()` method
- [ ] **Services:** Retrofit `@RestApi` with proper annotations
- [ ] **Use cases:** Single-responsibility, one class per business operation in `lib/domain/usecase/<feature>/`

## Design Tokens

- [ ] **Colors:** `AppColors.*` only — never raw `Colors.*` (except `Colors.transparent`)
- [ ] **Spacing:** `AppTheme.spacing*` tokens (spacingXs through spacing4xl) — never hardcoded px in `SizedBox`, `EdgeInsets`, or padding
- [ ] **Radii:** `AppTheme.radius*` tokens — cards use `radiusXl` (20), inputs `radiusLg` (16), buttons `radius2xl` (24)
- [ ] **Buttons:** `AppPrimaryButton` / `AppSecondaryButton` — never raw `ElevatedButton.styleFrom()` / `OutlinedButton.styleFrom()`
- [ ] **Typography:** `AppTypography.*` or `Theme.of(context).textTheme.*` — never inline `TextStyle()`
- [ ] **Shadows:** `AppTheme.shadowSm/Md/Lg/Up` — prefer border-based separation over Material shadows

## Quality

- [ ] **Line length:** 80 characters (enforced by `dart format`)
- [ ] **Linting:** `very_good_analysis` + `flutter_lints` — `make analyze` passes
- [ ] **No dead code:** Remove unused files, imports, variables
- [ ] **No `print()`:** Use `debugPrint()` guarded by `kDebugMode`, or remove
- [ ] **No hardcoded strings:** User-facing text should be localizable
- [ ] **Codegen clean:** `dart run build_runner build --delete-conflicting-outputs` runs without errors
