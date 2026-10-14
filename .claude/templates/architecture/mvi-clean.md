# Architecture Template: MVI + Clean Architecture (feature-based, single module)
 
## Tech Stack
- UI: Jetpack Compose
- DI: Koin
- Networking: Retrofit + OkHttp + Gson
- Image loading: Coil (coil-compose)
- Async: Kotlin Coroutines + Flow
- Navigation: Jetpack Navigation Compose (type-safe routes via kotlinx.serialization)
- Local storage: DataStore (Preferences) — default, no Room in shared scaffold
---
 
## Goal
Set up the entire skeleton of a NEW project — core infra, theme, base MVI classes, domain-layer contracts, DI wiring, navigation shell — so that features can be dropped in afterward using the standard feature-generation pattern below. Do NOT create any feature yet. Only scaffold shared foundations.
 
---
 
## Defaults
- Base URL: `BuildConfig.BASE_URL`, one placeholder per build type (debug / staging / release) with `// TODO` comments — see `.claude/templates/shared/project-setup.md`. `ApiConfig.kt` reads it, never declares a URL
- Local storage: DataStore (Preferences) — no Room, no SQL in shared scaffold
- JSON: Gson
These are fixed defaults — do not ask about them. Only deviate if explicitly stated.
 
---
 
## Project Structure
 
> **Claude Code instructions:**
> - Follow this structure exactly when creating files and folders.
> - Markers explained at bottom — read before creating any file.
> - `domain/` must never import from `data/`, `presentation/`, or any Android framework class.
> - `core/` must never import from any feature package.
> - `NetworkResult` is confined to `data/` only — `domain/` and `presentation/` never reference it.
> - Components placement rule (apply conditionally per `[LEVEL]` markers):
>   - Used across **all features** → `core/ui/components/`
>   - Used in **2+ screens within same feature** → `feature/featureX/presentation/components/`
>   - Used in **only one screen** → `feature/featureX/presentation/screenY/components/`
> - Start every new component at screen level. Move up only when reused.
 
