# Shared Hard Rules (all architectures)

> Applies to every project, whatever architecture pattern was chosen. `/init-project` copies the three `##` blocks
> below into every project's `CLAUDE.md` without changes. After them, it adds the chosen architecture template's own
> `## Architecture Hard Rules` block. Do not put architecture-specific rules (UseCases, MVI contracts, base classes)
> in this file. They belong in the architecture template.

## Hard Rules — Never Violate

- **When confused at any stage** — always ask, never guess
- **Never use `!!`** — use `?.`, `?: return`, or `requireNotNull` with a message
- **Never use `GlobalScope`** — always `viewModelScope` inside a ViewModel
- **Never hardcode a dispatcher** — do not use `Dispatchers.IO` / `Main` / `Default` outside `core/di/DispatcherModule.kt`. Inject a `CoroutineDispatcher` instead, for example with `get(named("io"))`
- **Never swallow `CancellationException`** — around suspend code, every `catch (e: Exception)` (or `Throwable`, or `runCatching`) must have `catch (e: CancellationException) { throw e }` before it. If a cancellation turns into an error, a dead coroutine keeps running, and the app can show a false "was cancelled" error
- **Never delay the network layer on purpose** (`Thread.sleep`, `delay`, a minimum-duration interceptor). If a loader flashes too briefly, fix it in the UI. Never fix it by slowing down the data
- **Network logging is debug-only and redacted**:
  - Add the logging interceptor only `if (BuildConfig.DEBUG)`. Never comment it out instead
  - Logs go through `AppLogger`
  - Mask passwords / OTPs / PINs / tokens in bodies
- **Never call API, Retrofit or DataStore directly from the UI layer**
- **Never put business logic inside a Composable**
- **Never use `collectAsState()`** — always `collectAsStateWithLifecycle()`
- **Never store sensitive data in `SharedPreferences`** — tokens, session and PII always go through the single DataStore from `LocalStorageModule`. You may use `SharedPreferences` only for non-sensitive values (e.g. app language), behind a `core/util` helper
- **Never compare server string fields against string literals** (e.g. `if (dto.type == "stage")`). Instead, in the mapping code, map each such field to a Kotlin `enum class` with a `fromValue(value: String?)` companion and an `Unknown` fallback. Then branch on the enum
- **Never add a library** without checking `gradle/libs.versions.toml` first
- **Strings:**
  - Never hardcode strings in UI code — always `stringResource()` with an entry in `strings.xml`
  - Before adding a string resource, check `values/strings.xml` for an existing one to reuse
  - Add every new string to `values/strings.xml` **and** to every other `values-<locale>/strings.xml` in the same change. Write natural, well-written text in each language. Never use a placeholder or a literal machine translation
- **Typography — every text uses a token** from `core/ui/theme/Typography.kt`, applied with `MaterialTheme.typography.*`:
  - Every `Text`/`TextField` gets its style in one of three ways. It sets `style = MaterialTheme.typography.<slot>`, or it inherits the app default (`LocalTextStyle` provided in `AppTheme`), or it uses a Material component's own default
  - **Never** set `fontSize`, `fontWeight`, `fontFamily`, `lineHeight`, `letterSpacing` or `fontStyle` inline. This covers a `Text` param, `.copy(...)`, a `SpanStyle` and a bare `TextStyle(...)`. You may use `.copy(...)` only for non-typographic values (`color`, `textAlign`)
  - Need bold/emphasis or a selected state? Switch to another existing token of the same size. Never override the weight
  - Once the design is known, record the design system's slot → size/weight mapping in `Typography.kt` (and in CLAUDE.md)
  - If no existing token fits the design, **stop and ask**. Add a truly new style only to `Typography.kt`, and only with explicit approval
- **UI components** — `core/ui/components/` (from `shared/ui-components.md`) is the only source of these widgets in feature code:
  - Every screen uses `AppScreen`, `AppTopBar`, `AppButton`, `AppChip`, `Avatar`, `AppOutlinedTextField`, `SearchTextField`, `AppBottomSheet`, `SearchableBottomSheet`, `AppConfirmDialog`, `ShimmerBox`/`ListSkeleton`, `LoadingIndicator`, `ErrorView`, `EmptyView`. Never use the raw `Scaffold`, `TopAppBar`, `Button`/`OutlinedButton`/`TextButton`, `FilterChip`/`AssistChip`, `TextField`/`OutlinedTextField`, `ModalBottomSheet`, `AlertDialog`, `CircularProgressIndicator` or a hand-rolled avatar/error/empty view
  - Whether a dialog or bottom sheet is showing is a State flag (`showXDialog: Boolean`), never an Effect
  - If a variant is missing, ask first. Then add it as a param to the core component. Never fork the component inside a feature
