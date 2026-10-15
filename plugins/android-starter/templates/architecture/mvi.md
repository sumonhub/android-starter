# Architecture Template: MVI (feature-based, single module)

## Tech stack
- UI: Jetpack Compose
- DI: Koin
- Networking: Retrofit + OkHttp
- Image loading: Coil (coil-compose)
- Async: Kotlin Coroutines + Flow
- Navigation: Jetpack Navigation Compose

## Goal
Set up the full skeleton of a NEW project: core infra, theme, base MVI classes, DI wiring and the navigation shell. Features are added later with the standard feature-generation pattern. Do NOT create any feature yet. Only create the shared foundations.

Note: this is plain MVI, with no Clean Architecture layers. Later, each feature gets a ViewModel plus State/Intent/Effect. The ViewModel talks directly to a single Repository. There is no domain UseCase layer and no Repository interface/impl split. This only affects how features are generated later. It does not change this shared scaffold.

## Defaults
- Base URL: `BuildConfig.BASE_URL`. Each build type (debug / staging / release) has one placeholder with a `// TODO` comment. See `templates/shared/project-setup.md`. `ApiConfig.kt` reads it and never declares a URL
- Local storage: Jetpack **DataStore (Preferences)** by default. No Room and no SQL entities in this shared scaffold
- JSON parsing: **Gson**

These are fixed defaults. Do not ask about them. Use a different choice only if the person explicitly asks for it in their own request.

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
│   ├── mvi/
│   │   ├── MviIntent.kt                    # marker interface: interface MviIntent
│   │   ├── MviState.kt                     # marker interface: interface MviState
│   │   ├── MviEffect.kt                    # marker interface: interface MviEffect
│   │   └── BaseViewModel.kt                # abstract class BaseViewModel<S: MviState, I: MviIntent, E: MviEffect>
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


## Marker Reference

| Marker | Meaning |
|---|---|
| `[CONDITIONAL]` | Create only if actually needed. Skip it otherwise (used in the feature template) |

Component placement:
- Start every new Composable at screen level.
- Move it up to the feature's `presentation/components/` once 2+ screens of that feature use it.
- Move it up to `core/ui/components/` once 2+ features use it.
- Never move a component down.

## Dependency Direction

```
feature/featureX/presentation  ──→  feature/featureX/data
                        ↑
                  core/ (all layers use)
                        ↑
         core/di/ + feature/featureX/di/ (binds everything)
```

- `core/` never imports any feature package
- `presentation/` never imports `data/remote/dto/`. It sees only the mapped model
- A feature never imports another feature's `presentation/` or DTOs. It injects that feature's Repository instead

---

## Step 1: Gradle dependencies
Add to `app/build.gradle.kts`:
- Compose BOM + compose-ui, compose-material3 (1.3+, because AppScreen uses `PullToRefreshBox`), material-icons-core, compose-tooling
- Koin: koin-android, koin-androidx-compose
- Retrofit: retrofit, converter-gson
- AndroidX DataStore: androidx-datastore-preferences
- OkHttp + logging-interceptor
- Coil 3: coil-compose + coil-network-okhttp (without it Coil 3 cannot load images from a URL)
- Kotlin Coroutines: kotlinx-coroutines-android
- Navigation: navigation-compose
- Lifecycle: lifecycle-viewmodel-compose, lifecycle-runtime-ktx, lifecycle-runtime-compose (`collectAsStateWithLifecycle()`)
- Build features: `android { buildFeatures { compose = true; buildConfig = true } }`. Since AGP 8, `buildConfig` is off by default. The network logging and the snackbar "Details" button read `BuildConfig.DEBUG`

## Step 2: Folder structure to create

See "## Project Structure" above. Create exactly that tree, nothing more and nothing less.

## Step 3: Koin setup in App.kt
- `startKoin { androidContext(this@App); modules(coreModules) }`
- `coreModules` = a list that combines networkModule, dispatcherModule, localStorageModule
- Leave this comment: `// Feature modules will be added here, e.g. modules(coreModules + listOf(orderModule, articleModule))`

## Step 4: NetworkModule.kt content
- Provide an OkHttpClient singleton. Add the logging interceptor only `if (BuildConfig.DEBUG)`. The interceptor writes through `AppLogger` and masks passwords / OTPs / tokens
- Provide a Retrofit singleton that uses ApiConfig.BASE_URL
- ALL features reuse this Retrofit instance (`get<Retrofit>().create(XApi::class.java)`). Features never build their own Retrofit

