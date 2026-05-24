# 🎯 Flutter Guidelines — Claude Skill

A Claude skill that enforces best-practice Flutter development guidelines covering **clean architecture**, **BLoC/Cubit state management**, **flutter_flavorizr**, **error handling**, **dependency injection**, and **performance optimization**.

---

## 📦 Installation

1. Download `flutter-guidelines.skill`
2. In Claude.ai → **Settings → Skills → Install from file**
3. Upload the `.skill` file

---

## 🚀 What It Does

Once installed, Claude automatically applies these guidelines whenever you work on Flutter. Just describe what you want to build and Claude will scaffold it correctly.

### Trigger Phrases

| You say… | Claude does… |
|---|---|
| "Create a Flutter feature for products" | Scaffolds full clean architecture |
| "Set up dev/staging/prod environments" | Configures flutter_flavorizr |
| "How should I handle errors in Flutter?" | Generates AppFailure hierarchy |
| "My Flutter app is slow / lagging" | Applies performance optimizations |
| "Create a BLoC/Cubit for user profile" | Generates State + Cubit with sentinel pattern |
| "Set up dependency injection" | Configures GetIt with proper scopes |

---

## 📚 What's Inside the Skill

```
flutter-guidelines/
├── SKILL.md                    ← Entry point + Quick Reference
└── references/
    ├── 01-structure.md         ← Project & Feature folder structure
    ├── 02-flavors.md           ← flutter_flavorizr setup (dev/staging/prod)
    ├── 03-errors.md            ← AppFailure hierarchy + exceptions
    ├── 04-architecture.md      ← Full code templates (State/Cubit/Repo/UseCase)
    ├── 05-performance.md       ← 60 FPS optimization rules
    └── 06-style.md             ← Code conventions & naming
```

---

## 🏛️ Architecture Overview

```
┌──────────┐   ┌──────────┐   ┌──────────┐   ┌──────────┐
│  Screen  │◄─►│  Cubit   │◄─►│ UseCase  │◄─►│   Repo   │
└──────────┘   └──────────┘   └──────────┘   └──────────┘
  const           sentinel      AppFailure    Remote+Local
  widgets         copyWith       Either<>     DataSources
```

### Key Decisions

| Decision | Choice | Why |
|---|---|---|
| State Management | BLoC / Cubit | Predictable, testable |
| Error Type | `AppFailure` hierarchy | Type-safe error handling |
| copyWith pattern | Sentinel Object | Correctly handles nullable → null |
| Environments | `flutter_flavorizr` | Build-time config, no Remote Config needed |
| DI | GetIt | Simple, performant |
| Network | Dio + Interceptors | Auth refresh, logging, retry |

---

## ⚡ Performance Rules Summary

1. **Widgets not helper methods** — always separate `StatelessWidget` classes
2. **`const` everywhere** — Flutter skips rebuild for const widgets
3. **`ListView.builder`** — never `SingleChildScrollView` with long lists
4. **`RepaintBoundary`** — isolate animations and real-time widgets
5. **`compute()`** — heavy work off the UI thread
6. **`FadeTransition` not `Opacity`** — for animations
7. **Test on real Android device in Release mode**

---

## 🍦 Flavor Structure (flutter_flavorizr)

```
dev     → https://dev.api.myapp.com     (debug logging on)
staging → https://staging.api.myapp.com (debug logging on)
prod    → https://api.myapp.com         (logging off, 30s timeout)
```

```bash
# Run
flutter run --flavor dev     -t lib/main_dev.dart
flutter run --flavor staging -t lib/main_staging.dart
flutter run --flavor prod    -t lib/main_prod.dart
```

---

## 🤖 Using with Other AI Models / Agents

The skill is designed as a **Markdown knowledge base** — any model can use it:

### With Claude API

