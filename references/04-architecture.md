# 04 — Feature Architecture — Full Code Templates

## State — sentinel copyWith pattern

```dart
import 'package:equatable/equatable.dart';
import '../../../../core/error/failures.dart';

const _unchanged = Object(); // sentinel

enum FeatureStatus { initial, initialLoading, loaded, loading, actionCompleted, error }

class FeatureState extends Equatable {
  final FeatureStatus status;
  final List<ItemType> data;
  final List<ItemType>? filteredData;
  final AppFailure? failure;
  final String? successMessage;
  final bool isActionLoading;
  final bool isRefreshing;
  final bool isLoadingMore;
  final bool hasMoreData;
  final bool isSearching;
  final String searchQuery;
  final DateTime? lastUpdated;
  final Set<String> selectedItems;

  const FeatureState._({
    required this.status,
    required this.data,
    this.filteredData,
    this.failure,
    this.successMessage,
    required this.isActionLoading,
    required this.isRefreshing,
    required this.isLoadingMore,
    required this.hasMoreData,
    required this.isSearching,
    required this.searchQuery,
    this.lastUpdated,
    required this.selectedItems,
  });

  const FeatureState.initial()
      : this._(
          status: FeatureStatus.initial,
          data: const [],
          isActionLoading: false,
          isRefreshing: false,
          isLoadingMore: false,
          hasMoreData: true,
          isSearching: false,
          searchQuery: '',
          selectedItems: const {},
        );

  FeatureState copyWith({
    FeatureStatus? status,
    List<ItemType>? data,
    Object? filteredData  = _unchanged,
    Object? failure       = _unchanged,
    Object? successMessage = _unchanged,
    bool? isActionLoading,
    bool? isRefreshing,
    bool? isLoadingMore,
    bool? hasMoreData,
    bool? isSearching,
    String? searchQuery,
    DateTime? lastUpdated,
    Set<String>? selectedItems,
  }) {
    return FeatureState._(
      status: status ?? this.status,
      data: data ?? this.data,
      filteredData:   filteredData   == _unchanged ? this.filteredData   : filteredData   as List<ItemType>?,
      failure:        failure        == _unchanged ? this.failure        : failure        as AppFailure?,
      successMessage: successMessage == _unchanged ? this.successMessage : successMessage as String?,
      isActionLoading: isActionLoading ?? this.isActionLoading,
      isRefreshing:    isRefreshing    ?? this.isRefreshing,
      isLoadingMore:   isLoadingMore   ?? this.isLoadingMore,
      hasMoreData:     hasMoreData     ?? this.hasMoreData,
      isSearching:     isSearching     ?? this.isSearching,
      searchQuery:     searchQuery     ?? this.searchQuery,
      lastUpdated:     lastUpdated     ?? this.lastUpdated,
      selectedItems:   selectedItems   ?? this.selectedItems,
    );
  }

  List<ItemType> get displayData => filteredData ?? data;
  bool get hasData               => data.isNotEmpty;
  bool get isFiltered            => filteredData != null;
  bool get isLoading             =>
      status == FeatureStatus.initialLoading ||
      status == FeatureStatus.loading ||
      isRefreshing || isLoadingMore || isSearching;

  @override
  List<Object?> get props => [
    status, data, filteredData, failure, successMessage,
    isActionLoading, isRefreshing, isLoadingMore, hasMoreData,
    isSearching, searchQuery, lastUpdated, selectedItems,
  ];
}
```

---

## Cubit

