# Feature Template: MVI + Clean Architecture (feature-based, single module)

> This template follows `.claude/templates/architecture/mvi-clean.md` and the Hard Rules in the
> project's `CLAUDE.md`. It assumes the shared scaffold already exists: `core/mvi/`, `core/domain/`,
> `core/error/`, `core/network/`, `core/ui/`, `navigation/`. Do not recreate any of these files
> here. Only reference or extend them.

> For naming, features without an API, and moving a feature from one screen to two, follow
> `.claude/templates/shared/feature-conventions.md`.

## Tech stack
- UI: Jetpack Compose
- DI: Koin
- Networking: Retrofit. Reuse the existing `core/network` `Retrofit` singleton. Never build a new one per feature.
- Image loading/caching: Coil (coil-compose)
- Async: Kotlin Coroutines + Flow

## Architecture Rules (MUST FOLLOW)
- Pattern: MVI + Clean Architecture, single module, feature-based.
- Each feature is self-contained. It lives under `feature/<feature_name>/` and has its own `domain/`, `data/`, `presentation/`, `di/`.
- Do NOT create a shared/common domain module. Each feature owns its own domain layer.
- If another feature needs this feature's model, it gets it only through this feature's UseCase or Repository interface. Never import another feature's `data/` or `presentation/` package directly.
- The theme/design system lives in `core/ui/theme/`. Reuse it. Do not redefine colors/typography here.
- Common UI widgets live in `core/ui/components/`: `AppScreen`, `AppTopBar`, `AppButton`, `AppChip`, `Avatar`, loaders, error/empty views. Reuse them. Do not duplicate them.
- A screen may have a secondary capability (e.g. search, filter, sort) that works on the same data and the same UI area. Merge it into that screen's State/Intent. Do not create a separate screen/ViewModel for it.
- Only create a separate screen when it has its own distinct route/navigation entry and its own distinct UI.

## Defaults (no need to ask)
- API base URL: already set at project level in `core/network/ApiConfig.kt`. Never ask about it again for a feature.
- Auth: do not add any auth header. Use only what OkHttp in `core/network/NetworkModule.kt` already attaches globally.
- No pagination. Use a plain "return the full list" endpoint. If the list could realistically grow large, add a `// TODO: add pagination` comment on the Api method.

Ask only if the feature description explicitly points to something different (e.g. the person says "paginated feed" or "requires a login token per-request"). Do not ask in advance.

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

> Add a per-screen subfolder only when the feature really has 2+ distinct screens (e.g. auth, work).
> A single-screen feature is the common case for a simple CRUD/picker feature. It keeps its
> Contract/ViewModel/Screen directly in `presentation/`. Never put them in a redundant
> `presentation/<featureName>/` subfolder.
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
> ([LEVEL 2]) only when 2+ screens in this feature reuse it. Never create it there in advance.

## Requirements per layer

### domain/model
- A plain Kotlin data class. No Android/Compose/Retrofit imports.
- It represents the pure business entity.

### domain/repository (interface)
- Defines the contract only. It returns the shared domain `Result<T>` (from `core/domain/Result.kt`).
- The return type is never a raw `List<Feature>` or a nullable type.
- Example: `suspend fun getList(): Result<List<Feature>>`, `suspend fun getDetail(id: String): Result<Feature>`
- No implementation details. No DTOs are referenced here.

### domain/usecase
- Every UseCase **extends `core/domain/UseCase.kt`**: `class GetFeatureListUseCase(private val repository: FeatureRepository, dispatcher: CoroutineDispatcher) : UseCase<Unit, List<Feature>>(dispatcher)`. It is not a bare class with its own `suspend operator fun invoke`.
- One class for each business action. Each class has a single responsibility.
- Depends only on the Repository interface. Never depends on Api/Dto directly.
- Business rules live here. They do not live in the ViewModel or RepositoryImpl.
- Field-format checks (`Validator`) stay in the ViewModel.
- The base `UseCase` already wraps `execute()` in try/catch. On unexpected exceptions it returns `Result.Error(Failure.Unknown(e))`. So UseCase code should not add another try/catch.

### data/remote
- `<Feature>ApiService.kt`: a Retrofit interface with standard REST endpoints (list, detail by id, search by query param).
  - Each endpoint is a suspend fun that returns `Response<ApiResponse<<Feature>Dto>>`. `ApiResponse` is the envelope from `core/network/`.
  - When success carries no data, it returns `Response<ApiResponse<Unit>>`.
- `dto/<Feature>Dto.kt`:
  - Lives in the `dto/` subfolder, because the R8 keep rule depends on it.
  - Is separate from the domain model.
  - Mirrors the API JSON shape exactly.
  - Every field has `@SerializedName`.

