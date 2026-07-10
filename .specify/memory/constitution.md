<!--
  Sync Impact Report
  ===================
  Version change: 1.0.0 → 1.1.0
  Modified principles:
    - III. Offline-First → renumbered IV, unchanged in substance, but now
      explicitly names Firestore as the remote sync backend and defines a
      default conflict-resolution strategy (previously left open)
  Added sections:
    - New Principle III: Theme-Driven UI (no hardcoded colors/fonts, light/dark)
    - New Principle VII: Data Privacy & Regulatory Compliance
    - Technology Constraints: Backend/Sync, State Management (Riverpod),
      Payments (RevenueCat), Environments/Flavors
    - New section: Environments & Flavors
    - New section: Monetization Reference (pointer to contexto-negocio.md)
  Removed sections: None
  Templates requiring updates:
    - .specify/templates/plan-template.md ✅ no changes needed
    - .specify/templates/spec-template.md ✅ no changes needed
    - .specify/templates/tasks-template.md ✅ no changes needed
  Follow-up TODOs:
    - Confirm final default conflict-resolution strategy (last-write-wins)
      is acceptable once the first sync-heavy feature (Fondo de Ahorro) is
      specified — flag any feature needing a different strategy explicitly
      in its own spec.md.
-->

# Finance Wrapped Constitution

## Core Principles

### I. Clean Architecture

All code MUST follow strict separation of concerns across three
layers: presentation, domain, and data. Dependencies MUST point
inward only (presentation → domain ← data). The domain layer
MUST NOT import Flutter, third-party packages, or data-layer
code. Each layer MUST reside in its own directory under `lib/`.

- Presentation layer: widgets, screens, state management
- Domain layer: entities, use cases, repository interfaces
- Data layer: repository implementations, data sources,
  models, API clients

Feature code MUST live under `lib/features/<feature_name>/` with
its own `data/`, `domain/`, and `presentation/` subfolders.
Cross-feature shared code (theme, routing, DI setup, common
utils) lives under `lib/core/`.

**Rationale**: Enforcing layer boundaries prevents coupling,
enables testability without Flutter test harness, and allows
swapping data sources without touching business logic.

### II. Widget-First Architecture

Every UI feature MUST start as a composable, reusable widget.
Widgets MUST be self-contained: they declare their own inputs
via constructor parameters and MUST NOT depend on global state
or singletons directly. Shared widgets MUST live in a dedicated
`lib/widgets/` directory and MUST be independently testable
via widget tests.

- Prefer composition over inheritance for widget reuse
- Extract repeated UI patterns into shared widgets
- Each screen MUST be composed of smaller, focused widgets

**Rationale**: Composable widgets reduce duplication, improve
testability, and make the UI layer easier to refactor.

### III. Theme-Driven UI

No widget MAY declare a hardcoded color, font size, spacing
value, or border radius. Every visual token MUST be consumed
from a central `AppTheme` exposed via `ThemeExtension`, itself
provided through the app's state management layer (see
Technology Constraints).

- The app MUST support light and dark mode from the first
  screen built, defaulting to `ThemeMode.system` with a manual
  override available to the user
- Semantic color tokens MUST be used instead of raw values
  (e.g., `theme.colors.income`, `theme.colors.expense`,
  `theme.colors.error`) so meaning survives a future palette
  change
- A PR introducing any `Color(0x...)`, raw `TextStyle(...)`, or
  raw `EdgeInsets` literal inside a feature widget MUST be
  rejected in review

**Rationale**: An app whose entire narrative hook (the
"Wrapped") depends on strong visual identity cannot afford
inconsistent, hand-tuned styling scattered across screens. A
single theming source also makes dark mode and future
rebranding low-risk.

### IV. Offline-First

The app MUST function without network connectivity. **Local
storage (SQLite via `drift`) is the source of truth on-device.**
All read operations MUST resolve from local data. Write
operations MUST persist locally first, then sync to Firebase
(Firestore) when connectivity is available. The UI MUST clearly
indicate sync status to the user (e.g., a subtle pending/synced
indicator, never a blocking spinner on core flows).

- Network failures MUST NOT block core user workflows,
  especially transaction registration (the sub-3-second flow)
- **Default conflict-resolution strategy**: last-write-wins at
  the document/field level, arbitrated by server timestamp on
  sync. Any feature whose data cannot tolerate last-write-wins
  (e.g., a running balance like the Fondo de Ahorro, where two
  offline devices could both apply deltas) MUST define its own
  merge strategy explicitly in that feature's `spec.md` before
  implementation — additive/delta operations (increment/decrement)
  MUST be modeled as operations to replay, not as overwritten
  absolute values, specifically to avoid this class of conflict
- Data migrations MUST be handled gracefully across app versions
  (drift migration scripts versioned and tested)
- Firestore acts as the sync/backup backend and the mechanism for
  multi-device access; it is NOT queried directly by the
  presentation layer — only through repositories that reconcile
  it with the local `drift` database

**Rationale**: Users track expenses on the go, often in areas
with poor connectivity. Offline-first ensures the app is always
usable, and a local relational store gives real transactional
guarantees that a remote cache cannot.

### V. Test-First Development (NON-NEGOTIABLE)

Tests MUST be written before implementation code. The
Red-Green-Refactor cycle is strictly enforced:

1. Write a failing test that defines expected behavior
2. Implement the minimum code to make the test pass
3. Refactor while keeping tests green

Unit tests MUST cover all domain-layer use cases. Widget
tests MUST cover all user-facing interactions. Integration
tests MUST cover critical user journeys (e.g., adding an
expense, viewing reports, generating a Wrapped).

