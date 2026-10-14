# Shared Hard Rules (all architectures)

> Applies regardless of which architecture pattern was chosen. `/init-project` writes the three `##` blocks
> below into every project's `CLAUDE.md`, unmodified, followed by the chosen architecture template's own
> `## Architecture Hard Rules` block. Keep architecture-specific rules (UseCases, MVI contracts, base classes)
> out of this file — they belong in the architecture template.

## Hard Rules — Never Violate

- **When confused at any stage** — always ask, never guess
- **Never use `!!`** — use `?.`, `?: return`, or `requireNotNull` with a message
- **Never use `GlobalScope`** — always `viewModelScope` inside a ViewModel
- **Never hardcode a dispatcher** — no `Dispatchers.IO` / `Main` / `Default` outside `core/di/DispatcherModule.kt`; inject a `CoroutineDispatcher` via `get(named("io"))` etc.
- **Never swallow `CancellationException`** — any `catch (e: Exception)` (or `Throwable`, or `runCatching`) around suspend code is preceded by `catch (e: CancellationException) { throw e }`. Turning a cancellation into an error keeps a dead coroutine running and can show a false "was cancelled" error
- **Never delay the network layer on purpose** (`Thread.sleep`, `delay`, a minimum-duration interceptor) — a loader that flashes too briefly is fixed in the UI, never by slowing the data down
- **Network logging is debug-only and redacted** — the logging interceptor is added only `if (BuildConfig.DEBUG)` (never commented out), logs go through `AppLogger`, and passwords / OTPs / PINs / tokens in bodies are masked
- **Never call API, Retrofit or DataStore directly from the UI layer**
- **Never put business logic inside a Composable**
- **Never use `collectAsState()`** — always `collectAsStateWithLifecycle()`
- **Never store sensitive data in `SharedPreferences`** — tokens, session, PII always go through the single DataStore from `LocalStorageModule`; `SharedPreferences` is allowed only for non-sensitive values (e.g. app language), behind a `core/util` helper
- **Never compare server string fields against string literals** (e.g. `if (dto.type == "stage")`) — map each such field to a Kotlin `enum class` with a `fromValue(value: String?)` companion (plus an `Unknown` fallback) in the mapping code, and branch on the enum
- **Never add a library** without checking `gradle/libs.versions.toml` first
- **Strings:**
  - Never hardcode strings in UI code — always `stringResource()` with an entry in `strings.xml`
  - Before adding a string resource, check `values/strings.xml` for an existing one to reuse
  - Every new string is added to `values/strings.xml` **and** every other `values-<locale>/strings.xml` in the same change — natural, well-written text in each language, never a placeholder or literal machine translation
- **Typography — every text uses a token** from `core/ui/theme/Typography.kt`, applied via `MaterialTheme.typography.*`:
  - Every `Text`/`TextField` either sets `style = MaterialTheme.typography.<slot>` or inherits the app default (`LocalTextStyle` provided in `AppTheme`) or a Material component's own default
  - **Never** set `fontSize`, `fontWeight`, `fontFamily`, `lineHeight`, `letterSpacing` or `fontStyle` inline — not as a `Text` param, not in `.copy(...)`, not in a `SpanStyle`, not in a bare `TextStyle(...)`. Only non-typographic `.copy(...)` is allowed (`color`, `textAlign`)
  - Need bold/emphasis or a selected state? Switch to another existing token of the same size — never override the weight
  - Record the design system's slot → size/weight mapping in `Typography.kt` (and in CLAUDE.md) once the design is known
  - If no existing token fits the design, **stop and ask** — a genuinely new style is added only to `Typography.kt`, with explicit approval
- **UI components** — `core/ui/components/` (from `shared/ui-components.md`) is the only source of these widgets in feature code:
  - Every screen uses `AppScreen`, `AppTopBar`, `AppButton`, `AppChip`, `Avatar`, `AppOutlinedTextField`, `SearchTextField`, `AppBottomSheet`, `SearchableBottomSheet`, `AppConfirmDialog`, `ShimmerBox`/`ListSkeleton`, `LoadingIndicator`, `ErrorView`, `EmptyView` — never the raw `Scaffold`, `TopAppBar`, `Button`/`OutlinedButton`/`TextButton`, `FilterChip`/`AssistChip`, `TextField`/`OutlinedTextField`, `ModalBottomSheet`, `AlertDialog`, `CircularProgressIndicator` or a hand-rolled avatar/error/empty view
  - Whether a dialog or bottom sheet is showing is a State flag (`showXDialog: Boolean`), never an Effect
  - A missing variant is added as a param to the core component (ask first) — never forked inside a feature
