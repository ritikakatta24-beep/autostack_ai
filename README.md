# AutoStack AI

A multi-agent Claude system that converts a natural language app description into a complete Flutter + Firebase + ML codebase.

## How It Works

```
You describe your app → Planner designs it → Builder codes it → ML agent handles AI features
```

## Quickstart

1. Open this project in Claude (with MCP enabled)
2. Describe your app — e.g.:
   > "Build a grocery delivery app with user login, product listings, cart, and order tracking"
3. AutoStack will ask any clarifying questions, then generate the full codebase

## What Gets Generated

- Complete Flutter project (feature-first clean architecture)
- Firebase Auth, Firestore, Storage, Functions setup
- Firestore and Storage security rules
- Riverpod state management
- GoRouter navigation
- GitHub Actions CI/CD
- Makefile for local dev
- ML integration (on-device TFLite or Cloud Functions) if needed

## Project Structure

```
AUTOSTACK_AI/
  .claude/
    agents/
      planner.md       ← orchestrator: reads prompt, writes system_plan.json
      builder.md       ← Flutter + Firebase + DevOps
      ml.md            ← ML pipeline (TFLite / Cloud / hybrid)
    skills/
      error_handler.md ← shared error resolution
      stack_guide.md   ← tech stack reference (arch + Firebase + models)
    workflows/
      main.md          ← entry point: planner → builder → ml
  mcp.json             ← MCP server + agent config
  claude.md            ← Claude's top-level instructions
  README.md
```

## Tech Stack

| Layer | Technology |
|---|---|
| UI | Flutter 3.x |
| State | Riverpod 2.x (code gen) |
| Navigation | GoRouter |
| Backend | Firebase (Auth, Firestore, Storage, Functions) |
| ML (on-device) | TFLite Flutter / ML Kit |
| ML (cloud) | Cloud Functions + TF.js-node |
| CI/CD | GitHub Actions + Firebase Hosting |

## Requirements

- Claude with MCP filesystem access enabled
- Flutter SDK 3.x
- Firebase CLI (`npm install -g firebase-tools`)
- Node.js 18+ (for Cloud Functions)
- A Firebase project (or create one at console.firebase.google.com)

## After Generation

```bash
# 1. Add your Firebase config files
#    android/app/google-services.json
#    ios/Runner/GoogleService-Info.plist

# 2. Install dependencies
make setup

# 3. Run locally
make run

# 4. Deploy
make deploy
```
