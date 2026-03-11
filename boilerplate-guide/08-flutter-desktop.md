# 08. Flutter Desktop Client

> Flutter macOS desktop app structure guide.
> Covers Riverpod + Freezed + macos_ui based patterns.
> Managed as an independent Dart toolchain outside the pnpm workspace.

---

## 1. Dependencies

### pubspec.yaml (Core Dependencies)

```yaml
name: projectname
description: macOS desktop client
publish_to: "none"
version: 1.0.0+1

environment:
  sdk: ^3.12.0

dependencies:
  flutter:
    sdk: flutter

  # State Management
  flutter_riverpod: ^2.6.1
  riverpod_annotation: ^2.6.1

  # macOS Native UI
  macos_ui: ^2.1.8
  cupertino_icons: ^1.0.8

  # System Integration
  tray_manager: ^0.2.3
  window_manager: ^0.4.3
  hotkey_manager: ^0.2.3

  # Data Models
  freezed_annotation: ^2.4.4
  json_annotation: ^4.9.0

  # Database (optional — MongoDB direct connectivity)
  mongo_dart: ^0.10.3

  # Utilities
  uuid: ^4.5.1
  path_provider: ^2.1.5
  intl: ^0.19.0
  crypto: ^3.0.6
  shared_preferences: ^2.3.4
  flutter_dotenv: ^5.2.1
  logger: ^2.6.2

dev_dependencies:
  flutter_test:
    sdk: flutter
  flutter_lints: ^6.0.0
  build_runner: ^2.4.13
  freezed: ^2.5.7
  json_serializable: ^6.8.0
  riverpod_generator: ^2.6.3

flutter:
  uses-material-design: true
  assets:
    - assets/icons/
```

---

## 2. Directory Structure (Feature-first)

```
apps/client/lib/
├── core/
│   ├── config/       # Application configuration, environment variables
│   ├── constants/    # Constants
│   ├── utils/        # Utility functions
│   ├── errors/       # Custom error types
│   └── logging/      # Logging configuration
└── features/
    ├── settings/     # General settings screens
    ├── system/       # System integration logic (tray, windows)
    ├── notes/        # Notes functionality logic
    └── clipboard/    # Clipboard functionality logic
```

**Feature-first Policy**:

- Each feature directory is standalone — subdivided into `models/`, `providers/`, `views/`, and `widgets/` architectures internally
- Distribute universal utilities within core solely
- Assorted inter-feature bindings funnel strictly via core

---

## 3. Freezed Models + Code Generation

### Model Definition

```dart
// lib/features/notes/models/note.dart
import 'package:freezed_annotation/freezed_annotation.dart';

part 'note.freezed.dart';
part 'note.g.dart';

@freezed
class Note with _$Note {
  const factory Note({
    required String id,
    required String title,
    required String content,
    required DateTime createdAt,
    required DateTime updatedAt,
    @Default([]) List<String> tags,
  }) = _Note;

  factory Note.fromJson(Map<String, dynamic> json) => _$NoteFromJson(json);
}
```

### Initializing the Code Generator

```bash
cd apps/client
dart run build_runner build --delete-conflicting-outputs
```

**Important**: After rectifying Dart files (such as Freezed models, Riverpod providers), you are compelled to rerun the code generation sequence.

---

## 4. Riverpod Hierarchy

### Provider (Generated Code)

```dart
// lib/features/notes/providers/notes_provider.dart
import 'package:riverpod_annotation/riverpod_annotation.dart';
import '../models/note.dart';
import '../repositories/notes_repository.dart';

part 'notes_provider.g.dart';

@riverpod
class NotesNotifier extends _$NotesNotifier {
  @override
  Future<List<Note>> build() async {
    final repository = ref.watch(notesRepositoryProvider);
    return repository.findAll();
  }

  Future<void> addNote(Note note) async {
    final repository = ref.read(notesRepositoryProvider);
    await repository.create(note);
    ref.invalidateSelf();
  }

  Future<void> deleteNote(String id) async {
    final repository = ref.read(notesRepositoryProvider);
    await repository.delete(id);
    ref.invalidateSelf();
  }
}
```

### Hierarchy Blueprint

```
View (Widget)
  → Provider (Riverpod, generated code)
    → Repository (Abstraction Layer)
      → Data Source (Local JSON / MongoDB)
```

---

## 5. Repository Abstraction

```dart
// lib/features/notes/repositories/notes_repository.dart
abstract class NotesRepository {
  Future<List<Note>> findAll();
  Future<Note?> findById(String id);
  Future<Note> create(Note note);
  Future<Note> update(Note note);
  Future<void> delete(String id);
}
```

### Local JSON Implementation

```dart
// Data Storage Path: ~/Library/Application Support/ProjectName/
class LocalNotesRepository implements NotesRepository {
  // Logic implementing JSON file-based CRUD ops
}
```

### MongoDB Implementation (Optional)

```dart
// Immediate direct binding via mongo_dart
class MongoNotesRepository implements NotesRepository {
  // Logic implementing MongoDB-based CRUD ops
}
```

Users may designate their preferred repository type within the application's configuration parameters.

---

## 6. macos_ui Guidelines

`macos_ui` packages are incorporated to impart a native macOS appearance:

```dart
import 'package:macos_ui/macos_ui.dart';

class App extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return MacosApp(
      title: 'Project Name',
      theme: MacosThemeData.light(),
      darkTheme: MacosThemeData.dark(),
      home: MacosWindow(
        sidebar: Sidebar(
          // ...
        ),
        child: ContentArea(
          // ...
        ),
      ),
    );
  }
}
```

**Regulations**:

- Leverage `MacosWindow`, `Sidebar`, and `ContentArea` in contrast to typical Material widgets
- Utilize `MacosThemeData` capabilities accommodating corresponding light and dark profiles
- Ensure preservation of default sidebar combined with content area layout arrangements

---

## 7. System Integration

```dart
// Tray manager
import 'package:tray_manager/tray_manager.dart';

// Window manager
import 'package:window_manager/window_manager.dart';

// Global hotkeys
import 'package:hotkey_manager/hotkey_manager.dart';
```

---

## Verification

```bash
cd apps/client

# Intercept dependencies
flutter pub get

# Launch Code generator
dart run build_runner build --delete-conflicting-outputs

# Linting
flutter analyze

# Verify tests
flutter test

# Boot up the instance
flutter run

# Transpile into a release macOS build artifact
flutter build macos --release
```
