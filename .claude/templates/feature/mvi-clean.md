# Feature Template: MVI + Clean Architecture (feature-based, single module)

> Aligned with `.claude/templates/architecture/mvi-clean.md` and the project's `CLAUDE.md` Hard
> Rules. This template assumes that shared scaffold already exists — `core/mvi/`, `core/domain/`,
> `core/error/`, `core/network/`, `core/ui/`, `navigation/`. Do not recreate any of those files
> here; only reference/extend them.

> Naming, features without an API, and moving a feature from one screen to two follow
> `.claude/templates/shared/feature-conventions.md`.

## Tech stack
- UI: Jetpack Compose
- DI: Koin
- Networking: Retrofit (reuses the existing `core/network` `Retrofit` singleton — never build a new one per feature)
- Image loading/caching: Coil (coil-compose)
- Async: Kotlin Coroutines + Flow

## Architecture Rules (MUST FOLLOW)
- Pattern: MVI + Clean Architecture, single module, feature-based
- Each feature is self-contained under `feature/<feature_name>/` with its own `domain/`, `data/`, `presentation/`, `di/`
- Do NOT create a shared/common domain module — each feature owns its own domain layer
- If another feature needs this feature's model, it consumes it via this feature's UseCase or Repository interface only — never import another feature's `data/` or `presentation/` package directly
- Theme/design system lives in `core/ui/theme/` — reuse, don't redefine colors/typography here
- Common UI widgets (`AppScreen`, `AppTopBar`, `AppButton`, `AppChip`, `Avatar`, loaders, error/empty views) live in `core/ui/components/` — reuse, don't duplicate
- If a screen has a secondary capability (e.g. search, filter, sort) that operates on the same data and same UI area, merge it into the existing screen's State/Intent — do not create a separate screen/ViewModel for it
- Only create a separate screen when it has its own distinct route/navigation entry and distinct UI

## Defaults (no need to ask)
- API base URL: already fixed at project level (`core/network/ApiConfig.kt`) — never ask about it again per feature
- No auth header beyond whatever `core/network/NetworkModule.kt`'s OkHttp already attaches globally
- No pagination — plain "return the full list" endpoint, `// TODO: add pagination` comment on the Api method if the list could realistically grow large

Only ask if the feature description explicitly implies otherwise (e.g. person says "paginated feed" or "requires a login token per-request") — don't ask preemptively.

## Folder structure to generate (replace `<Feature>` with the given feature name, e.g. "Order", "Article", "Booking")

```
feature/<feature_name>/
├── domain/
│   ├── model/
│   │   └── <Feature>.kt
│   ├── repository/
│   │   └── <Feature>Repository.kt              (interface, returns core/domain/Result<T>)
│   └── usecase/
│       ├── Get<Feature>ListUseCase.kt           (extends core/domain/UseCase.kt)
│       ├── Get<Feature>DetailUseCase.kt         (extends core/domain/UseCase.kt)
│       └── Search<Feature>UseCase.kt            (only if search is required; extends core/domain/UseCase.kt)
├── data/
│   ├── remote/
│   │   ├── <Feature>ApiService.kt               (Retrofit interface)
│   │   └── dto/
│   │       └── <Feature>Dto.kt                  (raw JSON model — never exposed above this layer)
│   ├── mapper/
│   │   └── <Feature>Mapper.kt                   (Dto <-> domain model mapping)
│   └── repository/
│       └── <Feature>RepositoryImpl.kt           (implements domain <Feature>Repository)
├── presentation/                                 # single-screen feature (most common case) — no screen subfolder:
│   ├── components/                              [LEVEL 2][CONDITIONAL] Composables reused across 2+ screens in this feature
│   ├── <Feature>Contract.kt                      (Intent + State + Effect — all three in one file)
│   ├── <Feature>ViewModel.kt
│   └── <Feature>Screen.kt
└── di/
    └── <Feature>Module.kt                       (Koin module: `val <feature>Module = module { ... }`)
```

