# Aim Brain Engine

## Purpose

The Aim Brain Engine is the core product layer of Aim Brain. It is not a UI layout, not a dashboard decoration, and not a spreadsheet clone. It is the deterministic logic layer that turns raw aim-training data into coaching decisions.

The central product rule is:

> The UI does not think. The Aim Brain Engine thinks. The UI only displays the engine output.

This keeps the app trustworthy, testable, and expandable. A clean dashboard can make the product look good, but the engine is what makes the product useful.

## Core data flow

```text
Raw score sources
→ import queue
→ validation and deduplication
→ scenario matching
→ raw run database
→ score statistics
→ rank conversion
→ category analysis
→ weakness diagnosis
→ training recommendation
→ coach summary
→ UI display
```

Aim Brain must be built around raw runs as the source of truth. Aggregates are derived from accepted runs, not manually maintained as primary data.

## Non-negotiable rules

1. **Raw runs are the source of truth.** Do not base the app on only PBs, averages, or UI-state summaries.
2. **The engine is separate from the UI.** React components must not contain performance-analysis logic.
3. **The engine is deterministic first.** The same database plus same threshold pack must return the same result.
4. **Rule-based recommendations come before AI text.** AI may later rewrite explanations, but it must not decide the actual analysis.
5. **No fake rank conversion.** If a scenario has no benchmark threshold data, show progress-only mode and say threshold data is missing.
6. **Aimlabs and KovaaK’s stay platform-aware.** Do not merge platform-specific scenario identities without a confirmed alias/mapping layer.
7. **Manual logging is a fallback/debug path only.** The main product direction is automatic/local import, especially Aimlabs local/import support and KovaaK’s stats support.
8. **Every recommendation must explain why.** Coach cards should show: recommendation, evidence, and success condition.

## Engine responsibilities

### 1. Score Engine

Calculates scenario-level and run-level statistics:

- latest score
- personal best
- all-run average
- last 5 average
- last 10 average
- total valid runs
- score volatility
- consistency
- improvement rate
- latest-vs-average delta
- PB-vs-last10 gap

The Score Engine must ignore rejected/deleted runs and should clearly separate accepted, duplicate, suspect, and needs-review data.

### 2. Scenario Engine

Matches imported scenario names to canonical scenarios.

It must handle cases like:

```text
VT Pasu Intermediate
Voltaic Pasu Int
Pasu Intermediate
```

Required responsibilities:

- normalize scenario names
- match exact vendor/task IDs where available
- match confirmed aliases
- use fuzzy search only as a candidate generator
- send uncertain matches to review
- persist confirmed aliases
- keep platform information attached to the scenario

The Scenario Engine protects the product from polluted data. Bad matching ruins every downstream metric.

### 3. Rank Engine

Converts scores into rank progress when threshold data exists.

Supported modes:

- `voltaic_energy`: for official Voltaic-style energy systems where available
- `simple_thresholds`: for benchmark sets with direct threshold bands
- `progress_only`: for practice scenarios or scenarios without known thresholds

Missing threshold rule:

```ts
{
  rank: null,
  nextRank: null,
  progressToNextRank: null,
  hasThresholdData: false,
  mode: "progress_only",
  note: "Progress only. No benchmark threshold data available."
}
```

Never invent rank data for unsupported scenarios.

### 4. Category Brain

Groups scenario results into aim categories and subcategories.

Default UI labels:

- Static Clicking
- Dynamic Clicking
- Smooth Tracking
- Reactive Tracking
- Speed Switching
- Evasive Switching
- Hybrid / Other

The Category Brain should calculate:

- strongest category
- weakest category
- most improved category
- most inconsistent category
- most neglected category
- closest-to-rank-up category
- category readiness
- category volume balance

### 5. Trend Engine

Detects progress direction over time.

Supported trend states:

- improving
- stable
- plateauing
- declining
- volatile
- insufficient data
- insufficient recent volume

Initial implementation can use simple slope/median logic. Later implementation can use robust slope methods such as Theil-Sen. The engine should avoid declaring a plateau without enough recent runs and training volume.

### 6. Weakness Detector

The weakness system must not simply pick the lowest PB. It should identify the biggest blocker toward the user’s goal.

Suggested first model:

```text
WeakLinkScore =
  0.45 * TargetGap
+ 0.20 * ConsistencyPenalty
+ 0.15 * RecencyPenalty
+ 0.10 * VolumeDeficit
+ 0.10 * DecayRisk
```

Explanation example:

```text
Reactive Tracking is your biggest blocker for Master because your last-10 average is farthest from the target, confidence is low, and the category has not had enough recent volume.
```

### 7. Training Recommender

Produces the next useful training action.

Inputs:

