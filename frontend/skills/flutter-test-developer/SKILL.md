# Flutter Test Developer — TableHoppingApp

Senior Flutter testing engineer guide. Covers unit, BLoC/Cubit, widget, and data layer tests.

---

## Project Conventions

- **Run tests:** `flutter test` or `make test`
- **Single file:** `flutter test test/path/to/file_test.dart`
- **Pre-PR validation:** `make pre-pr` (format + analyze + test + codegen check)
- **Linting:** `very_good_analysis` + `flutter_lints` (80-char line length)
- **Test structure:** `test/` mirrors `lib/` — e.g., `lib/presentation/blocs/catalog/` → `test/presentation/blocs/catalog/`

---

## Choosing Test Type

| What to test | Test type | Tools |
|-------------|-----------|-------|
| Use cases, validators, domain logic | Unit test | `flutter_test`, `mocktail` |
| BLoC/Cubit state transitions | BLoC test | `bloc_test`, `mocktail` |
| Data sources (API wrapping) | Unit test | `mocktail` (mock Dio/Service) |
| Repositories (DTO → domain mapping) | Unit test | `mocktail` (mock DataSource) |
| Widget rendering, interaction | Widget test | `flutter_test`, `mocktail`, Fake BLoC |
| DTO serialization | Unit test | `flutter_test` (manual JSON) |

---

## Core Patterns

### Arrange-Act-Assert

```dart
test('should return publications when successful', () async {
  // Arrange
  when(() => mockUseCase()).thenAnswer(
    (_) async => Right(testPublications),
  );

  // Act
  final result = await mockUseCase();

  // Assert
  expect(result, isA<Right<DomainException, List<Publication>>>());
});
```

### Mocking with mocktail

```dart
// Mock use cases
class MockGetPublicationsUseCase extends Mock
    implements GetPublicationsUseCase {}

// Mock repositories
class MockPublishRepository extends Mock
    implements PublishRepository {}

// Mock data sources
class MockCatalogRemoteDataSource extends Mock
    implements CatalogRemoteDataSource {}
```

### Either Stubbing

```dart
// Success
when(() => mockUseCase()).thenAnswer(
  (_) async => Right(expectedData),
);

// Failure
when(() => mockUseCase()).thenAnswer(
  (_) async => Left(
    DomainException(message: 'Server error'),
  ),
);

// Void success
when(() => mockUseCase(any())).thenAnswer(
  (_) async => const Right(null),
);
```

---

## BLoC/Cubit Testing

Use `bloc_test` package with `blocTest<Bloc, State>()`.

### Variant A — copyWith State BLoC

```dart
import 'package:bloc_test/bloc_test.dart';
import 'package:dartz/dartz.dart';
import 'package:flutter_test/flutter_test.dart';
import 'package:mocktail/mocktail.dart';

class MockGetPublicationsUseCase extends Mock
    implements GetPublicationsUseCase {}

void main() {
  late MockGetPublicationsUseCase mockGetPublications;

  setUp(() {
    mockGetPublications = MockGetPublicationsUseCase();
  });

  group('CatalogBloc', () {
    blocTest<CatalogBloc, CatalogState>(
      'emits loading then success when loadGames succeeds',
      setUp: () {
        when(() => mockGetPublications()).thenAnswer(
          (_) async => Right(testPublications),
        );
      },
      build: () => CatalogBloc(
        getPublications: mockGetPublications,
      ),
      act: (bloc) => bloc.add(
        const CatalogEvent.started(),
      ),
      expect: () => [
        const CatalogState(isLoading: true),
        CatalogState(
          isLoading: false,
          publications: testPublications,
        ),
      ],
    );

    blocTest<CatalogBloc, CatalogState>(
      'emits error when loadGames fails',
      setUp: () {
        when(() => mockGetPublications()).thenAnswer(
          (_) async => Left(
            DomainException(message: 'Network error'),
          ),
        );
      },
      build: () => CatalogBloc(
        getPublications: mockGetPublications,
      ),
      act: (bloc) => bloc.add(
        const CatalogEvent.started(),
      ),
      expect: () => [
        const CatalogState(isLoading: true),
        const CatalogState(
          isLoading: false,
          errorMessage: 'Network error',
        ),
      ],
    );
  });
}
```