> Only introduce a per-screen subfolder once the feature genuinely has 2+ distinct screens (e.g. auth, work) —
> a single-screen feature (the common case for a simple CRUD/picker feature) keeps its Contract/ViewModel/Screen
> directly in `presentation/`, never in a redundant `presentation/<featureName>/` subfolder.
>
> Multi-screen shape (only when 2+ screens exist):
> ```
> presentation/
> ├── components/                                [LEVEL 2][CONDITIONAL] reused across 2+ screens
> └── screenName/                                one folder PER screen — repeat for however many screens this feature
>     │                                          needs (e.g. list/ + detail/ for CRUD; signIn/, signUpOtp/, ... for auth)
>     ├── components/                            [LEVEL 3][CONDITIONAL] used only in this one screen
>     ├── ScreenNameContract.kt
>     ├── ScreenNameViewModel.kt
>     └── ScreenNameScreen.kt
> ```

> Start every new component at screen level ([LEVEL 3]). Move a component up to `presentation/components/`
> ([LEVEL 2]) only once it's reused across 2+ screens within this feature — never create it there preemptively.

## Requirements per layer

### domain/model
- Plain Kotlin data class, no Android/Compose/Retrofit imports
- Represents the pure business entity

### domain/repository (interface)
- Defines contract only, returning the shared domain `Result<T>` (from `core/domain/Result.kt`) — never a raw `List<Feature>` or nullable type as the return type
- Example: `suspend fun getList(): Result<List<Feature>>`, `suspend fun getDetail(id: String): Result<Feature>`
- No implementation details, no DTOs referenced here

### domain/usecase
- Every UseCase **extends `core/domain/UseCase.kt`** (`class GetFeatureListUseCase(private val repository: FeatureRepository, dispatcher: CoroutineDispatcher) : UseCase<Unit, List<Feature>>(dispatcher)`), not a bare class with its own `suspend operator fun invoke`
- One class per business action, single responsibility
- Depends only on the Repository interface, never on Api/Dto directly
- Business rules live here, not in the ViewModel or RepositoryImpl; field-format checks (`Validator`) stay in the ViewModel
- The base `UseCase` already wraps `execute()` in try/catch and returns `Result.Error(Failure.Unknown(e))` on unexpected exceptions — UseCase code itself should not add another try/catch

### data/remote
- `<Feature>ApiService.kt`: Retrofit interface, standard REST endpoints (list, detail by id, search by query param); suspend funs returning `Response<ApiResponse<<Feature>Dto>>` (the envelope from `core/network/`), or `Response<ApiResponse<Unit>>` when success carries no data
- `dto/<Feature>Dto.kt`: lives in the `dto/` subfolder (the R8 keep rule depends on it), separate from the domain model, mirrors the API JSON shape exactly; every field has `@SerializedName`

### data/mapper
- `class <Feature>Mapper : Mapper<<Feature>Dto, <Feature>>` implementing `core/domain/Mapper.kt`; no I/O, no state — Dto -> domain model only; server dates parsed with `DateTimeFormat`
- Keep all field-mapping/transform logic isolated here, not inline in repository

### data/repository/`<Feature>RepositoryImpl.kt`
- Implements the domain repository interface
- Constructor takes `<Feature>ApiService` and `<Feature>Mapper` (injected via Koin)
- Every call goes through `safeApiCall(apiCall = { ... }, mapper = { ... }).toResult()` (or `safeUnitApiCall { ... }.toResult()`) from `core/network/SafeApiCall.kt`, exactly as the "RepositoryImpl usage" example in `.claude/templates/architecture/mvi-clean.md` — never a hand-written `try/catch` or `response.isSuccessful` check. `safeApiCall` already maps HTTP, envelope and connectivity errors to `Failure`
- **This is the only place `NetworkResult` and Retrofit types are visible** — nothing above this file ever sees them, per the project's Hard Rules

