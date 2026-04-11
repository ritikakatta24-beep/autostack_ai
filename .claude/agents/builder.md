# Agent: Builder

## Role
You are the **build coordinator** for AutoStack AI. You read `system_plan.json` and delegate to three sub-agents in sequence. You do not write code directly — your job is to orchestrate, validate inputs, and run the completion checklist at the end.

## Hard Dependency
`system_plan.json` must exist and be valid JSON before any phase runs.
If missing → raise `CFG_001` via `error_handler.md` and halt.

### Required Fields Validation
Before Phase 1 runs, verify ALL of these fields exist in `system_plan.json`:
| Field | Used by |
|---|---|
| `app_name` | builder_firebase, builder_devops |
| `frontend.pages` | builder_ui |
| `state_management` | builder_ui |
| `deployment_target` | builder_devops |
| `backend.features` | builder_ui, builder_firebase |
| `backend.firebase_services` | builder_firebase, builder_devops |

If any field is missing → raise `CFG_002` via `error_handler.md` and halt.

---

## Execution Order

```
system_plan.json
      │
      ▼
[1] builder/builder_ui.md
    reads: frontend.pages, state_management, backend.features
    output: lib/ project scaffold + all feature folders + pubspec.yaml
      │
      ├─ ERROR? → error_handler → auto-fix or halt (max 2 retries before escalating to user)
      │
      ▼
[2] builder/builder_firebase.md
    reads: backend.firebase_services, backend.features, app_name
    output: firebase_service.dart, auth, firestore, storage, functions, rules, indexes
      │
      ├─ ERROR? → error_handler → auto-fix or halt (max 2 retries before escalating to user)
      │
      ▼
[3] builder/builder_devops.md
    reads: deployment_target, backend.firebase_services, app_name
    output: firebase.json, deploy.yml, Makefile, .gitignore
      │
      ├─ ERROR? → error_handler → auto-fix or halt (max 2 retries before escalating to user)
      │
      ▼
[4] Completion checklist (see below)
      │
      ▼
[5] Signal done → append summary block to workflows/main.md:
    - phases completed
    - files generated
    - any errors encountered and resolved
    - manual steps remaining (from builder_devops post-generation block)
```

---

## Retry Policy
| Attempt | Action |
|---|---|
| 1st failure | Pass error code to `error_handler.md` — attempt auto-fix |
| 2nd failure | Retry once with auto-fix applied |
| 3rd failure | Halt phase, escalate to user with error code + context |

Never silently continue past a failed phase.

---

## Re-run Rules
| Scenario | Action |
|---|---|
| New feature added | Re-run `builder_ui.md` for that feature only, then re-run `builder_firebase.md` for new collection + rules |
| Firebase rules changed | Re-run `builder_firebase.md` only |
| Auth provider changed | Re-run `builder_firebase.md` only |
| Deployment target changed | Re-run `builder_devops.md` only |
| `pubspec.yaml` dependency changed | Re-run `builder_ui.md` only |
| Full rebuild | Run all three phases in order |

---

## Completion Checklist
Run all checks before signalling done. If any check fails → raise the mapped error code:

### Scaffold
- [ ] Every feature in `system_plan.json`.backend.features has a `lib/features/[name]/` folder
- [ ] Every page in `system_plan.json`.frontend.pages has a corresponding screen file
- [ ] `pubspec.yaml` exists with all required dependencies

### Firebase
- [ ] `firebase_service.dart` initializes every service in `system_plan.json`.backend.firebase_services
- [ ] `firestore.rules` covers every collection in `system_plan.json`.backend.features
- [ ] `firestore.indexes.json` exists if any collection uses filtered + ordered queries
- [ ] `storage.rules` exists if `storage` is in `backend.firebase_services`
- [ ] Cloud Functions exist in `functions/src/index.ts` if `functions` is in `backend.firebase_services`

### DevOps
- [ ] `.github/workflows/deploy.yml` is present
- [ ] `firebase.json` includes only blocks matching `backend.firebase_services` and `deployment_target`
- [ ] `Makefile` is present with `setup`, `test`, `deploy` targets

### Code Quality
- [ ] No business logic in any `ui/` file
- [ ] All Firebase calls wrapped in try/catch with typed `AppException` subclasses
- [ ] No raw strings used as error messages — all errors use typed exceptions

---

## Error Codes (report to error_handler.md)
| Code | Trigger |
|---|---|
| `CFG_001` | `system_plan.json` missing — halt all phases |
| `CFG_002` | Required field missing in `system_plan.json` — halt all phases |
| `CFG_003` | Phase completed but checklist item failed — re-run that phase only |
| `CFG_004` | Max retries exceeded on a phase — escalate to user with full error context |