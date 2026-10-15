# Android Starter

A Claude Code plugin for starting Android projects with a consistent, structured architecture.

Install the plugin once, then give Claude Code one instruction in a new Android Studio project. Claude Code then scaffolds the shared foundation: Jetpack Compose, Koin DI, Retrofit networking, base ViewModel, theme, navigation shell and a project `CLAUDE.md`. The `CLAUDE.md` keeps every later session on the same rules.

## How to use

1. Install the plugin. Run this once inside Claude Code:
   ```
   /plugin install android-starter --marketplace sumonhub/android-starter
   ```
2. Create a new project in Android Studio (Empty Activity).
3. Open Claude Code in that project and run `/android-starter:init-project`.

Claude asks which architecture you want. Then it scaffolds the project. You can use these slash commands:

| Command | What it does |
|---|---|
| `/android-starter:init-project` | Scaffolds the shared foundation (core, DI, theme, navigation, `CLAUDE.md`). No feature code. |
| `/android-starter:generate-feature` | Adds a self-contained feature (`domain` / `data` / `presentation` / `di`) in the chosen architecture. |
| `/android-starter:generate-screen` | Builds one Compose screen from a Figma link or screenshot, reusing existing theme tokens and components. |

## Getting updates

When a new version is released, update the plugin:

```bash
claude plugin marketplace update android-starter
claude plugin update android-starter@android-starter
```

Then run `/reload-plugins` in any open session. To update automatically, open `/plugin`, go to the **Marketplaces** tab, select `android-starter` and turn on auto-update.

Updates change how future commands work. They don't change code that was already generated. If an update changes the rules that `/android-starter:init-project` writes into `CLAUDE.md`, run `/android-starter:init-project` again and pick **Re-run the same architecture**. It shows each changed section and asks before updating it.

### Turn it on for a whole team

Commit this to a project's `.claude/settings.json`. Everyone who opens the project and trusts the folder gets the plugin:

```json
{
  "extraKnownMarketplaces": {
    "android-starter": {
      "source": { "source": "github", "repo": "sumonhub/android-starter" }
    }
  },
  "enabledPlugins": {
    "android-starter@android-starter": true
  }
}
```

## Architectures

| Pattern | Status |
|---|---|
| MVI + Clean Architecture | Fully supported: complete scaffold template with full code |
| MVI, MVVM, MVVM + Clean | The scaffold template is an outline (structure, dependencies, base classes) |

Every architecture has its own feature template for `/android-starter:generate-feature`.

## What's in the kit

```
.claude-plugin/marketplace.json   # lets Claude Code install the plugin from this repo
plugins/android-starter/
├── .claude-plugin/plugin.json    # plugin name and version
├── skills/                       # the three slash commands, one folder each
└── templates/
    ├── architecture/             # scaffold templates, one per pattern
    ├── feature/                  # feature generation templates
    └── shared/                   # hard rules, UI components, core utils, feature conventions, project setup, coding standards, git workflow, response format, docs rules
```

Whichever architecture you pick, every generated project also gets:
- debug / staging / release environments, each with its own base URL
- release signing kept out of git (`keystore.properties`)
- R8 keep rules for API models, and a system splash screen
- shared UI components (`AppScreen`, buttons, text fields, sheets, dialogs, skeletons…)
- form validation, screen-result passing and date formatting helpers
- a unit-test setup, ktlint and a GitHub Actions workflow

`/android-starter:init-project` also creates a `docs/` folder in your project, the same for every architecture:

| Folder | Purpose |
|---|---|
| `docs/api/` | API tracker: endpoints, integration status, backend issues |
| `docs/ui/` | Design screenshots and Figma references, one subfolder per feature |
| `docs/oq/` | Open-questions tracker: pending product and architecture decisions |
| `docs/feedback/` | QA feedback tracker |

Each tracker is a `README.md` with colored status icons (🔴 🟡 🟢 ⚪). It starts with a list. After the list comes one details section per item. Click an ID in the list to jump to its details. Click "Back to list" to go back.

The Android code in this repo is Android Studio's Empty Activity (Jetpack Compose) template. It shows the starting point before `/android-starter:init-project` runs.

## Changing the plugin

1. Edit files under `plugins/android-starter/`. In a skill, refer to a template as `${CLAUDE_PLUGIN_ROOT}/templates/...`. Inside a template, write `templates/...`.
2. Try your change without installing:
   ```bash
   claude --plugin-dir ./plugins/android-starter
   ```
   Run `/reload-plugins` after each edit.
3. Check the files:
   ```bash
   claude plugin validate ./plugins/android-starter
   claude plugin validate .
   ```
4. Raise `version` in `plugins/android-starter/.claude-plugin/plugin.json`, then push. People only get an update when the version changes.

## Requirements

- Android Studio with AGP 9+
- A Compose project (Empty Activity template) with Kotlin DSL, `gradle/libs.versions.toml`, Kotlin 2.0+ and JDK 17+. `/android-starter:init-project` checks all of these first. If anything is missing, it explains how to fix it. XML / View-based projects are not supported.
- [Claude Code](https://claude.com/claude-code)
- Optional: the Figma MCP server, for `/android-starter:generate-screen` from Figma links
