# BLoC/Cubit Examples — TableHoppingApp

Real-world examples based on actual codebase patterns.

---

## Example 1: Variant A — copyWith State BLoC (CatalogBloc-style)

A BLoC managing multiple data fields with several use cases.

### `catalog_bloc.dart`

```dart
import 'package:flutter_bloc/flutter_bloc.dart';
import 'package:freezed_annotation/freezed_annotation.dart';
import 'package:injectable/injectable.dart';

import 'package:mobile_table_hopping/domain/model/catalog/game.dart';
import 'package:mobile_table_hopping/domain/model/catalog/publication_listing.dart';
import 'package:mobile_table_hopping/domain/usecase/catalog/get_publications_use_case.dart';
import 'package:mobile_table_hopping/domain/usecase/catalog/get_categories_use_case.dart';

part 'catalog_bloc.freezed.dart';
part 'catalog_event.dart';
part 'catalog_state.dart';

@injectable
class CatalogBloc extends Bloc<CatalogEvent, CatalogState> {
  CatalogBloc({
    required GetPublicationsUseCase getPublications,
    required GetCategoriesUseCase getCategories,
  }) : _getPublications = getPublications,
       _getCategories = getCategories,
       super(const CatalogState()) {
    on<_Started>(_onStarted);
    on<_SearchChanged>(_onSearchChanged);
  }

  final GetPublicationsUseCase _getPublications;
  final GetCategoriesUseCase _getCategories;

  Future<void> _onStarted(
    _Started event,
    Emitter<CatalogState> emit,
  ) async {
    emit(state.copyWith(isLoading: true, errorMessage: null));

    final publicationsResult = await _getPublications();

    await publicationsResult.fold<Future<void>>(
      (error) async {
        emit(state.copyWith(
          isLoading: false,
          errorMessage: error.message,
        ));
      },
      (publications) async {
        // Load categories (non-blocking)
        var categories = <GameCategory>[];
        final categoriesResult = await _getCategories();
        categoriesResult.fold(
          (_) {}, // Ignore category errors
          (data) => categories = data,
        );

        emit(state.copyWith(
          isLoading: false,
          publications: publications,
          categories: categories,
        ));
      },
    );
  }

  void _onSearchChanged(
    _SearchChanged event,
    Emitter<CatalogState> emit,
  ) {
    emit(state.copyWith(query: event.query));
  }
}
```

### `catalog_event.dart`

```dart
part of 'catalog_bloc.dart';

@freezed
abstract class CatalogEvent with _$CatalogEvent {
  const factory CatalogEvent.started() = _Started;
  const factory CatalogEvent.searchChanged(String query) =
      _SearchChanged;
}
```

### `catalog_state.dart`

```dart
part of 'catalog_bloc.dart';

@freezed
abstract class CatalogState with _$CatalogState {
  const factory CatalogState({
    @Default([]) List<PublicationListing> publications,
    @Default([]) List<GameCategory> categories,
    @Default('') String query,
    @Default(false) bool isLoading,
    String? errorMessage,
  }) = _CatalogState;

  const CatalogState._();

  bool get isSearchMode => query.trim().isNotEmpty;
}
```

### UI Wiring

```dart
// In app_router.dart
GoRoute(
  path: AppRoutes.home,
  builder: (context, state) => BlocProvider(
    create: (_) => getIt<CatalogBloc>()
      ..add(const CatalogEvent.started()),
    child: const CatalogPage(),
  ),
),

// In catalog_page.dart
BlocBuilder<CatalogBloc, CatalogState>(
  builder: (context, state) {
    if (state.isLoading) {
      return const Center(child: CircularProgressIndicator());
    }
    if (state.errorMessage != null) {
      return ErrorView(message: state.errorMessage!);
    }
    return PublicationGrid(
      publications: state.publications,
    );
  },
)
```

---

## Example 2: Variant B — Sealed Union States BLoC (MyRentalsBloc-style)

A BLoC with clear lifecycle states for a single data-loading concern.

### `my_rentals_bloc.dart`

