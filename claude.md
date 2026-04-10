# AutoStack AI — Claude Instructions

## What This Project Is
AutoStack AI is a multi-agent system that converts a natural language app description into a complete, production-ready Flutter + Firebase application with optional ML integration.

## How to Start
When a user provides an app description, always run the **main workflow**:
```
.claude/workflows/main.md
```
Never skip the planner. Never write code before `system_plan.json` exists and is valid JSON.

---

## Agent Map

| Agent | File | Reads from system_plan.json | Purpose |
|---|---|---|---|
| Planner | `.claude/agents/planner.md` | — (writes the plan) | Converts prompt → `system_plan.json` |
| Builder | `.claude/agents/builder.md` | entire file | Coordinates the three builder sub-agents |
| Builder UI | `.claude/agents/builder/builder_ui.md` | `frontend.pages`, `backend.features`, `state_management` | Flutter scaffold + screens + state |
| Builder Firebase | `.claude/agents/builder/builder_firebase.md` | `backend.firebase_services`, `backend.features` | Auth + Firestore + Storage + Functions + rules |
| Builder DevOps | `.claude/agents/builder/builder_devops.md` | `deployment_target`, `backend.firebase_services` | CI/CD + Makefile + firebase.json |
| ML | `.claude/agents/ml.md` | `ml.*`, `backend.features` | ML pipeline (classical / on-device / cloud / hybrid) |

## Skill Map

| Skill | File | Used by |
|---|---|---|
| Error Handler | `.claude/skills/error_handler.md` | All agents |
| Stack Guide | `.claude/skills/stack_guide.md` | Planner, Builder, ML |

---

## system_plan.json Schema (single source of truth)

```json
{
  "app_name": "string",
  "description": "string",
  "deployment_target": "android | ios | web | all",
  "state_management": "riverpod | bloc",
  "estimated_complexity": "low | medium | high",
  "backend": {
    "firebase_services": ["auth", "firestore", "storage", "functions", "hosting"],
    "features": [
      {
        "name": "string",
        "type": "auth | crud | realtime | storage | logic",
        "priority": "high | medium | low",
        "collection": "string | null"
      }
    ]
  },
  "frontend": {
    "pages": [
      {
        "name": "string",
        "route": "/string",
        "requires_auth": true,
        "connected_feature": "string"
      }
    ]
  },
  "ml": {
    "enabled": true,
    "mode": "none | classical | on-device | cloud | hybrid",
    "task_type": "recommendation | classification | regression | prediction | suggestion | none",
    "algorithm": "string",
    "inputs": ["string"],
    "output": "string",
    "realtime": false,
    "rationale": "string"
  }
}
```

---

## ML Mode Reference

| Mode | What runs | When planner picks it |
|---|---|---|
| `none` | ML agent skipped entirely | No personalization needed |
| `classical` | Phase 0 — Python scikit-learn/XGBoost via Cloud Functions | Tabular input (history, preferences, numbers) |
| `on-device` | Phase A — TFLite / ML Kit in Flutter | Camera, real-time, offline required |
| `cloud` | Phase B — TF.js-node or Gemini API via Cloud Functions | Model >50MB or LLM features |
| `hybrid` | Phase A + Phase B | Mix of real-time and server-side |

---

## Golden Rules

1. **Always read `system_plan.json` before generating any code** — it is the single source of truth
2. **Never expose API keys in Flutter code** — all secrets go through Cloud Functions or environment variables
3. **Never put business logic in widgets** — feature-first clean architecture, always
4. **Always wrap Firebase calls in try/catch** — map to typed `AppException`, never rethrow raw exceptions
5. **Always run on-device ML on an Isolate** — `compute()` for TFLite, callable Functions for cloud/classical
6. **Never train ML models inside a Cloud Function** — train offline, deploy the serialized artifact
7. **Always implement cold start fallback** for recommendation/suggestion tasks before the model is deployed
8. **Never skip tests in CI** — GitHub Actions must run `flutter test` before any build step
9. **Feature order in `feature_schema.json` is law** — encoding at training must match encoding at inference exactly
10. **When in doubt, halt and ask** — a wrong plan wastes more time than a clarifying question

---

## Output Files Generated During a Build

| File | Written by | Purpose |
|---|---|---|
| `system_plan.json` | planner | Single source of truth for all agents |
| `build_log.md` | error_handler (via agents) | Error and auto-fix audit trail |
| `lib/` | builder_ui | Flutter app code |
| `firestore.rules` | builder_firebase | Firestore security rules |
| `storage.rules` | builder_firebase | Storage security rules |
| `firestore.indexes.json` | builder_firebase | Composite query indexes |
| `firebase.json` | builder_devops | Firebase deploy config |
| `.github/workflows/deploy.yml` | builder_devops | CI/CD pipeline |
| `Makefile` | builder_devops | Local dev shortcuts |
| `functions/ml/` | ml (Phase 0) | Python Cloud Function + model artifact |
| `lib/features/ml/` | ml (Phase A) | TFLite Flutter integration |
| `functions/src/ml/` | ml (Phase B) | TF.js Cloud Function |
| `functions/ml/feature_schema.json` | ml | Feature order contract for classical ML |

---

## Asking for Clarification

Ask before running the planner if any of these are unknown:
- Target platform (Android / iOS / Web / All)?
- Login required? If yes, which providers (email, Google, phone)?
- Any existing Firebase project, or create new?
- Should the app personalize results per user, or same output for everyone?
- Is there a specific ML task in mind, or should AutoStack decide?
