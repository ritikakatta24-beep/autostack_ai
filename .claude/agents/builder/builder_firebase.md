# Sub-Agent: Builder Firebase

## Role
Wire all Firebase services, generate security rules, and set up Cloud Functions. Called by `builder.md` as Phase 2.

## Inputs
Read these fields from `system_plan.json`:
- `backend.firebase_services` → list of services to initialize
- `backend.features` → each feature's `type` and `collection` name
- `app_name` → used in function names and comments

Only generate sections for services that appear in `backend.firebase_services`.

---

## firebase_service.dart (always generate)

```dart
// lib/shared/services/firebase_service.dart
class FirebaseService {
  static Future<void> initialize() async {
    await Firebase.initializeApp(
      options: DefaultFirebaseOptions.currentPlatform,
    );
  }

  // Only include getters for services in backend.firebase_services:
  static FirebaseAuth get auth => FirebaseAuth.instance;           // if 'auth' in services
  static FirebaseFirestore get db => FirebaseFirestore.instance;   // if 'firestore' in services
  static FirebaseStorage get storage => FirebaseStorage.instance;  // if 'storage' in services
  static FirebaseFunctions get functions =>                        // if 'functions' in services
      FirebaseFunctions.instanceFor(region: 'asia-south1');
}
```

---

## Auth (include if `auth` in backend.firebase_services)

```dart
// lib/shared/services/auth_service.dart
class AuthService {
  final _auth = FirebaseService.auth;

  // Always a stream — never one-time read
  Stream<User?> get userStream => _auth.authStateChanges();

  Future<Either<AppException, UserCredential>> signIn(
      String email, String password) async {
    try {
      final cred = await _auth.signInWithEmailAndPassword(
          email: email, password: password);
      return Right(cred);
    } on FirebaseAuthException catch (e) {
      return Left(AuthException(e.code));
    }
  }

  Future<Either<AppException, UserCredential>> signUp(
      String email, String password) async {
    try {
      final cred = await _auth.createUserWithEmailAndPassword(
          email: email, password: password);
      // Always create a Firestore user document after sign-up
      await FirebaseService.db.collection('users').doc(cred.user!.uid).set({
        'uid': cred.user!.uid,
        'email': email,
        'createdAt': FieldValue.serverTimestamp(),
      });
      return Right(cred);
    } on FirebaseAuthException catch (e) {
      return Left(AuthException(e.code));
    }
  }

  Future<void> signOut() => _auth.signOut();
}

// Riverpod providers
@riverpod
AuthService authService(Ref ref) => AuthService();

@riverpod
Stream<User?> authState(Ref ref) =>
    ref.watch(authServiceProvider).userStream;
```

### Auth providers by plan
| Provider requested | Implementation |
|---|---|
| email | `signInWithEmailAndPassword` (above) |
| google | `GoogleSignIn` package + `GoogleAuthProvider.credential` |
| phone | `verifyPhoneNumber` + `PhoneAuthProvider.credential` |
| anonymous | `signInAnonymously()` |

### Callable functions from Flutter side (include if `functions` in services)

```dart
// How to call a Cloud Function from Flutter:
final result = await FirebaseService.functions
    .httpsCallable('[featureName]')
    .call({'param': value});
```

---

## Firestore (include if `firestore` in backend.firebase_services)

Generate one repository per feature where `feature.type != "auth"` and `feature.collection != null`:

```dart
// lib/features/[feature.name]/data/[feature]_repository.dart
class [Feature]Repository {
  final _db = FirebaseService.db;

  // Always .withConverter() — prevents runtime cast errors (FL_002)
  CollectionReference<[Feature]Model> get _col =>
      _db.collection('[feature.collection]').withConverter(
        fromFirestore: (snap, _) => [Feature]Model.fromJson(snap.data()!),
        toFirestore:   (item, _) => item.toJson(),
      );

  Stream<List<[Feature]Model>> watchAll() =>
      _col.snapshots().map((s) => s.docs.map((d) => d.data()).toList());

  // Watch filtered by current user
  Stream<List<[Feature]Model>> watchByOwner(String uid) =>
      _col.where('ownerId', isEqualTo: uid)
          .orderBy('createdAt', descending: true)
          .snapshots()
          .map((s) => s.docs.map((d) => d.data()).toList());

  Future<Either<AppException, [Feature]Model>> fetchById(String id) async {
    try {
      final doc = await _col.doc(id).get();
      if (!doc.exists) return Left(DatabaseException('Not found'));
      return Right(doc.data()!);
    } on FirebaseException catch (e) {
      return Left(DatabaseException(e.message ?? 'Unknown'));
    }
  }

  Future<Either<AppException, void>> save([Feature]Model item) async {
    try {
      await _col.doc(item.id).set(item);
      return const Right(null);
    } on FirebaseException catch (e) {
      return Left(DatabaseException(e.message ?? 'Unknown'));
    }
  }

  Future<Either<AppException, void>> delete(String id) async {
    try {
      await _col.doc(id).delete();
      return const Right(null);
    } on FirebaseException catch (e) {
      return Left(DatabaseException(e.message ?? 'Unknown'));
    }
  }
}

// Riverpod provider
@riverpod
[Feature]Repository [feature]Repository(Ref ref) => [Feature]Repository();
```

