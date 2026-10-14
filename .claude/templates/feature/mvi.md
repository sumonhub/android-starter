# Feature Template: MVI (feature-based, single module)

> Aligned with `.claude/templates/architecture/mvi.md` and the project's `CLAUDE.md` Hard Rules. This template
> assumes that shared scaffold already exists — `core/mvi/`, `core/network/` (`NetworkResult`, `safeApiCall`),
> `core/di/`, `core/ui/`, `navigation/`. Do not recreate any of those files here; only reference/extend them.

> Naming, features without an API, and moving a feature from one screen to two follow
> `.claude/templates/shared/feature-conventions.md`.

## Tech stack
- UI: Jetpack Compose
- DI: Koin
- Networking: Retrofit (reuses the existing `core/network` `Retrofit` singleton — never build a new one per feature)
- Image loading/caching: Coil (coil-compose)
- Async: Kotlin Coroutines + Flow

## Architecture Rules (MUST FOLLOW)
- Pattern: plain MVI, single module, feature-based — **no domain layer, no UseCases, no Repository interface/impl split**
- Each feature is self-contained under `feature/<feature_name>/` with its own `data/`, `presentation/`, `di/`
- The ViewModel calls the feature's Repository directly
- If another feature needs this feature's data, it injects this feature's Repository — never imports its `presentation/` package or its DTOs
- Theme/design system lives in `core/ui/theme/`, common widgets in `core/ui/components/` — reuse, don't redefine
- A secondary capability on the same data and same UI area (search, filter, sort) is merged into the existing screen's State/Intent — not a separate screen/ViewModel
- Only create a separate screen when it has its own distinct route and distinct UI

## Defaults (no need to ask)
- API base URL: already fixed at project level (`core/network/ApiConfig.kt`) — never ask about it again per feature
- No auth header beyond whatever `core/di/NetworkModule.kt`'s OkHttp already attaches globally
- No pagination — plain "return the full list" endpoint, `// TODO: add pagination` comment on the Api method if the list could realistically grow large

Only ask if the feature description explicitly implies otherwise (e.g. "paginated feed", "requires a login token per-request").

## Folder structure to generate (replace `<Feature>` with the given feature name, e.g. "Order", "Article")

```
feature/<feature_name>/
├── data/
│   ├── remote/
│   │   ├── <Feature>ApiService.kt               (Retrofit interface, suspend funs returning Response<Dto>)
│   │   └── dto/
│   │       └── <Feature>Dto.kt                  (raw JSON model — never reaches presentation/)
│   ├── model/
│   │   └── <Feature>.kt                         (model the UI works with)
│   ├── mapper/
│   │   └── <Feature>Mapper.kt                   (Dto -> model)
│   └── <Feature>Repository.kt                   (concrete class — no interface)
├── presentation/                                 # single-screen feature (most common case) — no screen subfolder
│   ├── components/                              [CONDITIONAL] Composables reused across 2+ screens in this feature
│   ├── <Feature>Contract.kt                      (State + Intent + Effect — all three in one file)
│   ├── <Feature>ViewModel.kt
│   └── <Feature>Screen.kt
└── di/
    └── <Feature>Module.kt                       (Koin module: `val <feature>Module = module { ... }`)
```

> Only introduce a per-screen subfolder (`presentation/<screenName>/` with its own Contract/ViewModel/Screen and
> optional `components/`) once the feature genuinely has 2+ distinct screens.

## Requirements per layer

### data/remote
- `<Feature>ApiService.kt`: Retrofit interface, standard REST endpoints, each a `suspend fun` returning `Response<<Feature>Dto>` (or `Response<List<...>>`)
- `dto/<Feature>Dto.kt`: mirrors the API JSON shape exactly; every field has `@SerializedName`, nullable where the backend may omit it. Must stay in `dto/` — the R8 keep rule depends on it

### data/model + data/mapper
- `model/<Feature>.kt`: plain Kotlin data class with non-null, UI-ready fields — no Retrofit/Gson annotations
- `mapper/<Feature>Mapper.kt`: pure extension functions (`fun <Feature>Dto.toModel(): <Feature>`); defaults for missing fields and server-string → enum mapping live here

### data/`<Feature>Repository.kt`
- Concrete class, constructor takes `<Feature>ApiService`
- Every call goes through `safeApiCall { api.getList() }` from `core/network` — never a hand-written `try/catch` or `response.isSuccessful` check here
- Maps a `NetworkResult.Success` DTO to the model before returning: `suspend fun getList(): NetworkResult<List<<Feature>>>`
- Retrofit suspend calls are main-safe — inject a dispatcher (`get(named("io"))`) only if the repository does blocking or heavy work itself

### presentation/ — for EACH screen
- **Contract.kt**:
  - `State`: `data class <Feature>State(...) : MviState` — only `val` + immutable collections; must include `isLoading: Boolean`, `error: NetworkResult.Error?` (not a display string — converted at render time) and, for server-data screens, `isRefreshing: Boolean`, plus screen data. Search/filter fields go here too
  - `Intent`: `sealed interface <Feature>Intent : MviIntent` — every user action (Load, Refresh, Retry, ErrorDismissed, ItemClicked, ...)
  - `Effect`: `sealed interface <Feature>Effect : MviEffect` — one-shot events only (Navigate, ShowToast). Anything that must survive rotation belongs in State
