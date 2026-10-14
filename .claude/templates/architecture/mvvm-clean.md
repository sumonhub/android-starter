# Architecture Template: MVVM + Clean Architecture (feature-based, single module)

## Tech stack
- UI: Jetpack Compose
- DI: Koin
- Networking: Retrofit + OkHttp
- Image loading: Coil (coil-compose)
- Async: Kotlin Coroutines + Flow
- Navigation: Jetpack Navigation Compose

## Goal
Set up the entire skeleton of a NEW project — core infra, theme, base MVVM classes, DI wiring, navigation shell — so that features can be dropped in afterward using the standard feature-generation pattern. Do NOT create any feature yet. Only scaffold shared foundations.

Note: Clean layering (domain UseCases, data Repository impl, presentation ViewModel) is applied inside each feature later, when features are generated. This shared scaffold only provides the base MVVM contract and infra that every feature's presentation layer will build on.

## Defaults
- Base URL: `BuildConfig.BASE_URL`, one placeholder per build type (debug / staging / release) with `// TODO` comments — see `.claude/templates/shared/project-setup.md`. `ApiConfig.kt` reads it, never declares a URL
- Local storage: Jetpack **DataStore (Preferences)** by default — no Room, no SQL entities in this shared scaffold
- JSON parsing: **Gson**

These are fixed defaults — do not ask about them. Only deviate if the person explicitly states a different choice in their own request.

## Project Structure

```
app/src/main/java/com/example/app/
│
├── App.kt                                  # Application class, starts Koin
├── MainActivity.kt                         # setContent { AppTheme { AppNavGraph() } }
│
├── core/
│   ├── di/
│   │   ├── NetworkModule.kt                # Koin module: Retrofit, OkHttp, Gson singletons
│   │   ├── LocalStorageModule.kt           # Koin module: provides DataStore<Preferences> singleton (default local storage, no Room)
│   │   └── DispatcherModule.kt             # Koin module: single(named("io")) / named("main") / named("default") CoroutineDispatchers — only file allowed to reference Dispatchers.*
│   ├── network/
│   │   ├── ApiConfig.kt                    # BASE_URL = BuildConfig.BASE_URL (per build type), timeouts
│   │   ├── NetworkResult.kt                # sealed interface: Success<T>(data) / Error(code, message, cause) — no Loading case
│   │   └── SafeApiCall.kt                  # safeApiCall { api.call() } → NetworkResult; rethrows CancellationException
│   ├── mvvm/
│   │   ├── UiState.kt                      # marker interface: interface UiState
│   │   ├── UiEvent.kt                      # marker interface: interface UiEvent (one-off events: navigate, show snackbar, etc.)
│   │   └── BaseViewModel.kt                # abstract class BaseViewModel<S: UiState>
│   ├── ui/
│   │   ├── theme/
│   │   │   ├── Color.kt
│   │   │   ├── Typography.kt
│   │   │   ├── Dimens.kt                   # spacing/sizing tokens — spacingXSmall/Small/Medium/Large; no hardcoded dp in screens
│   │   │   ├── Shape.kt
│   │   │   └── Theme.kt                    # @Composable AppTheme { ... }
│   │   └── components/                 # exactly as shared/ui-components.md: AppScreen, AppSnackbar (+ NetworkResult.Error.toErrorMessage()),
│   │                                       # AppTopBar, AppButton, AppChip, Avatar, LoadingIndicator, ErrorView, EmptyView
│   └── util/
│       ├── AppLogger.kt                    # central logger — d(tag, message) + e(tag, message, throwable); no-op in release
│       ├── Validator.kt / InputLimits.kt / DateTimeFormat.kt  # exactly as shared/core-utils.md
│       └── Extensions.kt
│
├── navigation/
│   ├── AppNavGraph.kt                      # NavHost shell, currently empty/placeholder route
│   ├── NavResult.kt                        # setNavResult / NavResultEffect + NavResultKeys — shared/core-utils.md
│   └── Routes.kt                           # route constants; features add their own later
│
└── feature/                                # empty — features added later via feature-generation pattern
    └── .gitkeep
```

Each feature generated later will internally follow: `data/` (RepositoryImpl, DTOs, remote/local sources) → `domain/` (Repository interface, UseCases, domain models) → `presentation/` (ViewModel extending `BaseViewModel`, UiState, Composable screen). This shared scaffold does not create any of that — only the base classes those layers will extend.


## Marker Reference

| Marker | Meaning |
|---|---|
| `[CONDITIONAL]` | Create only if actually needed — skip otherwise (used in the feature template) |

Component placement: start every new Composable at screen level, promote to the feature's `presentation/components/` once 2+ screens of that feature use it, and to `core/ui/components/` once 2+ features use it. Never move a component down.

## Dependency Direction

```
feature/featureX/presentation  ──→  feature/featureX/domain  ←──  feature/featureX/data
                                              ↑
                                        core/ (all layers use)
                                              ↑
                               core/di/ + feature/featureX/di/ (binds everything)
```