```dart
import 'package:bloc/bloc.dart';
import 'package:freezed_annotation/freezed_annotation.dart';
import 'package:injectable/injectable.dart';
import 'package:mobile_table_hopping/domain/model/rental/rental.dart';
import 'package:mobile_table_hopping/domain/usecase/rental/get_rentals_use_case.dart';
import 'package:mobile_table_hopping/domain/usecase/rental/cancel_rental_use_case.dart';

part 'my_rentals_event.dart';
part 'my_rentals_state.dart';
part 'my_rentals_bloc.freezed.dart';

@injectable
class MyRentalsBloc extends Bloc<MyRentalsEvent, MyRentalsState> {
  MyRentalsBloc(
    GetRentalsUseCase getMyRentals,
    CancelRentalUseCase cancelRental,
  ) : _getMyRentals = getMyRentals,
      _cancelRental = cancelRental,
      super(const _Initial()) {
    on<_Started>(_onStarted);
    on<_CancelRequested>(_onCancelRequested);
    on<_MessageDismissed>(_onMessageDismissed);
  }

  final GetRentalsUseCase _getMyRentals;
  final CancelRentalUseCase _cancelRental;

  Future<void> _onStarted(
    _Started event,
    Emitter<MyRentalsState> emit,
  ) async {
    emit(const MyRentalsState.loading());

    final result = await _getMyRentals(RentalRole.renter);

    result.fold(
      (error) => emit(MyRentalsState.failure(error.message)),
      (rentals) => emit(MyRentalsState.success(rentals)),
    );
  }

  Future<void> _onCancelRequested(
    _CancelRequested event,
    Emitter<MyRentalsState> emit,
  ) async {
    if (state is! _Success) return;
    final currentState = state as _Success;

    emit(currentState.copyWith(
      processingRentalId: event.rentalId,
    ));

    final result = await _cancelRental(event.rentalId);

    result.fold(
      (error) => emit(currentState.copyWith(
        processingRentalId: null,
        feedbackMessage: error.message,
      )),
      (_) {
        final updated = currentState.rentals.map((rental) {
          if (rental.id == event.rentalId) {
            return rental.copyWith(
              status: RentalStatus.cancelled,
            );
          }
          return rental;
        }).toList();

        emit(currentState.copyWith(
          rentals: updated,
          processingRentalId: null,
          feedbackMessage: 'Solicitud cancelada.',
        ));
      },
    );
  }

  void _onMessageDismissed(
    _MessageDismissed event,
    Emitter<MyRentalsState> emit,
  ) {
    if (state is! _Success) return;
    final currentState = state as _Success;
    emit(currentState.copyWith(feedbackMessage: null));
  }
}
```

### `my_rentals_event.dart`

```dart
part of 'my_rentals_bloc.dart';

@freezed
class MyRentalsEvent with _$MyRentalsEvent {
  const factory MyRentalsEvent.started() = _Started;
  const factory MyRentalsEvent.cancelRequested(String rentalId) =
      _CancelRequested;
  const factory MyRentalsEvent.messageDismissed() =
      _MessageDismissed;
}
```

### `my_rentals_state.dart`

```dart
part of 'my_rentals_bloc.dart';

@freezed
class MyRentalsState with _$MyRentalsState {
  const factory MyRentalsState.initial() = _Initial;
  const factory MyRentalsState.loading() = _Loading;
  const factory MyRentalsState.success(
    List<Rental> rentals, {
    String? processingRentalId,
    String? feedbackMessage,
  }) = _Success;
  const factory MyRentalsState.failure(String message) = _Failure;
}
```

### UI Wiring

```dart
// Sealed union states use .when() in builder
BlocBuilder<MyRentalsBloc, MyRentalsState>(
  builder: (context, state) => state.when(
    initial: () => const SizedBox.shrink(),
    loading: () => const Center(
      child: CircularProgressIndicator(),
    ),
    success: (rentals, processingId, feedback) =>
        RentalsList(
          rentals: rentals,
          processingId: processingId,
        ),
    failure: (message) => ErrorView(message: message),
  ),
)
```

---

## Example 3: Cubit — Form with FeedbackNotice (LoginCubit-style)

A Cubit managing a form with validation and API error feedback.

### `login_cubit.dart`

