---
description: Generate a single Compose screen from a Figma design or screenshot, following the project's chosen architecture, reusing existing colors/strings/dimens/shapes/typography/components before adding anything new
---

# /generate-screen

You are generating a **single screen** (state holder + ViewModel + Screen, plus data/domain wiring if it needs an API) inside `feature/<feature_name>/`. This is narrower than `/generate-feature`: it starts from a Figma design instead of a feature description, and it is biased toward *reusing* what already exists in the codebase before creating anything new.

## Rules
- When anything is unclear — ask before generating. Never guess and proceed.
- Read `CLAUDE.md` at the project root first. Its `## Architecture` section tells you which pattern is in use; its `## Hard Rules — Never Violate`, `## Architecture Hard Rules` (if present), `## Constraints`, and `## Code Style` sections are binding for everything generated here.

## Step 1 — Intake

Ask for whatever the person hasn't already given in their message:
1. Figma link, or a screenshot of the screen
2. Feature name + screen name (e.g. `feature: auth`, `screen: signIn`)
3. Whether this screen needs an API call — if yes, the real endpoint (check `docs/api/README.md` first) or "mock" if the backend isn't ready yet

Don't proceed past this step with guesses — a wrong feature/screen name or wrong endpoint is expensive to unwind later.

## Step 2 — Load the matching feature template

Based on `CLAUDE.md`'s `## Architecture` section, read the matching file fully — every later step follows its per-layer requirements:

| Architecture | Template |
|---|---|
| MVI + Clean | `.claude/templates/feature/mvi-clean.md` |
| MVI | `.claude/templates/feature/mvi.md` |
| MVVM + Clean | `.claude/templates/feature/mvvm-clean.md` |
| MVVM | `.claude/templates/feature/mvvm.md` |

## Step 3 — Read the design

- If a Figma link was given: invoke the `figma-design-to-code` skill first (mandatory prerequisite before calling `get_design_context`), then pull `get_design_context` and `get_variable_defs`. Variables matter here — they're what Step 4 diffs against the project's own tokens, not just a visual read of a screenshot.
- If only a screenshot was given: read it directly; there are no Figma variables to diff, so Step 4 relies on visual/measured comparison instead.

## Step 4 — Resource-reuse gate (before creating anything new)

Check each of the following, in order. Only create a new entry once you've confirmed nothing existing is an exact or close match — don't create first and dedupe later.

| Resource | Check here first |
|---|---|
| Color | `core/ui/theme/Color.kt` tokens → `MaterialTheme.colorScheme.*` roles |
| String | `values/strings.xml` — search by **key and by value text** (e.g. an existing "Try Again" should stop a new "Retry" from being added) |
| Dimen | `core/ui/theme/Dimens.kt`. If the design specifies a value close to an existing token (e.g. 14dp vs. an existing 12dp/16dp), ask whether to snap to the existing token or add a new one — don't silently add |
| Shape | `core/ui/theme/Shape.kt` before defining a new corner radius/shape |
| Typography | `core/ui/theme/Typography.kt` — **mandatory**: map every text in the design to an existing `MaterialTheme.typography.*` slot, per the typography Hard Rule in `CLAUDE.md`. If the design's text doesn't match any slot, stop and ask |
| Components | `core/ui/components/` first (`AppScreen`, `AppTopBar`, `AppButton`, `AppChip`, `Avatar`, `LoadingIndicator`, `ErrorView`, `EmptyView`, plus anything added since), then the feature's own `presentation/components/`. The body always composes inside `AppScreen`. Before generating a new button/input/chip/card, confirm nothing there already matches — a missing variant becomes a new param on the core component (ask first) |
| Model / DTO | If this screen's data already exists as a model anywhere: this feature's own model and DTO folders first, then other features' models for the same real-world entity. Never import another feature's model directly — consume it per the feature template's "Cross-feature consumption pattern" |

Report what was reused vs. newly added as part of Step 9's output — don't just assert it happened.

## Step 5 — Locate or scaffold the feature

- If `feature/<feature_name>/` already exists: generate directly into its `presentation/` (or `presentation/<screenName>/` if it already has 2+ screens). Reuse its existing data/domain/di code — don't duplicate models, DTOs, or repositories that already exist there.
- If `feature/<feature_name>/` does not exist yet: **scaffold presentation-only** (the template's state holder + ViewModel + Screen files) and **stop and ask** before creating the data, domain or di layers for a brand-new feature. Don't assume the full feature scaffold is wanted just because the screen needs an API; confirm first, then follow the feature template for those layers.

## Step 6 — Generate the screen

Follow the feature template's per-layer requirements exactly for whichever layers apply (state holder, ViewModel, Screen, and data/domain/di if confirmed). In every architecture:
- The Screen collects state with `collectAsStateWithLifecycle()`, holds no business logic, and has a `@Preview` with representative sample state
- **Real API:** the endpoint goes into the feature's ApiService and Repository exactly as the template describes, and is added to `docs/api/README.md` (status 🟡 In progress, or 🟢 Integrated once it works)
- **Mock:** keep the mock inside the data layer — the Repository returns sample data, marked `// TODO(mock): replace with <endpoint>` — so switching to the real endpoint later touches only the data layer, never the ViewModel, UseCase or Screen. Add the endpoint to `docs/api/README.md` as ⚪ Not started

## Step 7 — Strings in every locale

For every genuinely new string (it survived the Step 4 reuse check): add it to `values/strings.xml` **and** every other `values-<locale>/strings.xml` the project has, in this same step, with natural phrasing in each language — never a placeholder, never a literal word-for-word translation.

## Step 8 — Wire navigation

Add the route to `navigation/Routes.kt` and register the composable in `navigation/AppNavGraph.kt`, per the feature template's "Navigation wiring" section. This is the only shared/core file this command touches beyond whatever new resource entries Step 4/7 required.

## Step 9 — Verify and report

- Run `./gradlew ktlintCheck testDebugUnitTest assembleDebug` — the screen compiles, its ViewModel test passes, and ktlint is clean
- Confirm no rule in `CLAUDE.md`'s Hard Rules sections was violated
- Report back:
  - Feature + screen name, files created
  - **Resources reused** (existing color/string/dimen/shape/typography/component/model it matched onto) vs. **resources newly added** (with justification for why nothing existing matched)
  - Route(s) added
  - Whether this was presentation-only (existing feature) or paused for confirmation (new feature, Step 5)