```dart
import 'package:flutter_bloc/flutter_bloc.dart';
import '../../../../core/error/failures.dart';
import '../../../../core/use_cases/base_use_case.dart';
import '../../domain/use_cases/fetch_data_use_case.dart';
import '../../domain/use_cases/create_item_use_case.dart';
import '../../domain/use_cases/delete_item_use_case.dart';
import '../../dto/feature_dto.dart';
import 'feature_state.dart';

class FeatureCubit extends Cubit<FeatureState> {
  final FetchDataUseCase  _fetchUseCase;
  final CreateItemUseCase _createUseCase;
  final DeleteItemUseCase _deleteUseCase;

  FeatureCubit({
    required FetchDataUseCase fetchDataUseCase,
    required CreateItemUseCase createItemUseCase,
    required DeleteItemUseCase deleteItemUseCase,
  })  : _fetchUseCase  = fetchDataUseCase,
        _createUseCase = createItemUseCase,
        _deleteUseCase = deleteItemUseCase,
        super(const FeatureState.initial());

  Future<void> fetchInitialData() async {
    if (!state.hasData) {
      emit(state.copyWith(status: FeatureStatus.initialLoading));
    }
    final result = await _fetchUseCase.call(const NoParams());
    result.fold(
      _handleFailure,
      (data) => emit(state.copyWith(
        status: FeatureStatus.loaded,
        data: data,
        failure: null,
        lastUpdated: DateTime.now(),
      )),
    );
  }

  Future<void> refreshData() async {
    emit(state.copyWith(isRefreshing: true));
    final result = await _fetchUseCase.call(const NoParams());
    result.fold(
      (failure) => emit(state.copyWith(isRefreshing: false, failure: failure)),
      (data) => emit(state.copyWith(
        status: FeatureStatus.loaded,
        data: data,
        failure: null,
        isRefreshing: false,
        lastUpdated: DateTime.now(),
      )),
    );
  }

  Future<void> loadMoreData() async {
    if (state.isLoadingMore || !state.hasMoreData) return;
    emit(state.copyWith(isLoadingMore: true));
    const limit = 20;
    final result = await _fetchUseCase.call(FetchParams(
      offset: state.data.length,
      limit: limit,
    ));
    result.fold(
      (failure) => emit(state.copyWith(isLoadingMore: false, failure: failure)),
      (newData) => emit(state.copyWith(
        data: [...state.data, ...newData],
        isLoadingMore: false,
        hasMoreData: newData.length == limit,
        failure: null,
      )),
    );
  }

  Future<void> createItem(FeatureDto dto) async {
    emit(state.copyWith(status: FeatureStatus.loading, isActionLoading: true));
    final result = await _createUseCase.call(dto);
    result.fold(
      _handleFailure,
      (newItem) => emit(state.copyWith(
        status: FeatureStatus.actionCompleted,
        data: [newItem, ...state.data],
        successMessage: 'created_successfully',
        failure: null,
        isActionLoading: false,
      )),
    );
  }

  Future<void> deleteItem(String itemId) async {
    final index = state.data.indexWhere((i) => i.id == itemId);
    if (index == -1) return;
    final deleted = state.data[index];
    emit(state.copyWith(
      data: List.from(state.data)..removeAt(index),
      isActionLoading: true,
    ));
    final result = await _deleteUseCase.call(itemId);
    result.fold(
      (failure) => emit(state.copyWith(
        data: List.from(state.data)..insert(index, deleted),
        status: FeatureStatus.error,
        failure: failure,
        isActionLoading: false,
      )),
      (_) => emit(state.copyWith(
        status: FeatureStatus.actionCompleted,
        successMessage: 'deleted_successfully',
        failure: null,
        isActionLoading: false,
      )),
    );
  }

  void _handleFailure(AppFailure failure) {
    if (failure is UnauthorizedFailure) {
      AppRouter.toAndRemoveUntil(LoginScreen.routeName);
      return;
    }
    emit(state.copyWith(
      status: FeatureStatus.error,
      failure: failure,
      isActionLoading: false,
    ));
  }

  void clearError() => emit(state.copyWith(
    status: state.hasData ? FeatureStatus.loaded : FeatureStatus.initial,
    failure: null,
  ));

  void reset() => emit(const FeatureState.initial());
}
```

---

## UseCase

```dart
import 'package:dartz/dartz.dart';
import '../../../../core/error/failures.dart';
import '../../../../core/use_cases/base_use_case.dart';
import '../entities/feature_entity.dart';
import '../repositories/feature_repository.dart';

class FetchDataUseCase extends UseCase<List<FeatureEntity>, NoParams> {
  final FeatureRepository _repository;
  const FetchDataUseCase(this._repository);

  @override
  Future<Either<AppFailure, List<FeatureEntity>>> call(NoParams params) async {
    final result = await _repository.fetchData();
    return result.map(_applyBusinessRules);
  }

  List<FeatureEntity> _applyBusinessRules(List<FeatureEntity> entities) =>
      entities.where((e) => e.isActive).toList()
        ..sort((a, b) => b.priority.compareTo(a.priority));
}
```

---

## Repository Interface

