# triad 🔺

**A personal fitness intelligence pipeline.**

Triad unifies sleep, nutrition, and training data from your existing apps into a single structured dataset, then uses statistical feature engineering and LLM reasoning to generate personalized daily recommendations.

> *Built as a BYU-Idaho Data Science Senior Project — Spring 2026.*

![Python](https://img.shields.io/badge/python-3.11%20%7C%203.12-blue)
![Status](https://img.shields.io/badge/status-private%20repo%20%E2%80%94%20in%20development-red)
![Data](https://img.shields.io/badge/data-personal%20health-purple)

---

> **This repo is a public showcase.** The full source lives in a private repository while the project is under active development. This page documents the problem, the architecture, and how it works. See [Status &amp; Access](#status--access) below for how to request a look at the code.

---

## The Problem

When you're lifting weights, three things drive your results:

1. **Sleep** — recovery, hormone regulation, muscle repair
2. **Nutrition** — fuel for training, protein for muscle synthesis
3. **Training** — the stimulus for adaptation

When one is off, the others suffer. Bad sleep tanks your workout. Hard training without enough food crashes your energy. But every app tracks only one of these in isolation. Nothing connects them and adjusts based on how they interact.

**Triad solves this.** It pulls all three sources together and tells you — based on *your actual data* — what your body needs today.

---

## How It Works

```
┌─────────────────────────────────────────────────────────┐
│  DATA SOURCES                                           │
│  Zepp/Amazfit ──→ Apple Health ──┐                      │
│  Cal AI ────────→ Apple Health ──┼──→ triad.ingest()    │
│  Hevy ──────────→ REST API ──────┘         │            │
└────────────────────────────────────────────┼────────────┘
                                             ▼
┌─────────────────────────────────────────────────────────┐
│  DATABASE  (SQLite / triad.db)                          │
│  sleep_records + nutrition_logs + workout_sessions      │
│                    ↓ join by date                       │
│              daily_summary                              │
└────────────────────────────────────────────┬────────────┘
                                             ▼
┌─────────────────────────────────────────────────────────┐
│  FEATURE ENGINEERING  (triad.transform)                 │
│  sleep_debt_7d · recovery_score · training_load_7d      │
│  load_change_pct · caloric_deficit_3d · muscle_fatigue  │
└────────────────────────────────────────────┬────────────┘
                                             ▼
┌─────────────────────────────────────────────────────────┐
│  RECOMMENDATION ENGINE  (triad.recommend)               │
│  User profile + goal text + daily features              │
│                    ↓ Anthropic API                      │
│  Workout type · Calorie target · Recovery advice        │
└─────────────────────────────────────────────────────────┘
```

---

## Data Sources

| Source | Data | Access Method |
|---|---|---|
| Zepp / Amazfit band | Sleep stages (deep, REM, core, awake) | Apple Health sync → XML export |
| Cal AI | Calories, protein, carbs, fat | App PDF export |
| Hevy | Workout sessions, exercise sets, volume | Official REST API (Pro) |

---

## Feature Engineering

The feature layer translates raw daily numbers into signals the recommendation engine can reason about:

| Feature | Description |
|---|---|
| `sleep_debt_7d` | Cumulative sleep deficit over 7 days (hours) |
| `recovery_score` | 0–100 weighted composite of deep/REM/core sleep quality |
| `training_load_7d` | Total volume lifted in the last 7 days (lbs) |
| `load_change_pct` | Week-over-week training load change (%) |
| `caloric_deficit_3d` | Average daily caloric deficit over 3 days |
| `protein_avg_7d` | Average daily protein intake over 7 days (g) |
| `{muscle}_volume_48h` | Per-muscle-group volume in last 48 hours |

---

## Why LLM + Stats, Not Pure Stats

The original plan was a regression or classification model. Two problems got in the way: there isn't enough data yet to train one reliably (a real model needs 40-60+ complete days; early data only had a handful), and the user's goal is free text — "build muscle and lose fat," "train for a 5k," "I'm feeling run down" — which isn't something a numeric model can consume directly.

The hybrid approach solves both. Statistics compute the objective features (sleep debt, recovery score, training load, muscle fatigue per group) grounded in real data. The LLM handles the reasoning — it already understands exercise physiology and nutrition science, and the computed features give it personal context to reason from.

---

## Package Structure

```
triad/
├── __init__.py              Public API: setup, sync, recommend, today
├── db/store.py              All SQLite read/write operations
├── ingest/
│   ├── hevy.py              Hevy REST API puller + CSV fallback
│   ├── apple_health.py      Apple Health XML streaming parser
│   └── calai.py             Cal AI PDF parser
├── transform/
│   ├── features.py          Feature engineering: sleep debt, recovery, load
│   └── summarize.py         daily_summary builder
├── profile/
│   └── user.py              Biometrics, goals, TDEE/protein targets
└── recommend/
    └── engine.py            Prompt builder + Anthropic API + output formatter
```

---

## Built With

- [Anthropic Claude](https://anthropic.com) — recommendation reasoning
- [Hevy API](https://hevy.com) — workout data
- Apple HealthKit / Health Auto Export — sleep and nutrition data
- SQLite — local data store
- pdfplumber — Cal AI PDF parsing

---

## From Package to App

Triad started as a Python package — the data foundation. That package is now evolving into **[BodyByData](#)**, a full application built on top of it: the package handles data normalization and feature engineering, while the app adds the intelligence layer, daily recommendations, and a real interface people actually use.

---

## Status &amp; Access

The `triad` core package is currently in a **private repository** while the underlying senior project is still being developed and evaluated. This page exists so anyone who reaches it — professors, reviewers, recruiters — can see what the project is and how it works without needing repo access.

If you'd like to see the actual source, reach out and I'm happy to grant access on a case-by-case basis.

---

## Author

Jonathan Ascona · BYU-Idaho Data Science · Spring 2026
