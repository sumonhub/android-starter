# Feature Template: MVVM (feature-based, single module)

> This template follows `templates/architecture/mvvm.md` and the Hard Rules in the project's `CLAUDE.md`.
> It assumes the shared scaffold already exists: `core/mvvm/`, `core/network/` (`NetworkResult`, `safeApiCall`),
> `core/di/`, `core/ui/`, `navigation/`. Do not recreate any of these files here. Only reference or extend them.

> For naming, features without an API, and moving a feature from one screen to two, follow
> `templates/shared/feature-conventions.md`.

## Tech stack
- UI: Jetpack Compose
- DI: Koin
- Networking: Retrofit. Reuse the existing `core/network` `Retrofit` singleton. Never build a new one per feature.
- Image loading/caching: Coil (coil-compose)
- Async: Kotlin Coroutines + Flow

## Architecture Rules (MUST FOLLOW)
- Pattern: plain MVVM, single module, feature-based.
- **No domain layer, no UseCases, no Repository interface/impl split.**
- Each feature is self-contained. It lives under `feature/<feature_name>/` and has its own `data/`, `presentation/`, `di/`.
- The ViewModel calls the feature's Repository directly.
- The ViewModel exposes **explicit public functions** (`fun onRefresh()`, `fun onItemClicked(id: String)`). There is no `processIntent`.
- If another feature needs this feature's data, it injects this feature's Repository. It never imports this feature's `presentation/` package or its DTOs.
- The theme/design system lives in `core/ui/theme/`. Common widgets live in `core/ui/components/`. Reuse them. Do not redefine them.
- Some capabilities (search, filter, sort) work on the same data and the same UI area as an existing screen. Merge them into that screen's UiState/ViewModel. Do not create a separate screen for them.
- Only create a separate screen when it has its own distinct route and its own distinct UI.

## Defaults (no need to ask)
- API base URL: already set at project level in `core/network/ApiConfig.kt`. Never ask about it again for a feature.
- Auth: do not add any auth header. Use only what OkHttp in `core/di/NetworkModule.kt` already attaches globally.
- No pagination. Use a plain "return the full list" endpoint. If the list could realistically grow large, add a `// TODO: add pagination` comment on the Api method.

Ask only if the feature description explicitly points to something different (e.g. "paginated feed", "requires a login token per-request").

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
│   ├── <Feature>UiState.kt                       (UiState data class + UiEvent sealed interface)
│   ├── <Feature>ViewModel.kt
│   └── <Feature>Screen.kt
└── di/
    └── <Feature>Module.kt                       (Koin module: `val <feature>Module = module { ... }`)