- **Every screen's body composes inside `AppScreen`** (Scaffold + loading + snackbar + empty/error body + pull-to-refresh) — skip only for a static screen with no loading/error/empty state, and note the exception inline with a comment when skipping
  - Messages go through the one `snackbarMessage: AppSnackbar?` param (Error/Info/Success/Warning, each with its own `AppMessageDuration` and optional action — build error ones with the architecture's `toErrorMessage(...)` or `errorSnackbarMessage(...)`). Never add a separate snackbar/`SnackbarHost` to a screen
  - "Nothing to show" goes through `empty: EmptyState?` (`EmptyState(...).takeIf { nothingToShow }` — null means not empty), including after a failed first load: AppScreen then shows `ErrorView` with Retry instead of the empty view
  - Loading goes through the one `loading: LoadingState` param (`isLoading` + `display` Line/Skeleton/ReplaceContent + `blockInput`). A screen drawing its own first-load skeleton uses `LoadingDisplay.Skeleton`
  - Never disable a screen's buttons one by one while loading — AppScreen already blocks content input and locks `AppTopBar` actions; opt out with `LoadingState(blockInput = false)` only when the screen genuinely must stay interactive
  - `refresh: RefreshState?` has no default — every screen states it. **Screens that show server data** (lists, dashboards, details) pass `RefreshState(isRefreshing, onRefresh)`, backed by a refresh action + `isRefreshing` State field (refresh keeps content on screen: `isLoading = false` while `isRefreshing = true`). **Forms, auth, pickers/search and static screens** pass `refresh = null`. The content must scroll (`verticalScroll` / `LazyColumn`), or the pull gesture never reaches `PullToRefreshBox`
  - `AppSnackbar` is built in the Composable, never stored in ViewModel State (it holds lambdas) — State keeps the error, the Screen converts it
- **Components** — start every new Composable at screen level; promote to feature level when 2+ screens of the feature use it, and to `core/ui/components/` when 2+ features use it. Never move a component down
- **Every `@Composable` screen and `core/ui/components/` component includes a `@Preview`** with representative sample state
- **Forms** — State holds one `xxxError: ValidationError?` per field (never a display string), checked with `core/util/Validator` and converted to text at render time with `toMessage()`; max/min lengths come from `core/util/InputLimits`, shared by the field's `maxLength` and its validation. Validate on submit, then re-validate a field on each change
- **Screen results** — a picker or child screen returns its value with `setNavResult` / `NavResultEffect` from `navigation/NavResult.kt`, keyed by a `NavResultKeys` constant; inputs go through route arguments (ids and simple values). Never pass data between screens through a shared ViewModel, a singleton or a global event bus
- **Dates** — server dates are parsed into `java.time` types in the mapper (`DateTimeFormat.parseInstant` / `parseDate`); models never hold date strings. Display text comes only from `core/util/DateTimeFormat` — never `SimpleDateFormat` or a hand-written pattern in UI code
- **DTOs** live in `data/remote/dto/` and every field carries `@SerializedName` — the R8 keep rule depends on both
- **Base URLs** come only from `BuildConfig.BASE_URL` (one per build type) — never a URL hardcoded anywhere else
- **Tests** — every new ViewModel ships with a `<Name>ViewModelTest` (success, error and one Effect/Event path) and every new mapper with a mapper test, built on fakes (no mocking library) and `MainDispatcherRule`; `./gradlew ktlintCheck testDebugUnitTest` must pass before a change is done

## Do Not Touch

- `app/release/` — signing configs
- `*.keystore`, `*.jks`, `keystore.properties` — signing files; never committed, never read into chat
- `gradle/libs.versions.toml` — only update with explicit instruction

## Security

- No API keys or secrets in source — use `BuildConfig` fields
- Signing passwords live only in the git-ignored `keystore.properties` (or CI secrets) — never in a build file or `gradle.properties`
- No logging of tokens, passwords, or user PII
- Sensitive local data → DataStore (never `SharedPreferences`); non-sensitive values (e.g. app language) may use `SharedPreferences`
