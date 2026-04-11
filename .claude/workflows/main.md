# Workflow: Main

## Purpose
Entry point for every AutoStack AI build. Runs agents in the correct order and handles inter-agent coordination.

## Trigger
Run this workflow whenever a user provides a new app idea or feature request.

---

## Execution Order

User Prompt
    │
    ▼
[1] planner          → writes system_plan.json
    │
    ├─ ERROR? ──────→ error_handler → HALT if escalation needed
    │
    ▼
[2] builder          → reads system_plan.json, generates full codebase
    │                  internally runs: builder_ui → builder_firebase → builder_devops
    ├─ ERROR? ──────→ error_handler → auto-fix or HALT
    │
    ▼
[3] ml               → runs ONLY if ml.enabled == true in system_plan.json
    │
    ├─ ERROR? ──────→ error_handler → auto-fix or HALT
    │
    ▼
[4] Done             → show completion summary to user
```

---

## Step 1: Planner

```
Agent:          planner
Input:          user prompt (natural language)
Output:         system_plan.json (project root)
Halt condition: planner cannot resolve ambiguity → ask user before continuing
```

Do not proceed to Step 2 until `system_plan.json` exists and is valid JSON.

---

## Step 2: Builder

```
Agent:          builder (coordinator)
Input:          system_plan.json
Output:         full Flutter project codebase
Halt condition: CFG_001 (system_plan.json missing) or unresolvable Firebase error
```

Builder runs three sub-agents internally in this order:
1. `builder/builder_ui.md` → reads `system_plan.json`.frontend + state_management
2. `builder/builder_firebase.md` → reads `system_plan.json`.backend.firebase_services
3. `builder/builder_devops.md` → reads `system_plan.json`.deployment_target

Do not proceed to Step 3 until builder completes without escalation errors.

---

## Step 3: ML (conditional)

```
Condition:      ml.enabled == true in system_plan.json
Agent:          ml
Input:          system_plan.json (reads ml.mode, ml.task_type, ml.algorithm, ml.inputs, ml.output)
Output:         lib/features/ml/ and/or functions/ml/ depending on ml.mode
Halt condition: ML_006 (unknown task) → escalate to user
```

If `ml.enabled == false`, skip this step entirely.

ML mode routing inside the agent:
- `classical`  → Cloud Functions Python (scikit-learn / XGBoost)
- `on-device`  → TFLite / ML Kit in Flutter
- `cloud`      → TF.js-node or Gemini API via Cloud Functions
- `hybrid`     → on-device + cloud both

---

## Step 4: Completion Summary

After all agents complete without escalation errors, output to user:

 AutoStack Build Complete

App: [app_name]
─────────────────────────────
 Structure:    Feature-first clean architecture
 Pages:        [count] pages generated
 Firebase:     [firebase_services list]
 ML:           [ml.task_type] via [ml.algorithm] ([ml.mode])
 CI/CD:        GitHub Actions workflow ready
─────────────────────────────
Next steps:
1. Add google-services.json → android/app/
2. Add GoogleService-Info.plist → ios/Runner/
3. Run: make setup
4. Run: make run
5. Set FIREBASE_SERVICE_ACCOUNT in GitHub repo secrets for auto-deploy
```

---

## Re-run Rules

| Scenario | Re-run |
|---|---|
| User adds a new feature | Planner (update plan) → `builder_ui.md` for new feature only |
| Firebase rules need update | `builder_firebase.md` only |
| Deployment target changed | `builder_devops.md` only |
| ML mode or algorithm changed | `ml.md` only |
| Full rebuild | Entire workflow from Step 1 |

---

## build_log.md

Created at the start of every run in the project root. Every agent appends errors, auto-fixes, and escalations to it. Never delete during a build session — it is the audit trail for debugging.
