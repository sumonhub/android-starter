# Shared Core Infra (all architectures)

> This file applies to every architecture pattern.
> For every architecture, `/android-starter:init-project` creates every file below, **exactly as written here**.
> The architecture template adds only what depends on the pattern: `NetworkResult` + `SafeApiCall`, the base ViewModel and its markers, the domain contracts (Clean only) and the navigation shell.
>
> File locations follow the chosen template's `## Project Structure`. Most paths are the same in every template. One exception: `MainActivity.kt` sits in `navigation/` in MVI + Clean, and at the package root in the other three.

## Rules

- The whole app has one OkHttpClient and one Retrofit. `NetworkClient` builds them. `NetworkModule` provides them as `single`s.
- Features use `get<Retrofit>().create(FeatureXApiService::class.java)`. They never build their own Retrofit/OkHttp.
- Add the logging interceptor **only** `if (BuildConfig.DEBUG)`. Never comment it out "temporarily". It masks passwords / OTPs / PINs / tokens in bodies, even in debug.
- **Never slow down the network layer on purpose.** For example: `Thread.sleep`, `delay`, or a "minimum duration" interceptor. If a loader shows for too short a time, that is a UI problem. `rememberMinimumLoading` in `AppScreen` handles it without delaying the data.
- `DispatcherModule.kt` is the **only** file in the app that may reference `Dispatchers.*`.
- `LocalStorageModule.kt` is the **only** place that creates a DataStore.
- Log only through `AppLogger`. Never call `Log.*` / `println` anywhere else.

## Gradle

Besides the architecture template's Gradle step, make sure these exist:
- `androidx.activity:activity-compose` (`setContent`, `enableEdgeToEdge`). New Compose projects already have it.
- `android { testOptions { unitTests.isReturnDefaultValues = true } }`. `AppLogger` calls `android.util.Log`, and `BaseViewModel.launchSafely` logs through it. Without this flag, a ViewModel test that hits the error path fails with "Method ... not mocked".

## App.kt

```kotlin
class App : Application() {
    override fun onCreate() {
        super.onCreate()
        startKoin {
            if (BuildConfig.DEBUG) androidLogger()
            androidContext(this@App)
            modules(coreModules)
            // Feature modules added here later, e.g.:
            // modules(coreModules + listOf(authModule, orderModule))
        }
    }
}
```
- Register it in the manifest: `<application android:name=".App" ...>`.
- The feature that owns the session (usually `auth`) later adds one line here: it loads the saved token from DataStore into `AuthTokenProvider.token` at startup. The scaffold has no token yet, so it adds nothing.

## core/di/CoreModules.kt

```kotlin
val coreModules = listOf(networkModule, localStorageModule, dispatcherModule)
```

## core/network/ApiConfig.kt

```kotlin
object ApiConfig {
    val BASE_URL: String = BuildConfig.BASE_URL // one per build type — see shared/project-setup.md
    const val CONNECT_TIMEOUT_SECONDS = 30L
    const val READ_TIMEOUT_SECONDS = 30L
    const val WRITE_TIMEOUT_SECONDS = 30L
}
```

## core/network/AuthTokenProvider.kt

It holds the token synchronously, because OkHttp interceptors can't suspend on DataStore. The feature that owns the session keeps it up to date. This way `core/network` never imports a feature.
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
- While `token` is null, no `Authorization` header is sent. So an app without login works with no changes.
- When an auth feature is added later, `MainActivity` collects `sessionExpired` → navigates to sign-in → calls `consumeSessionExpired()`. The scaffold has no sign-in route, so it doesn't collect it yet.
- `CurrentUserProvider` (`@Volatile var userId: String?`) is [CONDITIONAL]. It uses the same pattern. Add it only when a feature needs the signed-in user's id without depending on the auth feature.

## core/network/NetworkClient.kt

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
`Buffer` is `okio.Buffer`. OkHttp already brings okio, so it needs no extra dependency.

## core/di/NetworkModule.kt

```kotlin
val networkModule = module {
    single { AuthTokenProvider() }
    single { NetworkClient.createOkHttpClient(get()) }
    single { NetworkClient.createRetrofit(get()) }
}
```

## core/di/LocalStorageModule.kt

```kotlin
private val Context.dataStore: DataStore<Preferences> by preferencesDataStore(name = "app_prefs")

val localStorageModule = module {
    single { androidContext().dataStore }
}
```
- Features inject `get<DataStore<Preferences>>()`. They never create their own instance.
- No Room, no SQL entities, no `@Dao` in the shared scaffold. If a feature later really needs relational data, decide that for that feature.
- **Sensitive data** (auth token, user session, anything PII) → **always** this DataStore. Never `SharedPreferences`.
- **Non-sensitive data** (e.g. app language, theme flag, "onboarding seen") → `SharedPreferences` is allowed. This is useful when the value must be read synchronously before Koin/DataStore are ready (e.g. locale in `attachBaseContext`).
  - Put it behind one small `core/util` helper (e.g. `LocaleManager`) with a named key constant. Never call `getSharedPreferences` / `PreferenceManager` directly from a Composable or from feature code.
  - If you are not sure whether a value is sensitive, treat it as sensitive and use DataStore.

## core/di/DispatcherModule.kt