```
app/src/main/java/com/example/app/
│
├── App.kt                                          # Application class — startKoin { androidContext(this@App); modules(coreModules) }
│                                                   # Leave comment: // Feature modules added here, e.g. modules(coreModules + listOf(authModule))
│
├── core/                                           # Shared across ALL features — pure Kotlin, minimal Android imports
│   ├── di/
│   │   ├── NetworkModule.kt                        # Koin: AuthTokenProvider + OkHttpClient + Retrofit singletons (built by NetworkClient)
│   │   ├── LocalStorageModule.kt                   # Koin: DataStore<Preferences> singleton — only instance in the app
│   │   ├── DispatcherModule.kt                     # Koin: named IO / Main / Default CoroutineDispatchers — inject, never hardcode
│   │   └── CoreModules.kt                          # val coreModules = listOf(networkModule, localStorageModule, dispatcherModule)
│   │
│   ├── network/
│   │   ├── ApiConfig.kt                            # BASE_URL = BuildConfig.BASE_URL (per build type) + timeouts
│   │   ├── ApiResponse.kt                          # backend response envelope { success, message, error, data } — adjust to the real backend
│   │   ├── AuthTokenProvider.kt                    # in-memory bearer token (hydrated from DataStore at App startup) + sessionExpired StateFlow
│   │   │                                           # read by NetworkClient's auth-header interceptor; 401 → notifySessionExpired()
│   │   ├── CurrentUserProvider.kt                  # [CONDITIONAL] in-memory signed-in userId, for features that mustn't depend on auth
│   │   ├── NetworkClient.kt                        # builds OkHttp (auth header → 401 → debug-only redacted logging) + Retrofit
│   │   ├── SafeApiCall.kt                          # safeApiCall / safeUnitApiCall / toResult() — the ONLY Response → NetworkResult → Result mapping
│   │   └── NetworkResult.kt                        # sealed class: Success<T> / Error(Failure) — no Loading (loading lives in UiStatus);
│   │                                               # DATA layer only, never leaks above RepositoryImpl
│   │
│   ├── domain/
│   │   ├── UseCase.kt                              # abstract class UseCase<in P, R>(dispatcher) — every feature UseCase extends this
│   │   ├── Result.kt                               # sealed class Result<T>: Success(data) / Error(failure: Failure) — domain only, decoupled from NetworkResult
│   │   └── Mapper.kt                               # interface Mapper<in I, out O> { fun map(input: I): O }
│   │
│   ├── error/
│   │   └── Failure.kt                              # sealed class: NoInternet / Timeout / Unauthorized / ServerError(code, message, rawResponse) / Unknown(throwable)
│   │                                               # + toDebugDetails() extension (toMessage() is UI — lives in AppSnackbar.kt)
│   │
│   ├── mvi/
│   │   ├── MviIntent.kt                            # marker interface MviIntent
│   │   ├── MviState.kt                             # marker interface MviState
│   │   ├── MviEffect.kt                            # marker interface MviEffect — plain interface, NOT sealed (a sealed type can't be
│   │   │                                           # implemented outside its own package); each feature's XEffect is the sealed interface
│   │   ├── UiStatus.kt                             # data class UiStatus(isLoading, failure: Failure?) — shared loading/error slice of a State
│   │   └── BaseViewModel.kt                        # abstract BaseViewModel<S: MviState, I: MviIntent, E: MviEffect>(initialState)
│   │                                               # — state: StateFlow<S>, effect: Flow<E> via Channel
│   │                                               # — setState(reducer), sendEffect(effect), launchSafely(onError, block)
│   │                                               # — abstract fun processIntent(intent: I)
│   │
│   ├── model/                                      # [CONDITIONAL] shared domain models used by 2+ features (e.g. User)
│   │   └── .gitkeep                                # placeholder — add models here only when genuinely shared across features
│   │
│   ├── ui/
│   │   ├── theme/
│   │   │   ├── Color.kt
│   │   │   ├── Typography.kt
│   │   │   ├── Shape.kt
│   │   │   ├── Dimens.kt                           # spacing/sizing tokens — e.g. val spacingMedium = 16.dp
│   │   │   └── Theme.kt                            # @Composable fun AppTheme { MaterialTheme(...) { content() } }
│   │   └── components/                             # [LEVEL 1] reused across ALL features and ALL screens — exactly as shared/ui-components.md:
│   │                                               # AppScreen, AppSnackbar (+ this architecture's UiStatus/Failure.toErrorMessage()),
│   │                                               # AppTopBar, AppButton, AppChip, Avatar, LoadingIndicator, ErrorView, EmptyView
│   │
│   └── util/
│       ├── AppLogger.kt                            # central logger — d(tag, message) + e(tag, message, throwable); no-op in release
│       │                                           # used by BaseViewModel.launchSafely and NetworkClient
│       ├── Validator.kt / InputLimits.kt / DateTimeFormat.kt  # exactly as shared/core-utils.md
│       └── Extensions.kt                           # Kotlin extension functions shared app-wide
│
│
├── navigation/
│   ├── MainActivity.kt                             # single Activity — setContent { AppTheme { AppNavGraph() } }
│   ├── AppNavGraph.kt                              # NavHost(startDestination = Routes.Home) — placeholder composable<Routes.Home> (the system splash comes first, no Splash route)
│   │                                               # Leave comment: // composable<Routes.FeatureX> { ... } added when features are generated
│   ├── NavResult.kt                                # setNavResult / NavResultEffect + NavResultKeys — shared/core-utils.md
│   ├── Routes.kt                                   # sealed interface Routes — @Serializable route objects/data classes, one block per feature
│   └── NavResult.kt                                # typed screen-to-screen passing via SavedStateHandle:
│                                                   # setNavInput/navInput (forward) + setNavResult/navResult (back, as State<T?>)
│
│
└── feature/                                        # empty at scaffold time — features added via feature-generation pattern below
    │
    └── featureX/                                   # [FEATURE TEMPLATE] one folder per feature, added later
        │
        ├── data/                                   # data layer — Retrofit, DataStore, mappers; only layer that sees NetworkResult
        │   ├── remote/
        │   │   ├── FeatureXApiService.kt           # Retrofit interface — provided via get<Retrofit>().create(...) in DI, never built manually
        │   │   └── dto/
        │   │       └── FeatureXDto.kt              # raw JSON model — never exposed above this layer
        │   ├── local/                              # [CONDITIONAL] only if this feature needs local persistence
        │   │   └── FeatureXLocalSource.kt          # DataStore read/write helpers for this feature
        │   ├── mapper/
        │   │   └── FeatureXMapper.kt               # implements Mapper<FeatureXDto, FeatureXModel> — DTO → domain conversion
        │   └── repository/
        │       └── FeatureXRepositoryImpl.kt       # implements domain FeatureXRepository
        │                                           # converts NetworkResult → Result<T>/Failure here — never return NetworkResult
        │
        ├── domain/                                 # pure Kotlin — zero Android/Retrofit/Room imports
        │   ├── model/
        │   │   └── FeatureXModel.kt                # domain entity — decoupled from DTO
        │   ├── repository/
        │   │   └── FeatureXRepository.kt           # interface only — RepositoryImpl implements this
        │   └── usecase/
        │       └── GetFeatureXUseCase.kt           # extends core/domain/UseCase.kt — one class, one operation, operator fun invoke()
        │                                           # [CONDITIONAL cross-feature] inject another feature's Repository here if needed
        │
        ├── presentation/                           # MVI — ViewModel + screen(s)
        │   │                                       # If this feature has only ONE screen: put ScreenNameContract.kt /
        │   │                                       # ScreenNameViewModel.kt / ScreenNameScreen.kt directly in presentation/
        │   │                                       # (no per-screen subfolder). Only introduce a screenName/ subfolder
        │   │                                       # once the feature genuinely has 2+ distinct screens.
        │   ├── components/                         # [LEVEL 2][CONDITIONAL] single-screen feature: that screen's components;
        │   │                                       # multi-screen feature: Composables reused across 2+ screens
        │   ├── ScreenNameContract.kt               # Intent + State + Effect — all three in one file
        │   ├── ScreenNameViewModel.kt              # extends BaseViewModel — processIntent() → launchSafely { useCase(...) } → setState/sendEffect
        │   └── ScreenNameScreen.kt                 # @Composable — collectAsStateWithLifecycle(); sends intents; includes @Preview
        │
        │   # --- multi-screen feature shape (only when 2+ screens exist) ---
        │   # presentation/
        │   #   ├── components/                     [LEVEL 2][CONDITIONAL] reused across 2+ screens
        │   #   └── screenName/                     one folder PER screen
        │   #       ├── components/                 [LEVEL 3][CONDITIONAL] used only in this one screen
        │   #       ├── ScreenNameContract.kt
        │   #       ├── ScreenNameViewModel.kt
        │   #       └── ScreenNameScreen.kt
        │   # When a single-screen feature gains a 2nd screen, move the first screen's files into its own screenName/ folder in the same change
        │
        └── di/
            └── FeatureXModule.kt                   # val featureXModule = module { repo + usecase + viewModel(::FeatureXViewModel) }
```
 
