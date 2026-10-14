# Shared Feature Conventions (all architectures)

> Applies to every feature, regardless of architecture. `/generate-feature` and `/generate-screen` read this
> file alongside the matching feature template. The feature template decides what each layer's code looks
> like; this file decides naming, features without an API, and how a feature grows from one screen to two.

## Naming

Derive every name from the feature/screen name the person gave (e.g. "Order History", "order detail"):

| Thing | Rule | Example |
|---|---|---|
| Feature package / folder | lowercase, words joined, no underscores | `feature/orderhistory/` |
| Feature class prefix (`<Feature>`) | PascalCase | `OrderHistoryRepository`, `OrderHistoryScreen` |
| Koin module | camelCase + `Module` | `val orderHistoryModule` |
| Screen folder (multi-screen features) | camelCase | `presentation/orderDetail/` |
| Screen class prefix | PascalCase | `OrderDetailViewModel` |
| Route | PascalCase inside `Routes` | `Routes.OrderHistory`, `Routes.OrderDetail(id)` |
| Tests | mirror the main package under `app/src/test/java/<package>/` | `feature/orderhistory/presentation/...` |
| Design references | per `reference-docs.md`: kebab-case folder, snake_case file | `docs/ui/order-history/order_detail.png` |

If the given name is ambiguous (e.g. "history" when an `orderhistory` feature already exists), ask before generating.

## Features without an API

Some features read and write only local data (settings, onboarding flags, a draft form). The layer rules stay the same as an API-backed feature, so every feature has the same shape. Only the data source changes:

- **Data source:** `data/local/<Feature>LocalDataSource.kt` wraps the shared `DataStore<Preferences>` (`get<DataStore<Preferences>>()`, never a new instance). Follow the architecture's Local storage rules for what may use `SharedPreferences`.
- **No** `data/remote/`, `dto/` or `data/mapper/` unless the stored shape differs from the model.
- **Clean architectures** (MVI + Clean, MVVM + Clean): keep the domain `Repository` interface, `RepositoryImpl` and UseCases. `RepositoryImpl` takes the local data source instead of an ApiService.
- **Plain architectures** (MVI, MVVM): keep the concrete `<Feature>Repository` class, taking the local data source.
- **Tests:** ViewModel test as usual. Skip the mapper test if there is no mapper.
- **Docs:** no `docs/api/README.md` entry.
- A feature with **no data at all** (a static screen) has only `presentation/` + `di/`; its ViewModel has no repository or UseCase dependencies.

## Growing from one screen to two

A single-screen feature keeps its files flat in `presentation/`. When its second screen is added, the existing screen moves into its own folder first, so the feature never has two screens side by side in a flat `presentation/`:

1. Pick the existing screen's folder name (camelCase, from its current name, e.g. `orderList/`).
2. Move its state file (Contract / UiState), ViewModel and Screen into `presentation/<existingScreen>/`.
3. Split `presentation/components/`: components used only by the existing screen move to `presentation/<existingScreen>/components/` (Level 3). Components the new screen will also use stay in `presentation/components/` (Level 2).
4. Update package declarations and every import: the feature's `di/` module, `navigation/AppNavGraph.kt`, and the moved tests under `app/src/test/`.
5. Route names don't change.
6. Then add the new screen in `presentation/<newScreen>/`.

Show this plan as a file list (old path → new path) and get confirmation before moving anything.
