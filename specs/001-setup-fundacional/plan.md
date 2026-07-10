# Implementation Plan: Setup Fundacional del Proyecto

**Branch**: `001-setup-fundacional` | **Date**: 2026-07-09 | **Spec**: [spec.md](./spec.md)

**Input**: Feature specification from `specs/001-setup-fundacional/spec.md`

## Summary

This feature establishes the technical foundation for Finance Wrapped: a
Flutter project with Clean Architecture (`lib/core/` + `lib/features/`),
3 environment flavors (`dev`/`staging`/`prod`) each backed by a separate
Firebase project, a central `AppTheme` with light/dark mode via
`ThemeExtension` + Riverpod, Firebase Authentication (Google, Apple,
email/password) with auto-provisioning of the user document in Firestore,
Firestore Security Rules with automated emulator tests, a local `drift`
database as source of truth with a `sync_queue` mechanism for offline-first
writes, and account deletion. No product screens are built — this is
purely infrastructure.

## Technical Context

**Language/Version**: Dart 3.10+ (latest stable via `fvm`)

**Framework**: Flutter 3.44+ (latest stable channel, pinned with `fvm`)

**Primary Dependencies**:
- State management: `flutter_riverpod` + `riverpod_annotation` + `riverpod_generator`
- Models: `freezed` (mixed mode 3.x) + `json_serializable`
- Codegen: `build_runner`
- Navigation: `go_router`
- Firebase: `firebase_core`, `firebase_auth`, `cloud_firestore`,
  `firebase_analytics`, `firebase_crashlytics`, `cloud_functions`
- Auth providers: `google_sign_in`, `sign_in_with_apple`
- Local DB: `drift` + `sqlite3_flutter_libs` + `path_provider`
- Connectivity: `connectivity_plus`
- Linting: `very_good_analysis` + `custom_lint` + `import_lint`
- Testing: `mocktail`, `fake_cloud_firestore`, `firebase_auth_mocks`,
  `integration_test`, `golden_toolkit`

**Storage**: SQLite via `drift` (local source of truth) + Firestore (sync/backup)

**Testing**: `flutter_test` (unit/widget), `integration_test` (e2e),
`@firebase/rules-unit-testing` (Firestore Security Rules via Node.js emulator)

**Target Platform**: iOS 15+ / iPadOS 15+ / Android 8.0 (API 26)+

**Project Type**: Mobile app (Flutter)

**Performance Goals**: App cold start < 3s on mid-range device; theme
switching < 1s; offline reads instantaneous from local DB

**Constraints**: Offline-capable (Principle IV); zero hardcoded visual
tokens (Principle III); 80% domain test coverage (constitution)

**Scale/Scope**: Single-user app; ~6 screens in this feature (splash,
login, register, password recovery, settings, example theme screen)

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

| # | Principle | Gate | Status |
|---|-----------|------|--------|
| I | Clean Architecture | `lib/core/` + `lib/features/<name>/{data,domain,presentation}/` structure. Domain layer imports no Flutter/Firebase/drift packages. Enforced by `import_lint` in CI. | ✅ Pass |
| II | Widget-First Architecture | All UI composed of focused, self-contained widgets. Shared widgets in `lib/core/widgets/`. Constructor-injected inputs, no direct global state. | ✅ Pass |
| III | Theme-Driven UI | Zero hardcoded `Color(0x...)`, `TextStyle(...)`, or raw `EdgeInsets`. All tokens from `AppTheme` via `ThemeExtension`. Light/dark from first screen. | ✅ Pass |
| IV | Offline-First | `drift` as local source of truth. Reads from local DB. Writes persist locally first → `sync_queue` → Firestore. Last-write-wins by server timestamp. Firestore never accessed from presentation layer. | ✅ Pass |
| V | Test-First Development | Tests written before implementation. Unit tests for domain use cases, widget tests for screens, integration tests for auth flow, emulator tests for Security Rules. | ✅ Pass |
| VI | Data Integrity | User entity: `uid` (string PK), no floating-point amounts in this feature (amounts come in later features). Soft-delete for account deletion (`deletedAt` timestamp before hard-delete via Cloud Function). Writes wrapped in drift transactions. | ✅ Pass |
| VII | Data Privacy & Regulatory | Security Rules enforce `uid`-scoped access, tested via emulator. No monetary data sent to analytics. Account self-delete available. No secrets committed. | ✅ Pass |

**Gate result**: All 7 principles pass. No violations to justify.

## Project Structure

### Documentation (this feature)

```text
specs/001-setup-fundacional/
├── plan.md              # This file
├── research.md          # Phase 0 output
├── data-model.md        # Phase 1 output
├── quickstart.md        # Phase 1 output
├── contracts/           # Phase 1 output
│   └── firestore-schema.md
└── tasks.md             # Phase 2 output (/speckit-tasks command)
```