```dart
import 'package:dartz/dartz.dart';
import '../../../../core/error/failures.dart';
import '../entities/feature_entity.dart';
import '../../dto/feature_dto.dart';

abstract class FeatureRepository {
  Future<Either<AppFailure, List<FeatureEntity>>> fetchData({
    int offset = 0, int limit = 20, String? searchQuery,
  });
  Future<Either<AppFailure, FeatureEntity>> fetchById(String id);
  Future<Either<AppFailure, FeatureEntity>> create(FeatureDto dto);
  Future<Either<AppFailure, FeatureEntity>> update(String id, FeatureDto dto);
  Future<Either<AppFailure, void>> delete(String id);
}
```

---

## Repository Implementation

```dart
import 'package:dartz/dartz.dart';
import '../../../../core/error/exceptions.dart';
import '../../../../core/error/failures.dart';
import '../../../../core/network/network_info.dart';
import '../../domain/repositories/feature_repository.dart';
import '../data_sources/local/feature_local_data_source.dart';
import '../data_sources/remote/feature_remote_data_source.dart';

class FeatureRepositoryImpl implements FeatureRepository {
  final FeatureRemoteDataSource _remote;
  final FeatureLocalDataSource  _local;
  final NetworkInfo             _networkInfo;

  const FeatureRepositoryImpl({
    required FeatureRemoteDataSource remote,
    required FeatureLocalDataSource  local,
    required NetworkInfo             networkInfo,
  })  : _remote      = remote,
        _local       = local,
        _networkInfo = networkInfo;

  @override
  Future<Either<AppFailure, List<FeatureEntity>>> fetchData({
    int offset = 0, int limit = 20, String? searchQuery,
  }) async {
    if (await _networkInfo.isConnected) {
      try {
        final models = await _remote.fetchData(
            offset: offset, limit: limit, searchQuery: searchQuery);
        if (offset == 0) await _local.cacheData(models);
        return Right(models.cast<FeatureEntity>());
      } on ServerException catch (e) {
        return _fetchLocal(offset: offset, limit: limit)
            .then((r) => r.fold((_) => Left(ServerFailure(e.message, statusCode: e.statusCode)), Right.new));
      } catch (e) {
        return Left(UnexpectedFailure(e.toString()));
      }
    }
    return _fetchLocal(offset: offset, limit: limit);
  }

  Future<Either<AppFailure, List<FeatureEntity>>> _fetchLocal({
    required int offset, required int limit,
  }) async {
    try {
      final models = await _local.getCachedData(offset: offset, limit: limit);
      if (models.isEmpty && offset == 0) return const Left(CacheFailure());
      return Right(models.cast<FeatureEntity>());
    } on CacheException catch (e) {
      return Left(CacheFailure(e.message));
    }
  }

  @override
  Future<Either<AppFailure, void>> delete(String id) async {
    if (!await _networkInfo.isConnected) return const Left(NetworkFailure());
    try {
      await _remote.delete(id);
      await _local.removeCachedItem(id);
      return const Right(null);
    } on ServerException catch (e) {
      return Left(ServerFailure(e.message, statusCode: e.statusCode));
    } catch (e) {
      return Left(UnexpectedFailure(e.toString()));
    }
  }

  // ... باقي implementations
}
```

---

## inject_[feature].dart

```dart
import 'package:get_it/get_it.dart';
// imports...

void injectFeature() {
  final getIt = GetIt.instance;

  getIt.registerFactory<FeatureCubit>(
    () => FeatureCubit(
      fetchDataUseCase:   getIt(),
      createItemUseCase:  getIt(),
      deleteItemUseCase:  getIt(),
    ),
  );

  getIt
    ..registerLazySingleton<FetchDataUseCase>  (() => FetchDataUseCase(getIt()))
    ..registerLazySingleton<CreateItemUseCase> (() => CreateItemUseCase(getIt()))
    ..registerLazySingleton<DeleteItemUseCase> (() => DeleteItemUseCase(getIt()));

  getIt.registerLazySingleton<FeatureRepository>(
    () => FeatureRepositoryImpl(remote: getIt(), local: getIt(), networkInfo: getIt()),
  );

  getIt
    ..registerLazySingleton<FeatureRemoteDataSource>(
        () => FeatureRemoteDataSourceImpl(apiProvider: getIt()))
    ..registerLazySingleton<FeatureLocalDataSource>(
        () => FeatureLocalDataSourceImpl(cacheManager: getIt()));
}
```
