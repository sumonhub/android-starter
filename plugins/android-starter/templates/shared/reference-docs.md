# Shared Reference Docs (all architectures)

> Applies to every project, whatever architecture pattern was chosen. `/android-starter:init-project` copies the `## Reference Docs`
> section below (with its subsections) into every project's `CLAUDE.md`. It also creates the matching folders and
> starter files from the starter-content blocks further down. Those blocks are never copied into `CLAUDE.md`.

## Reference Docs
- `docs/api/` — API tracker (`README.md`: every endpoint the app uses, its integration status, and backend issues). The raw backend API docs sit beside it
- `docs/ui/` — UI references: design screenshots / exported frames / Figma links. One subfolder per feature (`docs/ui/<feature>/`)
- `docs/oq/` — open-questions tracker (`README.md`): pending product/architecture decisions with context, options and trade-offs. Check here before you build anything it covers
- `docs/feedback/` — QA feedback tracker (`README.md`), plus any screenshots QA attaches
- `docs/CHANGELOG.md` — change log for QA, per build. Cut a release with `./gradlew updateChangelog`. It renames `[Unreleased]` to the current `versionName` + date

### Trackers (`docs/api/`, `docs/oq/`, `docs/feedback/`)
Each tracker's `README.md` has the same shape, in this order:
1. Legend and summary counts
2. The **list** (one table row per item)
3. The **details** (one section per item)

- **Two-way links:** the ID in every list row links down to its details (`[OQ-3](#oq-3)`). Every details section starts with `<a id="oq-3"></a>` and ends with `[↑ Back to list](#list)`.
- **Status:** use only the emoji from that file's legend. When a status changes, update the list row, the details section and the summary counts together.
- **IDs:** `API-n`, `OQ-n`, `FB-n`. They are sequential and never reused. Add new items at the end of both the list and the details.
- **When to write:** do not just answer in chat. Write to the tracker:
  - new endpoint shared, endpoint integrated, or "log this for backend" / "add an API issue" → `docs/api/README.md`. Backend problems get 🔴 `Blocked`, and the issue goes in the details
  - "park this question" / "add an open question" → `docs/oq/README.md`
  - QA feedback given → `docs/feedback/README.md`, one item per distinct problem
- Make each entry specific. Name the exact screen / feature / endpoint, so the reader can act without asking a follow-up question. If it is unclear which tracker or feature an item belongs to, ask. Do not guess.

### File naming (best practice)
- **Folders** — lowercase, one word or kebab-case: `api/`, `ui/`, `oq/`, `feedback/`, `ui/work-dashboard/`
- **Trackers** — `README.md` inside each tracker folder, so GitHub shows it when someone opens the folder
- **Top-level living docs** — `UPPER_SNAKE_CASE.md` (`CHANGELOG.md`)
- **Files inside folders** — lowercase `snake_case`. Name them by subject, not by date or person:
  - `docs/api/<area>_api_reference.md`, e.g. `work_api_reference.md`. When the backend shares a raw `<area>_api_reference.json`, put it beside that file
  - `docs/ui/<feature>/<screen>.png|jpg`, e.g. `docs/ui/rating/rating_details.jpg`. Name numbered variants `dashboard_1.jpg`, `dashboard_2.jpg`
  - `docs/feedback/<id>_<topic>.png|jpg` for QA attachments, e.g. `fb_3_login_crash.png`
- Never use spaces, uppercase letters or version suffixes like `_final` / `_v2` / `(1)` in a file name. Overwrite the file and let git keep the history
- Never commit real tokens, passwords or internal server paths. Remove them from shared API docs / Postman exports first

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

Add this to the app module's `build.gradle.kts`, after the `dependencies {}` block:

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