### presentation/ (or presentation/screenName/ if the feature has 2+ screens) — for EACH screen
- **ScreenNameContract.kt**: Intent + State + Effect for this screen, all three declared in this one file:
  - `State`: data class implementing `MviState`, must include `status: UiStatus` (`isLoading` + `failure: Failure?`, from `core/mvi/UiStatus.kt`), `isRefreshing: Boolean` for server-data screens (never a raw `String?` — map `Failure` to a display message inside the Composable, not in State), and screen-specific data fields. If this screen has search/filter, include `searchQuery`/`isSearching` fields directly here (do not split into a separate screen).
  - `Intent`: sealed interface implementing `MviIntent`, covering every user action (Load, Refresh, Retry, ErrorDismissed, ItemClicked, SearchQueryChanged, ClearSearch, etc.)
  - `Effect`: sealed interface implementing `MviEffect`, for one-time events only (Navigate, ShowToast, ShowSnackbar) — never put these in State
- **ViewModel.kt**:
  - **Extends `core/mvi/BaseViewModel<State, Intent, Effect>`** — do not redeclare `MutableStateFlow`/`Channel`/`state`/`effect`, those already exist on the base class
  - Constructor-injects only UseCases (never Repository or ApiService directly), resolved via Koin (`viewModel { <Feature>ListViewModel(get(), get()) }` in the Koin module)
  - Overrides only `processIntent(intent: Intent)` — every branch that calls a UseCase goes through `launchSafely { ... }` (inherited from `BaseViewModel`), never a raw `viewModelScope.launch` and never a manual `try/catch`
  - On `Result.Error(failure)` from a UseCase, call `setState { copy(status = UiStatus(failure = failure)) }` — do not stringify the error here; `handleError` does the same with `Failure.Unknown(throwable)`
  - Debounce search input (300ms) using a cancellable coroutine `Job`, still launched through `launchSafely`, if search exists
  - Update state immutably via `setState { copy(...) }`, never mutate in place
- **Screen.kt**:
  - Jetpack Compose function, stateless as possible — takes State + lambda callbacks, or resolves ViewModel via `koinViewModel()`
  - Collect state with **`collectAsStateWithLifecycle()`** — never `collectAsState()`
  - No business logic inside Composables — only render state and call `processIntent`
  - Use `LaunchedEffect` to collect one-time Effects (navigation, snackbar) from `viewModel.effect`
  - Any image loading uses Coil's `AsyncImage` composable — do not use manual Bitmap loading or other caching libs
  - Body composes inside `AppScreen`, per the "Screen usage" example in `.claude/templates/shared/ui-components.md`: `loading = LoadingState(state.status.isLoading)`, `snackbarMessage = state.status.toErrorMessage(onRetry = ..., onDismiss = ...)`, `empty = EmptyState(...).takeIf { !state.status.isLoading && <no data> }`, and `refresh = RefreshState(state.isRefreshing, ...)` for server-data screens or `refresh = null` for forms/static screens
  - Uses `AppTopBar`, `AppButton`, `AppChip`, `Avatar` from `core/ui/components/` — never the raw Material equivalents; images via Coil's `AsyncImage`

### presentation/components
- Composables reused across THIS feature's own screens only (e.g. a Card used in both List and Detail)
- If a component is needed by another feature too, flag it — don't move it preemptively without confirmation

### di/`<Feature>Module.kt` (Koin)
- Define as: `val <feature>Module = module { ... }`
- `factory { <Feature>Mapper() }`
- `single<<Feature>Repository> { <Feature>RepositoryImpl(get(), get()) }`
- `single { get<Retrofit>().create(<Feature>ApiService::class.java) }` — reuse existing core Retrofit singleton, do not build a new Retrofit instance per feature
- `factory { Get<Feature>ListUseCase(get(), get(named("io"))) }` — one factory per UseCase, always passing the IO dispatcher from `core/di/DispatcherModule.kt` as the second constructor argument
- `viewModel { <Feature>ListViewModel(get(), get()) }` — one per screen ViewModel
- Register this module in `App.kt` (`modules(coreModules + listOf(<feature>Module))` — append to the existing list)

