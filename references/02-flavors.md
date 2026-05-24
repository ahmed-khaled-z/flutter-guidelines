# 02 — Flavors — flutter_flavorizr

> بنستخدم [flutter_flavorizr](https://pub.dev/packages/flutter_flavorizr) **بدل** Firebase Remote Config.
> الفرق الجوهري: config في **build time** مش runtime.

## pubspec.yaml

```yaml
dev_dependencies:
  flutter_flavorizr: ^2.4.2
```

## flavorizr.yaml

```yaml
flavors:
  dev:
    app:
      name: "MyApp Dev"
      icon: "assets/icons/ic_launcher_dev.png"
    android:
      applicationId: "com.example.myapp.dev"
      buildConfigFields:
        base_url:
          type: "String"
          value: '"https://dev.api.myapp.com"'
        is_debug:
          type: "boolean"
          value: "true"
      firebase:
        config: ".firebase/dev/google-services.json"
    ios:
      bundleId: "com.example.myapp.dev"
      variables:
        BASE_URL:
          value: "https://dev.api.myapp.com"
      firebase:
        config: ".firebase/dev/GoogleService-Info.plist"

  staging:
    app:
      name: "MyApp Staging"
    android:
      applicationId: "com.example.myapp.staging"
      buildConfigFields:
        base_url:
          type: "String"
          value: '"https://staging.api.myapp.com"'
        is_debug:
          type: "boolean"
          value: "true"
      firebase:
        config: ".firebase/staging/google-services.json"
    ios:
      bundleId: "com.example.myapp.staging"
      variables:
        BASE_URL:
          value: "https://staging.api.myapp.com"
      firebase:
        config: ".firebase/staging/GoogleService-Info.plist"

  prod:
    app:
      name: "MyApp"
    android:
      applicationId: "com.example.myapp"
      buildConfigFields:
        base_url:
          type: "String"
          value: '"https://api.myapp.com"'
        is_debug:
          type: "boolean"
          value: "false"
      firebase:
        config: ".firebase/prod/google-services.json"
    ios:
      bundleId: "com.example.myapp"
      variables:
        BASE_URL:
          value: "https://api.myapp.com"
      firebase:
        config: ".firebase/prod/GoogleService-Info.plist"
```

## تشغيل الأمر

```bash
flutter pub get
flutter pub run flutter_flavorizr
```

---

## lib/config/flavor/app_flavor.dart

```dart
import '../../flavors.dart';

class AppFlavor {
  AppFlavor._();

  static String get baseUrl {
    switch (F.appFlavor) {
      case Flavor.dev:     return 'https://dev.api.myapp.com';
      case Flavor.staging: return 'https://staging.api.myapp.com';
      case Flavor.prod:    return 'https://api.myapp.com';
    }
  }

  static String get socketUrl {
    switch (F.appFlavor) {
      case Flavor.dev:     return 'wss://dev.socket.myapp.com';
      case Flavor.staging: return 'wss://staging.socket.myapp.com';
      case Flavor.prod:    return 'wss://socket.myapp.com';
    }
  }

  static bool get isDebug      => F.appFlavor != Flavor.prod;
  static bool get isProduction => F.appFlavor == Flavor.prod;
  static String get flavorName => F.name;
  static String get appTitle   => F.title;
  static int get connectTimeout => isProduction ? 30000 : 60000;
}
```

## lib/core/network/app_endpoints.dart

```dart
import '../../config/flavor/app_flavor.dart';

class AppUrls {
  AppUrls._();
  static String get _base => AppFlavor.baseUrl;

  // Auth
  static String get login    => '$_base/auth/login';
  static String get register => '$_base/auth/register';
  static String get logout   => '$_base/auth/logout';
  static String get refresh  => '$_base/auth/refresh';

  // User
  static String get profile              => '$_base/user/profile';
  static String updateProfile(String id) => '$_base/user/$id';

  // Dynamic endpoints
  static String itemDetails(String id) => '$_base/items/$id';
}
```

## Run & Build Commands

```bash
# Run
flutter run --flavor dev     -t lib/main_dev.dart
flutter run --flavor staging -t lib/main_staging.dart
flutter run --flavor prod    -t lib/main_prod.dart

# Build APK
flutter build apk --flavor prod -t lib/main_prod.dart --release

# Build iOS
flutter build ipa --flavor prod -t lib/main_prod.dart
```

## مقارنة AppConfig vs flutter_flavorizr

| | AppConfig (قديم) | flutter_flavorizr (جديد) |
|---|---|---|
| وقت التحديد | Runtime | Build time |
| مصدر الـ config | Firebase Remote Config | flavorizr.yaml |
| في main.dart | `await AppConfig.init()` | ❌ مش محتاج |
| الاستخدام | `AppConfig.baseUrl` | `AppFlavor.baseUrl` |
| Firebase config | ملف واحد | ملف منفصل لكل flavor |
| تغيير الـ URL | من Firebase dashboard | تغيير الـ flavor في البيلد |
