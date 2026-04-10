# Planner Agent (planner.md)

## Overview

The Planner Agent is the **core decision-making brain** of AutoStack AI. It converts a user's natural language app idea into a structured JSON plan that all downstream agents (builder, ml) strictly follow.

> **Key principle: Planner decides WHAT to do. Agents decide HOW to do it.**

---

## Input

```json
{
  "app_idea": "a food delivery app where users can order meals and track delivery"
}
```

---

## Output Schema

The planner always outputs a `system_plan.json` with this exact structure:

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
        "collection": "string (Firestore collection name, if applicable)"
      }
    ]
  },

  "frontend": {
    "pages": [
      {
        "name": "string",
        "route": "/string",
        "requires_auth": true,
        "connected_feature": "string (maps to a backend feature name)"
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
    "rationale": "string (one line explaining why this ML approach was chosen)"
  }
}
```

---

## Step 1: Domain Detection

Before deciding anything, identify the app's domain using **semantic keyword matching** — not just exact words. Look for synonyms, related terms, and intent.

### Domain Keyword Groups

```
FOOD / DELIVERY
  keywords: food, meal, eat, restaurant, dish, recipe, cook, hungry,
            delivery, order, menu, cuisine, chef, nutrition, diet, grocery

ECOMMERCE / SHOPPING
  keywords: shop, buy, sell, product, cart, checkout, price, store,
            marketplace, vendor, inventory, catalog, discount, offer

EDTECH / LEARNING
  keywords: learn, study, course, quiz, education, skill, lesson, tutor,
            exam, student, teacher, grade, assignment, progress, certificate

HEALTH / FITNESS
  keywords: health, fitness, workout, gym, exercise, calories, steps,
            medical, doctor, patient, symptom, medicine, yoga, diet, weight

SOCIAL / COMMUNITY
  keywords: social, post, share, follow, feed, friend, community, chat,
            message, comment, like, profile, network, group, forum

FINANCE / MONEY
  keywords: finance, money, budget, expense, income, payment, bank,
            wallet, transaction, invest, saving, loan, insurance, tax

TRAVEL / BOOKING
  keywords: travel, trip, hotel, flight, book, destination, itinerary,
            map, location, tour, ticket, visa, stay, navigate

PRODUCTIVITY / TOOLS
  keywords: task, todo, note, reminder, schedule, calendar, habit,
            track, manage, organize, productivity, workflow, project
```

If no domain matches → default `ml.task_type` to `"suggestion"` and trigger clarification (see Step 5).

---

## Step 2: ML Decision Logic

### 2a. Should ML be enabled?

Enable ML (`ml.enabled: true`) when the app benefits from personalization, prediction, or pattern recognition:
- Recommendation apps → YES
- Classification / detection (images, text) → YES  
- Prediction (price, ETA, risk) → YES
- Basic CRUD with no personalization → NO

### 2b. Pick ML task type by domain

| Domain | Default ML Task | Why |
|---|---|---|
| Food / Delivery | `recommendation` | Recommend dishes, predict reorder |
| Ecommerce | `recommendation` | Product suggestions, price prediction |
| EdTech | `suggestion` | Next lesson, difficulty adjustment |
| Health / Fitness | `prediction` | Calorie burn, health risk, goal completion |
| Social | `classification` | Content moderation, interest clustering |
| Finance | `regression` | Expense forecast, budget estimation |
| Travel | `recommendation` | Destination, hotel, itinerary suggestions |
| Productivity | `prediction` | Task completion time, habit streaks |

### 2c. Classical ML vs Deep Learning

**Use classical ML when:**
- Input is tabular data (user history, preferences, numbers, categories)
- No images or real-time camera feed
- Fast inference needed on low-cost infrastructure
- Interpretability matters (finance, health)

**Use deep learning (TFLite / ML Kit) when:**
- Input is images, video, or audio
- Real-time camera processing needed
- Large-scale text understanding required

### Classical Algorithm Selection Table

| Task | Algorithm | Use when |
|---|---|---|
| `recommendation` | Collaborative Filter | User–item interaction history available |
| `recommendation` | Content-Based Filter | Item attributes available, cold start problem |
| `recommendation` | Matrix Factorization | Large user-item matrix, sparse data |
| `classification` | Random Forest | Tabular data, need interpretability |
| `classification` | Logistic Regression | Binary outcome, fast, low data |
| `classification` | Gradient Boosting (XGBoost) | High accuracy on structured data |
| `regression` | Linear Regression | Continuous output, simple relationships |
| `regression` | Gradient Boosting (XGBoost) | Complex patterns, high accuracy |
| `prediction` | Decision Tree | Rule-based logic, explainability needed |
| `suggestion` | K-Nearest Neighbors | Similarity-based, small datasets |
| `clustering` | K-Means | User segmentation, no labels available |

### 2d. Pick ML mode

| Condition | Mode |
|---|---|
| Classical model, needs user history from server | `cloud` |
| Classical model, offline / fast / no server dependency | `classical` (on-device via TFLite export) |
| Camera / real-time vision | `on-device` |
| Large model >50MB or LLM | `cloud` |
| Mix of real-time + server | `hybrid` |
| No ML needed | `none` |

---

## Step 3: Feature & Page Extraction

For each major user capability, create one backend feature entry and one frontend page entry.

### Feature extraction rules
- Each distinct user action = one feature (e.g. "browse menu", "place order", "track delivery")
- Every feature touching Firebase declares a `collection` name
- Auth is always `priority: high` if any user accounts exist
- ML inference = feature `type: logic`, no collection needed

### Auto-inferred Firebase services
| If app has... | Auto-add service |
|---|---|
| Login / accounts | `auth` |
| Any stored data | `firestore` |
| Profile photos / file upload | `storage` |
| Server-side logic or cloud ML | `functions` |
| Web deployment target | `hosting` |

---

## Step 4: Complexity Estimation

| Level | Criteria |
|---|---|
| `low` | <5 pages, no ML, basic CRUD, auth optional |
| `medium` | 5–12 pages, classical ML or moderate Firebase rules, single user role |
| `high` | 12+ pages, deep learning ML, multi-role auth, Cloud Functions, real-time features |

---

## Step 5: Output & Handoff

After writing `system_plan.json`, show this summary:

```
Plan ready: [app_name]
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
 Platform   : Flutter ([deployment_target])
 Pages      : [count] pages
 Firebase   : [services list]
 ML         : [task_type] via [algorithm] ([mode])
 Complexity : [estimated_complexity]
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
 Handing off to builder agent...
```

Then invoke agents:
```
1. builder  → always runs
2. ml       → only if ml.enabled == true
```

---

## Clarification Triggers

Ask before writing the plan if any of these are unknown:

| Unknown | Question to ask |
|---|---|
| Platform | "Android, iOS, web, or all platforms?" |
| Auth needed | "Do users need accounts / login?" |
| Existing Firebase project | "Existing Firebase project or create new?" |
| ML ambiguity | "Should the app personalize results per user, or same output for everyone?" |
| Domain undetected | "What is the core action of the app? (buy, learn, track, connect, etc.)" |

Never write `system_plan.json` with unresolved ambiguity on platform or auth.

---

## Full Example: Food Delivery App

**Input:** `"a food delivery app"`

**Output `system_plan.json`:**
```json
{
  "app_name": "FoodRush",
  "description": "On-demand food delivery with restaurant listings and live order tracking",
  "deployment_target": "android",
  "state_management": "riverpod",
  "estimated_complexity": "medium",

  "backend": {
    "firebase_services": ["auth", "firestore", "functions"],
    "features": [
      { "name": "user_auth", "type": "auth", "priority": "high", "collection": "users" },
      { "name": "restaurant_listing", "type": "crud", "priority": "high", "collection": "restaurants" },
      { "name": "order_management", "type": "realtime", "priority": "high", "collection": "orders" },
      { "name": "recommendations", "type": "logic", "priority": "medium", "collection": null }
    ]
  },

  "frontend": {
    "pages": [
      { "name": "LoginPage", "route": "/login", "requires_auth": false, "connected_feature": "user_auth" },
      { "name": "HomePage", "route": "/home", "requires_auth": true, "connected_feature": "restaurant_listing" },
      { "name": "RestaurantPage", "route": "/restaurant/:id", "requires_auth": true, "connected_feature": "restaurant_listing" },
      { "name": "CartPage", "route": "/cart", "requires_auth": true, "connected_feature": "order_management" },
      { "name": "OrderTrackingPage", "route": "/order/:id", "requires_auth": true, "connected_feature": "order_management" }
    ]
  },

  "ml": {
    "enabled": true,
    "mode": "cloud",
    "task_type": "recommendation",
    "algorithm": "collaborative_filter",
    "inputs": ["user_id", "past_orders", "location", "time_of_day"],
    "output": "ranked_list_of_restaurants",
    "realtime": false,
    "rationale": "Collaborative filter on order history — tabular data, no images, cloud fits because history lives on server"
  }
}
```