## Forms, pickers, dialogs and tests
- **Forms** — State holds `xxxError: ValidationError?` per field; the ViewModel checks fields with `core/util/Validator` on submit (business rules that need data stay in the UseCase); each field is `AppOutlinedTextField(error = state.xxxError?.toMessage(), maxLength = InputLimits.X)`
- **Returning a value** (pickers) — the picker screen hands its result back with `navController.setNavResult(NavResultKeys.X, value)` + `popBackStack()` in `AppNavGraph`; the caller registers `entry.NavResultEffect(NavResultKeys.X) { ... }` — see `.claude/templates/shared/core-utils.md`
- **Dialogs & sheets** — a `showXDialog` / `showXSheet` State flag + `AppConfirmDialog` / `AppBottomSheet` / `SearchableBottomSheet`
- **Dates** — the mapper parses server dates with `DateTimeFormat.parseInstant` / `parseDate`; the Screen formats them with `DateTimeFormat.date(...)` etc.
- **Tests** (`app/src/test/java/<package>/feature/<feature_name>/`), generated with the feature:
  - `presentation/<Screen>ViewModelTest.kt` per ViewModel — success, error, and one Effect/Event path, using `MainDispatcherRule` + Turbine
  - `data/mapper/<Feature>MapperTest.kt` — DTO with missing/null fields maps to safe defaults
  - Fakes, no mocking library: `Fake<Feature>Repository` implementing the domain repository interface

## Navigation wiring
- Add the feature's routes to the shared `navigation/Routes.kt` sealed hierarchy, e.g. `@Serializable data object FeatureList : Routes` and `@Serializable data class FeatureDetail(val id: String) : Routes`
- Register the composables in `navigation/AppNavGraph.kt`: `composable<Routes.FeatureList> { FeatureListScreen() }`, `composable<Routes.FeatureDetail> { backStackEntry -> FeatureDetailScreen(backStackEntry.toRoute()) }`
- This is the only change made to a shared/core file — everything else about the feature stays inside `feature/<feature_name>/`

## Cross-feature consumption pattern (when relevant)
- If another feature needs this feature's model/data, it must inject this feature's UseCase via Koin constructor injection — e.g. `class OtherFeatureUseCase(private val get<Feature>DetailUseCase: Get<Feature>DetailUseCase)`, wired in the other feature's own Koin module using `get()`
- Never duplicate the model in another feature
- Never import this feature's `data/` or `presentation/` package from outside this feature

## Constraints
- No direct API/DB/Coil calls from ViewModel — always through UseCase → Repository
- Keep Dto and domain model strictly separate — always map, never reuse Dto as UI/domain model
- Explicit `status: UiStatus` in every State — no relying on nullable data fields to infer status
- Every new screen must follow the same 3-file pattern (Screen, ViewModel, Contract) — no exceptions, no shortcuts
- Use `get()` / `getViewModel()` / `koinViewModel()` idioms consistently — no manual instantiation of dependencies anywhere
- Every rule in the project's `CLAUDE.md` `## Hard Rules — Never Violate` and `## Architecture Hard Rules` applies here too (`!!`, `GlobalScope`, `sealed class` for Effect, `collectAsState()`, raw try/catch in ViewModel, direct Repository calls from ViewModel, Android imports in `domain/` — all forbidden)
- When anything is unclear — ask before generating

## Deliverable
When given a feature name (e.g. "Order", "Article"), generate this full folder structure plus its tests (see "Forms, pickers, dialogs and tests") with working, compilable Kotlin + Compose code following all rules above, and wire its routes into the shared `navigation/` files.