`core/network/NetworkResult.kt` + `SafeApiCall.kt` are the only place where a Retrofit `Response` / exception becomes a result:
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
Loading is UI state (`isLoading` in the screen's State). It is never a `NetworkResult` case. The Screen turns an `Error` into text at render time:
- Use `message` if it is present.
- Otherwise use a `stringResource`, such as `R.string.error_no_internet` for an `IOException` cause, or `R.string.error_generic`.

## Step 5: LocalStorageModule.kt content
- Provide a single `DataStore<Preferences>` singleton (e.g. `context.dataStore` via `preferencesDataStore(name = "app_prefs")`)
- This is the ONLY place that creates a DataStore instance. Features inject it (`get<DataStore<Preferences>>()`). They never build their own
- No Room, no SQL entities, no `@Dao` interfaces in this shared scaffold. If a feature later really needs relational data, decide that for that feature. It is not part of this infra step

## Step 6: BaseViewModel.kt content
```kotlin
abstract class BaseViewModel<S : MviState, I : MviIntent, E : MviEffect>(initialState: S) : ViewModel() {
    private val _state = MutableStateFlow(initialState)
    val state: StateFlow<S> = _state.asStateFlow()

    private val _effect = Channel<E>(Channel.BUFFERED)
    val effect = _effect.receiveAsFlow()

    protected fun setState(reducer: S.() -> S) {
        _state.update(reducer)
    }

    protected fun sendEffect(effect: E) {
        viewModelScope.launch { _effect.send(effect) }
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

    abstract fun processIntent(intent: I)
}
```

Later, each feature's ViewModel calls its Repository directly from `processIntent`. There is no UseCase layer in between.

## Step 7: UI components (`core/ui/components/`)
Create every component in `templates/shared/ui-components.md` exactly as written there: `AppScreen`, `AppSnackbar`, `AppTopBar`, `AppButton`, `AppChip`, `Avatar`, `LoadingIndicator`, `ErrorView`, `EmptyView`. Also create the `NetworkResult.Error?.toErrorMessage()` converter in `AppSnackbar.kt`. That file also lists the Dimens tokens and string resources the components need.

## Step 8: NavGraph shell
- `AppNavGraph.kt`: an empty `NavHost` with a placeholder `startDestination` "home". The system splash screen comes first, so there is no splash route
- Comment: `// composable(Routes.FeatureX.route) { FeatureXScreen() } — added when features are generated`

## Step 9: Project setup + core utilities
Apply all of `templates/shared/project-setup.md`: environments (debug/staging/release `BASE_URL`), release signing from `keystore.properties`, R8 keep rules for DTOs, splash screen, unit-test setup (`MainDispatcherRule`), ktlint + `.editorconfig`, and the GitHub Actions workflow. Then create every file in `templates/shared/core-utils.md`: `core/util/Validator.kt`, `InputLimits.kt`, `DateTimeFormat.kt` and `navigation/NavResult.kt`.

## Step 10: Verify
- `./gradlew ktlintCheck testDebugUnitTest assembleDebug` passes. The app launches and shows the system splash screen first
- The project builds successfully with zero features
- The app launches to a blank themed screen inside `AppScreen`. This proves that Koin, Theme, the NavGraph shell and the components are wired correctly

## Constraints
- The generic rules in `templates/shared/hard-rules.md` (Hard Rules, Do Not Touch, Security) also apply to this architecture
- Do not create any feature folder content beyond the empty `feature/` placeholder
- Do not hardcode any business logic, model, or screen here. This step is infra only
- Every core module must be feature-agnostic. It must not import anything feature-specific
- No domain/UseCase layer anywhere in this scaffold. Features will call their Repository directly from the ViewModel
- Every Composable screen includes a `@Preview` composable with representative/sample state. Reusable `core/ui/components/` components include one too, where practical

## CLAUDE.md content for this architecture

> The generic rules (`Hard Rules — Never Violate`, `Do Not Touch`, `Security`) live in `templates/shared/hard-rules.md`. They also apply to this architecture and are not repeated here.
> `/android-starter:init-project` Step 5 copies the `## Architecture Hard Rules` block below into the project's `CLAUDE.md` verbatim, right after those generic blocks. Keep exactly this one `##` block here. Anything outside it is not copied.

```markdown
## Architecture Hard Rules

> Plain MVI (no Clean layer). These add to the generic Hard Rules above.

- **ViewModel calls the feature's Repository directly**. No UseCase layer, no Repository interface/impl split
- **DTOs never reach `presentation/`**. The Repository maps every DTO to the feature's model before it returns
- **Repositories call the API only through `safeApiCall` from `core/network`**. Never write your own `try/catch` or `response.isSuccessful` check
- **ViewModel coroutines always go through `launchSafely { ... }`**, never raw `viewModelScope.launch`. **`handleError` override is mandatory** in every feature ViewModel: `setState { copy(isLoading = false, error = NetworkResult.Error(cause = throwable)) }`. Without it, the base no-op would silently swallow the error
- **Never use raw try/catch in a ViewModel**. `safeApiCall` already turned failures into `NetworkResult.Error`. The ViewModel checks the result with `when` and updates state
- **Errors are converted to text in the Screen at render time**. State holds `error: NetworkResult.Error?`, never a display string
- **ViewModel extends `core/mvi/BaseViewModel<State, Intent, Effect>`**. It never redeclares `MutableStateFlow`/`Channel`
- **Never use `sealed class` for a feature's Effect or Intent**. Always use `sealed interface XEffect : MviEffect` / `sealed interface XIntent : MviIntent`. The core markers stay plain `interface`
- **MVI contract rules:**
  - **State**: one immutable `data class XState : MviState` per screen. Use only `val` + immutable collections, with explicit `isLoading` and `error` fields. Change it **only** with `setState { copy(...) }`
  - **Intent**: the Screen talks to the ViewModel **only** through `viewModel.processIntent(...)`
  - **Effect**: **one-shot events only** (navigate, toast). Anything the screen must still show after rotation belongs in State
  - **Screen** collects `effect` only inside `LaunchedEffect(Unit) { viewModel.effect.collect { ... } }`
```
