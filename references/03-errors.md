# 03 — Error Handling — AppFailure Hierarchy

## core/error/failures.dart

```dart
import 'package:equatable/equatable.dart';

abstract class AppFailure extends Equatable {
  final String message;
  final int? statusCode;
  const AppFailure(this.message, {this.statusCode});

  @override
  List<Object?> get props => [message, statusCode];
}

class NetworkFailure   extends AppFailure {
  const NetworkFailure([super.message = 'No internet connection']);
}
class ServerFailure    extends AppFailure {
  const ServerFailure(super.message, {super.statusCode});
}
class CacheFailure     extends AppFailure {
  const CacheFailure([super.message = 'Cache error']);
}
class ValidationFailure extends AppFailure {
  const ValidationFailure(super.message);
}
class UnauthorizedFailure extends AppFailure {
  const UnauthorizedFailure([super.message = 'Unauthorized']);
}
class UnexpectedFailure extends AppFailure {
  const UnexpectedFailure([super.message = 'Unexpected error']);
}
```

## core/error/exceptions.dart

```dart
class ServerException implements Exception {
  final String message;
  final int? statusCode;
  const ServerException(this.message, {this.statusCode});
}
class CacheException     implements Exception {
  final String message;
  const CacheException([this.message = 'Cache error']);
}
class NetworkException   implements Exception {
  final String message;
  const NetworkException([this.message = 'No internet connection']);
}
class ValidationException implements Exception {
  final String message;
  const ValidationException(this.message);
}
class UnexpectedException implements Exception {
  final String message;
  const UnexpectedException([this.message = 'Unexpected error']);
}
```

## core/use_cases/base_use_case.dart

```dart
import 'package:dartz/dartz.dart';
import 'package:equatable/equatable.dart';
import '../error/failures.dart';

abstract class UseCase<Type, Params> {
  Future<Either<AppFailure, Type>> call(Params params);
}

class NoParams extends Equatable {
  const NoParams();
  @override List<Object?> get props => [];
}
```

## Error → Failure Mapping في Repository

```dart
Exception _mapToFailure(dynamic error) {
  if (error is ServerException)     return ServerFailure(error.message, statusCode: error.statusCode);
  if (error is CacheException)      return CacheFailure(error.message);
  if (error is NetworkException)    return NetworkFailure(error.message);
  if (error is ValidationException) return ValidationFailure(error.message);
  return UnexpectedFailure(error.toString());
}
```

## Handling في Cubit

```dart
result.fold(
  (failure) {
    // معالجة مركزية للـ Unauthorized
    if (failure is UnauthorizedFailure) {
      AppRouter.toAndRemoveUntil(LoginScreen.routeName);
      return;
    }
    emit(state.copyWith(
      status: FeatureStatus.error,
      failure: failure,
    ));
  },
  (data) => emit(state.copyWith(
    status: FeatureStatus.loaded,
    data: data,
    failure: null,
  )),
);
```

## Showing Error في Screen

```dart
// في BlocListener
case FeatureStatus.error:
  DialogUtils.hideDialog();
  DialogUtils.showErrorDialog(
    message: state.failure?.message ?? 'error'.tr(),
  );
  break;
```
