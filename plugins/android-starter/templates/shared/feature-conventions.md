# Shared Feature Conventions (all architectures)

> Applies to every feature, whatever the architecture. `/android-starter:generate-feature` and `/android-starter:generate-screen` read this
> file together with the matching feature template. The feature template decides what each layer's code looks
> like. This file decides naming, features without an API, and how a feature grows from one screen to two.

## Naming

Build every name from the feature/screen name the person gave (e.g. "Order History", "order detail"):

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

If the given name is unclear (e.g. "history" when an `orderhistory` feature already exists), ask before generating.

## Features without an API

Some features read and write only local data (settings, onboarding flags, a draft form). They follow the same layer rules as an API-backed feature, so every feature has the same shape. Only the data source changes:

- **Data source:** `data/local/<Feature>LocalDataSource.kt` wraps the shared `DataStore<Preferences>` (`get<DataStore<Preferences>>()`). Never create a new instance. To see what may use `SharedPreferences`, follow the architecture's Local storage rules.
- **No** `data/remote/`, `dto/` or `data/mapper/` unless the stored shape differs from the model.
- **Clean architectures** (MVI + Clean, MVVM + Clean): keep the domain `Repository` interface, `RepositoryImpl` and UseCases. `RepositoryImpl` takes the local data source instead of an ApiService.
- **Plain architectures** (MVI, MVVM): keep the concrete `<Feature>Repository` class, taking the local data source.
- **Tests:** ViewModel test as usual. Skip the mapper test if there is no mapper.
- **Docs:** no `docs/api/README.md` entry.
- A feature with **no data at all** (a static screen) has only `presentation/` + `di/`. Its ViewModel has no repository or UseCase dependencies.

## Growing from one screen to two

A single-screen feature keeps its files flat in `presentation/`. When you add its second screen, first move the existing screen into its own folder. This way the feature never has two screens side by side in a flat `presentation/`:

1. Pick the existing screen's folder name (camelCase, from its current name, e.g. `orderList/`).
2. Move its state file (Contract / UiState), ViewModel and Screen into `presentation/<existingScreen>/`.
3. Split `presentation/components/`:
   - Components that only the existing screen uses move to `presentation/<existingScreen>/components/` (Level 3).
   - Components that the new screen will also use stay in `presentation/components/` (Level 2).
4. Update package declarations and every import. This includes the feature's `di/` module, `navigation/AppNavGraph.kt`, and the moved tests under `app/src/test/`.
5. Route names do not change.
6. Then add the new screen in `presentation/<newScreen>/`.

Show this plan as a file list (old path → new path). Get confirmation before you move anything.
