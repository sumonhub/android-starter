---
description: Generate a single Compose screen from a Figma design or screenshot, following the project's chosen architecture, reusing existing colors/strings/dimens/shapes/typography/components before adding anything new
---

# /generate-screen

You are generating a **single screen** inside `feature/<feature_name>/`. A screen is a state holder + ViewModel + Screen, plus data/domain wiring if it needs data. This command is narrower than `/generate-feature`:
- It starts from a Figma design, not from a feature description.
- It prefers to *reuse* what already exists in the codebase before it creates anything new.

This command decides the order of work. The feature template decides what the code looks like. Never restate or reinterpret a template rule here. Follow the template.

## Rules
- If anything is unclear, ask before you generate anything. Never guess and continue.
- Read `CLAUDE.md` at the project root first. Its `## Architecture` section tells you which pattern the project uses. Everything you generate here must follow its `## Hard Rules — Never Violate`, `## Architecture Hard Rules` (if present), `## Constraints`, and `## Code Style` sections.

## Step 0 — Check the project is initialized

Use the same check as `/init-project`'s Step 0: `CLAUDE.md` exists at the root with an `## Architecture` section, and `core/` exists under the app module. If either is missing, stop. Tell the person that the shared foundation isn't scaffolded yet and that `/init-project` must run first. Never generate a screen without it. Never pick an architecture or a folder yourself.

## Step 1 — Intake

Ask for anything the person didn't already give in their message:
1. A Figma link, or a screenshot of the screen
2. The feature name + screen name (e.g. `feature: auth`, `screen: signIn`). Derive every name as described in the "Naming" section of `.claude/templates/shared/feature-conventions.md`
3. What data the screen needs:
   - **API** — the real endpoint (check `docs/api/README.md` first), or "mock" if the backend isn't ready yet
   - **Local only** (settings, flags, drafts) or **no data** (static screen) — follow "Features without an API" in `feature-conventions.md`

Don't go past this step with guesses. A wrong feature/screen name or a wrong endpoint is expensive to undo later.

## Step 2 — Load the matching feature template

Check `CLAUDE.md`'s `## Architecture` section. Read the matching file below fully, plus `.claude/templates/shared/feature-conventions.md`. Every later step follows their requirements:

| Architecture | Template |
|---|---|
| MVI + Clean | `.claude/templates/feature/mvi-clean.md` |
| MVI | `.claude/templates/feature/mvi.md` |
| MVVM + Clean | `.claude/templates/feature/mvvm-clean.md` |
| MVVM | `.claude/templates/feature/mvvm.md` |

If `CLAUDE.md` names an architecture that is not in this table, or the template file is missing, say so and stop.

## Step 3 — Read the design

- **If the person gave a Figma link:** first invoke the `figma-design-to-code` skill. This is required before you call `get_design_context`. Then call `get_design_context` and `get_variable_defs`. The variables matter here. Step 4 compares them with the project's own tokens, so a visual read of a screenshot is not enough.
- **If the Figma tools aren't available or the Figma server isn't authorized:** say so. The person must connect it in the claude.ai connector settings, or with `/mcp`. Then ask for a screenshot instead. Never build the screen from a guess about what the link shows.
- **If the person gave only a screenshot:** read it directly. There are no Figma variables to compare, so Step 4 uses visual/measured comparison instead.

Save the design reference as the project's `## Reference Docs` naming rules describe:
- A screenshot that is a file on disk → copy it to `docs/ui/<feature>/<screen>.png|jpg` (kebab-case folder, snake_case file).
- A screenshot pasted into the conversation can't be written to disk. Ask the person to save it there.
- A Figma link → add it as a line to `docs/ui/<feature>/figma_links.md` (`<screen>: <link>`).

## Step 4 — Resource-reuse gate (before creating anything new)

Check each resource below, in order. Create a new entry only after you confirm that nothing existing is an exact or close match. Don't create first and remove duplicates later.

