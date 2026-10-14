---
description: Generate a new self-contained feature (domain/data/presentation/di) matching the project's chosen architecture
---

# /generate-feature

You are generating a **new feature module** under `feature/<feature_name>/`. This assumes `/init-project` has already been run and the shared scaffold + `CLAUDE.md` exist.

## Rules
- When anything is unclear — ask before generating. Never guess and proceed.
- Read `CLAUDE.md` at the project root first. Its `## Architecture` section tells you which pattern is in use; its `## Hard Rules — Never Violate` and `## Architecture Hard Rules`, and `## Constraints` sections are binding for everything you generate here.

## Step 1 — Get the feature name

If the person already gave a feature name in their message (e.g. "generate the Order feature"), use it directly — don't ask again. Otherwise ask for one.

## Step 2 — Load the matching feature template

Based on `CLAUDE.md`'s `## Architecture` section, read the matching file fully before generating anything:

| Architecture | Template |
|---|---|
| MVI + Clean | `.claude/templates/feature/mvi-clean.md` |
| MVI | `.claude/templates/feature/mvi.md` |
| MVVM + Clean | `.claude/templates/feature/mvvm-clean.md` |
| MVVM | `.claude/templates/feature/mvvm.md` |

If the matching file doesn't exist yet, say so and stop — don't improvise an architecture that wasn't scaffolded.

## Step 3 — Apply the template's defaults

Each feature template has a "## Defaults (no need to ask)" section. Apply it directly. Only ask a question if the person's own description of the feature implies something the defaults don't cover (e.g. they mention pagination, auth tokens, or a non-REST data source).

## Step 4 — Check for cross-feature reuse

Before creating a new domain model, check whether an existing feature under `feature/` already exposes the same concept via its Repository interface or UseCases. If so, follow the template's "Cross-feature consumption pattern" instead of duplicating the model.

## Step 5 — Generate the feature

Follow the template's folder structure and per-layer requirements exactly:
- Every file listed, no more, no less (skip conditional files like a Search UseCase if the feature doesn't need search)
- State/Intent/Effect (MVI) or UiState/UiEvent (MVVM) implement the marker interfaces from `core/mvi/` or `core/mvvm/`
- ViewModel extends the shared `BaseViewModel`, never redeclares state/effect plumbing
- API calls go through the core helper the template names (`safeApiCall`) — never a hand-written `try/catch` in a repository
- DTOs never leave `data/`

## Step 6 — Wire navigation

Add the feature's route(s) to `navigation/Routes.kt` and register the composable(s) in `navigation/AppNavGraph.kt`, per the template's "Navigation wiring" section. This is the only shared/core file this command touches — everything else stays inside `feature/<feature_name>/`.

## Step 7 — Verify

- Run `./gradlew ktlintCheck testDebugUnitTest assembleDebug` — the feature compiles, its new ViewModel and mapper tests pass, and ktlint is clean
- Confirm no Hard Rule from `CLAUDE.md` was violated
- Report back: feature name, files created (tests included), and the routes added
