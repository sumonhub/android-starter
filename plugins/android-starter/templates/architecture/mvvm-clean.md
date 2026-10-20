# Architecture Template: MVVM + Clean Architecture (feature-based, single module)

## Tech stack
- UI: Jetpack Compose
- DI: Koin
- Networking: Retrofit + OkHttp + Gson
- Image loading: Coil (coil-compose)
- Async: Kotlin Coroutines + Flow
- Navigation: Jetpack Navigation Compose (string routes)
- Local storage: DataStore (Preferences). This is the default. The shared scaffold has no Room.

## Goal
Set up the full skeleton of a NEW project: core infra, theme, base MVVM classes, DI wiring and the navigation shell. Features are added later with the standard feature-generation pattern. Do NOT create any feature yet. Only create the shared foundations.

Note: each feature gets Clean layering (domain UseCases, data Repository impl, presentation ViewModel) later, when it is generated. This shared scaffold only provides the base MVVM contract and the infra. Every feature's presentation layer builds on them.

## Defaults
- Base URL: `BuildConfig.BASE_URL`. Each build type (debug / staging / release) has one placeholder with a `// TODO` comment. See `templates/shared/project-setup.md`. `ApiConfig.kt` reads it and never declares a URL
- Local storage: Jetpack **DataStore (Preferences)** by default. No Room and no SQL entities in this shared scaffold
- JSON parsing: **Gson**

These are fixed defaults. Do not ask about them. Use a different choice only if the person explicitly asks for it in their own request.

## Project Structure

