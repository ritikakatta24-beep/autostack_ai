# Skill: Stack Guide

## Purpose
Single reference for all technology decisions in AutoStack AI. Used by `planner` during system design and by `builder` + `ml` during code generation. Replaces `system_design.md`, `tech_stack_selector.md`, and `model_selector.md`.

---

## 1. Architecture Pattern

AutoStack AI always generates **Feature-First Clean Architecture**:

```
Presentation  →  Logic (Riverpod)  →  Data (Repository)  →  Firebase / ML
```

### Layer Rules
- **Presentation**: Widgets only. No Firebase calls, no business logic, no direct provider reads except `ref.watch`
- **Logic**: Providers + Notifiers. Orchestrates repository calls, manages state
- **Data**: Repositories + Models. All Firebase/API calls live here
- **No shortcuts**: Never call Firebase directly from a widget — this is the #1 maintainability error

---

## 2. Flutter Tech Stack

### Core Dependencies (always include)
```yaml
dependencies:
  flutter:
    sdk: flutter
  # Firebase
  firebase_core: ^3.0.0
  firebase_auth: ^5.0.0
  cloud_firestore: ^5.0.0
  firebase_storage: ^12.0.0
  cloud_functions: ^5.0.0

  # State
  flutter_riverpod: ^2.5.0
  riverpod_annotation: ^2.3.0

  # Navigation
  go_router: ^14.0.0

  # Utilities
  freezed_annotation: ^2.4.0
  json_annotation: ^4.9.0
  equatable: ^2.0.5
  dartz: ^0.10.1           # functional error handling (Either type)

dev_dependencies:
  build_runner: ^2.4.0
  freezed: ^2.5.0
  json_serializable: ^6.8.0
  riverpod_generator: ^2.4.0
  flutter_test:
    sdk: flutter
  mockito: ^5.4.0
```

### State Management: Riverpod
Always use **code generation** style (`@riverpod` annotation), not manual providers.

```dart
// ✅ Correct
@riverpod
class AuthNotifier extends _$AuthNotifier {
  @override
  AuthState build() => const AuthState.initial();
}

// ❌ Avoid
final authProvider = StateNotifierProvider<AuthNotifier, AuthState>(...);
```

### Navigation: GoRouter
```dart
// Always use named routes — prevents string typo bugs
GoRoute(
  name: AppRoutes.home,
  path: '/home',
  builder: (context, state) => const HomeScreen(),
)
```

### Error Handling: Either Type (dartz)
```dart
// All repository methods return Either<AppFailure, T>
// Never throw raw exceptions out of the data layer
Future<Either<AppFailure, User>> signIn(String email, String password) async {
  try {
    final cred = await _auth.signInWithEmailAndPassword(...);
    return Right(User.fromFirebase(cred.user!));
  } on FirebaseAuthException catch (e) {
    return Left(AppFailure.fromFirebaseAuth(e));
  }
}
```

---

## 3. Firebase Service Selection Guide

| Need | Service | Notes |
|---|---|---|
| User login/signup | Firebase Auth | Always use — never roll your own auth |
| Structured data, real-time | Firestore | Default choice for app data |
| File uploads (images, docs) | Firebase Storage | Pair with Firestore doc for metadata |
| Server-side logic | Cloud Functions | Only if truly needed client-side is insufficient |
| Web hosting | Firebase Hosting | Fast CDN, free tier generous |
| Analytics | Firebase Analytics | Add for any production app |
| Crash reporting | Firebase Crashlytics | Always add to production |

### Auth Providers by Feature
| Feature | Provider |
|---|---|
| Email/password login | `EmailAuthProvider` |
| Social login | `GoogleAuthProvider` / `AppleAuthProvider` |
| Phone OTP | `PhoneAuthProvider` |
| Anonymous | `AnonymousAuthProvider` — for guest mode |

### Firestore Data Modeling Rules
- Collections for **lists of things**: `users`, `posts`, `orders`
- Subcollections for **owned lists**: `users/{uid}/bookmarks`
- Avoid arrays of maps > 10 items — use subcollections instead
- Always add `createdAt` and `updatedAt` to every document
- Index queries are required for `where + orderBy` on different fields — generate `firestore.indexes.json`

---

## 4. ML Model Selection Guide

### Tier 0: Classical ML (use this first — most apps need this, not deep learning)

Classical ML works on **tabular/structured data** (user history, preferences, numbers, categories). It's faster to train, cheaper to serve, and easier to explain. Always prefer classical over deep learning unless the input is images, audio, or video.

