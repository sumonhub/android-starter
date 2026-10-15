---
description: Bootstrap a brand-new project's shared architecture foundation (infra only, no feature code)
disable-model-invocation: true
---

# /android-starter:init-project

> **Plugin files:** this skill's templates live in `${CLAUDE_PLUGIN_ROOT}/templates/`. When a template mentions a path that starts with `templates/`, it means `${CLAUDE_PLUGIN_ROOT}/templates/`.

You are scaffolding a **new project's shared foundation**. Do not write feature code. Create only infra, DI, theme, base classes and the nav shell.

## Rules
- If anything is unclear, ask before you generate anything. Never guess and continue. This rule applies to every step below: the architecture choice, anything the template's defaults don't cover, and anything else that comes up during the scaffold.

## Preflight — Check prerequisites (runs before Step 0)

Before you ask anything, check that the project is ready for the scaffold. Run every check below. Then report all the results at once. Don't stop at the first failure. This way the person can fix everything in one pass.

### Blockers: stop if any of these fail

| Check | How to detect | How to fix (tell the person) |
|---|---|---|
| Gradle Kotlin DSL | `settings.gradle.kts` is at the root and `build.gradle.kts` is in the app module (not Groovy `.gradle` files) | Create a new project in Android Studio (Kotlin DSL is the default). Or convert the build files to `.kts` |
| Version catalog | `gradle/libs.versions.toml` exists | Android Studio → **Refactor → Migrate to Version Catalog**. Or create a new project (new projects include one) |
| AGP 9+ | `agp` version in `libs.versions.toml` | Android Studio → **Tools → AGP Upgrade Assistant**. Upgrade to 9.x |
| Kotlin 2.0+ | `kotlin` version in `libs.versions.toml`. Compose needs the `org.jetbrains.kotlin.plugin.compose` plugin, and this plugin exists only from Kotlin 2.0 | Change `kotlin` in `libs.versions.toml` to 2.x and sync |
| Gradle wrapper + JDK 17+ | `gradlew` / `gradlew.bat` exist. `./gradlew --version` succeeds and shows JVM 17+. Step 6 depends on this | Android Studio → **Settings → Build, Execution, Deployment → Build Tools → Gradle → Gradle JDK**: pick 17+. Set `JAVA_HOME` to the same JDK for the terminal |
| Not an XML / View-based UI | See "XML / View-based projects" below | See below |

### Handled by the scaffold: report only, don't stop

- **Compose not set up yet**: the Compose BOM/material3 is missing, the `kotlin-compose` plugin is missing, or there is no `buildFeatures { compose = true }`. Step 4 adds all of these from the template's Gradle section. Say "will be added".
- **minSdk below 26**: `java.time` in `core-utils.md` needs core library desugaring. Step 4 turns it on. Say "desugaring will be enabled".

### XML / View-based projects

Treat the project as View-based if any of these exist: layout files under `res/layout/`, `setContentView(R.layout.…)`, `Fragment` subclasses, or `viewBinding`/`dataBinding = true`.

Every template is Compose-only. Step 4 rewrites `MainActivity` to `setContent { AppTheme { AppNavGraph() } }`. `/android-starter:generate-screen` and `/android-starter:generate-feature` only produce Compose. If you run the scaffold on a View-based app, its existing screens lose their connection to the entry point. The app also ends up with two UI systems side by side. So:

- **Untouched "Empty Views Activity" template**: the only View code is the default `MainActivity` + `activity_main.xml`, and nobody added anything by hand. Explain this and offer to convert it to Compose:
  - replace `MainActivity` with the template's version
  - delete `activity_main.xml`
  - let Step 4 add Compose

  Continue only after an explicit "yes, convert to Compose".
- **Anything beyond that** (real XML screens, Fragments, bindings): stop. Explain that this kit doesn't support View-based or hybrid projects. Recommend that the person creates a new project with Android Studio's **Empty Activity** (Compose) template, then ports the code into features with `/android-starter:generate-feature`. Never scaffold on top of it.

### Report format

Show one checklist: ✅ passes, ⚠️ handled by the scaffold, ❌ blocker. Add the fix for each ❌.
- If there is any ❌, stop here. Ask the person to fix the blockers and run `/android-starter:init-project` again.
- Never fix a blocker yourself. Blockers touch the build setup and `libs.versions.toml`, and the hard rules protect those.
- If everything is ✅/⚠️, continue to Step 0.

## Step 0 — Check for an existing scaffold