- `domain/` never imports `data/`, `presentation/`, or Android/Retrofit/Gson classes — the only `core/` type it uses is `NetworkResult`
- `core/` never imports any feature package
- DTOs never leave `data/`
- A feature never imports another feature's `data/` or `presentation/` — it injects that feature's UseCase or Repository interface instead

---

## Step 1: Gradle dependencies
Add to `app/build.gradle.kts`:
- Compose BOM + compose-ui, compose-material3 (1.3+ — AppScreen uses `PullToRefreshBox`), material-icons-core, compose-tooling
- Koin: koin-android, koin-androidx-compose
- Retrofit: retrofit, converter-gson
- AndroidX DataStore: androidx-datastore-preferences
- OkHttp + logging-interceptor
- Coil 3: coil-compose + coil-network-okhttp (without it Coil 3 cannot load images from a URL)
- Kotlin Coroutines: kotlinx-coroutines-android
- Navigation: navigation-compose
- Lifecycle: lifecycle-viewmodel-compose, lifecycle-runtime-ktx, lifecycle-runtime-compose (`collectAsStateWithLifecycle()`)
- Build features: `android { buildFeatures { compose = true; buildConfig = true } }` — `buildConfig` is off by default since AGP 8, and the network logging + snackbar "Details" button read `BuildConfig.DEBUG`

## Step 2: Folder structure to create

See "## Project Structure" above — create exactly that tree, nothing more, nothing less.

## Step 3: Koin setup in App.kt
- `startKoin { androidContext(this@App); modules(coreModules) }`
- `coreModules` = list combining networkModule, dispatcherModule, localStorageModule
- Leave comment: `// Feature modules will be added here, e.g. modules(coreModules + listOf(orderModule, articleModule))`

## Step 4: NetworkModule.kt content
- Provide OkHttpClient singleton — logging interceptor added only `if (BuildConfig.DEBUG)`, writing through `AppLogger`, with passwords / OTPs / tokens masked
- Provide Retrofit singleton using ApiConfig.BASE_URL
- This Retrofit instance is reused by ALL features (`get<Retrofit>().create(XApi::class.java)`) — features never build their own Retrofit
- Each feature's `RepositoryImpl` (data layer) is what calls this Retrofit-built API service — never the ViewModel or UseCase directly