### data/mapper
- `class <Feature>Mapper : Mapper<<Feature>Dto, <Feature>>` implements `core/domain/Mapper.kt`.
  - No I/O. No state.
  - It maps Dto -> domain model only.
  - It parses server dates with `DateTimeFormat`.
- Keep all field-mapping/transform logic here. Do not put it inline in the repository.

### data/repository/`<Feature>RepositoryImpl.kt`
- Implements the domain repository interface.
- Its constructor takes `<Feature>ApiService` and `<Feature>Mapper`. Koin injects both.
- Every call goes through `safeApiCall(apiCall = { ... }, mapper = { ... }).toResult()` (or `safeUnitApiCall { ... }.toResult()`) from `core/network/SafeApiCall.kt`.
  - Follow the "RepositoryImpl usage" example in `.claude/templates/architecture/mvi-clean.md` exactly.
  - Never write your own `try/catch` or `response.isSuccessful` check.
  - `safeApiCall` already maps HTTP, envelope and connectivity errors to `Failure`.
- **This is the only place `NetworkResult` and Retrofit types are visible.** Nothing above this file ever sees them. This follows the project's Hard Rules.

### presentation/ (or presentation/screenName/ if the feature has 2+ screens) — for EACH screen
- **ScreenNameContract.kt**: Intent + State + Effect for this screen. Declare all three in this one file.
  - `State`: a data class that implements `MviState`. It must include:
    - `status: UiStatus` (`isLoading` + `failure: Failure?`, from `core/mvi/UiStatus.kt`)
    - `isRefreshing: Boolean`, for server-data screens
    - screen-specific data fields
    - The error is never a raw `String?` in State. Map `Failure` to a display message inside the Composable, not in State.
    - If this screen has search/filter, put the `searchQuery`/`isSearching` fields directly in this State. Do not split them into a separate screen.
  - `Intent`: a sealed interface that implements `MviIntent`. It covers every user action (Load, Refresh, Retry, ErrorDismissed, ItemClicked, SearchQueryChanged, ClearSearch, etc.).
  - `Effect`: a sealed interface that implements `MviEffect`. Use it for one-time events only (Navigate, ShowToast, ShowSnackbar). Never put these in State.
- **ViewModel.kt**:
  - **Extends `core/mvi/BaseViewModel<State, Intent, Effect>`.** Do not redeclare `MutableStateFlow`/`Channel`/`state`/`effect`. The base class already has them.
  - Gets only UseCases by constructor injection. Never injects the Repository or ApiService directly. Koin resolves them (`viewModel { <Feature>ListViewModel(get(), get()) }` in the Koin module).
  - Overrides only `processIntent(intent: Intent)`.
  - Every branch that calls a UseCase goes through `launchSafely { ... }`, which it inherits from `BaseViewModel`. Never use a raw `viewModelScope.launch`. Never use a manual `try/catch`.
  - When a UseCase returns `Result.Error(failure)`, call `setState { copy(status = UiStatus(failure = failure)) }`. Do not turn the error into a string here. `handleError` does the same thing with `Failure.Unknown(throwable)`.
  - If search exists, debounce search input (300ms) with a cancellable coroutine `Job`. Still launch it through `launchSafely`.
  - Update state immutably with `setState { copy(...) }`. Never mutate it in place.
- **Screen.kt**:
  - A Jetpack Compose function. Keep it as stateless as possible. It takes State + lambda callbacks, or it resolves the ViewModel with `koinViewModel()`.
  - Collect state with **`collectAsStateWithLifecycle()`**. Never use `collectAsState()`.
  - No business logic inside Composables. They only render state and call `processIntent`.
  - Use `LaunchedEffect` to collect one-time Effects (navigation, snackbar) from `viewModel.effect`.
  - All image loading uses Coil's `AsyncImage` composable. Do not use manual Bitmap loading or other caching libs.
  - Compose the body inside `AppScreen`, as in the "Screen usage" example in `.claude/templates/shared/ui-components.md`. Pass:
    - `loading = LoadingState(state.status.isLoading)`
    - `snackbarMessage = state.status.toErrorMessage(onRetry = ..., onDismiss = ...)`
    - `empty = EmptyState(...).takeIf { !state.status.isLoading && <no data> }`
    - `refresh = RefreshState(state.isRefreshing, ...)` for server-data screens, or `refresh = null` for forms/static screens
  - Uses `AppTopBar`, `AppButton`, `AppChip`, `Avatar` from `core/ui/components/`. Never uses the raw Material equivalents.
  - Loads images with Coil's `AsyncImage`.

### presentation/components
- Only Composables reused across THIS feature's own screens (e.g. a Card used in both List and Detail).
- If another feature also needs a component, flag it. Do not move it in advance without confirmation.

