# Shared Reference Docs (all architectures)

> Applies regardless of which architecture pattern was chosen. `/init-project` writes the `## Reference Docs`
> section below (with its subsections) into every project's `CLAUDE.md`, and creates the matching folders and
> starter files from the starter-content blocks further down (those blocks are never copied into `CLAUDE.md`).

## Reference Docs
- `docs/api/` — API tracker (`README.md`: every endpoint the app uses, its integration status, and backend issues) + raw backend API docs beside it
- `docs/ui/` — UI references: design screenshots / exported frames / Figma links, one subfolder per feature (`docs/ui/<feature>/`)
- `docs/oq/` — open-questions tracker (`README.md`): pending product/architecture decisions with context, options and trade-offs. Check here before implementing anything it covers
- `docs/feedback/` — QA feedback tracker (`README.md`) + any screenshots QA attaches
- `docs/CHANGELOG.md` — per-build change log for QA; cut a release with `./gradlew updateChangelog` (renames `[Unreleased]` to the current `versionName` + date)

### Trackers (`docs/api/`, `docs/oq/`, `docs/feedback/`)
Each tracker's `README.md` has the same shape: legend + summary counts, then the **list** (one table row per item), then the **details** (one section per item).
- **Two-way links:** every list row's ID links down to its details (`[OQ-3](#oq-3)`); every details section starts with `<a id="oq-3"></a>` and ends with `[↑ Back to list](#list)`.
- **Status:** use only that file's legend emoji. When a status changes, update the list row, the details section and the summary counts together.
- **IDs:** `API-n`, `OQ-n`, `FB-n` — sequential, never reused. Append new items at the end of both the list and the details.
- **When to write:** do not just answer in chat —
  - new endpoint shared, endpoint integrated, or "log this for backend" / "add an API issue" → `docs/api/README.md` (backend problems get 🔴 `Blocked` and the issue in the details)
  - "park this question" / "add an open question" → `docs/oq/README.md`
  - QA feedback given → `docs/feedback/README.md`, one item per distinct problem
- Each entry is specific: name the exact screen / feature / endpoint so the reader can act without a follow-up question. If unclear which tracker or feature an item belongs to, ask rather than guessing.

### File naming (best practice)
- **Folders** — lowercase, one word or kebab-case: `api/`, `ui/`, `oq/`, `feedback/`, `ui/work-dashboard/`
- **Trackers** — `README.md` inside each tracker folder, so GitHub shows it when the folder is opened
- **Top-level living docs** — `UPPER_SNAKE_CASE.md` (`CHANGELOG.md`)
- **Files inside folders** — lowercase `snake_case`, named by subject, not by date or person:
  - `docs/api/<area>_api_reference.md` (+ the raw `<area>_api_reference.json` beside it when the backend shares one), e.g. `work_api_reference.md`
  - `docs/ui/<feature>/<screen>.png|jpg`, e.g. `docs/ui/rating/rating_details.jpg`; numbered variants as `dashboard_1.jpg`, `dashboard_2.jpg`
  - `docs/feedback/<id>_<topic>.png|jpg` for QA attachments, e.g. `fb_3_login_crash.png`
- Never spaces, uppercase letters or version suffixes like `_final` / `_v2` / `(1)` in a file name — overwrite and let git keep history
- Never commit real tokens, passwords or internal server paths — strip them from shared API docs / Postman exports first

### `docs/api/README.md` starter content