```

> Add a per-screen subfolder only when the feature really has 2+ distinct screens. Each subfolder is
> `presentation/<screenName>/`. It has its own UiState/ViewModel/Screen and an optional `components/`.

## Requirements per layer

### data/remote
- `<Feature>ApiService.kt`: a Retrofit interface with standard REST endpoints. Each endpoint is a `suspend fun` that returns `Response<<Feature>Dto>` (or `Response<List<...>>`).
- `dto/<Feature>Dto.kt`: mirrors the API JSON shape exactly.
  - Every field has `@SerializedName`.
  - A field is nullable when the backend may omit it.
  - The file must stay in `dto/`, because the R8 keep rule depends on it.

### data/model + data/mapper
- `model/<Feature>.kt`: a plain Kotlin data class with non-null, UI-ready fields. No Retrofit/Gson annotations.
- `mapper/<Feature>Mapper.kt`: pure extension functions (`fun <Feature>Dto.toModel(): <Feature>`). Defaults for missing fields go here. Server-string → enum mapping also goes here.

### data/`<Feature>Repository.kt`
- A concrete class. Its constructor takes `<Feature>ApiService`.
- Every call goes through `safeApiCall { api.getList() }` from `core/network`, or `safeUnitApiCall { ... }` for an endpoint with no response body. Never write your own `try/catch` or `response.isSuccessful` check here.
- Before returning, it maps the DTO to the model with `NetworkResult.map`: `suspend fun getList(): NetworkResult<List<<Feature>>> = safeApiCall { api.getList() }.map { dtos -> dtos.map { it.toModel() } }`
- Retrofit suspend calls are main-safe. Inject a dispatcher (`get(named("io"))`) only if the repository itself does blocking or heavy work.

### presentation/ — for EACH screen
- **UiState.kt**:
  - `data class <Feature>UiState(...) : UiState`. Use only `val` and immutable collections. It must include:
    - `isLoading: Boolean`
    - `error: NetworkResult.Error?`. This is not a display string. It is converted when the screen renders.
    - `isRefreshing: Boolean`, for server-data screens
    - the screen data
    - search/filter fields, if any
  - `sealed interface <Feature>UiEvent : UiEvent`. Use it for one-shot events only (NavigateToDetail, ShowToast). Anything that must survive rotation belongs in UiState.
- **ViewModel.kt**:
  - Extends `core/mvvm/BaseViewModel<<Feature>UiState>`. Never redeclares `MutableStateFlow`/`Channel`.
  - Gets the Repository by constructor injection (via Koin).
  - Loads initial data in `init { load() }`.
  - Has one public function for each user action. This includes `onRefresh()`, `onRetry()` and `onErrorDismissed()`. `onErrorDismissed()` clears `error`.
  - Runs work in `launchSafely { ... }`, which it inherits from `BaseViewModel`. Never uses raw `viewModelScope.launch`.
  - Handles results with `when (val result = repository.getList())` → `setState { copy(...) }`.
  - Sends one-shot events with `sendEvent(...)`.
  - Needs no `try/catch`. `safeApiCall` already turned failures into `NetworkResult.Error`.
  - Must override `handleError(throwable)`: `setState { copy(isLoading = false, error = NetworkResult.Error(cause = throwable)) }`
  - If search exists, debounces search input (300ms) with a cancellable `Job`.
- **Screen.kt**:
  - Resolves the ViewModel with `koinViewModel()`.
  - Collects `state` with `collectAsStateWithLifecycle()`.
  - Collects `event` inside `LaunchedEffect(Unit) { viewModel.event.collect { ... } }`.
  - Is split into two functions:
    - a stateful `<Feature>Screen()` that wires the ViewModel
    - a stateless `<Feature>Content(state, onRefresh, onItemClick, ...)`. The `@Preview` renders it with sample state.
  - Composes its body inside `AppScreen`, as in the "Screen usage" example in `templates/shared/ui-components.md`. Pass:
    - `loading = LoadingState(state.isLoading)`
    - `snackbarMessage = state.error.toErrorMessage(onRetry = ..., onDismiss = ...)`
    - `empty = EmptyState(...).takeIf { !state.isLoading && <no data> }`
    - `refresh = RefreshState(state.isRefreshing, ...)` for server-data screens, or `refresh = null` for forms/static screens
  - Uses `AppTopBar`, `AppButton`, `AppChip`, `Avatar` from `core/ui/components/`. Never uses the raw Material equivalents.
  - Loads images with Coil's `AsyncImage`.

### di/`<Feature>Module.kt` (Koin)
- `single { get<Retrofit>().create(<Feature>ApiService::class.java) }`. Reuse the core Retrofit singleton.
- `single { <Feature>Repository(get()) }`
- `viewModel { <Feature>ViewModel(get()) }`. Add one for each screen ViewModel.
- Register the module in `App.kt` (`modules(coreModules + listOf(<feature>Module))`).

## Forms, pickers, dialogs and tests
- **Forms**:
  - State holds one `xxxError: ValidationError?` for each field.
  - On submit, the ViewModel checks the fields with `core/util/Validator`.
  - Business rules that need data stay in the Repository / API.
  - Each field is `AppOutlinedTextField(error = state.xxxError?.toMessage(), maxLength = InputLimits.X)`.
- **Returning a value** (pickers):
  - The picker screen sends its result back with `navController.setNavResult(NavResultKeys.X, value)` + `popBackStack()` in `AppNavGraph`.
  - The caller registers `entry.NavResultEffect(NavResultKeys.X) { ... }`.
  - See `templates/shared/core-utils.md`.
- **Dialogs & sheets**: use a `showXDialog` / `showXSheet` State flag with `AppConfirmDialog` / `AppBottomSheet` / `SearchableBottomSheet`.
- **Dates**: the mapper parses server dates with `DateTimeFormat.parseInstant` / `parseDate`. The Screen formats them with `DateTimeFormat.date(...)` etc.
- **Tests** (`app/src/test/java/<package>/feature/<feature_name>/`): generate them with the feature.
  - `presentation/<Screen>ViewModelTest.kt`: one for each ViewModel. Cover success, error, and one Effect/Event path. Use `MainDispatcherRule` + Turbine.
  - `data/mapper/<Feature>MapperTest.kt`: check that a DTO with missing/null fields maps to safe defaults.
  - Use fakes, not a mocking library. `Fake<Feature>ApiService` implements the Retrofit interface and returns `Response.success(...)` / `Response.error(...)`. The real Repository runs on top of it.

## Navigation wiring
- Add the feature's route constants to the shared `navigation/Routes.kt` (e.g. `const val FEATURE_LIST = "feature_list"`, `const val FEATURE_DETAIL = "feature_detail/{id}"`).
- Register the composables in `navigation/AppNavGraph.kt`.
- This is the only change to a shared/core file. Everything else stays inside `feature/<feature_name>/`.

## Cross-feature consumption pattern (when relevant)
- If another feature needs this feature's data, it injects `<Feature>Repository` via Koin (`get()`) in its own module.
- Never duplicate the model in another feature.
- Never import this feature's `presentation/` or `dto/` from outside this feature.

## Constraints
- DTOs never reach `presentation/`. Always map them to the model in the Repository.
- Every UiState has explicit `isLoading` / `error` fields. Never infer status from nullable data.
- Every screen follows the same 3-file pattern (UiState, ViewModel, Screen).
- Dependencies come from Koin (`get()`, `koinViewModel()`). Never create dependencies by hand.
- Every rule in the project's `CLAUDE.md` `## Hard Rules — Never Violate` applies here too.
- When anything is unclear, ask before generating.

## Deliverable
When given a feature name, generate this full folder structure and its tests (see "Forms, pickers, dialogs and tests"). Write working, compilable Kotlin + Compose code that follows all rules above. Wire the feature's routes into the shared `navigation/` files.
