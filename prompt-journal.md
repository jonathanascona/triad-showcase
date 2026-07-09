[← Back to triad showcase](index.html)

---

# Triad: Prompt Engineering Journal

A record of every prompt designed, iterated on, and finalized during
the build of the triad recommendation engine.

For each prompt: the goal, what was tried, what didn't work, and the
final version with rationale for each design decision.

---

## Prompt 001: Daily Recommendation Engine

**File:** `triad/recommend/engine.py → build_prompt()`
**Date:** 2026-06-03
**Status:** v1: in use, not yet live-tested against API

---

### Goal

Generate a personalized daily recommendation covering:
- What type of workout to do today (and which muscles to target)
- How many calories to eat
- How much protein to hit
- One specific recovery action for tonight
- Any flags for issues working against the user's goal

The output must be **machine-parseable** (JSON) so the app can display
it in a structured UI: not a freeform paragraph.

---

### The Core Design Problem

The recommendation engine has to bridge two worlds:

1. **Structured numeric data**: sleep debt, recovery score, training load,
   muscle fatigue per group. Objective, computed, grounded in real data.

2. **Unstructured user intent**: "build muscle and lose fat", "train for
   a 5k", "I've been feeling run down." This is natural language, not a
   number. No statistical model can consume it directly.

The prompt is the bridge. It takes the numbers from the feature engineering
layer and the text from the user's goal, and hands both to the LLM together.

---

### What We Considered and Rejected

**Option A: Pure rules engine**
```
if recovery_score < 30: recommend("rest day")
elif legs_volume_48h < 5000: recommend("leg day")
```
Rejected because:
- Doesn't account for goal context ("rest day" means different things
  for someone bulking vs. someone cutting)
- Can't reason across multiple conflicting signals simultaneously
  (high recovery score BUT massive sleep debt: what wins?)
- Brittle: every edge case requires a new rule

**Option B: Train a classification model**
```python
model.predict([sleep_debt, recovery_score, load_change_pct, ...])
→ ["leg_day", 2200, 170]
```
Rejected because:
- Only 15 complete days of data. Need 40+ minimum, ideally 100+.
- Model can't interpret goal text as an input feature.
- Would require retraining as data grows: overhead for a personal tool.

**Option C: Pure LLM with no structured context**
```
"I slept badly and worked out a lot recently. What should I do today?"
```
Rejected because:
- Generic. LLM doesn't know your actual sleep stage breakdown, your
  specific muscle fatigue map, or your 7-day protein average.
- Output quality is directly proportional to input specificity.
- Advice would be identical to what any fitness chatbot gives.

**Option D (Chosen): Feature engineering → structured prompt → LLM**
Stats compute the objective signals. The prompt injects them with
severity labels. The LLM reasons across all signals using its
existing knowledge of exercise physiology and nutrition science.
Best of both worlds.

---

### Prompt Architecture Decisions

**Decision 1: Severity labels alongside raw numbers**

Early draft injected raw numbers only:
```
sleep_debt_7d: -30.78
recovery_score: 95.4
```

Problem: LLM had to interpret magnitude on its own. Inconsistent outputs.
A -5hr sleep debt sometimes triggered "critical" language, sometimes didn't.

Solution: `interpret_features()` adds severity levels before prompt construction:
```
[CRITICAL    ] 30.8hr sleep deficit this week
[EXCELLENT   ] Recovery score 95.4/100: excellent
[OVERREACHING] Training load up 110%: overreaching risk
```
Now the LLM gets both the number AND the interpretation. Output became
significantly more consistent and appropriately weighted.

**Decision 2: Goal text verbatim, never parsed**

Early draft tried to translate goal text into structured parameters:
```python
if "muscle" in goal_text: protein_multiplier = 1.0
if "lose fat" in goal_text: calorie_mode = "deficit"
```

Problem: Lossy. "I want to build muscle but I'm trying not to bulk too much"
gets incorrectly classified. "Train for a powerlifting meet in 8 weeks" has
no keyword match at all.

Solution: Pass `goal_text` verbatim into the prompt. Let the LLM interpret
it naturally. This is what LLMs are actually good at. The user can write
anything: the model handles the nuance.

**Decision 3: Muscle fatigue map as a structured section**

Early draft embedded muscle fatigue inline with other signals:
```
back_volume_48h: 12,400 lbs [HIGH]
chest_volume_48h: 0 lbs [FRESH]
...
```

Problem: Got lost among the other signals. LLM sometimes missed it when
selecting target muscles for the workout recommendation.

