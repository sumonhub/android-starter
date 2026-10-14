---
description: Bootstrap a brand-new project's shared architecture foundation (infra only, no feature code)
---

# /init-project

You are scaffolding a **new project's shared foundation**. No feature code — infra, DI, theme, base classes, nav shell only.

## Rules
- When anything is unclear — ask before generating. This applies to every step below: the architecture choice, anything not covered by the template's defaults, and anything else that comes up mid-scaffold. Never guess and proceed.

## Step 0 — Check for an existing scaffold

Before asking anything else, check whether this project has already been initialized:
- Does `CLAUDE.md` exist at the root with an `## Architecture` section?
- Does `core/` already exist under the app module?

If neither exists, skip straight to Step 1.

If either exists, stop and tell the person what's already there (including which architecture `CLAUDE.md` currently records, if any), then ask explicitly what they want:
1. **Skip** — project is already initialized, do nothing further
2. **Re-run the same architecture** — re-applies the template to `core/`/`navigation/` per Step 4, and updates `CLAUDE.md` per the surgical (not whole-file) procedure in Step 5; warn that this may overwrite any hand-edited files under `core/` or `navigation/` before doing it. Step 7's Alignment Check always runs afterward.
3. **Switch to a different architecture** — treat this as destructive: the existing `core/` files were built for the old pattern and won't automatically match the new one. Require an explicit confirmation naming the switch (e.g. "yes, switch from MVVM to MVI + Clean") before touching anything

Regardless of which option is chosen, never touch anything under `feature/` — existing features are always preserved.

## Step 1 — Ask which architecture pattern

Ask the user, in a single message with clear options, which pattern they want:

1. MVVM — *outline only*
2. MVVM + Clean Architecture — *outline only*
3. MVI — *outline only*
4. MVI + Clean Architecture — **fully supported (recommended)**

"Outline only" templates list the structure, dependencies and base classes but not full file contents for every core file (a matching feature template exists for all four) — say this when presenting the options, so the person knows more will be inferred if they pick one.

Wait for their answer. Do not proceed on a guess — if their answer is ambiguous, ask again.

## Step 2 — Load the matching template

Once they answer, read the matching file fully before doing anything else:

| Answer | Template |
|---|---|
| MVVM | `.claude/templates/architecture/mvvm.md` |
| MVVM + Clean | `.claude/templates/architecture/mvvm-clean.md` |
| MVI | `.claude/templates/architecture/mvi.md` |
| MVI + Clean | `.claude/templates/architecture/mvi-clean.md` |

## Step 3 — Apply the template's defaults

Each template has a "## Defaults" section (base URL placeholder, local storage choice, JSON parser). Apply these directly — do not ask about any of them. Only deviate if the person's own request already specified a different value (e.g. they named a real base URL up front).

## Step 4 — Generate the skeleton

Follow the template's folder structure and file contents exactly.