### Variant B — Sealed Union States BLoC

```dart
blocTest<MyRentalsBloc, MyRentalsState>(
  'emits loading then success with rentals',
  setUp: () {
    when(() => mockGetRentals(any())).thenAnswer(
      (_) async => Right(testRentals),
    );
  },
  build: () => MyRentalsBloc(
    mockGetRentals,
    mockCancelRental,
  ),
  act: (bloc) => bloc.add(
    const MyRentalsEvent.started(),
  ),
  expect: () => [
    const MyRentalsState.loading(),
    MyRentalsState.success(testRentals),
  ],
);

blocTest<MyRentalsBloc, MyRentalsState>(
  'emits failure when loading fails',
  setUp: () {
    when(() => mockGetRentals(any())).thenAnswer(
      (_) async => Left(
        DomainException(message: 'Server error'),
      ),
    );
  },
  build: () => MyRentalsBloc(
    mockGetRentals,
    mockCancelRental,
  ),
  act: (bloc) => bloc.add(
    const MyRentalsEvent.started(),
  ),
  expect: () => [
    const MyRentalsState.loading(),
    const MyRentalsState.failure('Server error'),
  ],
);
```

### Cubit Testing

```dart
blocTest<LoginCubit, LoginState>(
  'emits submitting then feedbackNotice on DomainException',
  setUp: () {
    when(
      () => mockLogin(
        email: any(named: 'email'),
        password: any(named: 'password'),
      ),
    ).thenThrow(
      DomainException(message: 'Invalid credentials'),
    );
  },
  build: () => LoginCubit(
    login: mockLogin,
    authBloc: mockAuthBloc,
  ),
  seed: () => const LoginState(
    email: 'test@email.com',
    password: 'password123',
  ),
  act: (cubit) => cubit.submit(),
  expect: () => [
    isA<LoginState>()
        .having((s) => s.isSubmitting, 'isSubmitting', true),
    isA<LoginState>()
        .having((s) => s.isSubmitting, 'isSubmitting', false)
        .having(
          (s) => s.feedbackNotice?.message,
          'feedbackMessage',
          'Invalid credentials',
        ),
  ],
);
```

### Test Lifecycle

```dart
late FeatureBloc bloc;

setUp(() {
  bloc = FeatureBloc(getItems: mockGetItems);
});

tearDown(() {
  bloc.close(); // CRITICAL: always close BLoC
});
```

---

## Data Layer Testing

### Data Source Tests (BaseDataSource.getStateOf)

```dart
class MockUserService extends Mock implements UserService {}

void main() {
  late UserRemoteDataSourceImpl dataSource;
  late MockUserService mockService;

  setUp(() {
    mockService = MockUserService();
    dataSource = UserRemoteDataSourceImpl(mockService);
  });

  test('returns ApiResult.success when service succeeds', () async {
    when(() => mockService.getUser(any())).thenAnswer(
      (_) async => testUserDto,
    );

    final result = await dataSource.getUser('123');

    expect(
      result,
      isA<ApiResult<UserDto>>().having(
        (r) => r.whenOrNull(success: (data) => data),
        'data',
        testUserDto,
      ),
    );
  });

  test('returns ApiResult.failure on DioException', () async {
    when(() => mockService.getUser(any())).thenThrow(
      DioException(
        requestOptions: RequestOptions(),
        type: DioExceptionType.connectionError,
      ),
    );

    final result = await dataSource.getUser('123');

    expect(
      result,
      isA<ApiResult<UserDto>>().having(
        (r) => r.whenOrNull(
          failure: (error) => error,
        ),
        'error',
        isA<DataException>(),
      ),
    );
  });
}
```

### Repository Tests (BaseRepository.executeDataSource)

