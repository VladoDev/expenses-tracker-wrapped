# Quickstart: Setup Fundacional del Proyecto

**Feature**: 001-setup-fundacional
**Date**: 2026-07-09

---

## Prerequisites

1. **Flutter** (latest stable) installed via `fvm`:
   ```bash
   dart pub global activate fvm
   fvm install stable
   fvm use stable
   ```
   Verify: `fvm flutter --version`

2. **Firebase CLI** installed:
   ```bash
   npm install -g firebase-tools
   firebase login
   ```

3. **FlutterFire CLI** installed:
   ```bash
   dart pub global activate flutterfire_cli
   ```

4. **Firebase Emulator Suite** for Security Rules testing:
   ```bash
   firebase init emulators
   # Select Firestore emulator
   ```

5. **Access** to the 3 Firebase projects:
   - `finance-wrapped-dev`
   - `finance-wrapped-staging`
   - `finance-wrapped-prod`

6. **Xcode** (latest stable) for iOS builds

7. **Android Studio** with SDK 26+ for Android builds

---

## Setup Steps

### 1. Clone and Install Dependencies

```bash
git clone <repo-url>
cd expenses_tracker_wrapped
fvm flutter pub get
```

### 2. Run Code Generation

```bash
fvm flutter pub run build_runner build --delete-conflicting-outputs
```

This generates:
- Riverpod providers (`*.g.dart`)
- Freezed models (`*.freezed.dart`, `*.g.dart`)
- Drift database code (`*.g.dart`)

### 3. Configure Firebase per Flavor

```bash
# Dev
flutterfire configure \
  --project=finance-wrapped-dev \
  --out=lib/core/firebase/firebase_options_dev.dart \
  --ios-bundle-id=com.financewrapped.app.dev \
  --android-app-id=com.financewrapped.app.dev \
  --flavor=dev

# Staging
flutterfire configure \
  --project=finance-wrapped-staging \
  --out=lib/core/firebase/firebase_options_staging.dart \
  --ios-bundle-id=com.financewrapped.app.staging \
  --android-app-id=com.financewrapped.app.staging \
  --flavor=staging

# Prod
flutterfire configure \
  --project=finance-wrapped-prod \
  --out=lib/core/firebase/firebase_options_prod.dart \
  --ios-bundle-id=com.financewrapped.app \
  --android-app-id=com.financewrapped.app \
  --flavor=prod
```

### 4. Deploy Firestore Security Rules

```bash
# To all 3 projects
firebase deploy --only firestore:rules --project finance-wrapped-dev
firebase deploy --only firestore:rules --project finance-wrapped-staging
firebase deploy --only firestore:rules --project finance-wrapped-prod
```

### 5. Deploy Cloud Functions

```bash
cd functions
npm install
cd ..
firebase deploy --only functions --project finance-wrapped-dev
firebase deploy --only functions --project finance-wrapped-staging
firebase deploy --only functions --project finance-wrapped-prod
```

---

## Running the App

```bash
# Dev (default for development)
fvm flutter run --flavor dev -t lib/main_dev.dart

# Staging
fvm flutter run --flavor staging -t lib/main_staging.dart

# Prod (only for release testing)
fvm flutter run --flavor prod -t lib/main_prod.dart
```

Each flavor connects to its own Firebase project. Dev and staging show
an environment badge in the app UI; prod does not.

---

## Running Tests

### Unit & Widget Tests

```bash
fvm flutter test
```

### Integration Tests

```bash
fvm flutter test integration_test/auth_flow_test.dart \
  --flavor dev -t lib/main_dev.dart
```

### Firestore Security Rules Tests

```bash
# Terminal 1: Start emulator
firebase emulators:start --only firestore

# Terminal 2: Run tests
cd firestore-tests
npm install
npm test
```

### Golden Tests

```bash
fvm flutter test test/golden/
```

To update golden files after intentional visual changes:
```bash
fvm flutter test test/golden/ --update-goldens
```

### Linting

```bash
fvm flutter analyze
dart run custom_lint
```

Both MUST pass with zero warnings before merging.

---

## Verification Checklist

After completing setup, verify:

- [ ] `flutter run --flavor dev` starts and shows "DEV" badge
- [ ] `flutter run --flavor staging` starts and shows "STAGING" badge
- [ ] `flutter run --flavor prod` starts with no badge
- [ ] All 3 flavors connect to their correct Firebase project (check
      Analytics dashboard for test event)
- [ ] Register with Google → user doc created in Firestore
- [ ] Register with Apple → user doc created in Firestore
- [ ] Register with email/password → verification email sent, user doc
      created
- [ ] Sign out and sign back in → same user data accessible
- [ ] Password recovery → reset email sent and works
- [ ] Toggle theme to dark mode → all colors update automatically
- [ ] Toggle theme to light mode → all colors update correctly
- [ ] Kill and reopen app → theme preference persists
- [ ] Turn on airplane mode → app shows local data without errors
- [ ] Turn off airplane mode → pending sync items flush to Firestore
- [ ] Delete account → user doc removed, signed out, redirected to
      login
- [ ] `flutter analyze` → zero warnings
- [ ] `dart run custom_lint` → zero warnings
- [ ] `flutter test` → all tests pass
- [ ] Firestore Security Rules tests → all pass against emulator
