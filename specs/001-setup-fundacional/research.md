# Research: Setup Fundacional del Proyecto

**Feature**: 001-setup-fundacional
**Date**: 2026-07-09

---

## R1. Flutter Flavors: `flutter_flavorizr` vs Manual Configuration

**Decision**: Manual configuration.

**Rationale**: With only 3 flavors, manual setup is a one-time cost that
gives full control. `flutter_flavorizr` adds a code-generation layer
that can conflict with FlutterFire CLI's generated
`firebase_options_*.dart` files and creates opaque configuration harder
to debug. Manual aligns exactly with our plan structure.

**Alternatives considered**:
- `flutter_flavorizr`: Rejected — generator conflicts, third-party
  dependency risk, unnecessary abstraction for 3 flavors.

### Flavor Setup Details

**Android** (`build.gradle.kts`):
- `productFlavors` block with `dev`, `staging`, `prod`
- Each flavor sets `applicationIdSuffix`, `resValue("string", "app_name", ...)`
- Per-flavor resource directories for launcher icons:
  `android/app/src/{dev,staging,prod}/res/mipmap-*/`

**iOS** (Xcode):
- 3 schemes: `dev`, `staging`, `prod`
- 6 build configurations: `Debug-dev`, `Release-dev`, etc.
- User-defined `APP_DISPLAY_NAME` build setting per configuration
- `Info.plist` uses `$(APP_DISPLAY_NAME)` for `CFBundleDisplayName`
- Separate `AppIcon-{dev,staging,prod}` asset catalogs

**Entrypoints**:
```bash
flutter run --flavor dev -t lib/main_dev.dart
flutter run --flavor staging -t lib/main_staging.dart
flutter run --flavor prod -t lib/main_prod.dart
```

Each `main_<flavor>.dart` imports the corresponding
`firebase_options_<flavor>.dart` and initializes Firebase with
flavor-specific options before calling the shared `App()`.

---

## R2. FlutterFire CLI Multi-Flavor Firebase Configuration

**Decision**: Run `flutterfire configure --flavor=X --project=Y` once
per flavor (3 times total).

**Rationale**: This is the official FlutterFire approach. It generates
per-flavor `google-services.json` (Android), `GoogleService-Info.plist`
(iOS), and Dart `DefaultFirebaseOptions` classes automatically.

**Commands**:
```bash
flutterfire configure \
  --project=finance-wrapped-dev \
  --out=lib/core/firebase/firebase_options_dev.dart \
  --ios-bundle-id=com.financewrapped.app.dev \
  --android-app-id=com.financewrapped.app.dev \
  --flavor=dev

flutterfire configure \
  --project=finance-wrapped-staging \
  --out=lib/core/firebase/firebase_options_staging.dart \
  --ios-bundle-id=com.financewrapped.app.staging \
  --android-app-id=com.financewrapped.app.staging \
  --flavor=staging

flutterfire configure \
  --project=finance-wrapped-prod \
  --out=lib/core/firebase/firebase_options_prod.dart \
  --ios-bundle-id=com.financewrapped.app \
  --android-app-id=com.financewrapped.app \
  --flavor=prod
```

**What gets generated**:
- `android/app/src/<flavor>/google-services.json`
- `ios/flavors/<flavor>/GoogleService-Info.plist` (build phase script
  copies correct plist per scheme)
- `lib/core/firebase/firebase_options_<flavor>.dart`

---

## R3. Drift Sync Queue Pattern (Offline-First)

**Decision**: Single `sync_queue` table in drift with operation type,
entity reference, JSON payload, retry metadata, and status. Processed
by a regular Dart class (not an Isolate) listening to connectivity
changes.

**Rationale**:
- Firestore plugin is tied to the main isolate (platform channels) —
  background isolates cannot call Firestore without workarounds.
- Sync processing is I/O-bound, not CPU-bound — async/await handles
  it perfectly on the main isolate.