- Only create what the template lists. No feature folders, no business logic, no sample screens beyond the placeholder/blank screen the template asks for.
- If Android Studio's default Compose theme exists (`ui/theme/` with `Color.kt`, `Type.kt`, `Theme.kt`), move its contents into `core/ui/theme/` per the template, rename the theme function to the template's `AppTheme` (if it isn't already), and delete the old `ui/theme/` and the default `Greeting` composable — never leave two themes side by side.
- Every core/shared file must stay feature-agnostic — no imports of anything feature-specific.
- Use the template's defaults from Step 3 to fill in the base URL placeholder, JSON parser, and local storage.
- Regardless of architecture, create `core/ui/components/` from `.claude/templates/shared/ui-components.md` (every component, plus the converter for the chosen architecture), along with the Dimens tokens and string resources it lists.
- Regardless of architecture, apply `.claude/templates/shared/project-setup.md` (environments, signing, R8 keep rules, splash screen, unit-test setup, ktlint, CI workflow) and create every file in `.claude/templates/shared/core-utils.md` (`Validator`, `InputLimits`, `DateTimeFormat`, `NavResult`).
- Regardless of architecture, also create the reference docs from `.claude/templates/shared/reference-docs.md` at the repo root:
  - `docs/api/README.md`, `docs/oq/README.md`, `docs/feedback/README.md` and `docs/CHANGELOG.md` — each from its starter content, copied verbatim
  - `docs/ui/.gitkeep` — starts empty
  - the `updateChangelog` Gradle task, added to the app module's `build.gradle.kts`

## Step 5 — Write / update CLAUDE.md at the project root

### First run (`CLAUDE.md` doesn't exist yet)

Create `CLAUDE.md` at the repo root with these sections, in this order:

1. **`## Architecture`** — chosen pattern + full tech stack (from the template header)
2. **`## Defaults`** — the template's full "Defaults" section, copied verbatim. Keeping these visible in `CLAUDE.md` is what lets a future session know these choices were deliberate and already fixed, instead of re-deriving or re-asking about them.
3. **`## Project Structure`** — the template's full "Project Structure" section, copied verbatim, so a future session can see the shape of the codebase without re-reading the template file.
4. **`## Constraints`** — the template's full "Constraints" section, copied verbatim. If the template has no `## Constraints` heading, stop and tell the person — never silently skip it.
5. Copy `.claude/templates/shared/hard-rules.md` — its `Hard Rules — Never Violate`, `Do Not Touch` and `Security` blocks, verbatim, in that order — always, regardless of which architecture was chosen. These are session-standing rules for every future coding session on this project, not just this scaffold step.
   Then, if the template has a **"## CLAUDE.md content for this architecture"** section, copy its `## Architecture Hard Rules` block verbatim right after them. If the section is missing, tell the person explicitly that the chosen template ships no architecture-specific rules (only the generic ones apply), rather than skipping it silently. Match headings exactly — never guess at a similarly-named section.
6. Copy `.claude/templates/shared/coding-standards.md` in full (Code Style, Error Handling, Logging) — always, regardless of which architecture was chosen.
7. Copy `.claude/templates/shared/git-workflow.md` in full (Commits, Pull Requests, Claude's own behavior) — always, regardless of which architecture was chosen.
8. Copy `.claude/templates/shared/response-format.md` in full (Issue/Root Cause/Action Plan before changes, What/Why Changed/Issue Resolved after) — always, regardless of which architecture was chosen.
9. Copy the `## Reference Docs` section of `.claude/templates/shared/reference-docs.md` — its doc list plus the `Trackers` and `File naming (best practice)` subsections — always, regardless of which architecture was chosen. The starter-content blocks and the Gradle task are for Step 4 only; never copy them into `CLAUDE.md`.
10. **`## Feature Generation`** — a pointer: "New features are added with the feature-generation command/skill, following the pattern already scaffolded under `feature/`."

### Re-run (`CLAUDE.md` already exists — "Re-run the same architecture" from Step 0)

Do **not** regenerate the whole file and do **not** blindly overwrite section-by-section. A project's `CLAUDE.md` accumulates legitimate, deliberate customizations that aren't in the raw template (e.g. a documented legacy-code carve-out, a project-specific file-split convention, a moved file location) — a blind rewrite silently destroys those. Instead, split the sections into two groups and handle each differently:

**A. Verbatim sections** — meant to be an exact copy of the template/shared files, never project-specific: `Hard Rules — Never Violate`, `Do Not Touch`, `Security`, `Architecture Hard Rules`, `Code Style`, `Error Handling`, `Logging`, `Commits`, `Pull Requests`, `Claude's own behavior`, `Response Format`, `Reference Docs`, `Feature Generation`.
- Diff each section's current text against the template/shared-file's canonical text.
- Identical → leave untouched.
- Missing entirely → add it verbatim.
- Present but different → **do not silently overwrite**. Show the person the diff and ask whether to keep the project's edited version or reset it to the template's canonical text.

**B. Project-state sections** — meant to reflect this project's actual current reality, not a static template copy: `Architecture`, `Defaults`, `Project Structure`.
- Missing entirely → add from the template as a starting point.
- Already present → do **not** blindly overwrite with the generic template text (this would revert a real `BASE_URL` back to the placeholder, or replace an already-customized file tree with the generic `featureX` skeleton). Leave these as-is; keeping them accurate is Step 7's job, not this step's.

Regardless of first-run or re-run, never touch anything under `feature/`, and never disturb unrelated content already in the file that falls outside the sections above.

## Step 6 — Verify

- Run `./gradlew ktlintCheck testDebugUnitTest assembleDebug` and fix anything that fails before reporting — never report success without a green run.
- Report back a short summary: architecture chosen, files created, and where CLAUDE.md now lives.

## Step 7 — Alignment Check (runs after every scaffold or re-run)

Compare the actual project against the chosen template's Project Structure, Marker Reference, and Dependency Direction sections, and report drift — this step never edits code, only reports.

Check at least:
- Do `core/domain/Mapper.kt` and `core/network/NetworkResult.kt` (or equivalents) actually get used anywhere outside their own definition? (grep for each type name across the codebase — if the only match is the definition file itself, it's defined but unused)
- Does `core/ui/theme/` have a spacing/sizing token file (e.g. `Dimens.kt`)?
- Does `core/model/` exist if 2+ features share a model, or are features instead importing a shared model from somewhere ad hoc (a legacy package, a random feature's `data/model/`)?
- Does each feature's `data/` and `domain/` follow the template's subfolder split (`data/{remote,local,mapper,repository}`, `domain/{model,repository,usecase}`), or is either flat?
- Does every feature domain layer own its own model, or does it re-export a DTO/legacy model directly as if it were the domain model?
- Does anything under `core/` import a feature-specific or legacy-only symbol (violates feature-agnostic)?
- Is the component-placement Level 1/2/3 rule actually followed (anything duplicated across features that should have been promoted to `core/ui/components/`; anything feature-wide sitting inline in a single screen that should have been promoted to `feature/x/presentation/components/`)?

Report the result as a concrete, file-referenced list split into two groups — Matches and Deviations — the same shape as a manual architecture-compliance review. **Never auto-fix anything found here.** End by asking the person explicitly what they want to do next: fix specific items now, note them for a tracked follow-up, or leave as-is for this run.
