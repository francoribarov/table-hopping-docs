# Create BLoC / Cubit

Comprehensive guide for creating BLoCs and Cubits in TableHoppingApp following Clean Architecture.

---

## Quick-Start Decision Tree

```
Need new state management?
├── Complex event-driven logic? → Bloc
│   ├── Simple load/display flow? → Variant B (sealed union states)
│   └── Multi-concern / form-heavy? → Variant A (copyWith state)
└── Simple form / imperative flow? → Cubit
```

**When to choose:**
- **Bloc (Variant A — copyWith):** Multiple use cases, many state fields, form inputs, filters, combined loading states. Examples: `CatalogBloc`, `PublishBloc`.
- **Bloc (Variant B — sealed union):** Single primary operation with clear initial/loading/success/failure lifecycle. Examples: `MyRentalsBloc`, `MyPublicationsBloc`.
- **Cubit:** Simple forms, direct method calls, no complex event streams. Examples: `LoginCubit`, `RegisterCubit`, `ProfileCubit`.

---

## File Structure

```
lib/presentation/blocs/<feature>/
├── <feature>_bloc.dart          # Main file (imports, class, handlers)
├── <feature>_event.dart         # Events (part of main file)
├── <feature>_state.dart         # State (part of main file)
└── <feature>_bloc.freezed.dart  # Generated
```

For Cubits, omit the event file:
```
lib/presentation/blocs/<feature>/
├── <feature>_cubit.dart
├── <feature>_state.dart         # part of cubit file
└── <feature>_cubit.freezed.dart
```

---

## Variant A: copyWith State (Multi-Concern BLoC)

Use when the BLoC manages multiple data fields, forms, or parallel concerns.

### State Definition

```dart
part of '<feature>_bloc.dart';

@freezed
abstract class FeatureState with _$FeatureState {
  const factory FeatureState({
    @Default([]) List<Item> items,
    @Default('') String query,
    @Default(false) bool isLoading,
    String? errorMessage,
    FeedbackNotice? feedbackNotice,
  }) = _FeatureState;

  // Add const constructor if you need computed getters:
  // const FeatureState._();
  // bool get hasItems => items.isNotEmpty;
}
```

**Key conventions:**
- `@Default(...)` for all fields with sensible defaults
- `String? errorMessage` for inline field-level validation errors
- `FeedbackNotice? feedbackNotice` for API/submit errors shown via snackbar
- Use `const FeatureState._()` only when adding computed getters

### Event Definition

```dart
part of '<feature>_bloc.dart';

@freezed
abstract class FeatureEvent with _$FeatureEvent {
  const factory FeatureEvent.started() = _Started;
  const factory FeatureEvent.refresh() = _Refresh;
  const factory FeatureEvent.itemSelected(String id) = _ItemSelected;
}
```

**Naming:** Use private factory constructors (`_Started`) for internal events. Use public names (`LoadGames`) when UI code references the event type directly.

### Main BLoC File

```dart
import 'package:flutter_bloc/flutter_bloc.dart';
import 'package:freezed_annotation/freezed_annotation.dart';
import 'package:injectable/injectable.dart';

// Domain imports...

part '<feature>_bloc.freezed.dart';
part '<feature>_event.dart';
part '<feature>_state.dart';

@injectable
class FeatureBloc extends Bloc<FeatureEvent, FeatureState> {
  FeatureBloc({
    required GetItemsUseCase getItems,
  }) : _getItems = getItems,
       super(const FeatureState()) {
    on<_Started>(_onStarted);
  }

  final GetItemsUseCase _getItems;

  Future<void> _onStarted(
    _Started event,
    Emitter<FeatureState> emit,
  ) async {
    emit(state.copyWith(isLoading: true, errorMessage: null));

    final result = await _getItems();

    result.fold(
      (error) => emit(
        state.copyWith(
          isLoading: false,
          errorMessage: error.message,
        ),
      ),
      (items) => emit(
        state.copyWith(
          isLoading: false,
          items: items,
        ),
      ),
    );
  }
}
```