```dart
class MockUserRemoteDataSource extends Mock
    implements UserRemoteDataSource {}

void main() {
  late UserRepositoryImpl repository;
  late MockUserRemoteDataSource mockDataSource;

  setUp(() {
    mockDataSource = MockUserRemoteDataSource();
    repository = UserRepositoryImpl(mockDataSource);
  });

  test('returns Right with domain model on success', () async {
    when(() => mockDataSource.getUser(any())).thenAnswer(
      (_) async => ApiResult.success(data: testUserDto),
    );

    final result = await repository.getUser('123');

    expect(result, isA<Right<DomainException, User>>());
    result.fold(
      (_) => fail('Expected Right'),
      (user) => expect(user.id, '123'),
    );
  });

  test('returns Left with DomainException on failure', () async {
    when(() => mockDataSource.getUser(any())).thenAnswer(
      (_) async => ApiResult.failure(
        dataException: DataException(message: 'Not found'),
      ),
    );

    final result = await repository.getUser('123');

    expect(result, isA<Left<DomainException, User>>());
  });
}
```

---

## Widget Testing

### Fake BLoC Pattern

Use `MockCubit` / `MockBloc` from `bloc_test` for widget tests:

```dart
import 'package:bloc_test/bloc_test.dart';
import 'package:flutter_bloc/flutter_bloc.dart';
import 'package:flutter_test/flutter_test.dart';

class MockLoginCubit extends MockCubit<LoginState>
    implements LoginCubit {}

void main() {
  late MockLoginCubit mockCubit;

  setUp(() {
    mockCubit = MockLoginCubit();
  });

  testWidgets('shows error message', (tester) async {
    when(() => mockCubit.state).thenReturn(
      const LoginState(errorMessage: 'Invalid email'),
    );

    await tester.pumpWidget(
      MaterialApp(
        home: BlocProvider<LoginCubit>.value(
          value: mockCubit,
          child: const LoginPage(),
        ),
      ),
    );

    expect(find.text('Invalid email'), findsOneWidget);
  });

  testWidgets('calls submit on button tap', (tester) async {
    when(() => mockCubit.state).thenReturn(
      const LoginState(
        email: 'test@email.com',
        password: 'pass123',
      ),
    );

    await tester.pumpWidget(
      MaterialApp(
        home: BlocProvider<LoginCubit>.value(
          value: mockCubit,
          child: const LoginPage(),
        ),
      ),
    );

    await tester.tap(find.text('Log In'));
    verify(() => mockCubit.submit()).called(1);
  });
}
```

### Widget Test with Sealed Union States

```dart
class MockMyRentalsBloc extends MockBloc<MyRentalsEvent, MyRentalsState>
    implements MyRentalsBloc {}

testWidgets('shows rentals list on success', (tester) async {
  when(() => mockBloc.state).thenReturn(
    MyRentalsState.success(testRentals),
  );

  await tester.pumpWidget(
    MaterialApp(
      home: BlocProvider<MyRentalsBloc>.value(
        value: mockBloc,
        child: const MyRentalsPage(),
      ),
    ),
  );

  expect(find.byType(RentalCard), findsNWidgets(testRentals.length));
});
```

---

## DTO Serialization Testing

Test JSON round-trip with manual JSON construction:

```dart
test('UserDto.fromJson maps correctly', () {
  final json = <String, dynamic>{
    'id': '123',
    'email': 'test@email.com',
    'first_name': 'John',  // snake_case from API
    'last_name': 'Doe',
  };

  final dto = UserDto.fromJson(json);

  expect(dto.id, '123');
  expect(dto.email, 'test@email.com');
  expect(dto.firstName, 'John');
});

test('UserDto.toDomainModel maps to User', () {
  final dto = UserDto(
    id: '123',
    email: 'test@email.com',
    firstName: 'John',
    lastName: 'Doe',
  );

  final user = dto.toDomainModel();

  expect(user, isA<User>());
  expect(user.id, '123');
  expect(user.fullName, 'John Doe');
});
```

---

## Reliability Guidelines

- Always use `const` constructors in test fixtures when possible
- Use `setUp` / `tearDown` for consistent test isolation
- **Always close BLoCs** in `tearDown`
- Prefer `blocTest` over manual stream subscriptions
- Use `seed` parameter in `blocTest` to set initial state for specific scenarios
- Use `.having()` matchers for partial state assertions
- Register fallback values for custom types: `registerFallbackValue(MyType())`
