# 06 — Code Style & Conventions

## Constructor Rules

```dart
// ✅ دائماً const + super.key
const HomeScreen({super.key});

// ✅ required للمطلوب، named للاختياري، default للـ booleans
const FeatureCard({
  super.key,
  required this.item,
  this.onTap,
  this.onLongPress,
  this.isSelected = false,
  this.showActions = true,
  this.isCompact = false,
});
```

## Import Organization

```dart
// 1. Dart core
import 'dart:async';
import 'dart:convert';

// 2. Flutter
import 'package:flutter/material.dart';
import 'package:flutter/services.dart';

// 3. Third-party (alphabetically)
import 'package:dio/dio.dart';
import 'package:easy_localization/easy_localization.dart';
import 'package:flutter_bloc/flutter_bloc.dart';
import 'package:get_it/get_it.dart';

// 4. Internal (alphabetically)
import '../../../core/error/failures.dart';
import '../../../core/utils/constants.dart';
import '../domain/entities/feature_entity.dart';
```

## AppConstants

```dart
class AppConstants {
  AppConstants._();
  static const String appName = 'MyApp';
  // ...
}
```

## AppGaps

```dart
// AppGaps.extraSmallGap → 2px
// AppGaps.soSmallGap    → 5px
// AppGaps.smallGap      → 10px
// AppGaps.defaultGap    → 20px
// AppGaps.bigGap        → 40px
// AppGaps.soBigGap      → 80px
// AppGaps.extraBigGap   → 120px

Column(children: [
  Text('Title'),
  AppGaps.defaultGap,
  Text('Subtitle'),
]);
```

## AppPadding

```dart
Padding(
  padding: EdgeInsets.all(AppPadding.defaultPadding),
  child: Text('Hello'),
);
```

## Naming Conventions

| Type | Convention | Example |
|---|---|---|
| Classes | PascalCase | `FeatureCubit` |
| Variables/Functions | camelCase | `fetchInitialData` |
| Constants | camelCase | `appName` |
| Files | snake_case | `feature_cubit.dart` |
| Routes | `/kebab-case` | `/feature-detail` |
| Enums | PascalCase values | `FeatureStatus.loaded` |

## DI Registration Rules

| Type | Registration | Reason |
|---|---|---|
| Cubit | `registerFactory` | كل screen تاخد instance جديد |
| UseCase | `registerLazySingleton` | Stateless — آمن يتشارك |
| Repository | `registerLazySingleton` | Cache management |
| DataSource | `registerLazySingleton` | Connection pooling |
| Dio/ApiProvider | `registerLazySingleton` | واحد في المشروع كله |

## Data Flow

```
1. User Action     → Cubit method
2. Cubit           → UseCase.call()
3. UseCase         → Repository method
4. Repository      → DataSource (remote or local)
5. DataSource      → API/Cache
6. Response        ← نفس الـ layers بالعكس
7. Cubit emits     → UI rebuilds
```

## Localization — easy_localization

```dart
// في الكود
Text('feature_title'.tr())
Text('items_count'.plural(count))

// في ملفات الترجمة
// assets/lang/ar.json
{
  "feature_title": "اسم الميزة",
  "items_count": {
    "zero":  "لا توجد عناصر",
    "one":   "عنصر واحد",
    "other": "{} عنصر"
  }
}
```

## Network Interceptors Setup

```dart
// في injection_container.dart
getIt.registerLazySingleton<Dio>(() => Dio()
  ..options = BaseOptions(
    connectTimeout: Duration(milliseconds: AppFlavor.connectTimeout),
    receiveTimeout: const Duration(seconds: 30),
  )
  ..interceptors.addAll([
    AuthInterceptor(
      getToken: () => authManager.currentUser?.token,
      onUnauthorized: () async => AppRouter.toAndRemoveUntil(LoginScreen.routeName),
    ),
    if (AppFlavor.isDebug) LoggingInterceptor(),
  ]));
```