---

## Variant B: Sealed Union States (Load/Display BLoC)

Use when the BLoC has a clear lifecycle: initial → loading → success/failure.

### State Definition

```dart
part of '<feature>_bloc.dart';

@freezed
class FeatureState with _$FeatureState {
  const factory FeatureState.initial() = _Initial;
  const factory FeatureState.loading() = _Loading;
  const factory FeatureState.success(
    List<Item> items, {
    String? feedbackMessage,
  }) = _Success;
  const factory FeatureState.failure(String message) = _Failure;
}
```

### Main BLoC File

```dart
@injectable
class FeatureBloc extends Bloc<FeatureEvent, FeatureState> {
  FeatureBloc(
    GetItemsUseCase getItems,
  ) : _getItems = getItems,
      super(const _Initial()) {
    on<_Started>(_onStarted);
  }

  final GetItemsUseCase _getItems;

  Future<void> _onStarted(
    _Started event,
    Emitter<FeatureState> emit,
  ) async {
    emit(const FeatureState.loading());

    final result = await _getItems();

    result.fold(
      (error) => emit(FeatureState.failure(error.message)),
      (items) => emit(FeatureState.success(items)),
    );
  }
}
```

**Updating within a success state:**

```dart
if (state is! _Success) return;
final currentState = state as _Success;
emit(currentState.copyWith(feedbackMessage: 'Updated'));
```

---

## Cubit Pattern (Form / Imperative Flows)

Use for simpler state machines without complex event processing.

### State Definition

```dart
part of '<feature>_cubit.dart';

@freezed
abstract class FeatureState with _$FeatureState {
  const factory FeatureState({
    @Default('') String fieldOne,
    @Default('') String fieldTwo,
    @Default(false) bool isSubmitting,
    String? errorMessage,
    FeedbackNotice? feedbackNotice,
  }) = _FeatureState;
}
```

### Main Cubit File

```dart
@injectable
class FeatureCubit extends Cubit<FeatureState> {
  FeatureCubit({
    required SubmitUseCase submit,
  }) : _submit = submit,
       super(const FeatureState());

  final SubmitUseCase _submit;

  void fieldOneChanged(String value) {
    emit(state.copyWith(fieldOne: value, errorMessage: null));
  }

  Future<void> submit() async {
    if (state.isSubmitting) return;

    // Validate
    if (state.fieldOne.isEmpty) {
      emit(state.copyWith(errorMessage: 'Field is required'));
      return;
    }

    emit(state.copyWith(
      isSubmitting: true,
      errorMessage: null,
      feedbackNotice: null,
    ));

    try {
      await _submit(field: state.fieldOne);
      emit(state.copyWith(isSubmitting: false));
    } on DomainException catch (e) {
      emit(state.copyWith(
        isSubmitting: false,
        feedbackNotice: FeedbackNotice(
          message: e.message,
          severity: FeedbackSeverity.error,
        ),
      ));
    } on Exception {
      emit(state.copyWith(
        isSubmitting: false,
        feedbackNotice: const FeedbackNotice(
          message: 'Something went wrong. Please try again.',
          severity: FeedbackSeverity.error,
        ),
      ));
    }
  }

  void clearNotice() {
    emit(state.copyWith(feedbackNotice: null));
  }
}
```

---

## Error Handling Patterns

### Standard: `Either.fold()` (preferred)

Use cases return `Future<Either<DomainException, T>>`. BLoC folds the result:

```dart
final result = await _useCase();
result.fold(
  (error) => emit(state.copyWith(errorMessage: error.message)),
  (data) => emit(state.copyWith(items: data)),
);
```

### Alternative: `DomainException` catch (imperative flows)

Some use cases throw `DomainException` instead of returning Either (e.g., auth flows with session persistence). Handle with try/catch:

```dart
try {
  final session = await _login(email: email, password: password);
  emit(state.copyWith(isSubmitting: false));
  _authBloc.add(AuthEvent.sessionObtained(session));
} on DomainException catch (e) {
  emit(state.copyWith(
    isSubmitting: false,
    feedbackNotice: FeedbackNotice(
      message: e.message,
      severity: FeedbackSeverity.error,
    ),
  ));
} on Exception {
  emit(state.copyWith(
    isSubmitting: false,
    feedbackNotice: const FeedbackNotice(
      message: 'An unexpected error occurred.',
      severity: FeedbackSeverity.error,
    ),
  ));
}
```

**Rule:** Always catch `on DomainException` first, then `on Exception` as fallback. Never do string surgery on exception messages.

### FeedbackNotice Pattern (User-Facing Errors)

1. State holds `FeedbackNotice? feedbackNotice`
2. `BlocConsumer.listener` shows snackbar via `FeedbackMessenger.showError()`
3. Listener calls `clearNotice()` after displaying

```dart
BlocConsumer<FeatureCubit, FeatureState>(
  listener: (context, state) {
    final notice = state.feedbackNotice;
    if (notice != null) {
      FeedbackMessenger.showError(context, notice.message);
      context.read<FeatureCubit>().clearNotice();
    }
  },
  builder: (context, state) {
    // Build UI...
  },
)
```

---

## Dependency Injection

| Scope | Annotation | When |
|-------|-----------|------|
| Fresh per page | `@injectable` | Most BLoCs/Cubits |
| App-wide singleton | `@lazySingleton` | AuthBloc only |

Constructor injection for all use cases:

```dart
@injectable
class FeatureBloc extends Bloc<FeatureEvent, FeatureState> {
  FeatureBloc({
    required GetItemsUseCase getItems,
    required DeleteItemUseCase deleteItem,
  }) : _getItems = getItems,
       _deleteItem = deleteItem,
       super(const FeatureState()) {
    // Register handlers...
  }

  final GetItemsUseCase _getItems;
  final DeleteItemUseCase _deleteItem;
}
```

After creating, run codegen: `dart run build_runner build --delete-conflicting-outputs`

---

## UI Wiring

### Route-Level Provider (preferred)

```dart
GoRoute(
  path: AppRoutes.feature,
  builder: (context, state) => BlocProvider(
    create: (_) => getIt<FeatureBloc>()
      ..add(const FeatureEvent.started()),
    child: const FeaturePage(),
  ),
),
```

### BlocConsumer vs BlocBuilder

- **`BlocConsumer`** — When you need both side-effects (snackbar, navigation) AND UI rebuilds.
- **`BlocBuilder`** — When you only need UI rebuilds (no side-effects).

```dart
// BlocBuilder with buildWhen for performance
BlocBuilder<FeatureBloc, FeatureState>(
  buildWhen: (prev, curr) => prev.items != curr.items,
  builder: (context, state) {
    if (state.isLoading) return const LoadingWidget();
    return ItemList(items: state.items);
  },
)
```

For sealed union states, use `.when()` or `.maybeWhen()`:

```dart
BlocBuilder<FeatureBloc, FeatureState>(
  builder: (context, state) => state.when(
    initial: () => const SizedBox.shrink(),
    loading: () => const LoadingWidget(),
    success: (items, _) => ItemList(items: items),
    failure: (message) => ErrorWidget(message: message),
  ),
)
```

---

## Validation Checklist

Before finishing a new BLoC/Cubit, verify:

- [ ] `@freezed` on all state and event classes
- [ ] `part` directives for `.freezed.dart`, event, and state files
- [ ] `@injectable` or `@lazySingleton` annotation on BLoC/Cubit class
- [ ] Constructor injects use cases (not repositories or data sources)
- [ ] `Either.fold()` or `DomainException` catch for error handling
- [ ] `FeedbackNotice` for user-facing API errors (if applicable)
- [ ] `clearNotice()` method (if using FeedbackNotice)
- [ ] `BlocProvider(create:)` in route builder (not `.value`)
- [ ] Codegen run: `dart run build_runner build --delete-conflicting-outputs`
- [ ] `make analyze` passes