- A custom sync queue gives full control over conflict resolution
  strategy, matching constitution Principle IV.

### Sync Queue Table Schema

```dart
class SyncQueue extends Table {
  IntColumn get id => integer().autoIncrement()();
  TextColumn get entityType => text()();       // 'usuario', 'expense', etc.
  TextColumn get entityId => text()();         // Local UUID
  TextColumn get operation => text()();        // 'insert', 'update', 'delete'
  TextColumn get payload => text()();          // JSON-serialized fields
  DateTimeColumn get createdAt =>
      dateTime().withDefault(currentDateAndTime)();
  IntColumn get retryCount =>
      integer().withDefault(const Constant(0))();
  TextColumn get lastError => text().nullable()();
  IntColumn get status =>
      integer().withDefault(const Constant(0))(); // 0=pending, 1=in_progress, 2=failed
}
```

### Processing Strategy

1. `connectivity_plus` stream triggers drain attempt
2. Verify actual Firestore reachability before processing
3. Process in FIFO order, batches of 20
4. Claim items (set `status=1`) to prevent concurrent processing
5. On success: delete from queue
6. On failure: increment `retryCount`, log error, reset to pending

### Operation Compaction (Pre-Processing)

Before draining, compact the queue:
- `insert` + multiple `update` for same entity → single `insert` with
  final payload
- `insert` + `delete` for same entity → discard both
- This reduces network calls significantly for offline editing bursts

### Conflict Resolution: Last-Write-Wins

- Every Firestore write sets `updatedAt: FieldValue.serverTimestamp()`
- Server timestamp eliminates clock-skew between devices
- `SetOptions(merge: true)` for partial updates
- On read (Firestore → local): compare timestamps, only overwrite
  local if remote is newer
- Constitution Principle IV requires this as the default strategy;
  features needing different strategies define their own in `spec.md`

### Retry Logic

- Max 5 retries per item
- Implicit backoff: retries only on next connectivity event or app
  resume (no timer — saves battery)
- Non-retryable errors (`permission-denied`, `not-found`) → dead-letter
  immediately
- Retryable errors (timeout, `unavailable`) → back to pending

### Sync Worker Architecture

```dart
class SyncWorker {
  final AppDatabase _db;
  final Connectivity _connectivity;
  bool _isSyncing = false;

  void start() {
    _connectivity.onConnectivityChanged.listen((result) {
      if (result != ConnectivityResult.none) scheduleDrain();
    });
  }

  Future<void> scheduleDrain() async {
    if (_isSyncing) return;
    _isSyncing = true;
    try { await _drainQueue(); }
    finally { _isSyncing = false; }
  }
}
```

Also trigger `scheduleDrain()` on app resume via
`WidgetsBindingObserver.didChangeAppLifecycleState`.

---

## R4. ThemeExtension + Riverpod Pattern

**Decision**: Three `ThemeExtension` subclasses (colors, typography,
spacing) assembled into `ThemeData` by an `AppTheme` class. `ThemeMode`
managed by a Riverpod `Notifier` persisted with `SharedPreferences`.

**Rationale**:
- Three extensions avoid a monolithic class and follow separation of
  concerns
- `SharedPreferences` is the lightest persistence for a single
  key-value (theme preference) — no need to use drift for this
- Convenience `BuildContext` extensions make the correct API shorter
  than hardcoding, nudging developers toward compliance

### ThemeExtension Structure

| File | Class | Purpose |
|------|-------|---------|
| `app_colors.dart` | `AppColorsExtension` | Semantic color tokens with `light()`/`dark()` factories |
| `app_typography.dart` | `AppTypographyExtension` | Full M3 type scale, derives text color from colors extension |
| `app_spacing.dart` | `AppSpacingExtension` | `xs`(4), `sm`(8), `md`(16), `lg`(24), `xl`(32), `xxl`(48) |
| `app_theme.dart` | `AppTheme` | Assembles `ThemeData.light()`/`.dark()` with all 3 extensions |
| `theme_provider.dart` | `ThemeModeNotifier` | Riverpod notifier persisted via SharedPreferences |
| `theme_extensions.dart` | `AppThemeX` on `BuildContext` | `context.colors`, `context.typography`, `context.spacing` |