Before you ask anything else, check whether this project is already initialized:
- Does `CLAUDE.md` exist at the root with an `## Architecture` section?
- Does `core/` already exist under the app module?

If neither exists, go straight to Step 1.

If either exists, stop. Tell the person what is already there. Include the architecture that `CLAUDE.md` records now, if any. Then ask them clearly which option they want:
1. **Skip** — the project is already initialized. Do nothing more.
2. **Re-run the same architecture** — apply the template to `core/`/`navigation/` again, as in Step 4. Update `CLAUDE.md` with the surgical (not whole-file) procedure in Step 5. Before you do this, warn the person that it may overwrite hand-edited files under `core/` or `navigation/`. Step 7's Alignment Check always runs afterward.
3. **Switch to a different architecture** — this is destructive. Follow "Switching architecture" below exactly.

Whichever option the person chooses, never touch anything under `feature/`. Existing features are always kept.

### Switching architecture

Run these steps in order. If any step stops, nothing has changed yet.

1. **Block if features exist.** List the folders under `feature/`. Ignore `.gitkeep`. If there are any folders, stop and explain why:
   - Those features use the current pattern's base classes. Switching `core/` would break the build (Step 6 can't go green).
   - This command never edits `feature/`.

   The person can keep the current architecture. Or they can remove or migrate those features themselves and run `/android-starter:init-project` again. Never offer to migrate them as part of this command.
2. **Ask which architecture** (Step 1). Then read **both** templates fully: the old one (named in `CLAUDE.md`'s `## Architecture`) and the new one.
3. **Build the change plan** from the Project Structure sections of the two templates:
   - **Delete:** files under `core/` and `navigation/` that the old template lists and the new one doesn't (e.g. MVI-only base classes)
   - **Replace:** files that both templates list, but with different contents (e.g. the base ViewModel, `MainActivity`, the error → snackbar converter in `AppSnackbar.kt`)
   - **Add:** files that only the new template lists
   - **Keep, untouched:** files under `core/` or `navigation/` that neither template lists (someone added them by hand). List them, so the person knows they may no longer fit the new pattern. Let Step 7 flag them
   - **Keep, untouched:** everything already in place that doesn't depend on the architecture: `project-setup.md` output (build types, signing, keep rules, splash, ktlint, CI), `core-utils.md` files, `core/ui/components/` (except the converter), `core/ui/theme/`, `docs/`, and any real `BASE_URL` values
   - **Gradle:** if the new template needs dependencies that are missing, add them. For dependencies that only the old template used, list them and ask whether to remove them. Never remove one silently
4. **Show the plan and confirm.** Show the full plan as file lists. Include the `CLAUDE.md` changes from Step 5's "Switch" procedure. Continue only after an explicit confirmation that names the switch, e.g. "yes, switch from MVI + Clean to MVVM". Any other answer, including a plain "yes", means you ask again.
5. **Apply** the plan. Then continue with Step 5 ("Switch"), Step 6 and Step 7.

## Step 1 — Ask which architecture pattern

Ask the user which pattern they want. Use a single message with clear options:

1. MVVM — *outline only*
2. MVVM + Clean Architecture — *outline only*
3. MVI — *outline only*
4. MVI + Clean Architecture — **fully supported (recommended)**

"Outline only" templates list the structure, dependencies and base classes. They don't give the full contents of every core file. (A matching feature template exists for all four.) Say this when you present the options, so the person knows that more will be inferred if they pick one.

Wait for their answer. Do not continue on a guess. If their answer is unclear, ask again.

## Step 2 — Load the matching template

When they answer, read the matching file fully before you do anything else:

| Answer | Template |
|---|---|
| MVVM | `${CLAUDE_PLUGIN_ROOT}/templates/architecture/mvvm.md` |
| MVVM + Clean | `${CLAUDE_PLUGIN_ROOT}/templates/architecture/mvvm-clean.md` |
| MVI | `${CLAUDE_PLUGIN_ROOT}/templates/architecture/mvi.md` |
| MVI + Clean | `${CLAUDE_PLUGIN_ROOT}/templates/architecture/mvi-clean.md` |

## Step 3 — Apply the template's defaults

Each template has a "## Defaults" section (base URL placeholder, local storage choice, JSON parser). Apply these directly. Do not ask about any of them. Use a different value only if the person's own request already gave one (e.g. they named a real base URL at the start).

## Step 4 — Generate the skeleton

Follow the template's folder structure and file contents exactly.

- Create only what the template lists. No feature folders, no business logic, and no sample screens except the placeholder/blank screen the template asks for.
- If Android Studio's default Compose theme exists (`ui/theme/` with `Color.kt`, `Type.kt`, `Theme.kt`):
  - move its contents into `core/ui/theme/`, as the template shows
  - rename the theme function to the template's `AppTheme` (if it isn't already)
  - delete the old `ui/theme/` and the default `Greeting` composable

  Never leave two themes side by side.
