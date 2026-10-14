# Android Starter

A reusable Claude Code kit for starting Android projects with a consistent, structured architecture.

Copy one folder into a new Android Studio project and give Claude Code one instruction. Claude Code then scaffolds the shared foundation: Jetpack Compose, Koin DI, Retrofit networking, base ViewModel, theme, navigation shell and a project `CLAUDE.md`. The `CLAUDE.md` keeps every later session on the same rules.

## How to use

1. Create a new project in Android Studio (Empty Activity).
2. Copy this repo's `.claude/` folder into the new project's root.
3. Open Claude Code in that project and run `/init-project`.

Claude asks which architecture you want. Then it scaffolds the project. You can use these slash commands:

| Command | What it does |
|---|---|
| `/init-project` | Scaffolds the shared foundation (core, DI, theme, navigation, `CLAUDE.md`). No feature code. |
| `/generate-feature` | Adds a self-contained feature (`domain` / `data` / `presentation` / `di`) in the chosen architecture. |
| `/generate-screen` | Builds one Compose screen from a Figma link or screenshot, reusing existing theme tokens and components. |

## Architectures

| Pattern | Status |
|---|---|
| MVI + Clean Architecture | Fully supported: complete scaffold template with full code |
| MVI, MVVM, MVVM + Clean | The scaffold template is an outline (structure, dependencies, base classes) |

Every architecture has its own feature template for `/generate-feature`.

## What's in the kit

```
.claude/
├── commands/                 # slash commands
└── templates/
    ├── architecture/         # scaffold templates, one per pattern
    ├── feature/              # feature generation templates
    └── shared/               # hard rules, UI components, core utils, feature conventions, project setup, coding standards, git workflow, response format, docs rules
```

Whichever architecture you pick, every generated project also gets:
- debug / staging / release environments, each with its own base URL
- release signing kept out of git (`keystore.properties`)
- R8 keep rules for API models, and a system splash screen
- shared UI components (`AppScreen`, buttons, text fields, sheets, dialogs, skeletons…)
- form validation, screen-result passing and date formatting helpers
- a unit-test setup, ktlint and a GitHub Actions workflow

`/init-project` also creates a `docs/` folder in your project, the same for every architecture:

| Folder | Purpose |
|---|---|
| `docs/api/` | API tracker: endpoints, integration status, backend issues |
| `docs/ui/` | Design screenshots and Figma references, one subfolder per feature |
| `docs/oq/` | Open-questions tracker: pending product and architecture decisions |
| `docs/feedback/` | QA feedback tracker |

Each tracker is a `README.md` with colored status icons (🔴 🟡 🟢 ⚪). It starts with a list. After the list comes one details section per item. Click an ID in the list to jump to its details. Click "Back to list" to go back.

The Android code in this repo is Android Studio's Empty Activity (Jetpack Compose) template. It shows the starting point before `/init-project` runs.

## Requirements

- Android Studio with AGP 9+
- A Compose project (Empty Activity template) with Kotlin DSL, `gradle/libs.versions.toml`, Kotlin 2.0+ and JDK 17+. `/init-project` checks all of these first. If anything is missing, it explains how to fix it. XML / View-based projects are not supported.
- [Claude Code](https://claude.com/claude-code)
- Optional: the Figma MCP server, for `/generate-screen` from Figma links