```
app/src/main/java/com/example/app/
│
├── App.kt                                  # Application class — startKoin { modules(coreModules) } — shared/core-infra.md
├── MainActivity.kt                         # installSplashScreen + setContent { AppTheme { AppNavGraph() } } — shared/core-infra.md
│
├── core/
│   ├── di/
│   │   ├── NetworkModule.kt                # Koin: AuthTokenProvider + OkHttpClient + Retrofit singletons (built by NetworkClient)
│   │   ├── LocalStorageModule.kt           # Koin: DataStore<Preferences> singleton — only instance in the app
│   │   ├── DispatcherModule.kt             # Koin: named("io") / ("main") / ("default") CoroutineDispatchers — only file allowed to reference Dispatchers.*
│   │   └── CoreModules.kt                  # val coreModules = listOf(networkModule, localStorageModule, dispatcherModule)
│   ├── network/
│   │   ├── ApiConfig.kt                    # BASE_URL = BuildConfig.BASE_URL (per build type), timeouts
│   │   ├── AuthTokenProvider.kt            # in-memory bearer token + sessionExpired StateFlow — read by NetworkClient
│   │   ├── NetworkClient.kt                # builds OkHttp (auth header → 401 → debug-only redacted logging) + Retrofit
│   │   ├── NetworkResult.kt                # sealed interface: Success<T>(data) / Error(code, message, cause) + map() — pure Kotlin, also the domain's result type
│   │   └── SafeApiCall.kt                  # safeApiCall / safeUnitApiCall → NetworkResult; rethrows CancellationException — data layer only
│   ├── mvvm/
│   │   ├── UiState.kt                      # marker interface: interface UiState
│   │   ├── UiEvent.kt                      # marker interface: interface UiEvent (one-off events: navigate, show snackbar, etc.)
│   │   └── BaseViewModel.kt                # abstract class BaseViewModel<S: UiState> — state, event, setState, sendEvent, launchSafely
│   ├── model/                              # [CONDITIONAL] shared domain models used by 2+ features (e.g. User) — not created at scaffold time
│   ├── ui/
│   │   ├── theme/
│   │   │   ├── Color.kt
│   │   │   ├── Typography.kt
│   │   │   ├── Dimens.kt                   # spacing/sizing tokens — spacingXSmall/Small/Medium/Large; no hardcoded dp in screens
│   │   │   ├── Shape.kt
│   │   │   └── Theme.kt                    # @Composable AppTheme { ... }
│   │   └── components/                     # exactly as shared/ui-components.md: AppScreen, AppSnackbar (+ NetworkResult.Error.toErrorMessage()),
│   │                                       # AppTopBar, AppButton, AppChip, Avatar, LoadingIndicator, ErrorView, EmptyView, ...
│   └── util/
│       ├── AppLogger.kt                    # central logger — d(tag, message) + e(tag, message, throwable); no-op in release
│       ├── Validator.kt / InputLimits.kt / DateTimeFormat.kt  # exactly as shared/core-utils.md
│       └── Extensions.kt
│
├── navigation/
│   ├── AppNavGraph.kt                      # NavHost(startDestination = Routes.HOME) with a blank placeholder Home
│   ├── NavResult.kt                        # setNavResult / NavResultEffect + NavResultKeys — shared/core-utils.md
│   └── Routes.kt                           # object Routes { const val HOME = "home" } — features add their own later
│
└── feature/                                # empty at scaffold time — features added later via feature-generation pattern
    ├── .gitkeep
    │
    └── featureX/                           # [FEATURE TEMPLATE] not created now — the shape templates/feature/mvvm-clean.md generates
        ├── domain/                         # pure Kotlin — no Android/Retrofit/Gson imports
        │   ├── model/FeatureX.kt           # domain model — decoupled from the DTO
        │   ├── repository/FeatureXRepository.kt   # interface, returns NetworkResult<T>
        │   └── usecase/GetFeatureXListUseCase.kt  # one class per operation — suspend operator fun invoke()
        ├── data/
        │   ├── remote/
        │   │   ├── FeatureXApiService.kt   # Retrofit interface — provided via get<Retrofit>().create(...) in DI
        │   │   └── dto/FeatureXDto.kt      # raw JSON model — never leaves data/
        │   ├── mapper/FeatureXMapper.kt    # fun FeatureXDto.toDomain(): FeatureX
        │   └── repository/FeatureXRepositoryImpl.kt  # implements the domain interface — safeApiCall { ... }.map { it.toDomain() }
        ├── presentation/
        │   ├── components/                 # [CONDITIONAL]
        │   ├── FeatureXUiState.kt          # UiState data class + UiEvent sealed interface
        │   ├── FeatureXViewModel.kt        # extends BaseViewModel — calls UseCases only
        │   └── FeatureXScreen.kt           # stateful Screen + stateless Content + @Preview
        └── di/FeatureXModule.kt            # val featureXModule = module { api + repository + useCases + viewModel }
```


## Marker Reference

| Marker | Meaning |
|---|---|
| `[CONDITIONAL]` | Create only if actually needed. Skip it otherwise |
| `[FEATURE TEMPLATE]` | Not created at scaffold time. Shows the pattern for generating features later |

Component placement:
- Start every new Composable at screen level.
- Move it up to the feature's `presentation/components/` once 2+ screens of that feature use it.
- Move it up to `core/ui/components/` once 2+ features use it.
- Never move a component down.

## Dependency Direction

```
feature/featureX/presentation  ──→  feature/featureX/domain  ←──  feature/featureX/data
                                              ↑
                                        core/ (all layers use)
                                              ↑
                               core/di/ + feature/featureX/di/ (binds everything)
```

- `domain/` never imports `data/`, `presentation/`, or Android/Retrofit/Gson classes. The only `core/` type it uses is `NetworkResult`
- `core/` never imports any feature package
- DTOs never leave `data/`
- A feature never imports another feature's `data/` or `presentation/`. It injects that feature's UseCase or Repository interface instead

---