### di/`<Feature>Module.kt` (Koin)
- Define as: `val <feature>Module = module { ... }`
- `factory { <Feature>Mapper() }`
- `single<<Feature>Repository> { <Feature>RepositoryImpl(get(), get()) }`
- `single { get<Retrofit>().create(<Feature>ApiService::class.java) }`. Reuse the existing core Retrofit singleton. Do not build a new Retrofit instance per feature.
- `factory { Get<Feature>ListUseCase(get(), get(named("io"))) }`. Add one factory for each UseCase. Always pass the IO dispatcher from `core/di/DispatcherModule.kt` as the second constructor argument.
- `viewModel { <Feature>ListViewModel(get(), get()) }`. Add one for each screen ViewModel.
- Register this module in `App.kt` (`modules(coreModules + listOf(<feature>Module))`). Append it to the existing list.

## Forms, pickers, dialogs and tests
- **Forms**:
  - State holds one `xxxError: ValidationError?` for each field.
  - On submit, the ViewModel checks the fields with `core/util/Validator`.
  - Business rules that need data stay in the UseCase.
  - Each field is `AppOutlinedTextField(error = state.xxxError?.toMessage(), maxLength = InputLimits.X)`.
- **Returning a value** (pickers):
  - The picker screen sends its result back with `navController.setNavResult(NavResultKeys.X, value)` + `popBackStack()` in `AppNavGraph`.
  - The caller registers `entry.NavResultEffect(NavResultKeys.X) { ... }`.
  - See `.claude/templates/shared/core-utils.md`.
- **Dialogs & sheets**: use a `showXDialog` / `showXSheet` State flag with `AppConfirmDialog` / `AppBottomSheet` / `SearchableBottomSheet`.
- **Dates**: the mapper parses server dates with `DateTimeFormat.parseInstant` / `parseDate`. The Screen formats them with `DateTimeFormat.date(...)` etc.
- **Tests** (`app/src/test/java/<package>/feature/<feature_name>/`): generate them with the feature.
  - `presentation/<Screen>ViewModelTest.kt`: one for each ViewModel. Cover success, error, and one Effect/Event path. Use `MainDispatcherRule` + Turbine.
  - `data/mapper/<Feature>MapperTest.kt`: check that a DTO with missing/null fields maps to safe defaults.
  - Use fakes, not a mocking library. `Fake<Feature>Repository` implements the domain repository interface.

## Navigation wiring
- Add the feature's routes to the shared `navigation/Routes.kt` sealed hierarchy, e.g. `@Serializable data object FeatureList : Routes` and `@Serializable data class FeatureDetail(val id: String) : Routes`.
- Register the composables in `navigation/AppNavGraph.kt`: `composable<Routes.FeatureList> { FeatureListScreen() }`, `composable<Routes.FeatureDetail> { backStackEntry -> FeatureDetailScreen(backStackEntry.toRoute()) }`
- This is the only change to a shared/core file. Everything else about the feature stays inside `feature/<feature_name>/`.

## Cross-feature consumption pattern (when relevant)
- If another feature needs this feature's model/data, it must inject this feature's UseCase by Koin constructor injection.
  - Example: `class OtherFeatureUseCase(private val get<Feature>DetailUseCase: Get<Feature>DetailUseCase)`.
  - Wire it in the other feature's own Koin module with `get()`.
- Never duplicate the model in another feature.
- Never import this feature's `data/` or `presentation/` package from outside this feature.

## Constraints
- The ViewModel makes no direct API/DB/Coil calls. It always goes through UseCase → Repository.
- Keep the Dto and the domain model strictly separate. Always map. Never reuse a Dto as the UI/domain model.
- Every State has an explicit `status: UiStatus`. Never rely on nullable data fields to infer status.
- Every new screen must follow the same 3-file pattern (Screen, ViewModel, Contract). No exceptions, no shortcuts.
- Use the `get()` / `getViewModel()` / `koinViewModel()` idioms consistently. Never create dependencies by hand anywhere.
- Every rule in the project's `CLAUDE.md` `## Hard Rules — Never Violate` and `## Architecture Hard Rules` applies here too. These are all forbidden: `!!`, `GlobalScope`, `sealed class` for Effect, `collectAsState()`, raw try/catch in ViewModel, direct Repository calls from ViewModel, Android imports in `domain/`.
- When anything is unclear, ask before generating.

## Deliverable
When given a feature name (e.g. "Order", "Article"), generate this full folder structure and its tests (see "Forms, pickers, dialogs and tests"). Write working, compilable Kotlin + Compose code that follows all rules above. Wire the feature's routes into the shared `navigation/` files.