- **Every screen's body composes inside `AppScreen`** (Scaffold + loading + snackbar + empty/error body + pull-to-refresh). Skip it only for a static screen with no loading/error/empty state. When you skip it, add an inline comment that notes the exception
  - Messages go through the one `snackbarMessage: AppSnackbar?` param. It supports Error/Info/Success/Warning, each with its own `AppMessageDuration` and an optional action. Build error messages with the architecture's `toErrorMessage(...)` or `errorSnackbarMessage(...)`. Never add a separate snackbar/`SnackbarHost` to a screen
  - "Nothing to show" goes through `empty: EmptyState?` (`EmptyState(...).takeIf { nothingToShow }`; null means not empty). This also applies after a failed first load. In that case AppScreen shows `ErrorView` with Retry instead of the empty view
  - Loading goes through the one `loading: LoadingState` param (`isLoading` + `display` Line/Skeleton/ReplaceContent + `blockInput`). If a screen draws its own first-load skeleton, it uses `LoadingDisplay.Skeleton`
  - Never disable a screen's buttons one by one while loading. AppScreen already blocks input to the content and locks `AppTopBar` actions. Opt out with `LoadingState(blockInput = false)` only when the screen really must stay interactive
  - `refresh: RefreshState?` has no default, so every screen must set it:
    - **Screens that show server data** (lists, dashboards, details) pass `RefreshState(isRefreshing, onRefresh)`. Back it with a refresh action and an `isRefreshing` State field. A refresh keeps the content on screen: `isLoading = false` while `isRefreshing = true`
    - **Forms, auth, pickers/search and static screens** pass `refresh = null`
    - The content must scroll (`verticalScroll` / `LazyColumn`). If it does not, the pull gesture never reaches `PullToRefreshBox`
  - Build `AppSnackbar` in the Composable. Never store it in ViewModel State, because it holds lambdas. State keeps the error, and the Screen converts it
- **Components** — start every new Composable at screen level. Move it up to feature level when 2+ screens of the feature use it. Move it up to `core/ui/components/` when 2+ features use it. Never move a component down
- **Every `@Composable` screen and `core/ui/components/` component includes a `@Preview`** with typical sample state
- **Forms**:
  - State holds one `xxxError: ValidationError?` per field, never a display string
  - Check each field with `core/util/Validator`. Convert the error to text at render time with `toMessage()`
  - Max/min lengths come from `core/util/InputLimits`. The field's `maxLength` and its validation use the same limit
  - Validate on submit. After that, re-validate a field on each change
- **Screen results** — a picker or child screen returns its value with `setNavResult` / `NavResultEffect` from `navigation/NavResult.kt`, keyed by a `NavResultKeys` constant. Inputs go through route arguments (ids and simple values). Never pass data between screens through a shared ViewModel, a singleton or a global event bus
- **Dates** — the mapper parses server dates into `java.time` types (`DateTimeFormat.parseInstant` / `parseDate`). Models never hold date strings. Display text comes only from `core/util/DateTimeFormat`. Never use `SimpleDateFormat` or a hand-written pattern in UI code
- **DTOs** live in `data/remote/dto/`, and every field has `@SerializedName`. The R8 keep rule depends on both
- **Base URLs** come only from `BuildConfig.BASE_URL` (one per build type). Never hardcode a URL anywhere else
- **Tests**:
  - Every new ViewModel comes with a `<Name>ViewModelTest` (success, error and one Effect/Event path)
  - Every new mapper comes with a mapper test
  - Build tests on fakes (no mocking library) and `MainDispatcherRule`
  - `./gradlew ktlintCheck testDebugUnitTest` must pass before a change is done

## Do Not Touch

- `app/release/` — signing configs
- `*.keystore`, `*.jks`, `keystore.properties` — signing files. Never commit them. Never read them into chat
- `gradle/libs.versions.toml` — update only with explicit instruction

## Security

- No API keys or secrets in source. Use `BuildConfig` fields
- Signing passwords live only in the git-ignored `keystore.properties` (or CI secrets). Never put them in a build file or `gradle.properties`
- Never log tokens, passwords, or user PII
- Store sensitive local data in DataStore (never `SharedPreferences`). Non-sensitive values (e.g. app language) may use `SharedPreferences`