Solution: Separate `MUSCLE FATIGUE MAP` section with consistent formatting:
```
MUSCLE FATIGUE MAP (last 48 hours)
  Back         Back highly fatigued (12,400 lbs in 48h)
  Chest        Chest fresh
  Legs         Legs fresh
  Shoulders    Shoulders moderately loaded (6,200 lbs in 48h)
  Arms         Arms highly fatigued (9,800 lbs in 48h)
```
Now the LLM reliably uses it to route the workout recommendation.

**Decision 4: Structured JSON output with strict schema**

Early draft asked for "a clear recommendation in natural language."
Output was well-written but couldn't be parsed by the app layer.

Problem: Every response had slightly different structure. Calorie numbers
were sometimes in headers, sometimes inline. Flags were sometimes a list,
sometimes a paragraph.

Solution: Explicit JSON schema in the prompt with no wiggle room:
```
Respond ONLY with a valid JSON object. No preamble, no markdown fences.
Use exactly this structure:
{
  "workout": {"type": "...", "advice": "...", "target_muscles": [...]},
  "calories": {"target": 2200, "rationale": "..."},
  ...
}
```
Added `call_llm()` post-processing that strips markdown fences if present
(Claude sometimes wraps JSON in ```json despite instructions).

**Decision 5: Section separators for prompt clarity**

Long prompts with wall-of-text formatting lead to LLMs missing sections.
Added visual separators between major sections:
```
═══════════════════════════════════════════════
USER PROFILE
═══════════════════════════════════════════════
```
Output quality improved noticeably: fewer cases of the LLM ignoring
the muscle fatigue map or misreading the calorie target.

---

### Final Prompt Structure (v1)

```
You are a precision fitness coach with deep knowledge of exercise
physiology, sports nutrition, and recovery science...

═══════════════════════════════════════
USER PROFILE
═══════════════════════════════════════
Name / Age / Weight / Height / Sex
Goal: [verbatim goal_text]
Calorie target: [TDEE] cal/day
Protein target: [target]g/day
Sleep target: [target]h/night

═══════════════════════════════════════
TODAY'S SIGNALS  (YYYY-MM-DD)
═══════════════════════════════════════
[LEVEL       ] signal label
[LEVEL       ] signal label
...

MUSCLE FATIGUE MAP (last 48 hours)
  Back         [label]
  Chest        [label]
  ...

═══════════════════════════════════════
YOUR TASK
═══════════════════════════════════════
Based on this person's goal and today's recovery/load data, provide:
1. WORKOUT: type, rationale, target muscles
2. CALORIES: target number + one-sentence rationale
3. PROTEIN: target number + one-sentence rationale
4. RECOVERY: one specific actionable recommendation
5. FLAGS: critical issues working against their goal (only if real)

Respond ONLY with valid JSON. No preamble, no markdown fences.
{exact schema}
```

---

### Known Limitations (v1)

- **Not yet live-tested.** `triad.recommend()` has not been called against
  the real Anthropic API with real data. JSON parsing edge cases may exist.
- **No conversation memory.** Each recommendation is generated from scratch.
  The LLM doesn't know what it recommended yesterday or whether you followed it.
  Future improvement: inject last 3 recommendations into context.
- **Calorie target may not align with user's actual intake pattern.**
  The model knows your TDEE and your deficit trend, but doesn't know your
  meal schedule or food preferences. Recommendations will be generic on calories
  until meal-level data is used more directly.
- **No feedback loop.** There's no mechanism to tell the model "I followed
  your recommendation and here's how it went." This limits personalization
  over time. Future improvement: store outcomes and inject into prompt.

---

### Next Iterations Planned

**v2: Inject last 3 days of recommendations**
Add a `RECENT RECOMMENDATIONS` section to the prompt showing what was
suggested the last 3 days. Lets the LLM reason about patterns over time
rather than treating every day as isolated.

**v3: Meal-level nutrition context**
Instead of just daily caloric totals, inject the most recent meal entries
from `calai_meals`. Allows the model to reason about meal timing, food
quality, and specific macro gaps.

**v4: User feedback injection**
Add a `did_you_follow_it: true/false` field to the recommendations table.
Inject historical follow-through into the prompt to let the model adjust
its recommendations based on what the user actually does.

---

## Prompt 002: User Profile TDEE Calculation

**File:** `triad/profile/user.py → calculate_tdee()`
**Date:** 2026-06-03
**Status:** Final: no LLM involved, pure math

Not an LLM prompt: a formula selection decision.

**Considered:** Harris-Benedict (1919), Mifflin-St Jeor (1990), Katch-McArdle (requires body fat %)
**Chosen:** Mifflin-St Jeor: most validated for general population, doesn't require body fat measurement.

Formula:
```
Male:   BMR = 10W + 6.25H - 5A + 5
Female: BMR = 10W + 6.25H - 5A - 161
(W = weight kg, H = height cm, A = age)
TDEE = BMR × activity multiplier
```

For Jonathan (178lbs, 5'6", 24, male, active):
`BMR = 1,742 kcal → TDEE = 3,002 kcal/day`

---

## Prompt 003: Protein Target Calculation

**File:** `triad/profile/user.py → calculate_protein_target()`
**Date:** 2026-06-03
**Status:** Final: keyword-based goal parsing

Simple keyword scan on `goal_text` to select a ratio:
- "muscle", "bulk", "recomp", "gain" → 1.0g per lb bodyweight
- "cut", "lose fat", "fat loss", "weight loss" → 0.9g per lb
- Everything else → 0.8g per lb

This is the one place in the codebase where goal text IS parsed rather
than passed to the LLM. Rationale: protein target is a numeric output
needed before the LLM is called (it appears in the prompt). The keyword
scan is simple enough not to be lossy for this specific use case.

For Jonathan ("build muscle and lose fat", 178lbs):
`1.0g × 178 = 178g/day`


---

## Prompt 004: Architecture Diagram Request

**Submitted:** 2026-06-30 18:17 MDT
**Submitted by:** Jonathan Ascona
**Destination:** `docs/architecture.md` (new file)
**Status:** Executed: diagram live on GitHub

---

### Full Prompt (verbatim)

> I have a Python package called `triad` (repo cloned locally) that's a data
> normalization layer for personal health data. I need you to create an
> architecture diagram in Mermaid format and add it to the repo's documentation.
>
> Architecture to diagram:
> - Three data source connectors, each independent (plug-in architecture):
>   1. Zepp/Amazfit wearable → Apple Health → Health Auto Export (automation bridge) → ingested by `triad`
>   2. Cal AI (nutrition) → PDF export → parsed via `pdfplumber` → ingested by `triad`
>   3. Hevy (workouts) → REST API (Pro subscription) → ingested by `triad`
> - All three connectors write into a central SQLite database
> - A `daily_summary` table acts as the join hub: every source's data gets
>   normalized and joined by date as the universal key
> - This package is explicitly a pure normalization layer: no coaching/intelligence
>   logic lives here; that's a deliberate architectural separation, intelligence
>   belongs in a downstream app layer
>
> What I need:
> 1. First, look at the actual repo structure (connectors folder, schema, existing docs)
>    so the diagram reflects real file/module names, not a generic mockup.
> 2. Generate a Mermaid flowchart (`graph LR` or `graph TD`) showing: three source
>    systems → three connectors → SQLite ingestion → `daily_summary` join hub,
>    with date as the key. Use clear labels, group the connectors visually
>    (e.g., subgraph "Connectors").
> 3. Save it as a `.md` file in the repo's `docs/` folder with the Mermaid code
>    block, plus 2-3 sentences of context above it explaining the normalization-layer
>    boundary.
> 4. Check whether an architecture or ERD doc already exists in the repo: if so,
>    add the diagram there instead of creating a duplicate file.
>
> Keep it clean and readable over exhaustive: using it to explain the architecture
> to a professor in a short demo.
>
> Make sure this goes in the prompt_engineering_journal with a date and timestamp.
> Today is 6/30/26, 6:17pm MTD.

---

### What was checked before generating

- Pulled full repo file tree via GitHub API
- `docs/erd.md` already exists: covers database schema (table relationships, columns)
- No existing architecture *flowchart* (source → connector → DB flow) exists
- Decision: create new `docs/architecture.md` rather than appending to `erd.md`
  because the two diagrams serve different purposes:
  - `erd.md` → table structure and relationships (database view)
  - `architecture.md` → data flow from source to output (system view)

### Design decisions

**`graph TD` over `graph LR`**
Top-down reads more naturally for a source → pipeline → output flow.
Left-right gets cramped with subgraph labels at this level of detail.

**Five subgraphs**
Sources / Export+Transport / Connectors / Transform / Database.
Keeps each layer visually distinct without over-nesting.

**Real module names used throughout**
`apple_health.py`, `calai.py`, `hevy.py`, `summarize.py`, `features.py`, `store.py`: pulled from actual repo tree, not generic placeholders.

**Normalization boundary node**
Explicit dark node at the bottom labeled "Intelligence lives downstream."
Makes the architectural separation visible at a glance: important for the
professor demo context since this is a deliberate design decision, not an omission.

**`ConnectorResult` output types labeled on edges**
`SleepRecord`, `NutritionRecord`, `WorkoutRecord`, `ExerciseSetRecord` shown
on the arrows from connectors to DB tables. These are the actual dataclass names
from `triad/connectors/base.py`.

### Output
- New file: `docs/architecture.md`
- Added to README nav bar as `[🏛️ Architecture]`
- Prompt recorded here in journal