```dart
import 'package:flutter/foundation.dart';
import 'package:flutter_bloc/flutter_bloc.dart';
import 'package:freezed_annotation/freezed_annotation.dart';
import 'package:injectable/injectable.dart';
import 'package:mobile_table_hopping/core/errors/domain/domain_exception.dart';
import 'package:mobile_table_hopping/domain/usecase/auth/login.dart';
import 'package:mobile_table_hopping/domain/validators/auth/auth_validator.dart';
import 'package:mobile_table_hopping/presentation/blocs/auth/auth_bloc.dart';
import 'package:mobile_table_hopping/presentation/blocs/common/feedback_notice.dart';
import 'package:mobile_table_hopping/presentation/validators/auth_validation_error_mapper.dart';

part 'login_cubit.freezed.dart';
part 'login_state.dart';

@injectable
class LoginCubit extends Cubit<LoginState> {
  LoginCubit({
    required Login login,
    required AuthBloc authBloc,
  }) : _login = login,
       _authBloc = authBloc,
       super(const LoginState());

  final Login _login;
  final AuthBloc _authBloc;

  void emailChanged(String email) {
    emit(state.copyWith(email: email, errorMessage: null));
  }

  void passwordChanged(String password) {
    emit(state.copyWith(
      password: password,
      errorMessage: null,
    ));
  }

  Future<void> submit() async {
    if (state.isSubmitting) return;

    final email = state.email.trim();
    final password = state.password;

    // Client-side validation
    final emailError = AuthValidationErrorMapper.mapEmailError(
      AuthValidator.validateEmail(email),
    );
    if (emailError != null) {
      emit(state.copyWith(errorMessage: emailError));
      return;
    }

    emit(state.copyWith(
      isSubmitting: true,
      errorMessage: null,
      feedbackNotice: null,
    ));

    try {
      final session = await _login(
        email: email,
        password: password,
      );
      emit(state.copyWith(isSubmitting: false, password: ''));
      _authBloc.add(AuthEvent.sessionObtained(session));
    } on DomainException catch (e) {
      if (kDebugMode) debugPrint('LoginCubit: error: $e');
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

### `login_state.dart`

```dart
part of 'login_cubit.dart';

@freezed
abstract class LoginState with _$LoginState {
  const factory LoginState({
    @Default('') String email,
    @Default('') String password,
    @Default(false) bool isSubmitting,
    String? errorMessage,
    FeedbackNotice? feedbackNotice,
  }) = _LoginState;
}
```

### UI Wiring with BlocConsumer

```dart
BlocConsumer<LoginCubit, LoginState>(
  listener: (context, state) {
    final notice = state.feedbackNotice;
    if (notice != null) {
      FeedbackMessenger.showError(context, notice.message);
      context.read<LoginCubit>().clearNotice();
    }
  },
  builder: (context, state) {
    return Column(
      children: [
        TextField(
          onChanged: context.read<LoginCubit>().emailChanged,
          decoration: InputDecoration(
            errorText: state.errorMessage,
          ),
        ),
        SizedBox(height: AppTheme.spacingLg),
        AppPrimaryButton(
          onPressed: state.isSubmitting
              ? null
              : context.read<LoginCubit>().submit,
          isLoading: state.isSubmitting,
          expand: true,
          child: const Text('Log In'),
        ),
      ],
    );
  },
)
```

---

## Key Differences Summary

| Aspect | Variant A (copyWith) | Variant B (sealed union) | Cubit |
|--------|---------------------|------------------------|-------|
| State class | Single factory with `@Default` fields | Multiple named factories | Single factory with `@Default` fields |
| Initial state | `const FeatureState()` | `const _Initial()` | `const FeatureState()` |
| Loading | `state.copyWith(isLoading: true)` | `FeatureState.loading()` | `state.copyWith(isSubmitting: true)` |
| Error handling | `Either.fold()` | `Either.fold()` | `DomainException` catch |
| UI builder | `BlocBuilder` with field checks | `.when()` on state | `BlocConsumer` for side-effects |
| Use case count | Multiple | One primary | One or few |
| Events | Yes (event file) | Yes (event file) | No (direct methods) |
| Best for | Forms, filters, dashboards | Load/display/CRUD lists | Simple forms, toggles |
