---
description: Generate a single Compose screen from a Figma design or screenshot, following the project's chosen architecture, reusing existing colors/strings/dimens/shapes/typography/components before adding anything new
---

# /generate-screen

You are generating a **single screen** (state holder + ViewModel + Screen, plus data/domain wiring if it needs data) inside `feature/<feature_name>/`. This is narrower than `/generate-feature`: it starts from a Figma design instead of a feature description, and it is biased toward *reusing* what already exists in the codebase before creating anything new.

This command decides the order of work; the feature template decides what the code looks like. Never restate or reinterpret a template rule here — follow the template.

## Rules
- When anything is unclear — ask before generating. Never guess and proceed.
- Read `CLAUDE.md` at the project root first. Its `## Architecture` section tells you which pattern is in use; its `## Hard Rules — Never Violate`, `## Architecture Hard Rules` (if present), `## Constraints`, and `## Code Style` sections are binding for everything generated here.

## Step 0 — Check the project is initialized

Use the same check as `/init-project`'s Step 0: `CLAUDE.md` exists at the root with an `## Architecture` section, and `core/` exists under the app module. If either is missing, stop. Tell the person the shared foundation isn't scaffolded yet and that `/init-project` must run first. Never generate a screen without it, and never pick an architecture or a folder yourself.

## Step 1 — Intake

Ask for whatever the person hasn't already given in their message:
1. Figma link, or a screenshot of the screen
2. Feature name + screen name (e.g. `feature: auth`, `screen: signIn`). Derive every name per the "Naming" section of `.claude/templates/shared/feature-conventions.md`
3. What data the screen needs:
   - **API** — the real endpoint (check `docs/api/README.md` first), or "mock" if the backend isn't ready yet
   - **Local only** (settings, flags, drafts) or **no data** (static screen) — follow "Features without an API" in `feature-conventions.md`

Don't proceed past this step with guesses — a wrong feature/screen name or wrong endpoint is expensive to unwind later.

## Step 2 — Load the matching feature template

Based on `CLAUDE.md`'s `## Architecture` section, read the matching file fully, plus `.claude/templates/shared/feature-conventions.md` — every later step follows their requirements:

| Architecture | Template |
|---|---|
| MVI + Clean | `.claude/templates/feature/mvi-clean.md` |
| MVI | `.claude/templates/feature/mvi.md` |
| MVVM + Clean | `.claude/templates/feature/mvvm-clean.md` |
| MVVM | `.claude/templates/feature/mvvm.md` |

If `CLAUDE.md` names an architecture not in this table, or the file is missing, say so and stop.

## Step 3 — Read the design

- If a Figma link was given: invoke the `figma-design-to-code` skill first (mandatory prerequisite before calling `get_design_context`), then pull `get_design_context` and `get_variable_defs`. Variables matter here — they're what Step 4 diffs against the project's own tokens, not just a visual read of a screenshot.
- If the Figma tools aren't available or the Figma server isn't authorized, say so: it has to be connected in the claude.ai connector settings, or via `/mcp`. Then ask for a screenshot instead. Never build the screen from a guess at what the link shows.
- If only a screenshot was given: read it directly; there are no Figma variables to diff, so Step 4 relies on visual/measured comparison instead.

Keep the design reference per the project's `## Reference Docs` naming rules:
- A screenshot that is a file on disk → copy it to `docs/ui/<feature>/<screen>.png|jpg` (kebab-case folder, snake_case file).
- A screenshot pasted into the conversation can't be written to disk. Ask the person to save it there.
- A Figma link → add it as a line to `docs/ui/<feature>/figma_links.md` (`<screen>: <link>`).

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

Decide where the screen goes before generating any file:

- **Not a new screen.** If the design is a secondary capability of an existing screen on the same data and UI area (search, filter, sort), the template says to merge it into that screen's state and ViewModel. Say so and confirm before doing that instead.
- **Feature exists, with 2+ screens.** Generate into `presentation/<screenName>/`. Reuse the feature's existing data/domain/di code — don't duplicate models, DTOs or repositories.
- **Feature exists, with 1 screen (flat `presentation/`).** Follow "Growing from one screen to two" in `feature-conventions.md`: show the move plan, get confirmation, move the existing screen, then generate the new one in `presentation/<screenName>/`.
- **Feature exists, with no screens yet.** Generate flat into `presentation/`.
- **Feature doesn't exist.** Always create `presentation/` (flat) and `di/` together. The ViewModel can't resolve through Koin without its module. If the screen needs data (API, mock or local), ask before generating anything:
  1. **Full feature now** (recommended): also create the data/domain layers the template lists for this data source.
  2. **Presentation only**: the ViewModel has no data dependencies yet. It starts from an empty state with a `// TODO: connect <data source>` and no mock data (mocks belong in the data layer, Step 6).

## Step 6 — Generate the screen

Follow the feature template's per-layer requirements exactly for whichever layers apply (state holder, ViewModel, Screen, `di/`, and data/domain if they're being created). In every architecture:
- The Screen collects state with `collectAsStateWithLifecycle()`, holds no business logic, and has a `@Preview` with representative sample state
- **Tests:** generate the tests the template's "Forms, pickers, dialogs and tests" section lists for the layers created here: always the ViewModel test, plus the mapper test if a mapper was created. For a presentation-only ViewModel, test its initial state and one user action
- **Real API:** the endpoint goes into the feature's ApiService and Repository exactly as the template describes, and is added to `docs/api/README.md` (status 🟡 In progress, or 🟢 Integrated once it works)
- **Mock:** keep the mock inside the data layer — the Repository returns sample data, marked `// TODO(mock): replace with <endpoint>` — so switching to the real endpoint later touches only the data layer, never the ViewModel, UseCase or Screen. Add the endpoint to `docs/api/README.md` as ⚪ Not started

## Step 7 — Strings in every locale

For every genuinely new string (it survived the Step 4 reuse check): add it to `values/strings.xml` **and** every other `values-<locale>/strings.xml` the project has, in this same step, with natural phrasing in each language — never a placeholder, never a literal word-for-word translation.

## Step 8 — Wire navigation and DI

- Add the route to `navigation/Routes.kt` and register the composable in `navigation/AppNavGraph.kt`, per the feature template's "Navigation wiring" section.
- If this command created the feature's Koin module, register it in `App.kt`, per the template's `di/` section (append to the existing module list).

These three files, plus the resource entries from Steps 4 and 7, are the only shared files this command touches.

## Step 9 — Verify and report

- Run `./gradlew ktlintCheck testDebugUnitTest assembleDebug` — the screen compiles, its tests pass, and ktlint is clean
- Confirm no rule in `CLAUDE.md`'s Hard Rules sections was violated
- Report back:
  - Feature + screen name, files created (tests included), and any files moved (Step 5)
  - **Resources reused** (existing color/string/dimen/shape/typography/component/model it matched onto) vs. **resources newly added** (with justification for why nothing existing matched)
  - Route(s) added, and whether `App.kt` was touched
  - Which Step 5 path was taken (existing feature, new full feature, or new presentation-only)
  - Where the design reference was saved under `docs/ui/`, or that the person still needs to save it
