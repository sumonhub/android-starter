# Shared Core Utilities (all architectures)

> Applies regardless of which architecture pattern was chosen. `/init-project` creates every file below. The usage
> rules live in `shared/hard-rules.md`, so they reach every project's `CLAUDE.md`. All of these are pure Kotlin except
> `NavResult.kt`, so a Clean domain layer may use `Validator` and `InputLimits` too.

```
core/util/
├── Validator.kt        # field-format checks → ValidationError?
├── InputLimits.kt      # max/min lengths shared by fields and validation
└── DateTimeFormat.kt   # the one place server dates are parsed and dates are formatted for display
navigation/
└── NavResult.kt        # return a value to the previous screen (pickers)
```

## Validator.kt

```kotlin
sealed interface ValidationError {
    data object Required : ValidationError
    data object InvalidEmail : ValidationError
    data object InvalidPhone : ValidationError
    data class TooShort(val min: Int) : ValidationError
    data class TooLong(val max: Int) : ValidationError
}

object Validator {
    private val emailRegex = Regex("^[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\\.[A-Za-z]{2,}$")
    private val phoneRegex = Regex("^\\+?[0-9]{7,15}$")

    fun required(value: String): ValidationError? =
        if (value.isBlank()) ValidationError.Required else null

    fun email(value: String): ValidationError? =
        required(value) ?: if (emailRegex.matches(value.trim())) null else ValidationError.InvalidEmail

    fun phone(value: String): ValidationError? =
        required(value) ?: if (phoneRegex.matches(value.filterNot { it == ' ' || it == '-' })) null else ValidationError.InvalidPhone

    fun length(value: String, min: Int = 0, max: Int = Int.MAX_VALUE): ValidationError? = when {
        value.length < min -> ValidationError.TooShort(min)
        value.length > max -> ValidationError.TooLong(max)
        else -> null
    }

    fun password(value: String): ValidationError? =
        required(value) ?: length(value, min = InputLimits.PASSWORD_MIN, max = InputLimits.PASSWORD_MAX)
}
```
The user-facing text lives in the UI layer — `@Composable fun ValidationError.toMessage(): String` in `core/ui/components/AppOutlinedTextField.kt` (see `shared/ui-components.md`).

**Form pattern** (every form screen):
- State holds one `xxxError: ValidationError?` per field, never a display string
- Validate on submit; after the first submit attempt, re-validate a field on each change so the error clears as the user fixes it
- Submit runs only when every field's error is null
- Field-format checks (empty, email, length) use `Validator`; business rules that need data (e.g. "email already taken") come from the API/UseCase

## InputLimits.kt

```kotlin
object InputLimits {
    const val NAME_MAX = 50
    const val EMAIL_MAX = 100
    const val PHONE_MAX = 15
    const val PASSWORD_MIN = 8
    const val PASSWORD_MAX = 64
    const val DESCRIPTION_MAX = 500
}
```
Add a limit here when a new field needs one — `AppOutlinedTextField(maxLength = InputLimits.X)` and `Validator.length(max = InputLimits.X)` use the same constant.

## DateTimeFormat.kt

```kotlin
object DateTimeFormat {
    /** Server timestamps (ISO-8601) → Instant. Called in mappers only; null for missing/unparseable values. */
    fun parseInstant(value: String?): Instant? =
        value?.let { runCatching { Instant.parse(it) }.getOrNull() }

    /** Server dates (yyyy-MM-dd) → LocalDate. */
    fun parseDate(value: String?): LocalDate? =
        value?.let { runCatching { LocalDate.parse(it) }.getOrNull() }

    fun date(instant: Instant, zone: ZoneId = ZoneId.systemDefault(), locale: Locale = Locale.getDefault()): String =
        DateTimeFormatter.ofLocalizedDate(FormatStyle.MEDIUM).withLocale(locale).format(instant.atZone(zone))

    fun date(date: LocalDate, locale: Locale = Locale.getDefault()): String =
        DateTimeFormatter.ofLocalizedDate(FormatStyle.MEDIUM).withLocale(locale).format(date)

    fun time(instant: Instant, zone: ZoneId = ZoneId.systemDefault(), locale: Locale = Locale.getDefault()): String =
        DateTimeFormatter.ofLocalizedTime(FormatStyle.SHORT).withLocale(locale).format(instant.atZone(zone))

    fun dateTime(instant: Instant, zone: ZoneId = ZoneId.systemDefault(), locale: Locale = Locale.getDefault()): String =
        DateTimeFormatter.ofLocalizedDateTime(FormatStyle.MEDIUM, FormatStyle.SHORT).withLocale(locale).format(instant.atZone(zone))
}
```
- `java.time` needs minSdk 26+; below that, enable core library desugaring
- Localized styles (`FormatStyle`) follow the device language — no hand-written patterns like `"dd/MM/yyyy"` unless the design explicitly requires a fixed format, in which case add a named function here

## NavResult.kt

```kotlin
/** Keys for values passed back between screens — one constant per result, never an inline string. */
object NavResultKeys {
    // e.g. const val SELECTED_CATEGORY = "selected_category"
}

/** Return a value to the previous screen. Call right before navController.popBackStack(). */
fun <T> NavController.setNavResult(key: String, value: T) {
    previousBackStackEntry?.savedStateHandle?.set(key, value)
}

/** In the waiting screen: runs onResult once per returned value, then clears it so it never fires twice. */
@Composable
fun <T : Any> NavBackStackEntry.NavResultEffect(key: String, onResult: (T) -> Unit) {
    val handle = savedStateHandle
    val result by handle.getStateFlow<T?>(key, null).collectAsStateWithLifecycle()
    val currentOnResult by rememberUpdatedState(onResult)
    LaunchedEffect(result) {
        result?.let {
            currentOnResult(it)
            handle.remove<T>(key)
        }
    }
}
```
- Values must be storable in a `Bundle`: primitives, `String`, `@Parcelize` classes, or lists of those — pass a small result (an id, or a tiny `@Parcelize` model), never a whole screen's data
- **Inputs** go the other way through route arguments (ids and simple values) — never through `savedStateHandle` or a shared ViewModel
- Wiring: the picker's screen calls `navController.setNavResult(...)` + `popBackStack()` from its Effect/Event handler in `AppNavGraph`; the caller's `composable(...) { entry -> entry.NavResultEffect<Category>(NavResultKeys.SELECTED_CATEGORY) { viewModel.processIntent(...) /* or viewModel.onCategorySelected(it) */ } }`
