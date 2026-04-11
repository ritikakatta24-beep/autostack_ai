 Skill: Error Handler

## Purpose
Shared error resolution layer for all AutoStack AI agents. When any agent hits an error it cannot self-resolve, it invokes this skill with a structured error payload.

## Invocation Format

```yaml
error_code: string          # from the agent's own error table
affected_file: string       # e.g. lib/features/auth/data/auth_repo.dart
agent: planner | builder_ui | builder_firebase | builder_devops | ml
context:
  - key: value              # any relevant state from system_plan.json
```

---

## Error Categories & Resolutions

### Category 1: Configuration Errors

| Code | Description | Auto-fix | Escalate |
|---|---|---|---|
| `CFG_001` | `system_plan.json` not found or invalid JSON | Halt, tell planner to re-run | Yes |
| `CFG_002` | Firebase project not initialized | Run `firebase init` instructions | No |
| `CFG_003` | Missing asset declared in `pubspec.yaml` | Add asset entry automatically | No |
| `CFG_004` | Flutter SDK version mismatch | Output correct SDK constraint | No |
| `CFG_005` | `google-services.json` or `GoogleService-Info.plist` missing | Provide download instructions | Yes |

### Category 2: Firebase Errors

| Code | Description | Auto-fix | Escalate |
|---|---|---|---|
| `FB_001` | `permission-denied` on Firestore | Regenerate rules from `system_plan.json`.backend.features | No |
| `FB_002` | `FirebaseApp not initialized` | Confirm `FirebaseService.initialize()` called in `main.dart` | No |
| `FB_003` | Auth provider not enabled in Firebase Console | Output Firebase Console steps | Yes |
| `FB_004` | Functions cold start timeout | Add model/data caching outside handler | No |
| `FB_005` | Storage CORS error | Provide `gsutil cors set` command | No |
| `FB_006` | Firestore index missing | Generate composite index entry in `firestore.indexes.json` | No |

### Category 3: Flutter / Dart Errors

| Code | Description | Auto-fix | Escalate |
|---|---|---|---|
| `FL_001` | `MissingPluginException` | Add `WidgetsFlutterBinding.ensureInitialized()` to `main.dart` | No |
| `FL_002` | Null check on null value | Add null guards, fix model with nullable types | No |
| `FL_003` | `StateError: Stream closed` | Add `ref.onDispose` cleanup to provider | No |
| `FL_004` | Build runner conflict | Run `dart run build_runner build --delete-conflicting-outputs` | No |
| `FL_005` | Riverpod provider not found | Confirm `ProviderScope` wraps `App` in `main.dart` | No |

### Category 4: ML Errors

| Code | Description | Auto-fix | Escalate |
|---|---|---|---|
| `ML_001` | `TfLiteException: model load failed` | Verify asset path matches `pubspec.yaml` declaration | No |
| `ML_002` | OOM / app crash during inference | Move inference to `compute()` isolate | No |
| `ML_003` | Shape mismatch on input tensor | Fix `ml_preprocessor.dart` to match model input spec | No |
| `ML_004` | NaN output from model | Check normalization range (should be [0,1] or [-1,1]) | No |
| `ML_005` | `HttpsError: deadline-exceeded` on cloud function | Set `timeoutSeconds: 60`, cache model outside handler | No |
| `ML_006` | Unknown ML task in `system_plan.json`.ml.task_type | Escalate — request user to clarify | Yes |
| `ML_007` | `invalid_features` from Phase 0 function | Feature count/order mismatch — check `feature_schema.json` | No |
| `ML_008` | Cold start returns empty results | Implement popularity-based fallback in cold start protocol | No |

### Category 5: CI/CD Errors

| Code | Description | Auto-fix | Escalate |
|---|---|---|---|
| `CI_001` | `FIREBASE_SERVICE_ACCOUNT` secret missing | Output GitHub secrets setup steps | Yes |
| `CI_002` | Flutter tests fail in CI | Show failing test — never skip tests | Yes |
| `CI_003` | `flutter build web` fails in CI | Run `make build-web` locally first to diagnose | No |
| `CI_004` | iOS build missing certs or provisioning profile | Output Apple cert setup steps | Yes |

---

## Escalation Protocol

When `Escalate: Yes`:

1. **Stop** the current agent pipeline immediately
2. **Output** this message to the user:

```
⚠️  AutoStack needs your input

Error   : [error_code] — [description]
File    : [affected_file]
Agent   : [agent]

What's needed:
[specific action the user must take]

Once resolved, re-run: [which step/agent to restart from]
```

3. **Never** continue silently past an escalation error

---

## Auto-fix Protocol

When `Escalate: No`:

1. Apply the documented fix
2. Log it:

```
🔧 Auto-fixed [error_code] in [affected_file]
   Applied: [what was changed]
   Continuing pipeline...
```

3. Return control to the calling agent
4. If the same error recurs after auto-fix → escalate anyway

---

## build_log.md Format

All errors (auto-fixed or escalated) must be appended to `build_log.md` in project root:

```markdown
## [timestamp] [error_code] — [agent]
- File: [affected_file]
- Description: [what happened]
- Resolution: auto-fixed | escalated
- Fix applied: [description, or N/A if escalated]
```

This file is created fresh at the start of each build run and never deleted during a session.