### firestore.rules (always write — one block per collection in backend.features)

```
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {

    match /users/{userId} {
      allow read, write: if request.auth != null
                         && request.auth.uid == userId;
    }

    // generated from system_plan.json backend.features — one block per feature.collection
    match /[feature.collection]/{docId} {
      allow read: if request.auth != null;
      allow create: if request.auth != null
                    && request.resource.data.ownerId == request.auth.uid;
      allow update, delete: if request.auth != null
                            && resource.data.ownerId == request.auth.uid;
    }
  }
}
```

### firestore.indexes.json (generate one block per feature.collection that uses watchByOwner)

```json
{
  "indexes": [
    {
      "collectionGroup": "[feature.collection]",
      "queryScope": "COLLECTION",
      "fields": [
        { "fieldPath": "ownerId",   "order": "ASCENDING"  },
        { "fieldPath": "createdAt", "order": "DESCENDING" }
      ]
    }
  ],
  "fieldOverrides": []
}
```

> **Note:** Generate one index block per feature collection that calls `watchByOwner()`. If multiple features exist, repeat the index block for each `feature.collection`.

---

## Storage (include if `storage` in backend.firebase_services)

### storage_service.dart

```dart
// lib/shared/services/storage_service.dart
class StorageService {
  final _storage = FirebaseService.storage;

  // Upload to user-scoped path — prevents cross-user access
  Future<Either<AppException, String>> uploadFile({
    required String uid,
    required String fileName,
    required Uint8List bytes,
    required String contentType,
  }) async {
    try {
      final ref = _storage.ref('users/$uid/$fileName');
      final task = await ref.putData(
        bytes,
        SettableMetadata(contentType: contentType),
      );
      final url = await task.ref.getDownloadURL();
      return Right(url);
    } on FirebaseException catch (e) {
      return Left(DatabaseException(e.message ?? 'Upload failed'));
    }
  }

  Future<Either<AppException, void>> deleteFile(String path) async {
    try {
      await _storage.ref(path).delete();
      return const Right(null);
    } on FirebaseException catch (e) {
      return Left(DatabaseException(e.message ?? 'Delete failed'));
    }
  }
}

@riverpod
StorageService storageService(Ref ref) => StorageService();
```

### storage.rules (always write)

```
rules_version = '2';
service firebase.storage {
  match /b/{bucket}/o {

    // User-scoped uploads — full isolation per user
    match /users/{userId}/{allPaths=**} {
      allow read, write: if request.auth != null
                         && request.auth.uid == userId;
    }

    // Shared uploads — size + type restricted, read by any auth user
    match /uploads/{allPaths=**} {
      allow write: if request.auth != null
        && request.resource.size < 10 * 1024 * 1024
        && request.resource.contentType.matches('image/.*');
      allow read: if request.auth != null;
    }
  }
}
```

---

## Cloud Functions (include if `functions` in backend.firebase_services)

### functions/package.json (always generate)

```json
{
  "name": "functions",
  "scripts": {
    "build": "tsc",
    "serve": "npm run build && firebase emulators:start --only functions",
    "deploy": "firebase deploy --only functions"
  },
  "dependencies": {
    "firebase-admin": "^12.0.0",
    "firebase-functions": "^5.0.0"
  },
  "devDependencies": {
    "typescript": "^5.3.0",
    "firebase-functions-test": "^3.1.0"
  }
}
```

### functions/src/index.ts

```typescript
// functions/src/index.ts — always TypeScript
import * as admin from 'firebase-admin';
import * as functions from 'firebase-functions/v2';

admin.initializeApp();

// Generated from backend.features — one callable per feature that needs server logic
export const [featureName] = functions.https.onCall(
  { region: 'asia-south1' },
  async (request) => {
    // Always auth-guard every callable function
    if (!request.auth) {
      throw new functions.https.HttpsError('unauthenticated', 'Login required');
    }
    try {
      // logic here
      return { success: true };
    } catch (error) {
      // Never expose internal error details to client
      throw new functions.https.HttpsError('internal', 'Function failed');
    }
  }
);
```

Rules:
- Always TypeScript — never plain JS in Functions
- Always `region: 'asia-south1'` — lower latency for India
- Always auth-guard every callable function
- Always call `admin.initializeApp()` at top of `index.ts`
- Never expose internal error details to client
- Always return a plain object `{ success: true, ... }` — never return `undefined`

---

## Error Codes (report to error_handler.md)
| Code | Trigger |
|---|---|
| `FB_001` | `permission-denied` on Firestore — regenerate rules from `system_plan.json`.backend.features |
| `FB_002` | `FirebaseApp not initialized` — check `firebase_service.dart` initialize() called in main.dart |
| `FB_003` | Auth provider not enabled in Firebase Console — escalate to user |
| `FB_004` | Functions cold start timeout — verify region set, `admin.initializeApp()` outside handler |
| `FB_005` | Storage CORS error — provide `gsutil cors set cors.json gs://[bucket]` command to user |
| `FB_006` | Firestore index missing — add composite index block to `firestore.indexes.json` per collection |
| `FB_007` | `signUp` succeeds but user doc missing — check Firestore write in `signUp` after `createUserWithEmailAndPassword` |
| `FB_008` | Functions returns `undefined` — ensure every callable returns a plain object |
