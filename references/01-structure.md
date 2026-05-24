# 01 — Project Structure & Feature Scaffolding

## Project Structure

```
lib/
├── config/
│   ├── app_helper/
│   │   ├── app_extension.dart
│   │   ├── app_formats.dart
│   │   ├── app_functions.dart
│   │   ├── app_gaps.dart
│   │   └── app_padding.dart
│   ├── flavor/
│   │   └── app_flavor.dart
│   ├── router/
│   │   ├── app_router.dart
│   │   └── unknown_route.dart
│   └── theme/
│       ├── dark_theme.dart
│       ├── light_theme.dart
│       └── theme_manager.dart
├── core/
│   ├── error/
│   │   ├── failures.dart
│   │   └── exceptions.dart
│   ├── network/
│   │   ├── interceptors/
│   │   │   ├── auth_interceptor.dart
│   │   │   ├── logging_interceptor.dart
│   │   │   └── retry_interceptor.dart
│   │   ├── api_provider.dart
│   │   ├── app_endpoints.dart
│   │   └── network_info.dart
│   ├── use_cases/
│   │   └── base_use_case.dart
│   └── utils/
│       ├── dialogs.dart
│       ├── logger.dart
│       ├── snack_bar_utils.dart
│       └── validator.dart
├── features/
├── flavors.dart                 ← مولَّد من flutter_flavorizr
├── main_dev.dart                ← مولَّد من flutter_flavorizr
├── main_staging.dart            ← مولَّد من flutter_flavorizr
├── main_prod.dart               ← مولَّد من flutter_flavorizr
├── app.dart
├── injection_container.dart
└── main.dart
```

---

## Feature Structure

```
lib/features/[feature_name]/
├── data/
│   ├── data_sources/
│   │   ├── local/
│   │   │   └── [feature]_local_data_source.dart
│   │   └── remote/
│   │       └── [feature]_remote_data_source.dart
│   ├── models/
│   │   └── [feature]_model.dart
│   └── repositories/
│       └── [feature]_repository_impl.dart
├── domain/
│   ├── entities/
│   │   └── [feature]_entity.dart
│   ├── repositories/
│   │   └── [feature]_repository.dart
│   └── use_cases/
│       ├── fetch_[feature]_use_case.dart
│       ├── create_[feature]_use_case.dart
│       ├── update_[feature]_use_case.dart
│       └── delete_[feature]_use_case.dart
├── dto/
│   └── [feature]_dto.dart
├── presentation/
│   ├── cubit/
│   │   ├── [feature]_cubit.dart
│   │   └── [feature]_state.dart
│   ├── screens/
│   │   └── [feature]_screen.dart
│   └── widgets/
│       └── [feature]_card.dart
└── inject_[feature].dart
```

---

## File Roles

| File | Role | Dependencies |
|---|---|---|
| `[feature]_screen.dart` | Main UI + state observation | Cubit, Widgets |
| `[feature]_cubit.dart` | State + logic coordination | UseCases, State |
| `[feature]_state.dart` | Immutable state + sentinel copyWith | AppFailure, Models |
| `[feature]_card.dart` | Reusable UI component (const) | Models, Theme |
| `[feature]_repository.dart` | Interface/Contract | AppFailure, Entities |
| `[action]_use_case.dart` | Single business operation | Repository, AppFailure |
| `[feature]_model.dart` | JSON serialization | json_annotation |
| `[feature]_repository_impl.dart` | Coordinates DataSources | Remote, Local, NetworkInfo |
| `[feature]_remote_data_source.dart` | API calls | ApiProvider, DTO |
| `[feature]_local_data_source.dart` | Cache operations | Storage |
| `[feature]_dto.dart` | Request/Response formatting | JSON |
| `inject_[feature].dart` | GetIt registration | All feature classes |

---

## injection_container.dart — Service Locator

```dart
final GetIt getIt = GetIt.instance;

class ServiceLocator {
  Future<void> setup() async {
    // Core
    getIt.registerLazySingleton<Dio>(() => Dio()
      ..interceptors.addAll([
        AuthInterceptor(
          getToken: () => authManager.currentUser?.token,
          onUnauthorized: () async => AppRouter.toAndRemoveUntil(LoginScreen.routeName),
        ),
        LoggingInterceptor(),
      ]));

    getIt.registerLazySingleton<ApiProvider>(() => ApiProvider(getIt()));
    getIt.registerLazySingleton<NetworkInfo>(() => NetworkInfoImpl(Connectivity()));

    // Features
    injectAuth();
    injectHome();
    // ...
  }
}
```

---

## main.dart

```dart
Future<void> main() async {
  WidgetsFlutterBinding.ensureInitialized();
  await initializeDateFormatting();
  await Firebase.initializeApp(); // بدون options — flavorizr بيتولاها
  
  languageManager = await LanguageManager.loadLanguage();
  themeManager    = await ThemeManager.loadTheme(languageManager.currentLanguage);
  authManager     = await AuthManager.loadUser(
    fromJson: (json) => UserWithToken.fromJson(json),
    toJson:   (user) => user.toJson(),
  );
  
  await EasyNotify.init();
  await NotificationService.instance.initialize();
  ServiceLocator().setup();
  
  runApp(EasyLocalization(
    path: 'assets/lang',
    startLocale: languageManager.currentLocale,
    supportedLocales: const [Locale('ar'), Locale('en')],
    child: const App(),
  ));
}
```