## Step 1: Gradle dependencies
Add to `app/build.gradle.kts`:
- Compose BOM + compose-ui, compose-material3 (1.3+, because AppScreen uses `PullToRefreshBox`), material-icons-core, compose-tooling, activity-compose
- Koin: koin-android, koin-androidx-compose
- Retrofit: retrofit, converter-gson
- AndroidX DataStore: androidx-datastore-preferences
- OkHttp: okhttp, logging-interceptor
- Coil 3: coil-compose + coil-network-okhttp (without it Coil 3 cannot load images from a URL)
- Kotlin Coroutines: kotlinx-coroutines-android
- Navigation: navigation-compose
- Lifecycle: lifecycle-viewmodel-compose, lifecycle-runtime-ktx, lifecycle-runtime-compose (`collectAsStateWithLifecycle()`)
- Build features: `android { buildFeatures { compose = true; buildConfig = true } }`. Since AGP 8, `buildConfig` is off by default. The network logging and the snackbar "Details" button read `BuildConfig.DEBUG`

## Step 2: Folder structure to create

See "## Project Structure" above. Create exactly that tree, nothing more and nothing less. The `featureX/` block is only the pattern for later, and `core/model/` is [CONDITIONAL]. At scaffold time `feature/` holds only `.gitkeep`.

## Step 3: Shared core infra
Create every file in `templates/shared/core-infra.md`, exactly as written there:
- `App.kt`, `MainActivity.kt` (package root)
- `core/di/`: `CoreModules.kt`, `NetworkModule.kt`, `LocalStorageModule.kt`, `DispatcherModule.kt`
- `core/network/`: `ApiConfig.kt`, `AuthTokenProvider.kt`, `NetworkClient.kt`
- `core/util/`: `AppLogger.kt`, `Extensions.kt`
- `core/ui/theme/`: `Color.kt`, `Typography.kt`, `Shape.kt`, `Dimens.kt`, `Theme.kt`

That file also holds the rules for the network client, local storage and dispatchers.

## Step 4: Network results (`core/network/`)

**Rules**
- Only a feature's `RepositoryImpl` (data layer) calls the Retrofit-built API service. The ViewModel or UseCase never calls it directly.
- `NetworkResult.kt` + `SafeApiCall.kt` are the **only** place where a Retrofit `Response` / exception becomes a result. Never write a `try/catch` or a `response.isSuccessful` check by hand in a RepositoryImpl.
- `NetworkResult.kt` is pure Kotlin (no Retrofit/Android imports), so `domain/` may use it as its result type. `SafeApiCall.kt` imports Retrofit, so only `data/` uses it.