- Every core/shared file must stay feature-agnostic. It must not import anything feature-specific.
- Use the template's defaults from Step 3 for the base URL placeholder, JSON parser and local storage.
- For every architecture, create `core/ui/components/` from `${CLAUDE_PLUGIN_ROOT}/templates/shared/ui-components.md` (every component, plus the converter for the chosen architecture). Also add the Dimens tokens and string resources it lists.
- For every architecture, apply `${CLAUDE_PLUGIN_ROOT}/templates/shared/project-setup.md` (environments, signing, R8 keep rules, splash screen, unit-test setup, ktlint, CI workflow). Also create every file in `${CLAUDE_PLUGIN_ROOT}/templates/shared/core-utils.md` (`Validator`, `InputLimits`, `DateTimeFormat`, `NavResult`).
- For every architecture, also create the reference docs from `${CLAUDE_PLUGIN_ROOT}/templates/shared/reference-docs.md` at the repo root:
  - `docs/api/README.md`, `docs/oq/README.md`, `docs/feedback/README.md` and `docs/CHANGELOG.md` — create each one from its starter content, copied verbatim
  - `docs/ui/.gitkeep` — starts empty
  - the `updateChangelog` Gradle task — add it to the app module's `build.gradle.kts`

## Step 5 — Write / update CLAUDE.md at the project root

### First run (`CLAUDE.md` doesn't exist yet)

Create `CLAUDE.md` at the repo root with these sections, in this order:

1. **`## Architecture`** — the chosen pattern + the full tech stack (from the template header)
2. **`## Defaults`** — the template's full "Defaults" section, copied verbatim. These must stay visible in `CLAUDE.md`. Then a future session knows that these choices were deliberate and are already fixed. It won't work them out again or ask about them again.
3. **`## Project Structure`** — the template's full "Project Structure" section, copied verbatim. A future session can then see the shape of the codebase without reading the template file again.
4. **`## Constraints`** — the template's full "Constraints" section, copied verbatim. If the template has no `## Constraints` heading, stop and tell the person. Never skip it silently.
5. Copy `${CLAUDE_PLUGIN_ROOT}/templates/shared/hard-rules.md`: its `Hard Rules — Never Violate`, `Do Not Touch` and `Security` blocks, verbatim, in that order. Always do this, whatever architecture was chosen. These rules apply to every future coding session on this project, not just to this scaffold step.
   Then, if the template has a **"## CLAUDE.md content for this architecture"** section, copy its `## Architecture Hard Rules` block verbatim right after them. If that section is missing, don't skip it silently. Tell the person clearly that the chosen template has no architecture-specific rules, so only the generic ones apply. Match headings exactly. Never guess at a section with a similar name.
