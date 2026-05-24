---
name: flutter-guidelines
description: >
  Comprehensive Flutter development guidelines covering clean architecture,
  feature scaffolding, state management with BLoC/Cubit, environment flavors
  (flutter_flavorizr), error handling, dependency injection, and performance
  optimization. Use this skill whenever the user is building a Flutter app,
  creating a new feature, setting up project structure, asking about BLoC/Cubit
  patterns, handling errors, configuring dev/staging/prod environments, or
  optimizing Flutter performance. Trigger for any mention of: Flutter feature,
  Flutter screen, Cubit, BLoC, clean architecture, flavors, AppFailure,
  UseCase, Repository, DI, GetIt, performance 60fps, RepaintBoundary.
---

# Flutter Development Guidelines

## Quick Reference — What's Inside

| Section | File | Use When |
|---|---|---|
| Project & Feature Structure | `references/01-structure.md` | إنشاء مشروع أو feature جديدة |
| Flavors (dev/staging/prod) | `references/02-flavors.md` | إعداد البيئات |
| Error Handling | `references/03-errors.md` | إنشاء Failures أو Exceptions |
| Feature Architecture | `references/04-architecture.md` | كتابة State/Cubit/Repository/UseCase |
| Performance | `references/05-performance.md` | تحسين الأداء أو 60 FPS |
| Code Style | `references/06-style.md` | قواعد الكود العامة |

---

## Core Principles

1. **Clean Architecture** — UI ← Cubit ← UseCase ← Repository ← DataSource
2. **AppFailure not Exception** — استخدم `AppFailure` hierarchy دائماً بدل `Exception` خام
3. **Sentinel Pattern** — في `copyWith` للـ nullable fields
4. **flutter_flavorizr** — إدارة البيئات في build time مش runtime
5. **60 FPS Target** — كل إطار في أقل من 16.66ms
6. **Widgets لا Helper Methods** — دائماً Widget منفصلة بـ `const`

---

## Workflow — عند إنشاء Feature جديدة

```
1. اقرأ references/01-structure.md  ← هيكل المجلدات
2. اقرأ references/03-errors.md     ← AppFailure types
3. اقرأ references/04-architecture.md ← الكود الكامل للـ layers
4. اقرأ references/02-flavors.md    ← لو محتاج baseUrl أو config
5. اقرأ references/05-performance.md ← قبل كتابة أي Widget
```

---

## Component Interaction Flow

```
┌──────────┐   ┌──────────┐   ┌──────────┐   ┌──────────┐
│  Screen  │◄─►│  Cubit   │◄─►│ UseCase  │◄─►│   Repo   │
└──────────┘   └──────────┘   └──────────┘   └──────────┘
      │               │               │              │
  Widgets          State         AppFailure     DataSources
  (const)       (sentinel        Either<>      Remote/Local
               copyWith)
```

---

## Critical Rules — لا تكسرها

```dart
// ✅ دائماً AppFailure مش Exception
Future<Either<AppFailure, List<Entity>>> fetchData();

// ✅ دائماً Sentinel في copyWith
const _unchanged = Object();
FeatureState copyWith({Object? failure = _unchanged, ...}) {
  return FeatureState._(
    failure: failure == _unchanged ? this.failure : failure as AppFailure?,
  );
}

// ✅ دائماً const + super.key
const MyWidget({super.key});

// ✅ دائماً Widget منفصلة مش helper method
class MyCard extends StatelessWidget { ... }   // ✅
Widget _buildCard() => Card(...);              // ❌
```

---

## اقرأ الـ Reference Files بالترتيب ده

**للـ Feature الكاملة**: اقرأ 01 ← 03 ← 04
**للـ Flavors فقط**: اقرأ 02
**للـ Performance فقط**: اقرأ 05
**للـ Code Review**: اقرأ 06