---
 
## Marker Reference
 
| Marker | Meaning |
|---|---|
| `[CONDITIONAL]` | Create only if actually needed — skip otherwise |
| `[LEVEL 1]` | Global component — reused across all features |
| `[LEVEL 2]` | Feature-wide component — reused across 2+ screens in same feature |
| `[LEVEL 3]` | Screen-specific component — used in exactly one screen |
| `[FEATURE TEMPLATE]` | Not created at scaffold time — shows the pattern for later feature generation |
| `[CONDITIONAL cross-feature]` | Inject another feature's Repository into this UseCase only when genuinely needed |
 
---
 
## Component Placement — Decision Tree
 
```
Creating a new Composable?
        │
        ▼
Used in only one screen?
   YES → feature/featureX/presentation/screenName/components/      [LEVEL 3]
   NO  ↓
        ▼
Used in 2+ screens of the same feature?
   YES → feature/featureX/presentation/components/                 [LEVEL 2]
   NO  ↓
        ▼
Used across all features?
   YES → core/ui/components/                                       [LEVEL 1]
```
 
> **Rule:** Always start at LEVEL 3 (screen). Promote upward only when reuse appears — never move a component down.
> In a single-screen feature (flat `presentation/`, no `screenName/` folder), LEVEL 3 components go in `presentation/components/`.
 
---
 
## Dependency Direction
 
```
feature/featureX/presentation  ──→  feature/featureX/domain  ←──  feature/featureX/data
                                              ↑
                                        core/ (all layers use)
                                              ↑
                                   core/di/ + feature/di/ (binds everything)
```
 
See **Constraints** below for the rules this diagram implies.

---

## Constraints

- `domain/` never imports `data/`, `presentation/`, or Android framework classes
- `core/` never imports any feature package
- `NetworkResult` never crosses above `RepositoryImpl`
- `core/model/` used only when a model is genuinely shared across 2+ features
---
 
## Step-by-Step Scaffold Instructions
 
### Step 1: Gradle (`app/build.gradle.kts`)
Add:
- Compose BOM + ui, material3 (1.3+ — AppScreen uses `PullToRefreshBox`), material-icons-core, tooling-preview
- Koin: koin-android, koin-androidx-compose
- Retrofit: retrofit, converter-gson
- OkHttp: okhttp, logging-interceptor
- DataStore: androidx-datastore-preferences
- Coil 3: coil-compose + coil-network-okhttp (without it Coil 3 cannot load images from a URL)
- Coroutines: kotlinx-coroutines-android
- Navigation: navigation-compose (2.8+)
- Serialization: kotlinx-serialization-json + `kotlin("plugin.serialization")` plugin
- Lifecycle: lifecycle-viewmodel-compose, lifecycle-runtime-ktx, lifecycle-runtime-compose (`collectAsStateWithLifecycle()`)
- Build features: `android { buildFeatures { compose = true; buildConfig = true } }` — `buildConfig` is off by default since AGP 8, and the network logging + snackbar "Details" button read `BuildConfig.DEBUG`
### Step 2: Create folder structure
Exactly as shown above. `feature/` folder is empty at scaffold time — only `core/` and `navigation/` are populated.
 
### Step 3: App.kt
```kotlin
startKoin {
    androidLogger()
    androidContext(this@App)
    modules(coreModules)
    // Feature modules added here later, e.g.:
    // modules(coreModules + listOf(authModule, orderModule))
}
```
 