| Resource | Check here first |
|---|---|
| Color | `core/ui/theme/Color.kt` tokens → `MaterialTheme.colorScheme.*` roles |
| String | `values/strings.xml`. Search by **key and by value text**. For example, an existing "Try Again" should stop you from adding a new "Retry" |
| Dimen | `core/ui/theme/Dimens.kt`. If the design uses a value close to an existing token (e.g. 14dp vs. an existing 12dp/16dp), ask whether to snap to the existing token or add a new one. Don't add one silently |
| Shape | `core/ui/theme/Shape.kt`, before you define a new corner radius/shape |
| Typography | `core/ui/theme/Typography.kt`. This is **mandatory**: map every text in the design to an existing `MaterialTheme.typography.*` slot, as the typography Hard Rule in `CLAUDE.md` requires. If a text in the design doesn't match any slot, stop and ask |
| Components | Check `core/ui/components/` first (`AppScreen`, `AppTopBar`, `AppButton`, `AppChip`, `Avatar`, `LoadingIndicator`, `ErrorView`, `EmptyView`, plus anything added since). Then check the feature's own `presentation/components/`. The body always composes inside `AppScreen`. Before you generate a new button/input/chip/card, confirm that nothing there already matches. If only a variant is missing, add it as a new param on the core component (ask first) |
| Model / DTO | Check whether this screen's data already exists as a model anywhere. Look in this feature's own model and DTO folders first. Then look at other features' models for the same real-world entity. Never import another feature's model directly. Use it as the feature template's "Cross-feature consumption pattern" describes |

Report what you reused vs. newly added as part of Step 9's output. Don't just claim that it happened.

## Step 5 — Locate or scaffold the feature

Decide where the screen goes before you generate any file:

- **Not a new screen.** The design may be a secondary capability of an existing screen on the same data and UI area (search, filter, sort). In that case, the template says to merge it into that screen's state and ViewModel. Say so, and confirm with the person before you do that instead.
- **Feature exists, with 2+ screens.** Generate into `presentation/<screenName>/`. Reuse the feature's existing data/domain/di code. Don't duplicate models, DTOs or repositories.
- **Feature exists, with 1 screen (flat `presentation/`).** Follow "Growing from one screen to two" in `feature-conventions.md`:
  1. Show the move plan.
  2. Get confirmation.
  3. Move the existing screen.
  4. Generate the new one in `presentation/<screenName>/`.
- **Feature exists, with no screens yet.** Generate flat into `presentation/`.
- **Feature doesn't exist.** Always create `presentation/` (flat) and `di/` together, because the ViewModel can't resolve through Koin without its module. If the screen needs data (API, mock or local), ask before you generate anything:
  1. **Full feature now** (recommended): also create the data/domain layers that the template lists for this data source.
  2. **Presentation only**: the ViewModel has no data dependencies yet. It starts from an empty state with a `// TODO: connect <data source>`. It has no mock data, because mocks belong in the data layer (Step 6).

## Step 6 — Generate the screen

Follow the feature template's per-layer requirements exactly for every layer that applies: state holder, ViewModel, Screen, `di/`, and data/domain if you are creating them. In every architecture:
- The Screen collects state with `collectAsStateWithLifecycle()`. It holds no business logic. It has a `@Preview` with representative sample state
- **Tests:** generate the tests that the template's "Forms, pickers, dialogs and tests" section lists for the layers you create here. Always generate the ViewModel test. Also generate the mapper test if you created a mapper. For a presentation-only ViewModel, test its initial state and one user action
- **Real API:** put the endpoint into the feature's ApiService and Repository exactly as the template describes. Add it to `docs/api/README.md` (status 🟡 In progress, or 🟢 Integrated once it works)
- **Mock:** keep the mock inside the data layer. The Repository returns sample data, marked `// TODO(mock): replace with <endpoint>`. Then, when you switch to the real endpoint later, you change only the data layer, never the ViewModel, UseCase or Screen. Add the endpoint to `docs/api/README.md` as ⚪ Not started

## Step 7 — Strings in every locale

This step covers every string that is really new (it passed the Step 4 reuse check). In this same step, add it to `values/strings.xml` **and** to every other `values-<locale>/strings.xml` in the project. Use natural wording in each language. Never use a placeholder. Never use a literal word-for-word translation.

## Step 8 — Wire navigation and DI

- Add the route to `navigation/Routes.kt`. Register the composable in `navigation/AppNavGraph.kt`. Follow the feature template's "Navigation wiring" section.
- If this command created the feature's Koin module, register it in `App.kt`, as the template's `di/` section describes. Append it to the existing module list.

This command touches only these three shared files, plus the resource entries from Steps 4 and 7.

## Step 9 — Verify and report

- Run `./gradlew ktlintCheck testDebugUnitTest assembleDebug`. The screen must compile, its tests must pass, and ktlint must be clean
- Confirm that no rule in `CLAUDE.md`'s Hard Rules sections was broken
- Report back:
  - The feature + screen name, the files created (tests included), and any files moved (Step 5)
  - **Resources reused** (the existing color/string/dimen/shape/typography/component/model it matched) vs. **resources newly added** (with a reason why nothing existing matched)
  - The route(s) added, and whether `App.kt` was touched
  - Which Step 5 path you took (existing feature, new full feature, or new presentation-only)
  - Where the design reference was saved under `docs/ui/`, or that the person still needs to save it