```kotlin
val dispatcherModule = module {
    single<CoroutineDispatcher>(named("io")) { Dispatchers.IO }
    single<CoroutineDispatcher>(named("main")) { Dispatchers.Main }
    single<CoroutineDispatcher>(named("default")) { Dispatchers.Default }
}
```
- Every class that switches threads takes a `CoroutineDispatcher` constructor param. Koin passes the named one (`get(named("io"))`). This applies to UseCases, repositories that do blocking work, and helpers (e.g. a `core/util/LocationProvider` doing `withContext(ioDispatcher) { ... }`).
- Never write `withContext(Dispatchers.IO)` / `launch(Dispatchers.Default)` inline. Tests can't swap it for a test dispatcher.

## core/util/AppLogger.kt

```kotlin
/** The only logger in the app. Silent in release builds — logs can carry user data. */
object AppLogger {
    fun d(tag: String, message: String) {
        if (BuildConfig.DEBUG) Log.d(tag, message)
    }

    fun e(tag: String, message: String, throwable: Throwable? = null) {
        if (BuildConfig.DEBUG) Log.e(tag, message, throwable)
    }
}
```
If the project later adds crash reporting (e.g. Crashlytics), `e(...)` is the one place to forward release errors to it.

## core/util/Extensions.kt

Create it with only the package line and this comment. Add an extension here only when 2+ features need it:
```kotlin
// Kotlin extension functions shared app-wide. Feature-only extensions stay inside that feature.
```

## core/ui/theme/

If Android Studio's default theme exists (`ui/theme/`), move it here as `init-project` Step 4 says. Keep the project's real brand colors if it already has them. Otherwise use these files.

**`Color.kt`**
```kotlin
// Brand palette — replace with the real brand colors. Screens never use these directly; they read MaterialTheme.colorScheme.
val Primary = Color(0xFF3F51B5)
val OnPrimary = Color(0xFFFFFFFF)
val Secondary = Color(0xFF00897B)
val OnSecondary = Color(0xFFFFFFFF)
val ErrorColor = Color(0xFFB3261E)

val BackgroundLight = Color(0xFFFDFBFF)
val SurfaceLight = Color(0xFFFFFFFF)
val OnSurfaceLight = Color(0xFF1B1B1F)

val BackgroundDark = Color(0xFF121316)
val SurfaceDark = Color(0xFF1B1B1F)
val OnSurfaceDark = Color(0xFFE4E1E6)
```

**`Typography.kt`**
```kotlin
val AppTypography = Typography(
    titleLarge = TextStyle(fontWeight = FontWeight.SemiBold, fontSize = 22.sp, lineHeight = 28.sp),
    titleMedium = TextStyle(fontWeight = FontWeight.Medium, fontSize = 16.sp, lineHeight = 24.sp, letterSpacing = 0.15.sp),
    bodyLarge = TextStyle(fontWeight = FontWeight.Normal, fontSize = 16.sp, lineHeight = 24.sp, letterSpacing = 0.5.sp),
    bodyMedium = TextStyle(fontWeight = FontWeight.Normal, fontSize = 14.sp, lineHeight = 20.sp, letterSpacing = 0.25.sp),
    labelLarge = TextStyle(fontWeight = FontWeight.Medium, fontSize = 14.sp, lineHeight = 20.sp, letterSpacing = 0.1.sp),
)
```
Styles not listed keep the Material 3 defaults.

**`Shape.kt`**
```kotlin
val AppShapes = Shapes(
    small = RoundedCornerShape(8.dp),
    medium = RoundedCornerShape(12.dp),
    large = RoundedCornerShape(16.dp),
)
```

**`Dimens.kt`**: exactly the tokens listed in `shared/ui-components.md` ("Needs"), as an `object Dimens { val spacingMedium = 16.dp ... }`.

**`Theme.kt`**
```kotlin
private val LightColors = lightColorScheme(
    primary = Primary,
    onPrimary = OnPrimary,
    secondary = Secondary,
    onSecondary = OnSecondary,
    error = ErrorColor,
    background = BackgroundLight,
    surface = SurfaceLight,
    onBackground = OnSurfaceLight,
    onSurface = OnSurfaceLight,
)

private val DarkColors = darkColorScheme(
    primary = Primary,
    onPrimary = OnPrimary,
    secondary = Secondary,
    onSecondary = OnSecondary,
    error = ErrorColor,
    background = BackgroundDark,
    surface = SurfaceDark,
    onBackground = OnSurfaceDark,
    onSurface = OnSurfaceDark,
)

@Composable
fun AppTheme(
    darkTheme: Boolean = isSystemInDarkTheme(),
    // Off by default: the app shows its own brand colors, not the wallpaper's.
    dynamicColor: Boolean = false,
    content: @Composable () -> Unit,
) {
    val colorScheme = when {
        dynamicColor && Build.VERSION.SDK_INT >= Build.VERSION_CODES.S -> {
            val context = LocalContext.current
            if (darkTheme) dynamicDarkColorScheme(context) else dynamicLightColorScheme(context)
        }
        darkTheme -> DarkColors
        else -> LightColors
    }
    MaterialTheme(colorScheme = colorScheme, typography = AppTypography, shapes = AppShapes, content = content)
}
```

## MainActivity.kt

```kotlin
class MainActivity : ComponentActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        installSplashScreen() // before super.onCreate — see shared/project-setup.md
        super.onCreate(savedInstanceState)
        enableEdgeToEdge() // AppScreen applies the safe-drawing insets itself
        setContent {
            AppTheme {
                AppNavGraph()
            }
        }
    }
}
```
- If the start destination later depends on async data (e.g. the saved session), keep the splash with `setKeepOnScreenCondition { !isReady }`. Never use a fake delay.
- The session-expired redirect is added here when an auth feature exists (see `AuthTokenProvider` above).
