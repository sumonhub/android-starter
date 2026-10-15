---
description: Generate a new self-contained feature (domain/data/presentation/di) matching the project's chosen architecture
disable-model-invocation: true
---

# /android-starter:generate-feature

> **Plugin files:** this skill's templates live in `${CLAUDE_PLUGIN_ROOT}/templates/`. When a template mentions a path that starts with `templates/`, it means `${CLAUDE_PLUGIN_ROOT}/templates/`.

You are generating a **new feature module** under `feature/<feature_name>/`. This command decides the order of work. The feature template decides what the code looks like. Never restate or reinterpret a template rule here. Follow the template.

## Rules
- If anything is unclear, ask before you generate anything. Never guess and continue.
- Read `CLAUDE.md` at the project root first. Its `## Architecture` section tells you which pattern the project uses. Everything you generate here must follow its `## Hard Rules — Never Violate`, `## Architecture Hard Rules` (if present), `## Constraints` and `## Code Style` sections.

## Step 0 — Check the project is initialized

Use the same check as `/android-starter:init-project`'s Step 0: `CLAUDE.md` exists at the root with an `## Architecture` section, and `core/` exists under the app module. If either is missing, stop. Tell the person that the shared foundation isn't scaffolded yet and that `/android-starter:init-project` must run first. Never generate a feature without it. Never pick an architecture yourself.

## Step 1 — Get the feature name

If the person already gave a feature name in their message (e.g. "generate the Order feature"), use it. Don't ask again. Otherwise, ask for one.

Derive every name (package, class prefix, Koin module, routes) as described in the "Naming" section of `${CLAUDE_PLUGIN_ROOT}/templates/shared/feature-conventions.md`.

If `feature/<feature_name>/` already exists, stop. Never overwrite or regenerate an existing feature. Tell the person, and point them to `/android-starter:generate-screen` to add a screen to it.

## Step 2 — Load the matching feature template

Check `CLAUDE.md`'s `## Architecture` section. Read the matching file below fully, plus `${CLAUDE_PLUGIN_ROOT}/templates/shared/feature-conventions.md`, before you generate anything:

| Architecture | Template |
|---|---|
| MVI + Clean | `${CLAUDE_PLUGIN_ROOT}/templates/feature/mvi-clean.md` |
| MVI | `${CLAUDE_PLUGIN_ROOT}/templates/feature/mvi.md` |
| MVVM + Clean | `${CLAUDE_PLUGIN_ROOT}/templates/feature/mvvm-clean.md` |
| MVVM | `${CLAUDE_PLUGIN_ROOT}/templates/feature/mvvm.md` |

If `CLAUDE.md` names an architecture that is not in this table, or the template file is missing, say so and stop. Don't invent an architecture that wasn't scaffolded.

## Step 3 — Apply the template's defaults

Each feature template has a "## Defaults (no need to ask)" section. Apply it directly. Ask a question only if the person's own description of the feature suggests something the defaults don't cover (e.g. pagination, per-request auth tokens).

If the feature has no API (local data only, or no data at all), follow "Features without an API" in `feature-conventions.md`. Do not generate `data/remote/`. If it isn't clear whether the feature needs an API, ask.

## Step 4 — Check for cross-feature reuse

Before you create a new model, check whether an existing feature under `feature/` already exposes the same concept. If it does, don't duplicate the model. Use it as the template's "Cross-feature consumption pattern" describes: through its UseCase in Clean architectures, or through its Repository in plain ones.

## Step 5 — Generate the feature

Generate exactly what the template lists: its folder structure, per-layer requirements and tests ("Forms, pickers, dialogs and tests"). Include every listed file, no more and no less. Skip conditional files that the feature doesn't need (e.g. a Search UseCase when there is no search).

For the feature's screens, also apply these steps from `/android-starter:generate-screen` (`${CLAUDE_PLUGIN_ROOT}/skills/generate-screen/SKILL.md`):
- **Step 4** — the resource-reuse gate (color, string, dimen, shape, typography, components). Run it before you add any new resource
- **Step 7** — add every new string to `values/strings.xml` and to every `values-<locale>/strings.xml`

For every endpoint the feature calls, add or update its entry in `docs/api/README.md`. Follow the project's `## Reference Docs` rules: 🟡 In progress while wiring, 🟢 Integrated once it works, ⚪ Not started if mocked.

## Step 6 — Wire navigation and DI

- Add the feature's route(s) to `navigation/Routes.kt`. Register the composable(s) in `navigation/AppNavGraph.kt`. Follow the template's "Navigation wiring" section.
- Register the feature's Koin module in `App.kt`, as the template's `di/` section describes. Append it to the existing module list.

This command touches only these three shared files, plus the resource entries from Step 5. Everything else stays inside `feature/<feature_name>/` and its test folder.

## Step 7 — Verify

- Run `./gradlew ktlintCheck testDebugUnitTest assembleDebug`. The feature must compile, its new tests must pass, and ktlint must be clean
- Confirm that no Hard Rule from `CLAUDE.md` was broken
- Report back: the feature name, the files created (tests included), the resources reused vs. newly added, the routes added, and the `docs/api/README.md` entries you touched