### Source Code (repository root)

```text
lib/
├── main_dev.dart                    # Entrypoint for dev flavor
├── main_staging.dart                # Entrypoint for staging flavor
├── main_prod.dart                   # Entrypoint for prod flavor
├── app.dart                         # MaterialApp.router with AppTheme
├── core/
│   ├── theme/
│   │   ├── app_theme.dart           # ThemeExtension definitions
│   │   ├── app_colors.dart          # Semantic color tokens (light + dark)
│   │   ├── app_typography.dart      # Type scale (display→label)
│   │   ├── app_spacing.dart         # Standardized spacing constants
│   │   └── theme_provider.dart      # Riverpod provider for ThemeMode
│   ├── router/
│   │   └── app_router.dart          # GoRouter config with auth redirect
│   ├── firebase/
│   │   └── firebase_options_*.dart  # Per-flavor DefaultFirebaseOptions
│   ├── sync/
│   │   ├── sync_worker.dart         # Processes sync_queue on connectivity
│   │   └── sync_provider.dart       # Riverpod provider for sync status
│   ├── database/
│   │   ├── app_database.dart        # Drift database definition
│   │   ├── app_database.g.dart      # Generated
│   │   └── tables/                  # Drift table definitions
│   │       ├── usuarios_table.dart
│   │       └── sync_queue_table.dart
│   ├── di/
│   │   └── providers.dart           # Core Riverpod providers (DB, Firebase)
│   ├── widgets/
│   │   ├── environment_badge.dart   # Shows DEV/STAGING badge (not in prod)
│   │   └── sync_status_indicator.dart
│   └── utils/
│       └── constants.dart           # App-wide constants
├── features/
│   └── auth/
│       ├── data/
│       │   ├── repositories/
│       │   │   └── auth_repository_impl.dart
│       │   ├── datasources/
│       │   │   ├── firebase_auth_datasource.dart
│       │   │   ├── firestore_user_datasource.dart
│       │   │   └── local_user_datasource.dart
│       │   └── models/
│       │       └── usuario_dto.dart  # Freezed DTO for Firestore ↔ drift
│       ├── domain/
│       │   ├── entities/
│       │   │   └── usuario.dart      # Pure Dart entity (no Flutter imports)
│       │   ├── repositories/
│       │   │   └── auth_repository.dart  # Abstract interface
│       │   └── usecases/
│       │       ├── sign_in_with_google.dart
│       │       ├── sign_in_with_apple.dart
│       │       ├── sign_in_with_email.dart
│       │       ├── sign_up_with_email.dart
│       │       ├── sign_out.dart
│       │       ├── reset_password.dart
│       │       ├── delete_account.dart
│       │       └── get_current_user.dart
│       └── presentation/
│           ├── screens/
│           │   ├── login_screen.dart
│           │   ├── register_screen.dart
│           │   ├── password_recovery_screen.dart
│           │   └── settings_screen.dart
│           ├── widgets/
│           │   ├── social_login_button.dart
│           │   └── auth_form.dart
│           └── providers/
│               └── auth_providers.dart  # Riverpod providers for auth state

test/
├── unit/
│   └── features/auth/domain/usecases/
│       ├── sign_in_with_google_test.dart
│       ├── sign_in_with_email_test.dart
│       ├── sign_out_test.dart
│       ├── delete_account_test.dart
│       └── ...
├── widget/
│   └── features/auth/presentation/
│       ├── login_screen_test.dart
│       ├── settings_screen_test.dart
│       └── ...
├── golden/
│   └── core/theme/
│       └── theme_screen_test.dart
└── integration/
    └── auth_flow_test.dart

firestore-tests/                     # Node.js project (NOT Flutter)
├── package.json
├── tsconfig.json
└── tests/
    └── security-rules.test.ts       # @firebase/rules-unit-testing

firestore.rules                      # Single source, deployed to all 3 projects

ios/                                 # Flutter-managed, per-flavor config
android/                             # Flutter-managed, per-flavor config
functions/                           # Cloud Functions (TypeScript)
└── src/
    └── onUserDeleted.ts             # Cascade-delete trigger (doc raíz only for now)
```

**Structure Decision**: Flutter mobile app with Clean Architecture.
Feature-based modular structure under `lib/features/`. Shared
infrastructure under `lib/core/`. Firestore Security Rules tests live
in a separate Node.js project (`firestore-tests/`) because the
`@firebase/rules-unit-testing` SDK requires Node. Cloud Functions live
in `functions/` (TypeScript, standard Firebase layout).

## Complexity Tracking

> No violations detected. All decisions align with constitution principles.

| Violation | Why Needed | Simpler Alternative Rejected Because |
|-----------|------------|-------------------------------------|
| N/A | — | — |
