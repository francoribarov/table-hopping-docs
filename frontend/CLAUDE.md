# CLAUDE.MD

Flutter board-game rental marketplace. FVM `3.44.0` (`.fvmrc`). Dart `^3.8.0`. Package `mobile_table_hopping`.

## Commands

```bash
flutter pub get
dart run build_runner build --delete-conflicting-outputs

make format          # dart format --line-length=80
make format-check
make analyze
make test
flutter test test/path/to/file_test.dart

make pre-pr          # format + analyze + test + codegen check
./scripts/atomic_guardrails.sh
./scripts/atomic_guardrails.sh --strict
```

Coverage: `dart tool/test_and_coverage.dart` (optional `--use-fvm`). Threshold 80% lines in `tool/coverage_thresholds.json`, enforced by `tool/enforce_coverage.dart`.

```bash
flutter run --dart-define=API_BASE_URL=http://<ip>:8000
flutter run --dart-define=APP_LINK_HOSTS=stopper-program-gangrene.ngrok-free.dev,tablehopping.uy
flutter run --dart-define=MIXPANEL_TOKEN=<token>
```

`API_BASE_URL` default: `http://localhost:8000` (Android emulator: `http://10.0.2.2:8000`). Mercado Pago return hosts: `APP_LINK_HOSTS` (also in `.vscode/launch.json`). Mixpanel: `MIXPANEL_TOKEN`.

## Layout

```
lib/
├── core/            # analytics, auth, constants, data, deep_links, di,
│                    # errors, network, notifications, resources, routing,
│                    # security, theme, utils, widgets
├── domain/          # mapper, model, params, repository, service,
│                    # usecase/<feature>/, validators
├── data/            # datasource, dto, mapper, repository, services
└── presentation/    # blocs, gateway, l10n, notifications, pages,
                     # validators, widgets
```

Features: `auth`, `catalog`, `publish`, `rental`, `my_publications`, `my_rentals`, `search`, `reviews`, `announcements`, `rules`, `food_bundle`, `upload`, `notification`, `profile` / `public_profile`.

Widgets:

```
presentation/widgets/
├── atoms/ molecules/ organisms/ templates/
├── rental/            # checkout/rental UI, outside atomic folders
└── announcements/
```

`core/widgets/` also has `atoms/`, `molecules/`, `templates/`. Taxonomy: `docs/atomic-design/widget-taxonomy.md`.

## Conventions

**DI.** `get_it` + `injectable`. Locator: `lib/core/di/injection.dart` (`getIt`). `@injectable` = fresh per page (default). No `@Named`.

`@lazySingleton`: `AuthBloc`, `PendingReviewsBloc`, `DioClient`, `PushMessagingGateway`, `PasswordEncryptor`, `SessionExpiredNotifier`, `PushNotificationCoordinator`, `NotificationNavigationHolder`, `RegisteredFcmTokenStore`.

**Network.** `DioClient` interceptors: Auth → Refresh → Logging. No Error interceptor. Dio errors become `DataException` in `BaseDataSource.getStateOf` (`lib/core/resources/base_data_source.dart`) → `ApiResult<T>`. REST clients: `@RestApi` (Retrofit).

**Data sources.** Extend `BaseDataSource`. All HTTP through `getStateOf<T>(request: () => ...)`. No raw try/catch around service calls.

**Repositories.** Extend `BaseRepository` (`lib/core/data/base_repository.dart`). Use `executeDataSource`, `executeDataSourceList`, `executeDataSourceListMapped`, `executeVoidDataSource`, `toEither`, `unwrapOrThrow`. Return `Future<Either<DomainException, T>>`.

**DTOs.** Top-level responses implement `BaseDtoResponse<T>` with `toDomainModel()`. `@freezed` + `@JsonSerializable(fieldRename: FieldRename.snake)`. Domain stays camelCase.

**BLoC states.**
- copyWith: `CatalogBloc`, `LoginCubit` — one `@freezed` class, `@Default` fields, `isLoading` / `isSubmitting`, `errorMessage`.
- Sealed: `MyRentalsBloc` — `.initial()` / `.loading()` / `.success(...)` / `.failure(message)`.

**UI errors.**
- Inline: `String? errorMessage` → `InlineFeedbackText`.
- API/submit: `FeedbackNotice` in state → `BlocConsumer.listener` → `FeedbackMessenger.showError(context, message: notice.message)` → `clearNotice()`.

**Catch order.** `on DomainException catch (e)` first, then `on Exception`. No string surgery on messages.

**BlocProvider.** Route builders: `BlocProvider(create: (_) => getIt<Bloc>()..add(InitEvent))`. `BlocProvider.value` only for already-owned instances.

**Routing.** `StatefulShellRoute` in `lib/core/routing/app_router.dart`. Navigate via `lib/core/routing/navigation.dart` (`context.goToPublication(id)`, `context.goToLogin(from:)`, …). Widgets must not import `app_router.dart`.

**Atomic guardrails** (`scripts/atomic_guardrails.sh`): templates/organisms do not import blocs, do not `context.read/watch<Bloc>()`, do not `context.go/push`. No raw `ScaffoldMessenger.showSnackBar`. Page `steps/` same bloc rule.

**Tokens.** `AppColors` (`lib/core/theme/app_colors.dart`), `AppTheme.spacing*` (`lib/core/theme/app_theme.dart`), `AppTypography`. Buttons: `AppPrimaryButton` / `AppSecondaryButton` in `lib/presentation/widgets/atoms/common/`.

Format line length 80. Lint: `very_good_analysis` + `flutter_lints` (`lines_longer_than_80_chars` off).

## Skills / agents

Mirrored in `.claude/skills/`, `.cursor/skills/`, `.agents/skills/`:

- `create-bloc/` — BLoC/Cubit patterns
- `flutter-test-developer/` — BLoC, data source, repository tests
- `agent-review/` — multi-agent PR review (`develop` diff)
- `code-review-files/` — file/diff review
- `generate-pr/` — PR description

Review agents: `.cursor/agents/` (`architecture-reviewer`, `bug-reviewer`, `code-quality-reviewer`, `data-layer-reviewer`, `ui-reviewer`, `perf-reviewer`, `pre-pr-validator`, `finding-challenger`, `review-synthesizer`).

## Tests

`test/` mirrors `lib/`. `bloc_test` + `mocktail`. 80% line coverage (`tool/coverage_thresholds.json`) during `make pre-pr`.