- **ViewModel.kt**:
  - Extends `core/mvi/BaseViewModel<State, Intent, Effect>` — never redeclares `MutableStateFlow`/`Channel`
  - Constructor-injects the Repository (via Koin)
  - Overrides only `processIntent(intent)`; work runs in `launchSafely { ... }` (inherited from `BaseViewModel`, never raw `viewModelScope.launch`), results handled with `when (val result = repository.getList())` → `setState { copy(...) }`. No `try/catch` needed — `safeApiCall` already turned failures into `NetworkResult.Error`
  - Overrides `handleError(throwable)`: `setState { copy(isLoading = false, error = NetworkResult.Error(cause = throwable)) }` — mandatory
  - Debounce search input (300ms) with a cancellable `Job` if search exists
- **Screen.kt**:
  - Resolves the ViewModel with `koinViewModel()`, collects `state` with `collectAsStateWithLifecycle()` and `effect` inside `LaunchedEffect(Unit) { viewModel.effect.collect { ... } }`
  - Talks to the ViewModel **only** via `viewModel.processIntent(...)`
  - Split into a stateful `<Feature>Screen()` (ViewModel wiring) and a stateless `<Feature>Content(state, onIntent)` that the `@Preview` renders with sample state
  - Body composes inside `AppScreen`, per the "Screen usage" example in `.claude/templates/shared/ui-components.md`: `loading = LoadingState(state.isLoading)`, `snackbarMessage = state.error.toErrorMessage(onRetry = ..., onDismiss = ...)`, `empty = EmptyState(...).takeIf { !state.isLoading && <no data> }`, and `refresh = RefreshState(state.isRefreshing, ...)` for server-data screens or `refresh = null` for forms/static screens
  - Uses `AppTopBar`, `AppButton`, `AppChip`, `Avatar` from `core/ui/components/` — never the raw Material equivalents; images via Coil's `AsyncImage`

### di/`<Feature>Module.kt` (Koin)
- `single { get<Retrofit>().create(<Feature>ApiService::class.java) }` — reuse the core Retrofit singleton
- `single { <Feature>Repository(get()) }`
- `viewModel { <Feature>ViewModel(get()) }` — one per screen ViewModel
- Register the module in `App.kt` (`modules(coreModules + listOf(<feature>Module))`)

## Forms, pickers, dialogs and tests
- **Forms** — State holds `xxxError: ValidationError?` per field; the ViewModel checks fields with `core/util/Validator` on submit (business rules that need data stay in the Repository / API); each field is `AppOutlinedTextField(error = state.xxxError?.toMessage(), maxLength = InputLimits.X)`
- **Returning a value** (pickers) — the picker screen hands its result back with `navController.setNavResult(NavResultKeys.X, value)` + `popBackStack()` in `AppNavGraph`; the caller registers `entry.NavResultEffect(NavResultKeys.X) { ... }` — see `.claude/templates/shared/core-utils.md`
- **Dialogs & sheets** — a `showXDialog` / `showXSheet` State flag + `AppConfirmDialog` / `AppBottomSheet` / `SearchableBottomSheet`
- **Dates** — the mapper parses server dates with `DateTimeFormat.parseInstant` / `parseDate`; the Screen formats them with `DateTimeFormat.date(...)` etc.
- **Tests** (`app/src/test/java/<package>/feature/<feature_name>/`), generated with the feature:
  - `presentation/<Screen>ViewModelTest.kt` per ViewModel — success, error, and one Effect/Event path, using `MainDispatcherRule` + Turbine
  - `data/mapper/<Feature>MapperTest.kt` — DTO with missing/null fields maps to safe defaults
  - Fakes, no mocking library: `Fake<Feature>ApiService` implementing the Retrofit interface (returns `Response.success(...)` / `Response.error(...)`), with the real Repository on top

## Navigation wiring
- Add the feature's route constants to the shared `navigation/Routes.kt` (e.g. `const val FEATURE_LIST = "feature_list"`, `const val FEATURE_DETAIL = "feature_detail/{id}"`)
- Register the composables in `navigation/AppNavGraph.kt`
- This is the only change made to a shared/core file — everything else stays inside `feature/<feature_name>/`

## Cross-feature consumption pattern (when relevant)
- Another feature that needs this feature's data injects `<Feature>Repository` via Koin (`get()`) in its own module
- Never duplicate the model in another feature; never import this feature's `presentation/` or `dto/` from outside

## Constraints
- DTOs never reach `presentation/` — always map to the model in the Repository
- Explicit `isLoading` / `error` fields in every State — never infer status from nullable data
- Every screen follows the same 3-file pattern (Contract, ViewModel, Screen)
- Dependencies come from Koin (`get()`, `koinViewModel()`) — no manual instantiation
- Every rule in the project's `CLAUDE.md` `## Hard Rules — Never Violate` applies here too
- When anything is unclear — ask before generating

## Deliverable
When given a feature name, generate this full folder structure plus its tests (see "Forms, pickers, dialogs and tests") with working, compilable Kotlin + Compose code following all rules above, and wire its routes into the shared `navigation/` files.