### Step 4: Network layer (`core/network/` + `NetworkModule.kt`)

**Rules**
- One OkHttpClient + one Retrofit for the whole app, built in `NetworkClient` and provided as `single`s by `NetworkModule`
- Features use `get<Retrofit>().create(FeatureXApiService::class.java)` — never build their own Retrofit/OkHttp
- Only `RepositoryImpl` touches Retrofit, and only through `safeApiCall` / `safeUnitApiCall` — never a hand-written `try/catch` around an API call. `.toResult()` turns the `NetworkResult` into the domain `Result` before it leaves the repository
- The logging interceptor is added **only** `if (BuildConfig.DEBUG)` — never commented out "temporarily" — and it masks passwords / OTPs / PINs / tokens in bodies even in debug
- **Never slow down the network layer on purpose** (`Thread.sleep`, `delay`, a "minimum duration" interceptor). If a loader flashes too briefly, that's a UI concern — `rememberMinimumLoading` in `AppScreen` (Step 8) handles it without delaying the data

**`ApiConfig.kt`**
```kotlin
object ApiConfig {
    val BASE_URL: String = BuildConfig.BASE_URL // one per build type — see shared/project-setup.md
    const val CONNECT_TIMEOUT_SECONDS = 30L
    const val READ_TIMEOUT_SECONDS = 30L
    const val WRITE_TIMEOUT_SECONDS = 30L
}
```

**`ApiResponse.kt`** — the backend's response envelope. **Adjust to your backend's real shape** (ask if unknown — never guess):
```kotlin
data class ApiResponse<T>(
    val success: Boolean = false,
    val message: String? = null,
    val error: String? = null,
    val data: T? = null,
)
```

