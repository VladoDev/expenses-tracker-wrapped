# Firestore Schema Contract: Setup Fundacional

**Feature**: 001-setup-fundacional
**Date**: 2026-07-09

---

## Collection: `usuarios`

**Path**: `usuarios/{uid}`

This is the only Firestore collection created in this feature. Future
features will add collections (transactions, budgets, etc.) following
the same patterns established here.

### Document Schema

```json
{
  "uid": "string (matches doc ID and auth.uid)",
  "email": "string (non-empty)",
  "fechaCreacion": "Timestamp (server, immutable after creation)",
  "updatedAt": "Timestamp (server, updated on every write)",
  "is_premium_whitelisted": "boolean (default: false)",
  "locale": "string (optional, default: 'es')",
  "deletedAt": "Timestamp | null (soft-delete marker)"
}
```

### Write Contract

**Create** (on first login):
```javascript
firestore.collection('usuarios').doc(auth.uid).set({
  uid: auth.uid,
  email: auth.email,
  fechaCreacion: FieldValue.serverTimestamp(),
  updatedAt: FieldValue.serverTimestamp(),
  is_premium_whitelisted: false,
  locale: 'es',
  deletedAt: null
});
```

**Update** (any subsequent write):
```javascript
firestore.collection('usuarios').doc(auth.uid).set({
  ...changedFields,
  updatedAt: FieldValue.serverTimestamp()
}, { merge: true });
```

**Delete** (account deletion):
```javascript
// Step 1: Soft-delete (set marker)
firestore.collection('usuarios').doc(auth.uid).update({
  deletedAt: FieldValue.serverTimestamp(),
  updatedAt: FieldValue.serverTimestamp()
});
// Step 2: Hard-delete (via Cloud Function or client)
firestore.collection('usuarios').doc(auth.uid).delete();
```

### Read Contract

All reads of `usuarios` documents go through the local drift database,
NOT directly from Firestore. Firestore reads only happen during sync
reconciliation in the data layer.

---

## Firestore Security Rules

**File**: `firestore.rules` (repository root)

```javascript
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {

    // usuarios collection: strict uid isolation
    match /usuarios/{userId} {
      allow read: if request.auth != null
                  && request.auth.uid == userId;

      allow create: if request.auth != null
                    && request.auth.uid == userId
                    && request.resource.data.uid == userId
                    && request.resource.data.email is string
                    && request.resource.data.email.size() > 0
                    && request.resource.data.is_premium_whitelisted == false;

      allow update: if request.auth != null
                    && request.auth.uid == userId
                    && request.resource.data.uid == userId
                    // fechaCreacion is immutable
                    && request.resource.data.fechaCreacion
                       == resource.data.fechaCreacion
                    // is_premium_whitelisted cannot be changed by user
                    && request.resource.data.is_premium_whitelisted
                       == resource.data.is_premium_whitelisted;

      allow delete: if request.auth != null
                    && request.auth.uid == userId;
    }

    // Default deny all
    match /{document=**} {
      allow read, write: if false;
    }
  }
}
```

### Security Rules Test Cases

Tests live in `firestore-tests/tests/security-rules.test.ts` using
`@firebase/rules-unit-testing`.

| Test Case | Expected |
|-----------|----------|
| Authenticated user reads own doc | Allow |
| Authenticated user reads another user's doc | Deny |
| Authenticated user creates own doc with correct fields | Allow |
| Authenticated user creates doc with wrong `uid` field | Deny |
| Authenticated user creates doc with `is_premium_whitelisted: true` | Deny |
| Authenticated user updates own doc (mutable fields only) | Allow |
| Authenticated user tries to change `fechaCreacion` | Deny |
| Authenticated user tries to change `is_premium_whitelisted` | Deny |
| Authenticated user writes to another user's doc | Deny |
| Authenticated user deletes own doc | Allow |
| Authenticated user deletes another user's doc | Deny |
| Unauthenticated user reads any doc | Deny |
| Unauthenticated user writes any doc | Deny |
| Access to undefined collection | Deny |

---

## Cloud Function: `onUserDeleted`

**Trigger**: `onDocumentDeleted` on `usuarios/{uid}`
**Runtime**: Node.js (TypeScript)
**Location**: `functions/src/onUserDeleted.ts`

### Behavior (this feature)

1. Log the deletion event (uid, timestamp)
2. No cascade deletion needed yet (no subcollections exist)

### Future Expansion

When features add subcollections under `usuarios/{uid}/` (transactions,
budgets, etc.), this function will be expanded to cascade-delete them:

```typescript
export const onUserDeleted = onDocumentDeleted(
  "usuarios/{uid}",
  async (event) => {
    const uid = event.params.uid;
    // Future: delete subcollections
    // await deleteCollection(`usuarios/${uid}/transacciones`);
    // await deleteCollection(`usuarios/${uid}/presupuestos`);
    logger.info(`User ${uid} document deleted`, { uid });
  }
);
```

---

## Firebase Analytics Events (this feature)

Per Principle VII, only aggregate/anonymous events. No amounts, no
free-text fields.

| Event Name | Parameters | When |
|------------|------------|------|
| `sign_up` | `method` (google/apple/email) | First-time registration |
| `login` | `method` (google/apple/email) | Each login |
| `account_deleted` | — | Account deletion completed |
| `theme_changed` | `mode` (system/light/dark) | Theme preference changed |