| Task | Algorithm | When to use | Serve via |
|---|---|---|---|
| `recommendation` | Collaborative Filter | User–item history available | Cloud Functions (Python) |
| `recommendation` | Content-Based Filter | Item attributes, cold start | Cloud Functions (Python) |
| `recommendation` | Matrix Factorization | Large sparse user-item matrix | Cloud Functions (Python) |
| `classification` | Logistic Regression | Binary outcome, fast, low data | Cloud or on-device (TFLite export) |
| `classification` | Random Forest | Tabular data, interpretability | Cloud Functions (Python) |
| `classification` | Gradient Boosting (XGBoost) | High accuracy on structured data | Cloud Functions (Python) |
| `regression` | Linear Regression | Continuous output, simple patterns | Cloud or on-device |
| `regression` | Gradient Boosting (XGBoost) | Complex patterns, high accuracy | Cloud Functions (Python) |
| `prediction` | Decision Tree | Rule-based, explainability needed | Cloud or on-device |
| `suggestion` | K-Nearest Neighbors | Similarity-based, small datasets | On-device (TFLite export) |
| `clustering` | K-Means | User segmentation, no labels | Cloud (offline batch job) |

**Serving classical models:**
- **Cloud Functions (Python):** Train with scikit-learn/XGBoost, serve via Firebase Functions. Best for models that need user data from Firestore.
- **On-device (TFLite):** Export trained sklearn/XGBoost model to TFLite format using `tf.lite.TFLiteConverter`. Best for offline / low-latency needs.

---

### Tier 1: On-Device Deep Learning (camera, real-time, offline)

Use only when input is images, video, audio, or real-time sensor data.

#### Vision
| Task | Model | Flutter Package | Size |
|---|---|---|---|
| Image classification | MobileNetV2 | tflite_flutter | 14MB |
| Object detection (real-time) | SSD MobileNet | tflite_flutter | 20MB |
| Object detection (accurate) | EfficientDet | tflite_flutter | 45MB |
| Face detection | ML Kit | google_mlkit_face_detection | Built-in |
| Text in images / OCR | ML Kit | google_mlkit_text_recognition | Built-in |
| Barcode / QR | ML Kit | google_mlkit_barcode_scanning | Built-in |
| Image segmentation | DeepLab v3 | tflite_flutter | 30MB |

#### Language / Text
| Task | Model | Package | Notes |
|---|---|---|---|
| Text classification (large) | MobileBERT | tflite_flutter | 25MB |
| Language detection | ML Kit | google_mlkit_language_id | Built-in |
| Translation | ML Kit | google_mlkit_translation | Downloads language pack |
| Smart reply | ML Kit | google_mlkit_smart_reply | Built-in |

#### Body / Pose
| Task | Model | Package | Notes |
|---|---|---|---|
| Pose estimation | MoveNet Lightning | tflite_flutter | 9MB, 50fps |
| Pose estimation (accurate) | MoveNet Thunder | tflite_flutter | 29MB |
| Hand tracking | MediaPipe | mediapipe_flutter | Built-in |

---

### Tier 2: Cloud Deep Learning (large models, LLMs, batch)

| Task | Approach | Notes |
|---|---|---|
| Custom trained model | TFLite export + tflite_flutter | User must provide `.tflite` |
| Large model (>50MB) | Cloud Functions + TF.js-node | Never bundle large models in app |
| LLM / GPT features | Cloud Functions + Gemini API | Never expose API keys in Flutter |

---

### Model Decision Tree

```
Is input images / video / audio / camera?
  Yes → Tier 1 (TFLite or ML Kit)

Is input tabular / structured (numbers, categories, history)?
  Yes → Tier 0 (Classical ML)
        → Needs user history from server? → Cloud Functions (Python)
        → Needs offline / fast response? → On-device TFLite export

Is model >50MB or needs LLM?
  Yes → Tier 2 (Cloud Functions)

Does ML Kit support the task?
  Yes → Use ML Kit (built-in, no model management, Google-maintained)
  No  → Use TFLite

Needs offline ML?
  Yes → On-device only (Tier 0 TFLite export or Tier 1)
  No  → Cloud preferred for classical ML
```

---

## 5. Complexity & Timeline Estimates

| Complexity | Features | Firebase | ML | Estimated Build Time |
|---|---|---|---|---|
| Low | <5 screens, basic CRUD | Auth + Firestore | None | 1 agent pass |
| Medium | 5–15 screens, real-time data | Auth + Firestore + Storage | On-device | 2 agent passes |
| High | 15+ screens, custom logic | Full stack + Functions | Cloud/hybrid | 3+ agent passes |

---

## 6. Regional Configuration (India / South Asia)

Since this project targets India-based deployments:
- Firebase region: `asia-south1` (Mumbai) for Cloud Functions
- Firestore region: `asia-south1`
- Use `asia` bucket region for Storage
- Use Google Sign-In as primary social provider (high penetration)
- Support both English and Hindi if app is consumer-facing