```markdown
# 🔌 API

> **Source:** _where the backend docs come from_ · **Snapshot:** _YYYY-MM-DD_ — the live source may move on without this copy.

**Legend:** ⚪ Not started · 🟡 In progress · 🟢 Integrated · 🔴 Blocked (backend issue)
**Summary:** ⚪ 0 · 🟡 0 · 🟢 0 · 🔴 0

<a id="list"></a>
## 📋 List

| ID | Endpoint | Feature | Status |
|---|---|---|---|

---

## 📝 Details

<!-- One section per endpoint, in this shape:

<a id="api-1"></a>
### API-1 · `POST /auth/login`
**Status:** ⚪ Not started · **Feature:** auth · **Screen:** Sign in

- **Request:** `{ email, password }`
- **Response:** `{ token, user }`
- **Reference:** [auth_api_reference.md](auth_api_reference.md)
- **Issue:** _only when 🔴 Blocked — what is wrong and what the app expects_

[↑ Back to list](#list)
-->
```

### `docs/oq/README.md` starter content

```markdown
# ❓ Open Questions

> Pending product/architecture decisions — recorded so the context isn't lost while they're being decided.

**Legend:** 🟡 Open · 🟢 Decided · ⚪ Dropped
**Summary:** 🟡 0 · 🟢 0 · ⚪ 0

<a id="list"></a>
## 📋 List

| ID | Question | Area | Raised | Status |
|---|---|---|---|---|

---

## 📝 Details

<!-- One section per question, in this shape:

<a id="oq-1"></a>
### OQ-1 · Should the session survive app restarts?
**Status:** 🟡 Open · **Area:** auth · **Raised:** YYYY-MM-DD

**Context** — why this came up, what it affects.

| | A. … | B. … |
|---|---|---|
| How | | |
| Trade-off | | |

**Recommendation** — which option and why (or "needs product input").
**Decision** — _when 🟢 Decided: the decision, date, and the PR that implements it._

[↑ Back to list](#list)
-->
```

### `docs/feedback/README.md` starter content

```markdown
# 🧪 QA Feedback

> Feedback from QA on app builds. One item per distinct problem.

**Legend:** 🔴 Open · 🟡 In progress · 🟢 Fixed · ⚪ Won't fix
**Summary:** 🔴 0 · 🟡 0 · 🟢 0 · ⚪ 0

<a id="list"></a>
## 📋 List

| ID | Feedback | Screen | Build | Status |
|---|---|---|---|---|

---

## 📝 Details

<!-- One section per item, in this shape:

<a id="fb-1"></a>
### FB-1 · Login button stays disabled after valid input
**Status:** 🔴 Open · **Screen:** Sign in · **Build:** 1.0 · **Reported:** YYYY-MM-DD

- **Steps:** 1. … 2. …
- **Expected:** …
- **Actual:** …
- **Attachment:** [fb_1_login_button.png](fb_1_login_button.png)
- **Fix:** _when 🟢 Fixed: what changed and in which build._

[↑ Back to list](#list)
-->
```

### `docs/CHANGELOG.md` starter content

```markdown
# Changelog

## [Unreleased]
```

### `updateChangelog` Gradle task

Add to the app module's `build.gradle.kts`, after the `dependencies {}` block:

```kotlin
// Renames docs/CHANGELOG.md's [Unreleased] section to the current versionName + date, and opens
// a fresh empty [Unreleased] section above it. Run before cutting a release build:
//   ./gradlew updateChangelog
tasks.register("updateChangelog") {
    group = "release"
    description = "Cuts docs/CHANGELOG.md's [Unreleased] section into a dated release entry."
    val changelogFile = rootProject.file("docs/CHANGELOG.md")
    val version = android.defaultConfig.versionName
    doLast {
        check(changelogFile.exists()) { "docs/CHANGELOG.md not found at $changelogFile" }
        val content = changelogFile.readText()
        check("## [Unreleased]" in content) { "docs/CHANGELOG.md has no '## [Unreleased]' section to cut." }
        check("## [$version]" !in content) {
            "docs/CHANGELOG.md already has an entry for version $version — bump versionName in app/build.gradle.kts first."
        }
        val today = java.time.LocalDate.now().toString()
        changelogFile.writeText(content.replaceFirst("## [Unreleased]", "## [Unreleased]\n\n## [$version] - $today"))
    }
}
```