- weak categories
- recent category volume
- scenario confidence
- trend state
- plateau state
- decay/freshness risk
- user goal rank
- setup/profile context
- session type preference if provided

Example output:

```ts
{
  sessionType: "Weakness Focus",
  priorityCategories: ["Reactive Tracking", "Evasive Switching"],
  avoidOvertraining: ["Static Clicking"],
  recommendedScenarios: [
    {
      scenarioId: "...",
      name: "VT Reactive Tracking Intermediate",
      reason: "Weak category, low consistency, and high target gap"
    }
  ],
  sessionStructure: [
    "Warmup",
    "Benchmark check",
    "Weakness block",
    "Confidence finisher"
  ],
  successCondition: "Raise last-10 average or confidence, not just one lucky PB."
}
```

### 8. Coach Summary Engine

Turns engine results into readable text.

The first implementation should be rule-based templates, not generative AI.

Example:

```text
Your clicking categories are ahead of your tracking categories. Static Clicking is carrying your overall readiness, but Reactive Tracking is holding back your Master goal. Today should be a tracking-focused consistency session, not a PB-grind day.
```

Later, an AI layer may rewrite the wording, but the decision and evidence must come from deterministic engine output.

## Recommended TypeScript module structure

```text
src/
  engine/
    types/
      ScoreRun.ts
      Scenario.ts
      RankThreshold.ts
      PlayerStats.ts
      CategoryAnalysis.ts
      TrainingRecommendation.ts
      AimBrainResult.ts

    normalization/
      normalizeScoreRun.ts
      normalizeScenarioName.ts

    scenarios/
      scenarioDatabase.ts
      scenarioMatcher.ts
      scenarioCategoryMap.ts

    ranks/
      rankScale.ts
      rankThresholds.ts
      calculateRankProgress.ts

    stats/
      calculateScenarioStats.ts
      calculateCategoryStats.ts
      calculateConsistency.ts
      calculateTrend.ts

    diagnosis/
      detectWeakCategories.ts
      detectPlateaus.ts
      detectNeglectedCategories.ts
      detectMasterReadiness.ts

    recommendations/
      recommendTrainingSession.ts
      recommendScenarios.ts
      buildSessionStructure.ts

    coach/
      generateCoachSummary.ts
      generateCategoryNotes.ts

    index.ts
```

The frontend should import from the engine boundary:

```ts
import { analyzeAimProfile } from "@/engine";
```

React components should not directly import low-level calculation files unless there is a strong reason.

## Main engine API

The app should eventually call one main function:

```ts
const result = analyzeAimProfile({
  scoreRuns,
  scenarios,
  rankThresholds,
  userGoal: "Master",
  setupProfiles,
  now: new Date(),
});
```

Expected result shape:

```ts
{
  overview: {
    totalRuns: number;
    strongestCategory: string | null;
    weakestCategory: string | null;
    masterReadiness: number | null;
    overallTrend: "improving" | "stable" | "plateauing" | "declining" | "insufficient_data";
  };

  categories: CategoryAnalysis[];
  scenarios: ScenarioAnalysis[];
  weaknesses: WeaknessFinding[];
  recommendations: TrainingRecommendation;
  coachSummary: string;
  warnings: EngineWarning[];
}
```

## UI integration contract

The UI may display:

- Home dashboard cards
- radar chart
- category strength bars
- weakness cards
- today’s training recommendation
- rank progress
- weekly progress
- coach notes
- import health

But the UI must not calculate the underlying coaching logic. It should render the result returned by `analyzeAimProfile()`.

## Build order

### Phase 1 — Engine foundation

- create TypeScript types
- create sample data
- calculate PB/latest/averages/last-10
- calculate basic consistency
- return first `AimBrainResult`

### Phase 2 — Scenario and category logic

- scenario database
- scenario matcher
- category assignment
- platform-aware aliases
- review fallback for uncertain matches

### Phase 3 — Rank conversion

- rank scale
- threshold lookup
- score-to-rank conversion
- progress to next rank
- missing-threshold fallback

### Phase 4 — Diagnosis

- weak category detection
- plateau detection
- neglected category detection
- Master readiness
- category imbalance

### Phase 5 — Training recommendation

- today’s focus
- recommended scenarios
- session type
- success condition
- coach summary

### Phase 6 — UI integration

- connect Home to engine output
- connect Summary to engine output
- connect Coach Report to engine output
- make all explanation cards evidence-based

## Acceptance criteria

The engine is ready for first product integration when:

- it accepts raw run arrays and scenario/threshold data
- it produces stable output from the same input
- it calculates PB/latest/last-10 correctly
- it does not fake ranks for missing threshold data
- it identifies at least one weak link when enough data exists
- it returns a training recommendation with a reason
- it has tests or sample cases for core calculations
- React UI does not duplicate the engine’s analysis logic