**Rationale**: TDD prevents regressions, documents expected
behavior, and enforces the Clean Architecture boundaries
by requiring testable code from the start.

### VI. Data Integrity

All financial data (amounts, currencies, dates) MUST be
validated at system boundaries before persistence. Amounts
MUST be stored as integer minor units (e.g., cents) to
avoid floating-point errors. Currency MUST be explicitly
associated with every monetary value. Deletions MUST use
soft-delete with a `deletedAt` timestamp; hard deletes
are prohibited in production data paths.

- Input validation MUST reject negative amounts, missing
  currencies, and future-dated expenses (unless explicitly
  allowed by the feature spec)
- All write operations MUST be wrapped in transactions
- Backup/export functionality MUST preserve full fidelity
- Category type (`ingreso`/`egreso`) MUST be resolved from the
  category itself, not re-derived or guessed at query time —
  see `contexto-negocio.md` §1 for the income/expense detection
  model

**Rationale**: An expenses tracker handles real financial
data. Precision errors, silent data loss, or corruption
directly undermine user trust.

### VII. Data Privacy & Regulatory Compliance

Financial data is sensitive data and MUST be handled with the
same rigor as health data.

- Compliance with Mexico's LFPDPPP is the minimum bar; the data
  model MUST NOT preclude extending to GDPR-level guarantees if
  the app expands internationally
- Firestore Security Rules MUST enforce that a user can only
  read/write documents they own (`request.auth.uid ==
  resource.data.userId`), with no exceptions, verified by
  automated tests against the Firestore Emulator — not manual
  review alone
- No monetary amount, transaction concept/note, or category
  detail MAY be sent to third-party analytics. Firebase
  Analytics events MUST be aggregate/anonymous only (e.g.
  `wrapped_generated`), never containing amounts or free-text
  fields
- Users MUST be able to self-service delete their account and
  all associated data from within the app
- No secret, API key, or environment credential is ever
  committed to the repository

**Rationale**: Trust is the product for a finance app. A single
privacy or security failure is existential, not just a bug.

## Technology Constraints

- **Language**: Dart (latest stable)
- **Framework**: Flutter (latest stable channel)
- **Target Platforms**: iOS, iPadOS, and Android at MVP;
  Apple Watch and Wear OS are a later phase, not part of the
  initial scope
- **Minimum OS versions**: iOS/iPadOS 15+, Android 8.0 (API 26)+
- **Local Storage**: SQLite via `drift` — source of truth on
  device (Principle IV)
- **Backend / Sync**: Firebase — Firestore (sync + backup),
  Firebase Authentication (Apple, Google, Email/Password),
  Firebase Analytics, Crashlytics
- **State Management**: Riverpod, used consistently across all
  features — MUST NOT mix with another state management
  approach within the same feature
- **Dependency Injection**: Provided via Riverpod's own provider
  graph; no separate DI framework unless a future plan justifies
  it explicitly
- **Payments / Subscriptions**: RevenueCat (`purchases_flutter`)
  wrapping StoreKit and Google Play Billing. The final premium
  entitlement MUST be resolved as
  `isPremium = revenueCat.isActive OR firestore.is_premium_whitelisted`
- **Minimum Test Coverage**: 80% line coverage for domain layer;
  widget tests for every screen; Firestore Security Rules MUST
  have automated emulator tests

## Environments & Flavors

Three Flutter flavors MUST exist: `dev`, `staging`, `prod` — each
backed by its **own, separate Firebase project** (not just a
different config within one project):

| Flavor | Firebase Project | App ID |
|---|---|---|
| `dev` | `finance-wrapped-dev` | `com.financewrapped.app.dev` |
| `staging` | `finance-wrapped-staging` | `com.financewrapped.app.staging` |
| `prod` | `finance-wrapped-prod` | `com.financewrapped.app` |

- Each flavor MUST have a distinguishable app name/icon on
  device to avoid confusing installs during testing
- Firestore Security Rules MUST be a single source, versioned in
  the repo, and deployed identically to all three projects
- The `is_premium_whitelisted` manual flag is only ever set by
  hand in `prod` (and optionally `staging` for QA)
- `prod` builds MUST NEVER point at `dev`/`staging` Firebase
  projects or vice versa

## Monetization Reference

Pricing, competitor benchmarking, subscription tiers, and the
`is_premium_whitelisted` mechanism are documented in
`contexto-negocio.md`. That document is the source of truth for
*business* decisions; this constitution only fixes the
*technical* contract (`isPremium` resolution logic above) that
every feature spec MUST rely on.

## Development Workflow

- Every feature MUST have a specification (`spec.md`)
  approved before implementation begins
- Every feature MUST have a plan (`plan.md`) reviewed
  against this constitution before task generation
- Code reviews MUST verify compliance with all seven
  core principles
- Each pull request MUST be scoped to a single user
  story or a single infrastructure concern
- Commits MUST be atomic: one logical change per commit
- All linting rules (`flutter analyze`) MUST pass with
  zero warnings before merge

## Governance

This constitution supersedes all other development
practices for the Finance Wrapped project. Amendments require:

1. A written proposal describing the change and its
   rationale
2. An impact assessment against existing specifications
   and plans
3. A version bump following semantic versioning:
   - MAJOR: principle removal or incompatible redefinition
   - MINOR: new principle or materially expanded guidance
   - PATCH: clarification, wording, or typo fix
4. Update of all dependent templates and artifacts

Compliance review: every plan MUST include a
"Constitution Check" section verifying alignment with
all active principles before implementation proceeds.

**Version**: 1.1.0 | **Ratified**: 2026-07-09 | **Last Amended**: 2026-07-09