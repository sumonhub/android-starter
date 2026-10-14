---
description: Generate a new self-contained feature (domain/data/presentation/di) matching the project's chosen architecture
---

# /generate-feature

You are generating a **new feature module** under `feature/<feature_name>/`. This command decides the order of work; the feature template decides what the code looks like. Never restate or reinterpret a template rule here — follow the template.

## Rules
- When anything is unclear — ask before generating. Never guess and proceed.
- Read `CLAUDE.md` at the project root first. Its `## Architecture` section tells you which pattern is in use; its `## Hard Rules — Never Violate`, `## Architecture Hard Rules` (if present), `## Constraints` and `## Code Style` sections are binding for everything generated here.

## Step 0 — Check the project is initialized

Use the same check as `/init-project`'s Step 0: `CLAUDE.md` exists at the root with an `## Architecture` section, and `core/` exists under the app module. If either is missing, stop. Tell the person the shared foundation isn't scaffolded yet and that `/init-project` must run first. Never generate a feature without it, and never pick an architecture yourself.

## Step 1 — Get the feature name

If the person already gave a feature name in their message (e.g. "generate the Order feature"), use it directly — don't ask again. Otherwise ask for one.

Derive every name (package, class prefix, Koin module, routes) per the "Naming" section of `.claude/templates/shared/feature-conventions.md`.

If `feature/<feature_name>/` already exists, stop. Never overwrite or regenerate an existing feature. Tell the person, and point them to `/generate-screen` to add a screen to it.

## Step 2 — Load the matching feature template

Based on `CLAUDE.md`'s `## Architecture` section, read the matching file fully, plus `.claude/templates/shared/feature-conventions.md`, before generating anything:

| Architecture | Template |
|---|---|
| MVI + Clean | `.claude/templates/feature/mvi-clean.md` |
| MVI | `.claude/templates/feature/mvi.md` |
| MVVM + Clean | `.claude/templates/feature/mvvm-clean.md` |
| MVVM | `.claude/templates/feature/mvvm.md` |

If `CLAUDE.md` names an architecture not in this table, or the file is missing, say so and stop — don't improvise an architecture that wasn't scaffolded.

## Step 3 — Apply the template's defaults

Each feature template has a "## Defaults (no need to ask)" section. Apply it directly. Only ask a question if the person's own description of the feature implies something the defaults don't cover (e.g. pagination, per-request auth tokens).

If the feature has no API (local data only, or no data at all), follow "Features without an API" in `feature-conventions.md` instead of generating `data/remote/`. If it isn't clear whether the feature needs an API, ask.

## Step 4 — Check for cross-feature reuse

Before creating a new model, check whether an existing feature under `feature/` already exposes the same concept. If so, consume it per the template's "Cross-feature consumption pattern" (via its UseCase in Clean architectures, via its Repository in plain ones) instead of duplicating the model.

## Step 5 — Generate the feature

Generate exactly the template's folder structure, per-layer requirements and tests ("Forms, pickers, dialogs and tests"). Include every file listed, no more and no less, and skip conditional files the feature doesn't need (e.g. a Search UseCase without search).

For the feature's screens, also apply from `/generate-screen` (`.claude/commands/generate-screen.md`):
- **Step 4** — the resource-reuse gate (color, string, dimen, shape, typography, components) before adding any new resource
- **Step 7** — every new string goes into `values/strings.xml` and every `values-<locale>/strings.xml`

For every endpoint the feature calls, add or update its entry in `docs/api/README.md` per the project's `## Reference Docs` rules (🟡 In progress while wiring, 🟢 Integrated once it works, ⚪ Not started if mocked).

## Step 6 — Wire navigation and DI

- Add the feature's route(s) to `navigation/Routes.kt` and register the composable(s) in `navigation/AppNavGraph.kt`, per the template's "Navigation wiring" section.
- Register the feature's Koin module in `App.kt`, per the template's `di/` section (append to the existing module list).

These three files, plus resource entries from Step 5, are the only shared files this command touches. Everything else stays inside `feature/<feature_name>/` and its test folder.

## Step 7 — Verify

- Run `./gradlew ktlintCheck testDebugUnitTest assembleDebug` — the feature compiles, its new tests pass, and ktlint is clean
- Confirm no Hard Rule from `CLAUDE.md` was violated
- Report back: feature name, files created (tests included), resources reused vs. newly added, routes added, and `docs/api/README.md` entries touched
