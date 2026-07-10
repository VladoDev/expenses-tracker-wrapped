# Data Model: Setup Fundacional del Proyecto

**Feature**: 001-setup-fundacional
**Date**: 2026-07-09

---

## Entities

### 1. Usuario

Represents a user's identity and base profile. Exists in both Firestore
and the local drift database.

**Firestore path**: `usuarios/{uid}`

| Field | Type | Required | Default | Notes |
|-------|------|----------|---------|-------|
| `uid` | string | Yes | — | Primary key, matches Firebase Auth UID |
| `email` | string | Yes | — | From Firebase Auth provider |
| `fechaCreacion` | timestamp | Yes | Server timestamp | Set once on first login |
| `updatedAt` | timestamp | Yes | Server timestamp | Updated on every write (LWW arbiter) |
| `is_premium_whitelisted` | boolean | Yes | `false` | Manual flag, set by admin only |
| `locale` | string | No | `"es"` | Reserved for future i18n |
| `deletedAt` | timestamp | No | `null` | Soft-delete marker (set before hard-delete) |

**Validation rules**:
- `uid` MUST match the authenticated user's `auth.uid`
- `email` MUST be a non-empty string
- `fechaCreacion` MUST be a valid timestamp, immutable after creation
- `is_premium_whitelisted` MUST be a boolean

**Drift table** (`usuarios_table.dart`):

| Column | Dart Type | SQLite Type | Notes |
|--------|-----------|-------------|-------|
| `uid` | `String` | `TEXT` | Primary key |
| `email` | `String` | `TEXT` | Not null |
| `fechaCreacion` | `DateTime` | `INTEGER` (epoch ms) | Not null |
| `updatedAt` | `DateTime` | `INTEGER` (epoch ms) | Not null |
| `isPremiumWhitelisted` | `bool` | `INTEGER` (0/1) | Default false |
| `locale` | `String?` | `TEXT` | Nullable |
| `deletedAt` | `DateTime?` | `INTEGER` (epoch ms) | Nullable |

---

### 2. SyncQueueEntry

Represents a pending write operation in the offline sync queue. Exists
only in the local drift database (never synced to Firestore).

**Drift table** (`sync_queue_table.dart`):

| Column | Dart Type | SQLite Type | Notes |
|--------|-----------|-------------|-------|
| `id` | `int` | `INTEGER` | Auto-increment PK |
| `entityType` | `String` | `TEXT` | e.g. `'usuario'` |
| `entityId` | `String` | `TEXT` | UUID of the target entity |
| `operation` | `String` | `TEXT` | `'insert'`, `'update'`, `'delete'` |
| `payload` | `String` | `TEXT` | JSON-serialized field map |
| `createdAt` | `DateTime` | `INTEGER` (epoch ms) | Auto-set on insert |
| `retryCount` | `int` | `INTEGER` | Default 0 |
| `lastError` | `String?` | `TEXT` | Nullable, last failure message |
| `status` | `int` | `INTEGER` | 0=pending, 1=in_progress, 2=failed, 3=dead |

**Indexes**:
- `(status, createdAt)` — for efficient FIFO queue drain queries
- `(entityType, entityId)` — for operation compaction lookups

**Validation rules**:
- `operation` MUST be one of `'insert'`, `'update'`, `'delete'`
- `payload` MUST be valid JSON
- `status` MUST be 0, 1, 2, or 3
- `retryCount` MUST be >= 0

---

## Entity Relationships

```text
┌──────────────┐
│   Usuario    │ ← Source of truth: drift (local)
│              │   Synced to: Firestore (remote)
│  uid (PK)    │
│  email       │
│  fechaCreacion│
│  updatedAt   │
│  ...         │
└──────┬───────┘
       │ writes generate
       ▼
┌──────────────┐
│ SyncQueueEntry│ ← Local only (never synced)
│              │   Processed by SyncWorker
│  id (PK)     │
│  entityType  │ ── "usuario"
│  entityId    │ ── references Usuario.uid
│  operation   │
│  payload     │
│  status      │
└──────────────┘
```

---

## DTOs (Data Transfer Objects)

### UsuarioDto (Freezed)

Bridges between Firestore documents and drift rows. Lives in
`lib/features/auth/data/models/usuario_dto.dart`.

```dart
@freezed
class UsuarioDto with _$UsuarioDto {
  const factory UsuarioDto({
    required String uid,
    required String email,
    required DateTime fechaCreacion,
    required DateTime updatedAt,
    @Default(false) bool isPremiumWhitelisted,
    String? locale,
    DateTime? deletedAt,
  }) = _UsuarioDto;

  factory UsuarioDto.fromJson(Map<String, dynamic> json) =>
      _$UsuarioDtoFromJson(json);

  factory UsuarioDto.fromFirestore(DocumentSnapshot doc) { ... }
  factory UsuarioDto.fromDrift(UsuarioRow row) { ... }

  Map<String, dynamic> toFirestore() { ... }
  UsuariosCompanion toDrift() { ... }
}
```

---

## Domain Entity (Pure Dart)

### Usuario (Domain)

Lives in `lib/features/auth/domain/entities/usuario.dart`. No Flutter
or Firebase imports.

```dart
@freezed
class Usuario with _$Usuario {
  const factory Usuario({
    required String uid,
    required String email,
    required DateTime fechaCreacion,
    required DateTime updatedAt,
    required bool isPremiumWhitelisted,
    String? locale,
  }) = _Usuario;
}
```

Note: `deletedAt` is excluded from the domain entity — soft-deleted
users are filtered out at the data layer and never surfaced to the
domain/presentation layers.

---

## State Transitions

### User Lifecycle

```text
[Unauthenticated] ──login──▶ [Authenticated]
                                   │
                              create doc in
                              Firestore + drift
                                   │
                                   ▼
                            [Active User]
                                   │
                        ┌──────────┴──────────┐
                        │                     │
                    sign out              delete account
                        │                     │
                        ▼                     ▼
                [Unauthenticated]     [Soft-deleted]
                                          │
                                    Cloud Function
                                    hard-deletes doc
                                          │
                                          ▼
                                      [Purged]
```

### Sync Queue Entry Lifecycle

```text
[Created: status=0 pending]
        │
   SyncWorker claims
        │
        ▼
[Processing: status=1 in_progress]
        │
   ┌────┴────┐
   │         │
 success   failure
   │         │
   ▼         ▼
[Deleted]  retryCount < 5?
              │
         ┌────┴────┐
         │         │
        yes        no
         │         │
         ▼         ▼
   [Pending:    [Dead: status=3]
    status=0,    (manual intervention
    retryCount   or discard)
    incremented]
```

---

## Migration Strategy

This is the first schema version (v1). Future migrations will use
drift's schema migration API.

```dart
@DriftDatabase(tables: [Usuarios, SyncQueue])
class AppDatabase extends _$AppDatabase {
  @override
  int get schemaVersion => 1;

  @override
  MigrationStrategy get migration => MigrationStrategy(
    onCreate: (m) => m.createAll(),
  );
}
```

Each future schema change MUST:
1. Increment `schemaVersion`
2. Add a migration step in `onUpgrade`
3. Include a test verifying the migration from the previous version