**`NetworkResult.kt`**
```kotlin
sealed interface NetworkResult<out T> {
    data class Success<T>(val data: T) : NetworkResult<T>
    data class Error(val code: Int? = null, val message: String? = null, val cause: Throwable? = null) : NetworkResult<Nothing>
}

/** Maps the success value (e.g. DTO → domain model) and passes an Error through untouched. */
inline fun <T, R> NetworkResult<T>.map(transform: (T) -> R): NetworkResult<R> = when (this) {
    is NetworkResult.Success -> NetworkResult.Success(transform(data))
    is NetworkResult.Error -> this
}
```
There is no `Loading` case. A network call either succeeds or fails. Loading is UI state (`isLoading` in the screen's UiState).

**`SafeApiCall.kt`**. Import `kotlin.coroutines.cancellation.CancellationException`.
```kotlin
suspend fun <T> safeApiCall(call: suspend () -> Response<T>): NetworkResult<T> = try {
    val response = call()
    val body = response.body()
    if (response.isSuccessful && body != null) {
        NetworkResult.Success(body)
    } else {
        NetworkResult.Error(code = response.code(), message = response.message().ifBlank { null })
    }
} catch (e: CancellationException) {
    throw e // cancellation must stop the caller's coroutine — never an Error
} catch (e: Exception) {
    NetworkResult.Error(cause = e)
}

/** For endpoints whose success carries no body (e.g. 204 No Content on a DELETE). */
suspend fun safeUnitApiCall(call: suspend () -> Response<Unit>): NetworkResult<Unit> = try {
    val response = call()
    if (response.isSuccessful) {
        NetworkResult.Success(Unit)
    } else {
        NetworkResult.Error(code = response.code(), message = response.message().ifBlank { null })
    }
} catch (e: CancellationException) {
    throw e
} catch (e: Exception) {
    NetworkResult.Error(cause = e)
}
```
The Screen turns an `Error` into text at render time, with the `NetworkResult.Error?.toErrorMessage()` converter from `shared/ui-components.md`.

**Domain + data usage — every feature follows this shape:**
```kotlin
// domain/repository/OrderRepository.kt — interface only
interface OrderRepository {
    suspend fun getOrders(): NetworkResult<List<Order>>
    suspend fun cancelOrder(id: String): NetworkResult<Unit>
}

// domain/usecase/GetOrdersUseCase.kt — one class, one operation; business rules live here
class GetOrdersUseCase(private val repository: OrderRepository) {
    suspend operator fun invoke(): NetworkResult<List<Order>> = repository.getOrders()
}

// data/repository/OrderRepositoryImpl.kt
class OrderRepositoryImpl(private val api: OrderApiService) : OrderRepository {
    override suspend fun getOrders(): NetworkResult<List<Order>> =
        safeApiCall { api.getOrders() }.map { dtos -> dtos.map { it.toDomain() } }

    override suspend fun cancelOrder(id: String): NetworkResult<Unit> =
        safeUnitApiCall { api.cancelOrder(id) }
}
```

## Step 5: MVVM markers + BaseViewModel (`core/mvvm/`)

**Markers** are plain `interface`s, never `sealed`. Kotlin only lets code in the same package implement a sealed type. So `feature/*` could never implement a sealed core marker.
```kotlin
interface UiState
interface UiEvent
```

**`BaseViewModel.kt`**. Import `kotlin.coroutines.cancellation.CancellationException`.
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

Unlike MVI, there is no `processIntent` contract. Each feature ViewModel exposes its own explicit public functions and calls the feature's UseCases.

**Feature usage — every screen follows this shape (UiState + ViewModel + Screen):**
```kotlin
// OrderListUiState.kt — UiState + UiEvent in one file
data class OrderListUiState(
    val isLoading: Boolean = false,
    val isRefreshing: Boolean = false,
    val error: NetworkResult.Error? = null,
    val orders: List<Order> = emptyList(),        // immutable List, never MutableList / var
) : UiState

sealed interface OrderListUiEvent : UiEvent {
    data class OpenOrder(val id: String) : OrderListUiEvent
}

// OrderListViewModel.kt — UseCases only, one public function per user action
class OrderListViewModel(
    private val getOrders: GetOrdersUseCase,
) : BaseViewModel<OrderListUiState>(OrderListUiState()) {

    init {
        load()
    }

    fun onRefresh() = load(isRefresh = true)

    fun onRetry() = load()

    fun onErrorDismissed() = setState { copy(error = null) }

    fun onOrderClicked(id: String) = sendEvent(OrderListUiEvent.OpenOrder(id))

    private fun load(isRefresh: Boolean = false) = launchSafely {
        setState { copy(isLoading = !isRefresh, isRefreshing = isRefresh, error = null) }
        when (val result = getOrders()) {
            is NetworkResult.Success -> setState { copy(isLoading = false, isRefreshing = false, orders = result.data) }
            is NetworkResult.Error -> setState { copy(isLoading = false, isRefreshing = false, error = result) }
        }
    }

    override fun handleError(throwable: Throwable) {
        setState { copy(isLoading = false, isRefreshing = false, error = NetworkResult.Error(cause = throwable)) }
    }
}

// OrderListScreen.kt — events collected once, in a LaunchedEffect
@Composable
fun OrderListScreen(onOpenOrder: (String) -> Unit, viewModel: OrderListViewModel = koinViewModel()) {
    val state by viewModel.state.collectAsStateWithLifecycle()
    LaunchedEffect(Unit) {
        viewModel.event.collect { event ->
            when (event) {
                is OrderListUiEvent.OpenOrder -> onOpenOrder(event.id)
            }
        }
    }
    OrderListContent(
        state = state,
        onRefresh = viewModel::onRefresh,
        onRetry = viewModel::onRetry,
        onErrorDismissed = viewModel::onErrorDismissed,
        onOrderClick = viewModel::onOrderClicked,
    )
}
// OrderListContent(...) is stateless, composes AppScreen as in shared/ui-components.md "Screen usage", and has a @Preview.

// di/OrderModule.kt
val orderModule = module {
    single { get<Retrofit>().create(OrderApiService::class.java) }
    single<OrderRepository> { OrderRepositoryImpl(get()) }
    factory { GetOrdersUseCase(get()) }
    viewModel { OrderListViewModel(get()) }
}
```

## Step 6: UI components (`core/ui/components/`)
Create every component in `templates/shared/ui-components.md` exactly as written there: `AppScreen`, `AppSnackbar`, `AppTopBar`, `AppButton`, `AppChip`, `Avatar`, `LoadingIndicator`, `ErrorView`, `EmptyView` and the rest of its list. Also create the **MVI, MVVM, MVVM + Clean** converter `NetworkResult.Error?.toErrorMessage()` in `AppSnackbar.kt`. That file also lists the Dimens tokens and string resources the components need.

Screens follow the shared file's "Screen usage" example:
- pass `LoadingState(isLoading = state.isLoading)` to `loading`
- pass `state.error.toErrorMessage(onRetry = onRetry, onDismiss = onErrorDismissed)` to `snackbarMessage`

## Step 7: Project setup + core utilities
Apply all of `templates/shared/project-setup.md`: environments (debug/staging/release `BASE_URL`), release signing from `keystore.properties`, R8 keep rules for DTOs, splash screen, unit-test setup (`MainDispatcherRule`), ktlint + `.editorconfig`, and the GitHub Actions workflow. Then create every file in `templates/shared/core-utils.md`: `core/util/Validator.kt`, `InputLimits.kt`, `DateTimeFormat.kt` and `navigation/NavResult.kt`.

## Step 8: Navigation shell

**`Routes.kt`**
```kotlin
object Routes {
    const val HOME = "home"
    // Features add their own routes here later, e.g.:
    // const val ORDER_LIST = "order_list"
    // const val ORDER_DETAIL = "order_detail/{id}"
}
```

**`AppNavGraph.kt`**
```kotlin
@Composable
fun AppNavGraph(navController: NavHostController = rememberNavController()) {
    NavHost(navController = navController, startDestination = Routes.HOME) {
        composable(Routes.HOME) { HomePlaceholder() }
        // composable(Routes.FEATURE_X) { FeatureXScreen() } — added when features are generated
    }
}

/** Blank themed screen until the first feature replaces the start destination. */
@Composable
private fun HomePlaceholder() {
    AppScreen(refresh = null) {}
}
```
The system splash screen comes first, so there is no splash route.

## Step 9: Verify
- `./gradlew ktlintCheck testDebugUnitTest assembleDebug` passes. The app launches and shows the system splash screen first
- The project builds successfully with zero features
- The app launches to a blank themed screen inside `AppScreen`. This proves that Koin, Theme, the NavGraph shell and the components are wired correctly
- `core/mvvm/BaseViewModel.kt` and `core/network/NetworkResult.kt` compile with no feature imports

---

## Cross-Feature Data Sharing

When a Feature2 UseCase needs Feature1's data, it injects Feature1's Repository **interface** (or UseCase):

```kotlin
// feature/feature2/domain/usecase/GetFeature2UseCase.kt
class GetFeature2UseCase(
    private val feature2Repository: Feature2Repository,
    private val feature1Repository: Feature1Repository, // domain interface — never Feature1's data/ or presentation/
) {
    suspend operator fun invoke(): NetworkResult<Feature2> { ... } // combines both repositories
}

// feature/feature2/di/Feature2Module.kt
val feature2Module = module {
    factory { GetFeature2UseCase(get(), get()) } // Koin resolves Feature1Repository from feature1Module
    viewModel { Feature2ViewModel(get()) }
}
```

Load order in `App.kt`:
```kotlin
modules(coreModules + listOf(
    feature1Module,   // must come before feature2Module
    feature2Module,
))
```

## Constraints
- The generic rules in `templates/shared/hard-rules.md` (Hard Rules, Do Not Touch, Security) also apply to this architecture
- Do not create any feature folder content beyond the empty `feature/` placeholder
- Do not hardcode any business logic, model, or screen here. This step is infra only
- Every core module must be feature-agnostic. It must not import anything feature-specific
- `core/mvvm/BaseViewModel.kt` must not know about any concrete feature's UseCases or Repository
- Use `core/model/` only when 2+ features really share a model
- Every Composable screen includes a `@Preview` composable with representative/sample state. Reusable `core/ui/components/` components include one too, where practical

## CLAUDE.md content for this architecture

> The generic rules (`Hard Rules — Never Violate`, `Do Not Touch`, `Security`) live in `templates/shared/hard-rules.md`. They also apply to this architecture and are not repeated here.
> `/android-starter:init-project` Step 5 copies the `## Architecture Hard Rules` block below into the project's `CLAUDE.md` verbatim, right after those generic blocks. Keep exactly this one `##` block here. Anything outside it is not copied.

```markdown
## Architecture Hard Rules

> MVVM + Clean Architecture. These add to the generic Hard Rules above.

- **Never call Repository or ApiService from a ViewModel**. Always go through a UseCase
- **Never add Android/Retrofit/Gson imports in `domain/`**. It is pure Kotlin only. `NetworkResult` is the domain's result type
- **Repository interface lives in `domain/repository/`, its implementation in `data/repository/`**. DTOs never leave `data/`
- **UseCase**: one class per operation, with `suspend operator fun invoke()`. It depends only on Repository interfaces. Business rules live here, never in the ViewModel or RepositoryImpl. Exception: field-format checks with `Validator` stay in the ViewModel
- **Repositories call the API only through `safeApiCall` / `safeUnitApiCall` from `core/network`**. Never write your own `try/catch` or `response.isSuccessful` check
- **ViewModel coroutines always go through `launchSafely { ... }`**, never raw `viewModelScope.launch`. **`handleError` override is mandatory** in every feature ViewModel: `setState { copy(isLoading = false, error = NetworkResult.Error(cause = throwable)) }`. Without it, the base no-op would silently swallow the error
- **Never use raw try/catch in a ViewModel**. `safeApiCall` already turned failures into `NetworkResult.Error`. The ViewModel checks the result with `when` and updates state
- **Errors are converted to text in the Screen at render time**. State holds `error: NetworkResult.Error?`, never a display string
- **ViewModel extends `core/mvvm/BaseViewModel<UiState>`**. It never redeclares `MutableStateFlow`/`Channel` and never exposes a mutable flow
- **One public ViewModel function per user action** (`onRefresh()`, `onItemClicked(id)`). No `processIntent`. The Screen never mutates ViewModel fields
- **UiState**: one immutable `data class XUiState : UiState` per screen. Use only `val` + immutable collections, with explicit `isLoading` and `error` fields. Change it **only** with `setState { copy(...) }`
- **UiEvent**: `sealed interface XUiEvent : UiEvent`, for **one-shot events only** (navigate, toast). Send it with `sendEvent(...)`. Collect it only inside `LaunchedEffect(Unit) { viewModel.event.collect { ... } }`. Anything the screen must still show after rotation belongs in UiState
```