`core/network/NetworkResult.kt` + `SafeApiCall.kt` — the only place a Retrofit `Response` / exception becomes a result:
```kotlin
sealed interface NetworkResult<out T> {
    data class Success<T>(val data: T) : NetworkResult<T>
    data class Error(val code: Int? = null, val message: String? = null, val cause: Throwable? = null) : NetworkResult<Nothing>
}

suspend fun <T> safeApiCall(call: suspend () -> Response<T>): NetworkResult<T> = try {
    val response = call()
    val body = response.body()
    if (response.isSuccessful && body != null) NetworkResult.Success(body)
    else NetworkResult.Error(code = response.code(), message = response.message().ifBlank { null })
} catch (e: CancellationException) {
    throw e
} catch (e: Exception) {
    NetworkResult.Error(cause = e)
}
```
Loading is UI state (`isLoading` in the screen's State), never a `NetworkResult` case. The Screen turns an `Error` into text at render time (`message` if present, otherwise a `stringResource` such as `R.string.error_no_internet` for an `IOException` cause or `R.string.error_generic`).

## Step 5: LocalStorageModule.kt content
- Provide a single `DataStore<Preferences>` singleton (e.g. `context.dataStore` via `preferencesDataStore(name = "app_prefs")`)
- This is the ONLY place a DataStore instance is created — features inject it (`get<DataStore<Preferences>>()`) rather than building their own
- No Room, no SQL entities, no `@Dao` interfaces in this shared scaffold — if a feature later genuinely needs relational data, that's a per-feature decision, not part of this infra step

## Step 6: BaseViewModel.kt content
```kotlin
abstract class BaseViewModel<S : UiState>(initialState: S) : ViewModel() {
    private val _state = MutableStateFlow(initialState)
    val state: StateFlow<S> = _state.asStateFlow()

    private val _event = Channel<UiEvent>(Channel.BUFFERED)
    val event = _event.receiveAsFlow()

    protected fun setState(reducer: S.() -> S) {
        _state.update(reducer)
    }

    protected fun sendEvent(event: UiEvent) {
        viewModelScope.launch { _event.send(event) }
    }

    /** Every coroutine a feature ViewModel starts goes through here — an unexpected exception (e.g. a mapper failing on bad data) lands in handleError instead of crashing the app. */
    protected fun launchSafely(
        onError: (Throwable) -> Unit = ::handleError,
        block: suspend () -> Unit,
    ) {
        viewModelScope.launch {
            try {
                block()
            } catch (e: CancellationException) {
                throw e // normal teardown — not an error to show
            } catch (e: Exception) {
                AppLogger.e(tag = this@BaseViewModel.javaClass.simpleName, message = "Unhandled error in launchSafely", throwable = e)
                onError(e)
            }
        }
    }

    /** Override in every feature ViewModel: setState { copy(isLoading = false, error = NetworkResult.Error(cause = throwable)) }. */
    protected open fun handleError(throwable: Throwable) {}
}
```

Unlike MVI, there is no `processIntent` contract — each feature ViewModel exposes its own explicit public functions (e.g. `fun onSubmitClicked()`), which internally call the feature's UseCases and call `setState { ... }` with the result.

## Step 7: UI components (`core/ui/components/`)
Create every component in `.claude/templates/shared/ui-components.md` exactly as written there — `AppScreen`, `AppSnackbar`, `AppTopBar`, `AppButton`, `AppChip`, `Avatar`, `LoadingIndicator`, `ErrorView`, `EmptyView` — plus its `NetworkResult.Error?.toErrorMessage()` converter in `AppSnackbar.kt`. That file also lists the Dimens tokens and string resources the components need.

## Step 8: NavGraph shell
- `AppNavGraph.kt`: empty `NavHost` with a placeholder `startDestination` "home" (the system splash screen comes first — no splash route)
- Comment: `// composable(Routes.FeatureX.route) { FeatureXScreen() } — added when features are generated`

## Step 9: Project setup + core utilities
Apply `.claude/templates/shared/project-setup.md` in full — environments (debug/staging/release `BASE_URL`), release signing from `keystore.properties`, R8 keep rules for DTOs, splash screen, unit-test setup (`MainDispatcherRule`), ktlint + `.editorconfig`, and the GitHub Actions workflow. Then create every file in `.claude/templates/shared/core-utils.md` — `core/util/Validator.kt`, `InputLimits.kt`, `DateTimeFormat.kt` and `navigation/NavResult.kt`.

## Step 10: Verify
- `./gradlew ktlintCheck testDebugUnitTest assembleDebug` passes, and the app launches showing the system splash screen first
- Project builds successfully with zero features present
- App launches to a blank themed screen composed inside `AppScreen` (proves Koin + Theme + NavGraph shell + components are wired correctly)

## Constraints
- The generic rules in `.claude/templates/shared/hard-rules.md` (Hard Rules, Do Not Touch, Security) apply to this architecture too
- Do not create any feature folder content beyond the empty `feature/` placeholder
- Do not hardcode any business logic, model, or screen here — this step is infra-only
- Every core module must be feature-agnostic — no imports of anything feature-specific
- `core/mvvm/BaseViewModel.kt` must not know about any concrete feature's UseCases or Repository
- Every Composable screen (and reusable `core/ui/components/` component where practical) includes a `@Preview` composable with representative/sample state

## CLAUDE.md content for this architecture

> The generic rules (`Hard Rules — Never Violate`, `Do Not Touch`, `Security`) live in `.claude/templates/shared/hard-rules.md` and apply to this architecture too — they are not repeated here.
> `/init-project` Step 5 copies the `## Architecture Hard Rules` block below into the project's `CLAUDE.md` verbatim, right after those generic blocks. Keep exactly this one `##` block here — anything outside it is not copied.

```markdown
## Architecture Hard Rules

> MVVM + Clean Architecture. These add to the generic Hard Rules above.

- **Never call Repository or ApiService from a ViewModel** — always through a UseCase
- **Never add Android/Retrofit/Gson imports in `domain/`** — pure Kotlin only; `NetworkResult` is the domain's result type
- **Repository interface lives in `domain/repository/`, its implementation in `data/repository/`** — DTOs never leave `data/`
- **UseCase**: one class per operation, `suspend operator fun invoke()`, depends only on Repository interfaces; business rules live here, never in the ViewModel or RepositoryImpl (field-format checks with `Validator` stay in the ViewModel)
- **Repositories call the API only through `safeApiCall` from `core/network`** — never a hand-written `try/catch` or `response.isSuccessful` check
- **ViewModel coroutines always go through `launchSafely { ... }`**, never raw `viewModelScope.launch`; **`handleError` override is mandatory** in every feature ViewModel: `setState { copy(isLoading = false, error = NetworkResult.Error(cause = throwable)) }` — the base no-op would swallow the error silently
- **Never use raw try/catch in a ViewModel** — `safeApiCall` already turned failures into `NetworkResult.Error`; the ViewModel branches on the result with `when` and updates state
- **Errors are converted to text in the Screen at render time** — State holds `error: NetworkResult.Error?`, never a display string
- **ViewModel extends `core/mvvm/BaseViewModel<UiState>`** — never redeclares `MutableStateFlow`/`Channel`, never exposes a mutable flow
- **One public ViewModel function per user action** (`onRefresh()`, `onItemClicked(id)`) — no `processIntent`, and the Screen never mutates ViewModel fields
- **UiState** — one immutable `data class XUiState : UiState` per screen, only `val` + immutable collections, with explicit `isLoading` and `error` fields. Changed **only** via `setState { copy(...) }`
- **UiEvent** — `sealed interface XUiEvent : UiEvent`, **one-shot events only** (navigate, toast), sent with `sendEvent(...)` and collected only inside `LaunchedEffect(Unit) { viewModel.event.collect { ... } }`. Anything the screen must still show after rotation belongs in UiState
```