```python
import anthropic

# Load skill content
with open("flutter-guidelines/SKILL.md") as f:
    skill_md = f.read()
with open("flutter-guidelines/references/04-architecture.md") as f:
    arch_ref = f.read()

client = anthropic.Anthropic()
response = client.messages.create(
    model="claude-sonnet-4-20250514",
    max_tokens=4096,
    system=f"""You are a Flutter development assistant.
Follow these guidelines strictly:

{skill_md}

When creating features, use this architecture:
{arch_ref}
""",
    messages=[{"role": "user", "content": "Create a Flutter products feature"}]
)
```

### With OpenAI API

```python
from openai import OpenAI

with open("flutter-guidelines/SKILL.md") as f:
    skill = f.read()

client = OpenAI()
response = client.chat.completions.create(
    model="gpt-4o",
    messages=[
        {"role": "system", "content": f"Flutter guidelines:\n{skill}"},
        {"role": "user",   "content": "Create a products Cubit with state management"}
    ]
)
```

### With LangChain

```python
from langchain_anthropic import ChatAnthropic
from langchain_core.prompts import ChatPromptTemplate
from pathlib import Path

# Load all reference files
refs_dir = Path("flutter-guidelines/references")
context  = "\n\n".join(f.read_text() for f in sorted(refs_dir.glob("*.md")))

prompt = ChatPromptTemplate.from_messages([
    ("system", "Flutter expert. Guidelines:\n{guidelines}"),
    ("human",  "{question}"),
])

chain = prompt | ChatAnthropic(model="claude-sonnet-4-20250514")
result = chain.invoke({
    "guidelines": context,
    "question":   "Scaffold a clean architecture feature for authentication"
})
```

### With Any Agent Framework (AutoGen, CrewAI, etc.)

```python
# Load the reference files you need
import os

def load_flutter_guidelines(sections: list[str] = None) -> str:
    """Load Flutter guidelines. Pass section names to load specific files."""
    base_path = "flutter-guidelines/references"
    files = {
        "structure":    "01-structure.md",
        "flavors":      "02-flavors.md",
        "errors":       "03-errors.md",
        "architecture": "04-architecture.md",
        "performance":  "05-performance.md",
        "style":        "06-style.md",
    }
    if sections:
        to_load = {k: v for k, v in files.items() if k in sections}
    else:
        to_load = files

    content = []
    for name, filename in to_load.items():
        path = os.path.join(base_path, filename)
        with open(path) as f:
            content.append(f"## {name.upper()}\n{f.read()}")
    return "\n\n".join(content)

# Usage
guidelines = load_flutter_guidelines(["architecture", "errors"])
```

---

## 🔑 Core Patterns Reference

### AppFailure (never use raw Exception)

```dart
abstract class AppFailure extends Equatable {
  final String message;
  const AppFailure(this.message);
}
class NetworkFailure    extends AppFailure { ... }
class ServerFailure     extends AppFailure { ... }
class CacheFailure      extends AppFailure { ... }
class UnauthorizedFailure extends AppFailure { ... }
```

### Sentinel copyWith (fixes nullable field bug)

```dart
const _unchanged = Object();

FeatureState copyWith({Object? failure = _unchanged}) {
  return FeatureState._(
    failure: failure == _unchanged ? this.failure : failure as AppFailure?,
  );
}
```

### BaseUseCase

```dart
abstract class UseCase<Type, Params> {
  Future<Either<AppFailure, Type>> call(Params params);
}
class NoParams extends Equatable { const NoParams(); }
```

---

## 📋 Packages Used

| Package | Purpose |
|---|---|
| `flutter_bloc` | State management |
| `dartz` | Either type for functional error handling |
| `get_it` | Dependency injection |
| `dio` | HTTP client |
| `connectivity_plus` | Network info |
| `equatable` | Value equality |
| `easy_localization` | i18n |
| `flutter_flavorizr` | Build flavors (dev/staging/prod) |
| `cached_network_image` | Image caching |
| `json_annotation` + `build_runner` | JSON serialization |

---

## 📄 License

MIT