**`AuthTokenProvider.kt`** — synchronous token holder (OkHttp interceptors can't suspend on DataStore). Kept in sync by the feature that owns the session, so `core/network` never imports a feature:
```kotlin
class AuthTokenProvider {
    @Volatile
    var token: String? = null

    private val _sessionExpired = MutableStateFlow(false)

    /** StateFlow, not SharedFlow: replays to a collector that attaches late (e.g. MainActivity recreated), so a 401 redirect is never dropped. */
    val sessionExpired: StateFlow<Boolean> = _sessionExpired.asStateFlow()

    fun notifySessionExpired() {
        token = null
        _sessionExpired.value = true
    }

    /** Call once the redirect to sign-in has been handled. */
    fun consumeSessionExpired() {
        _sessionExpired.value = false
    }
}
```
`CurrentUserProvider` (`@Volatile var userId: String?`) — [CONDITIONAL] same pattern, add only when a feature needs the signed-in user's id without depending on the auth feature.

**`NetworkClient.kt`**
```kotlin
object NetworkClient {
    private const val LOG_TAG = "NetworkClient"

    /** Capped body peek — `peekBody(Long.MAX_VALUE)` blocks until EOF and can hang a response that never terminates. */
    private const val MAX_LOGGED_BODY_BYTES = 200_000L

    /** Keys whose values never reach a log line, debug builds included. */
    private val SENSITIVE_KEY = "[A-Za-z_]*(?:password|otp|pin|token|secret)[A-Za-z_]*"
    private val SENSITIVE_JSON_FIELD = Regex("(\"$SENSITIVE_KEY\"\\s*:\\s*)(\"(?:\\\\.|[^\"\\\\])*\"|[^,}\\s]+)", RegexOption.IGNORE_CASE)
    private val SENSITIVE_FORM_FIELD = Regex("((?:^|&)$SENSITIVE_KEY=)[^&]*", RegexOption.IGNORE_CASE)

    fun createOkHttpClient(authTokenProvider: AuthTokenProvider): OkHttpClient =
        OkHttpClient.Builder()
            .connectTimeout(ApiConfig.CONNECT_TIMEOUT_SECONDS, TimeUnit.SECONDS)
            .readTimeout(ApiConfig.READ_TIMEOUT_SECONDS, TimeUnit.SECONDS)
            .writeTimeout(ApiConfig.WRITE_TIMEOUT_SECONDS, TimeUnit.SECONDS)
            // 1. Bearer token
            .addInterceptor { chain ->
                val builder = chain.request().newBuilder()
                authTokenProvider.token?.takeIf { it.isNotEmpty() }?.let { builder.addHeader("Authorization", "Bearer $it") }
                chain.proceed(builder.build())
            }
            // 2. 401 on an authenticated request → session is dead, force sign-in
            .addInterceptor { chain ->
                val request = chain.request()
                val response = chain.proceed(request)
                if (response.code == 401 && request.header("Authorization") != null) {
                    AppLogger.e(tag = LOG_TAG, message = "Session expired: 401 on authenticated request ${request.method} ${request.url} — forcing logout")
                    authTokenProvider.notifySessionExpired()
                }
                response
            }
            // 3. Debug-only logging — bodies carry user data
            .apply { if (BuildConfig.DEBUG) addInterceptor(loggingInterceptor) }
            .build()

    /** One compact block per call: method, url, body, status, duration, body — secrets masked. */
    private val loggingInterceptor = Interceptor { chain ->
        val request = chain.request()
        val requestBody = request.body?.let { body ->
            runCatching { Buffer().apply { body.writeTo(this) }.readUtf8() }.getOrNull()
        }?.redactSecrets()
        val startNanos = System.nanoTime()
        val response = chain.proceed(request)
        val durationMs = (System.nanoTime() - startNanos) / 1_000_000
        // Best-effort: logging must never turn a fine response into a failure.
        val responseBody = runCatching { response.peekBody(MAX_LOGGED_BODY_BYTES).string() }.getOrNull()?.redactSecrets()
        AppLogger.d(
            tag = LOG_TAG,
            message = buildString {
                append("→ ${request.method} ${request.url}")
                if (!requestBody.isNullOrBlank()) append("\n  body: $requestBody")
                append("\n← ${response.code} ${response.message} (${durationMs}ms)")
                if (!responseBody.isNullOrBlank()) append("\n  body: $responseBody")
            },
        )
        response
    }

    private fun String.redactSecrets(): String =
        replace(SENSITIVE_JSON_FIELD) { "${it.groupValues[1]}\"***\"" }
            .replace(SENSITIVE_FORM_FIELD) { "${it.groupValues[1]}***" }

    fun createRetrofit(okHttpClient: OkHttpClient): Retrofit =
        Retrofit.Builder()
            .baseUrl(ApiConfig.BASE_URL)
            .client(okHttpClient)
            .addConverterFactory(GsonConverterFactory.create())
            .build()
}
```

**`NetworkModule.kt`**
```kotlin
val networkModule = module {
    single { AuthTokenProvider() }
    single { NetworkClient.createOkHttpClient(get()) }
    single { NetworkClient.createRetrofit(get()) }
}
```
`App.kt` hydrates `AuthTokenProvider.token` from DataStore at startup; `MainActivity` collects `sessionExpired` → navigates to sign-in → `consumeSessionExpired()`.

**`SafeApiCall.kt`** — the **only** place a Retrofit `Response` becomes a `NetworkResult`, so every feature maps errors identically (import `kotlin.coroutines.cancellation.CancellationException`):
```kotlin
@PublishedApi
internal val envelopeGson = Gson()

suspend inline fun <DTO, DOMAIN> safeApiCall(
    crossinline apiCall: suspend () -> Response<ApiResponse<DTO>>,
    crossinline mapper: (DTO) -> DOMAIN,
): NetworkResult<DOMAIN> = try {
    val response = apiCall()
    val body = response.body()
    val data = body?.data
    when {
        !response.isSuccessful -> NetworkResult.Error(response.toFailure())
        body == null -> NetworkResult.Error(Failure.ServerError(response.code(), "Empty response body"))
        !body.success -> NetworkResult.Error(body.toServerError(response.code()))
        data == null -> NetworkResult.Error(Failure.ServerError(response.code(), "Response data is null"))
        else -> NetworkResult.Success(mapper(data))
    }
} catch (e: CancellationException) {
    throw e // cancellation must stop the caller's coroutine — never a Failure
} catch (e: Exception) {
    NetworkResult.Error(e.toFailure())
}

/** For endpoints whose success carries no `data`. */
suspend inline fun safeUnitApiCall(
    crossinline apiCall: suspend () -> Response<ApiResponse<Unit>>,
): NetworkResult<Unit> = try {
    val response = apiCall()
    val body = response.body()
    when {
        !response.isSuccessful -> NetworkResult.Error(response.toFailure())
        body == null -> NetworkResult.Error(Failure.ServerError(response.code(), "Empty response body"))
        !body.success -> NetworkResult.Error(body.toServerError(response.code()))
        else -> NetworkResult.Success(Unit)
    }
} catch (e: CancellationException) {
    throw e // cancellation must stop the caller's coroutine — never a Failure
} catch (e: Exception) {
    NetworkResult.Error(e.toFailure())
}

fun <T> NetworkResult<T>.toResult(): Result<T> = when (this) {
    is NetworkResult.Success -> Result.Success(data)
    is NetworkResult.Error -> Result.Error(failure)
}

@PublishedApi
internal fun Exception.toFailure(): Failure = when (this) {
    is SocketTimeoutException -> Failure.Timeout          // before IOException — it is one
    is UnknownHostException, is IOException -> Failure.NoInternet
    else -> Failure.Unknown(this)
}

@PublishedApi
internal fun ApiResponse<*>.toServerError(code: Int): Failure =
    Failure.ServerError(code = code, message = message ?: error ?: "Unknown error", rawResponse = envelopeGson.toJson(this))

@PublishedApi
internal fun <T> Response<ApiResponse<T>>.toFailure(): Failure {
    val rawBody = errorBody()?.string()
    // Error bodies aren't always the envelope (HTML gateway pages, plain text) — fall back to no message.
    val message = rawBody?.let { raw ->
        runCatching { envelopeGson.fromJson(raw, ApiResponse::class.java) }.getOrNull()?.let { it.error ?: it.message }
    }
    return if (code() == 401) Failure.Unauthorized(message) else Failure.ServerError(code(), message, rawBody)
}
```

**RepositoryImpl usage — the shape every feature follows:**
```kotlin
class OrderRepositoryImpl(
    private val api: OrderApiService,
    private val mapper: OrderMapper,
) : OrderRepository {
    override suspend fun getOrders(): Result<List<Order>> =
        safeApiCall(apiCall = { api.getOrders() }, mapper = { dtos -> dtos.map(mapper::map) }).toResult()

    override suspend fun cancelOrder(id: String): Result<Unit> =
        safeUnitApiCall { api.cancelOrder(id) }.toResult()
}
```

### Step 5: LocalStorageModule.kt
- Single `DataStore<Preferences>` via `preferencesDataStore(name = "app_prefs")`
- Features inject `get<DataStore<Preferences>>()` — never create their own instance
- **Sensitive data** (auth token, user session, anything PII) → **always** this DataStore — never `SharedPreferences`
- **Non-sensitive data** (e.g. app language, theme flag, "onboarding seen") → `SharedPreferences` is allowed, especially when it must be read synchronously before Koin/DataStore are ready (e.g. locale in `attachBaseContext`)
  - Keep it behind one small `core/util` helper (e.g. `LocaleManager`) with a named key constant — never call `getSharedPreferences` / `PreferenceManager` from a Composable or feature code directly
  - If in doubt whether a value is sensitive, treat it as sensitive and use DataStore
### Step 5b: DispatcherModule.kt
```kotlin
val dispatcherModule = module {
    single(named("io")) { Dispatchers.IO }
    single(named("main")) { Dispatchers.Main }
    single(named("default")) { Dispatchers.Default }
}
```
- This is the **only** file in the app allowed to reference `Dispatchers.*`
- Every class that switches threads takes a `CoroutineDispatcher` constructor param and Koin passes the named one — UseCases (`factory { GetFeatureXUseCase(get(), get(named("io"))) }`) **and** non-UseCase helpers too (e.g. a `core/util/LocationProvider` doing `withContext(ioDispatcher) { ... }`)
- Never write `withContext(Dispatchers.IO)` / `launch(Dispatchers.Default)` inline — it can't be swapped for a test dispatcher
### Step 6: Domain contracts
 
**`Failure.kt`** — pure Kotlin, no UI or string resources (the user-facing `Failure.toMessage()` lives in `AppSnackbar.kt`, see Step 8):
```kotlin
sealed class Failure {
    data object NoInternet : Failure()
    /** Reachable but no response in time (SocketTimeoutException) — not "check your network": it may have gone through server-side. */
    data object Timeout : Failure()
    data class Unauthorized(val message: String? = null) : Failure()
    data class ServerError(val code: Int, val message: String? = null, val rawResponse: String? = null) : Failure()
    data class Unknown(val throwable: Throwable? = null) : Failure()
}

/** Debug-only details for AppSnackbar.details (shown in the snackbar's Details sheet). */
fun Failure.toDebugDetails(): String? = when (this) {
    is Failure.Unknown -> throwable?.stackTraceToString()
    is Failure.ServerError -> "HTTP $code\n" + (rawResponse ?: message ?: "")
    is Failure.Unauthorized, Failure.NoInternet, Failure.Timeout -> null
}
```
 
**`NetworkResult.kt`** (`core/network/`, data layer only)
```kotlin
sealed class NetworkResult<out T> {
    data class Success<T>(val data: T) : NetworkResult<T>()
    data class Error(val failure: Failure) : NetworkResult<Nothing>()
}
```
No `Loading` case — a network call is either done or failed; "loading" is UI state and lives in `UiStatus.isLoading`.

**`Result.kt`**
```kotlin
sealed class Result<out T> {
    data class Success<T>(val data: T) : Result<T>()
    data class Error(val failure: Failure) : Result<Nothing>()
}
```
 
**`UseCase.kt`**
```kotlin
abstract class UseCase<in P, R>(private val dispatcher: CoroutineDispatcher) {
    suspend operator fun invoke(params: P): Result<R> = withContext(dispatcher) {
        try {
            execute(params)
        } catch (e: CancellationException) {
            throw e // cancellation must propagate so the caller's coroutine actually stops — never a Failure
        } catch (e: Exception) {
            Result.Error(Failure.Unknown(e))
        }
    }
    protected abstract suspend fun execute(params: P): Result<R>
}
```
 
**`Mapper.kt`**
```kotlin
interface Mapper<in I, out O> {
    fun map(input: I): O
}
```
 
### Step 7: MVI markers + UiStatus + BaseViewModel.kt

**Markers** — plain `interface`, never `sealed` (Kotlin only lets a sealed type be implemented inside its own package, so a sealed core marker could never be implemented by `feature/*`):
```kotlin
interface MviState
interface MviIntent
interface MviEffect
```

**`UiStatus.kt`** — the shared loading/error slice every screen State carries and hands to `AppScreen`:
```kotlin
data class UiStatus(
    val isLoading: Boolean = false,
    /** Typed error — the Screen turns it into text (string resources) and debug details at render time. */
    val failure: Failure? = null,
)
```
A `Result.Error(failure)` from a UseCase and an unexpected exception caught by `launchSafely` (`Failure.Unknown(throwable)`) both land here — one error shape for every screen.

**`BaseViewModel.kt`**
```kotlin
abstract class BaseViewModel<S : MviState, I : MviIntent, E : MviEffect>(
    initialState: S
) : ViewModel() {
    private val _state = MutableStateFlow(initialState)
    val state: StateFlow<S> = _state.asStateFlow()
 
    private val _effect = Channel<E>(Channel.BUFFERED)
    val effect = _effect.receiveAsFlow()
 
    protected fun setState(reducer: S.() -> S) = _state.update(reducer)
 
    protected fun sendEffect(effect: E) {
        viewModelScope.launch { _effect.send(effect) }
    }
 
    protected fun launchSafely(
        onError: (Throwable) -> Unit = ::handleError,
        block: suspend () -> Unit,
    ) {
        viewModelScope.launch {
            try {
                block()
            } catch (e: CancellationException) {
                throw e // normal teardown (ViewModel cleared, collectLatest replacing a job) — not an error to show
            } catch (e: Exception) {
                AppLogger.e(tag = this@BaseViewModel.javaClass.simpleName, message = "Unhandled error in launchSafely", throwable = e)
                onError(e)
            }
        }
    }
 
    protected open fun handleError(throwable: Throwable) {
        // default: no-op — override per feature ViewModel
    }
 
    abstract fun processIntent(intent: I)
}
```

**Feature usage — the shape every screen follows (`ScreenNameContract.kt` + ViewModel + Screen):**
```kotlin
// Contract — all three in one file
data class OrderListState(
    val status: UiStatus = UiStatus(),
    val orders: List<Order> = emptyList(),        // immutable List, never MutableList / var
) : MviState

sealed interface OrderListIntent : MviIntent {
    data object Retry : OrderListIntent
    data object ErrorDismissed : OrderListIntent
    data class OrderClicked(val id: String) : OrderListIntent
}

sealed interface OrderListEffect : MviEffect {
    data class OpenOrder(val id: String) : OrderListEffect
}

// ViewModel
override fun processIntent(intent: OrderListIntent) = when (intent) {
    OrderListIntent.Retry -> load()
    OrderListIntent.ErrorDismissed -> setState { copy(status = status.copy(failure = null)) }
    is OrderListIntent.OrderClicked -> sendEffect(OrderListEffect.OpenOrder(intent.id))
}

override fun handleError(throwable: Throwable) {
    setState { copy(status = UiStatus(failure = Failure.Unknown(throwable))) }
}

// Screen — effects collected once, in a LaunchedEffect
LaunchedEffect(Unit) {
    viewModel.effect.collect { effect ->
        when (effect) { is OrderListEffect.OpenOrder -> onOpenOrder(effect.id) }
    }
}
```
 
### Step 8: UI components (`core/ui/components/`)

Create every component in `.claude/templates/shared/ui-components.md` exactly as written there — `AppScreen`, `AppSnackbar`, `AppTopBar`, `AppButton`, `AppChip`, `Avatar`, `LoadingIndicator`, `ErrorView`, `EmptyView` — plus its **MVI + Clean** converter (`UiStatus.toErrorMessage()`, `Failure?.toErrorMessage()`, `Failure.toMessage()`) in `AppSnackbar.kt`. That file also lists the Dimens tokens and string resources the components need.

Screens pass `state.status.isLoading` to `LoadingState` and `state.status.toErrorMessage(onRetry = { viewModel.processIntent(XIntent.Retry) }, onDismiss = { viewModel.processIntent(XIntent.ErrorDismissed) })` to `snackbarMessage`, per the shared file's "Screen usage" example.

### Step 9: Project setup + core utilities

Apply `.claude/templates/shared/project-setup.md` in full — environments (debug/staging/release `BASE_URL`), release signing from `keystore.properties`, R8 keep rules for DTOs, splash screen, unit-test setup (`MainDispatcherRule`), ktlint + `.editorconfig`, and the GitHub Actions workflow. Then create every file in `.claude/templates/shared/core-utils.md` — `core/util/Validator.kt`, `InputLimits.kt`, `DateTimeFormat.kt` and `navigation/NavResult.kt`.

### Step 10: Navigation shell
 
**`Routes.kt`**
```kotlin
sealed interface Routes {
    @Serializable data object Home : Routes
    // Features add their own routes here later:
    // @Serializable data class FeatureXDetail(val id: String) : Routes
}
```
 
**`AppNavGraph.kt`**
```kotlin
NavHost(navController, startDestination = Routes.Home) {
    composable<Routes.Home> { /* blank themed placeholder screen inside AppScreen */ }
    // composable<Routes.FeatureX> { FeatureXScreen() } — added when features are generated
}
```
 
### Step 11: Verify
- `./gradlew ktlintCheck testDebugUnitTest assembleDebug` passes, and the app launches showing the system splash screen first
- Project builds with zero features present
- App launches to blank themed screen → proves Koin + Theme + NavGraph are wired
- `core/domain/UseCase.kt` compiles with no feature imports
- `AppScreen.kt` compiles with all its params; its loading and Info-snackbar `@Preview`s render; the placeholder Home screen composes its body inside it
---
 
## Cross-Feature Data Sharing
 
When a Feature2 UseCase needs Feature1's data:
 
```kotlin
// feature/feature2/domain/usecase/GetFeature2UseCase.kt
class GetFeature2UseCase(
    private val feature2Repo: Feature2Repository,
    private val feature1Repo: Feature1Repository    // inject — never call data layer directly
) : UseCase<...>(dispatcher) { ... }
 
// feature/feature2/di/Feature2Module.kt
val feature2Module = module {
    factory { GetFeature2UseCase(get(), get()) }    // Koin resolves Feature1Repository automatically
    viewModel { Feature2ViewModel(get()) }
}
```
 
Load order in `App.kt`:
```kotlin
modules(coreModules + listOf(
    feature1Module,   // must come before feature2Module
    feature2Module
))
```
 
---
 
## CLAUDE.md content for this architecture

> The generic rules (`Hard Rules — Never Violate`, `Do Not Touch`, `Security`) live in `.claude/templates/shared/hard-rules.md` and apply to this architecture too — they are not repeated here.
> `/init-project` Step 5 copies the `## Architecture Hard Rules` block below into the project's `CLAUDE.md` verbatim, right after those generic blocks. Keep exactly this one `##` block here — anything outside it is not copied.

```markdown
## Architecture Hard Rules

> MVI + Clean Architecture. These add to the generic Hard Rules above.

- **Repositories call the API only through `safeApiCall` / `safeUnitApiCall` + `.toResult()`** — never a hand-written `try/catch` or `response.isSuccessful` check in a RepositoryImpl
- **Never call Repository directly from ViewModel** — always through a UseCase
- **Never add Android/Retrofit/Room imports in `domain/`** — pure Kotlin only
- **Never use `NetworkResult` above `RepositoryImpl`** — domain and presentation see only `Result<T>` and `Failure`
- **UseCase**: one class per operation, extends `core/domain/UseCase.kt`, always `operator fun invoke()`; inject its dispatcher, never hardcode one
- **Never use raw try/catch in ViewModel** — UseCase returns `Result<T>`; ViewModel maps via `setState`
- **ViewModel side-effects**: always go through `launchSafely`, never raw `viewModelScope.launch`
- **Never use `sealed class` for a feature's Effect or Intent** — always `sealed interface XEffect : MviEffect` / `sealed interface XIntent : MviIntent`. The core `MviEffect`/`MviIntent`/`MviState` markers stay plain `interface`
- **MVI contract rules:**
  - **State** — one immutable `data class XState : MviState` per screen, only `val` + immutable collections (`List`, never `MutableList`/`var`); carries `status: UiStatus` (`isLoading` + `failure: Failure?`) for loading/error. Changed **only** via `setState { copy(...) }`
  - **Intent** — `sealed interface XIntent : MviIntent`; the Screen talks to the ViewModel **only** via `viewModel.processIntent(...)` — never by calling other public VM functions or mutating VM fields
  - **Effect** — `sealed interface XEffect : MviEffect`, **one-shot events only** (navigate, toast, close sheet, open picker). Anything the screen must still show after rotation belongs in State, never in an Effect; and a one-shot event never goes in State as a flag
  - **Screen** collects `state` with `collectAsStateWithLifecycle()` and `effect` inside `LaunchedEffect(Unit) { viewModel.effect.collect { ... } }` — never collects `effect` anywhere else
  - **`handleError` override is mandatory** in every feature ViewModel: `setState { copy(status = UiStatus(failure = Failure.Unknown(throwable))) }` (also clears loading) — the base no-op would swallow the error silently
```