### Color Token List (FR-005)

Semantic tokens: `primary`, `onPrimary`, `primaryContainer`,
`background`, `onBackground`, `surface`, `onSurface`, `surfaceVariant`,
`success`, `onSuccess`, `warning`, `onWarning`, `error`, `onError`,
`ingreso`, `onIngreso`, `egreso`, `onEgreso`.

`Color(0x...)` literals allowed ONLY in `app_colors.dart`. Everywhere
else uses `context.colors.tokenName`.

### ThemeMode Persistence

`ThemeModeNotifier` reads from `SharedPreferences` synchronously
(prefs instance resolved in `main()` before `runApp`, passed via
Riverpod override). Supports `ThemeMode.system` (default),
`.light`, `.dark`.

### Enforcement Strategy (Zero Hardcoded Values)

1. **Convenience extensions** — `context.colors.X` is shorter than
   `Color(0xFFXXXXXX)`, making the right pattern the easy path
2. **`custom_lint` rules** — Flag `Color(`, `Colors.`, `TextStyle(`,
   `EdgeInsets.` outside `lib/core/theme/`. Runs in IDE + CI.
3. **Code review** — Principle III checklist item (backstop)

---

## R5. Firestore Security Rules Testing

**Decision**: Separate Node.js project in `firestore-tests/` using
`@firebase/rules-unit-testing` against the Firestore Emulator.

**Rationale**: The Rules Unit Testing SDK is JavaScript/TypeScript only
(Firebase requirement). Cannot run from Flutter test. Keeps security
tests in a language-appropriate project while still being versioned in
the same repo.

**Alternatives considered**:
- Testing rules manually via console: Rejected — not automated, not
  reproducible, violates Principle VII.
- Using `fake_cloud_firestore` in Flutter: Rejected — it fakes the
  client-side SDK, it does NOT evaluate actual Security Rules.

---

## R6. Account Deletion Strategy

**Decision**: Two-phase approach.

1. **Phase 1 (this feature)**: Client calls
   `FirebaseAuth.instance.currentUser.delete()` + deletes the root
   document `usuarios/{uid}`. User is signed out and redirected to
   login.

2. **Phase 2 (expanded in future features)**: A Cloud Function
   (`onUserDeleted`) triggers on document deletion and cascade-deletes
   subcollections. In this feature, it only logs the deletion event
   since no subcollections exist yet. The function skeleton is deployed
   so the pattern is established.

**Rationale**: Per the draft spec's suggestion — implement the root
document deletion now, expand the Cloud Function as subcollections are
added in future features. Avoids over-engineering a cascade delete with
no subcollections to cascade.

---

## Summary of All Decisions

| Topic | Decision | Key Rationale |
|-------|----------|---------------|
| Flavor tooling | Manual config | One-time cost, no generator conflicts |
| Firebase per flavor | FlutterFire CLI `--flavor` flag | Official approach, auto-generates all config |
| Sync queue | Custom drift table + regular Dart class | Full control over conflict strategy, Firestore can't run in isolate |
| Conflict resolution | Last-write-wins via `FieldValue.serverTimestamp()` | Server clock eliminates skew, sufficient for this domain |
| Theming | 3 ThemeExtension subclasses + Riverpod notifier | Separation of concerns, SharedPreferences for persistence |
| Theme enforcement | `custom_lint` rules + BuildContext extensions | Automated + ergonomic |
| Security Rules testing | Node.js project with `@firebase/rules-unit-testing` | Firebase SDK requirement, only way to test actual rules |
| Account deletion | Root doc delete now, Cloud Function skeleton for cascade | Avoids over-engineering with no subcollections yet |