6. Copy `${CLAUDE_PLUGIN_ROOT}/templates/shared/coding-standards.md` in full (Code Style, Error Handling, Logging). Always do this, whatever architecture was chosen.
7. Copy `${CLAUDE_PLUGIN_ROOT}/templates/shared/git-workflow.md` in full (Commits, Pull Requests, Claude's own behavior). Always do this, whatever architecture was chosen.
8. Copy `${CLAUDE_PLUGIN_ROOT}/templates/shared/response-format.md` in full (Issue/Root Cause/Action Plan before changes, What/Why Changed/Issue Resolved after). Always do this, whatever architecture was chosen.
9. Copy the `## Reference Docs` section of `${CLAUDE_PLUGIN_ROOT}/templates/shared/reference-docs.md`: its doc list plus the `Trackers` and `File naming (best practice)` subsections. Always do this, whatever architecture was chosen. The starter-content blocks and the Gradle task are only for Step 4. Never copy them into `CLAUDE.md`.
10. **`## Feature Generation`** — a pointer: "New features are added with the feature-generation command/skill, following the pattern already scaffolded under `feature/`."

### Re-run (`CLAUDE.md` already exists — "Re-run the same architecture" from Step 0)

Do **not** regenerate the whole file. Do **not** blindly overwrite it section by section. Over time, a project's `CLAUDE.md` collects real, deliberate customizations that aren't in the raw template (e.g. a documented legacy-code carve-out, a project-specific file-split convention, a moved file location). A blind rewrite silently destroys them. Instead, split the sections into two groups and handle each group differently:

**A. Verbatim sections** — these must be an exact copy of the template/shared files. They are never project-specific: `Constraints`, `Hard Rules — Never Violate`, `Do Not Touch`, `Security`, `Architecture Hard Rules`, `Code Style`, `Error Handling`, `Logging`, `Commits`, `Pull Requests`, `Claude's own behavior`, `Response Format`, `Reference Docs`, `Feature Generation`.
- Diff each section's current text against the canonical text in the template/shared file.
- Identical → leave it untouched.
- Missing entirely → add it verbatim.
- Present but different → **do not silently overwrite**. Show the person the diff. Ask whether to keep the project's edited version or reset it to the template's canonical text.

**B. Project-state sections** — these must describe this project as it really is now. They are not a static copy of the template: `Architecture`, `Defaults`, `Project Structure`.
- Missing entirely → add it from the template as a starting point.
- Already present → do **not** blindly overwrite it with the generic template text. That would turn a real `BASE_URL` back into the placeholder, or replace a customized file tree with the generic `featureX` skeleton. Leave these sections as they are. Keeping them accurate is Step 7's job, not this step's.

### Switch (`CLAUDE.md` exists — "Switch to a different architecture" from Step 0)

The architecture-specific sections describe the old pattern, so they must change. Everything else follows the re-run rules above.

- **Replace with the new template's text:** `Architecture` (pattern + tech stack), `Defaults`, `Project Structure`, `Constraints`, `Architecture Hard Rules`. If the new template has no `## Architecture Hard Rules`, remove the old block and tell the person. Never leave the old pattern's rules in place.
- **Carry over project-specific facts** from the old sections, but only facts that are still true under the new pattern (e.g. a real base URL noted in `Defaults`, a documented legacy-code carve-out). List each one and ask before you keep it.
- **Shared sections** (`Hard Rules — Never Violate`, `Do Not Touch`, `Security`, `Code Style`, `Error Handling`, `Logging`, `Commits`, `Pull Requests`, `Claude's own behavior`, `Response Format`, `Reference Docs`, `Feature Generation`) don't depend on the architecture. Handle them exactly like group A in the re-run.
- The person already saw and confirmed these replacements in Step 0's switch plan. Don't ask again. Just apply them.

In every case (first run, re-run or switch), never touch anything under `feature/`. Never change unrelated content in the file that is outside the sections above.

## Step 6 — Verify

- Run `./gradlew ktlintCheck testDebugUnitTest assembleDebug`. Fix anything that fails before you report. Never report success without a green run.
- Report a short summary: the architecture chosen, the files created, and where CLAUDE.md is now.

## Step 7 — Alignment Check (runs after every scaffold or re-run)

Compare the actual project with the chosen template's Project Structure, Marker Reference and Dependency Direction sections. Report any drift. This step never edits code. It only reports.

Check at least:
- Are `core/domain/Mapper.kt` and `core/network/NetworkResult.kt` (or equivalents) used anywhere outside their own definition? Grep for each type name across the codebase. If the only match is the definition file itself, the type is defined but unused.
- Does `core/ui/theme/` have a spacing/sizing token file (e.g. `Dimens.kt`)?
- If 2+ features share a model, does `core/model/` exist? Or do features import a shared model from some ad hoc place instead (a legacy package, a random feature's `data/model/`)?
- Does each feature's `data/` and `domain/` follow the template's subfolder split (`data/{remote,local,mapper,repository}`, `domain/{model,repository,usecase}`)? Or is either one flat?
- Does every feature's domain layer own its own model? Or does it re-export a DTO/legacy model directly as if it were the domain model?
- Does anything under `core/` import a feature-specific or legacy-only symbol? (This breaks the feature-agnostic rule.)
- Is the component-placement Level 1/2/3 rule actually followed? Look for:
  - anything duplicated across features that should have been promoted to `core/ui/components/`
  - anything used across a whole feature that sits inline in a single screen and should have been promoted to `feature/x/presentation/components/`

Report the result as a concrete list with file references. Split it into two groups: Matches and Deviations. Use the same shape as a manual architecture-compliance review. **Never auto-fix anything found here.** At the end, ask the person clearly what they want to do next: fix specific items now, note them for a tracked follow-up, or leave them as they are for this run.
