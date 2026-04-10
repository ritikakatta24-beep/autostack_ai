# AutoStack AI

Type one sentence. Get a complete full-stack app.

Built for the BITS Pilani Claude Hackathon.

## How to Run

1. Install Claude Code: npm install -g @anthropic-ai/claude-code
2. Open this folder in Claude Code
3. Type: Build a [your app idea]
4. Watch it build automatically

## Example Prompts

- Build a food delivery app
- Build a hospital appointment booking app
- Build a carbon footprint tracker
- Build a stock portfolio tracker
- Build a fitness coaching app

## What Gets Generated

- Flutter mobile app — all screens, navigation, state management
- Firebase backend — Firestore, Auth, Storage, Security Rules
- ML inference layer — on-device TFLite or cloud Gemini or custom FastAPI
- Docker container — ML deployment
- CI/CD pipelines — GitHub Actions
- Makefile — developer shortcuts

## Stack

- Frontend:  Flutter 3.22+ / Dart 3.4+
- Backend:   Firebase Firestore, Auth, Storage, Functions
- State:     Riverpod 2.x
- Navigation: go_router 13.x
- ML:        TFLite / Gemini / Scikit-learn + FastAPI
- DevOps:    Docker + GitHub Actions

## Agent Pipeline

Architect → Firebase → Flutter → ML → DevOps

## Folder Structure

.claude/
  agents/        — 5 specialist agents
  workflows/     — 6 phase orchestration files
  skills/        — 14 reusable skill files
  mcp.json       — MCP server config
claude.md        — entry point
README.md        — this file