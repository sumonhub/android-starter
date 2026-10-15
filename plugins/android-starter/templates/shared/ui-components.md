# Shared UI Components (all architectures)

> This file applies to every architecture pattern.
> For every architecture, `/android-starter:init-project` creates every component below in `core/ui/components/`.
> It also creates the error → snackbar converter for the chosen architecture (see the last section).
> The usage rules are in `shared/hard-rules.md` (`UI components`). That way they reach every project's `CLAUDE.md`.
>
> Every component file ends with one or more `@Preview`s. Each preview is wrapped in `AppTheme` and uses typical sample values.
> The previews are not shown below, to keep this file short.
> When a real need appears, add params to the component. Never make a separate copy of it for one screen or feature.

## Components

```
core/ui/components/
├── AppScreen.kt          # every screen's wrapper: Scaffold + loading + snackbar + empty/error body + pull-to-refresh
│                         # also LoadingDisplay, LoadingState, RefreshState, rememberMinimumLoading, LocalScreenInputBlocked
├── AppSnackbar.kt        # AppSnackbar + AppMessageType + AppMessageDuration + errorSnackbarMessage() + the architecture's toErrorMessage()
├── AppTopBar.kt          # top bar — locks its actions (not Back) while AppScreen blocks input
├── AppButton.kt          # Primary / Secondary / Text button
├── AppChip.kt            # selectable chip (filters, tags)
├── Avatar.kt             # circular image with initials fallback
├── LoadingIndicator.kt   # centered spinner
├── ErrorView.kt          # centered error message + Retry — AppScreen shows it when there is nothing else to show
├── EmptyView.kt          # EmptyView + EmptyState
├── AppOutlinedTextField.kt  # text field: label, error, max length, password toggle + ValidationError.toMessage()
├── SearchTextField.kt    # search field with clear button
├── AppBottomSheet.kt     # ModalBottomSheet wrapper (title, insets, padding)
├── SearchableBottomSheet.kt  # generic pick-one-from-a-list sheet with local search
├── Shimmer.kt            # ShimmerBox + ListSkeleton — what LoadingDisplay.Skeleton screens draw
└── AppConfirmDialog.kt   # confirm / discard / delete dialog
```

**Needs:**
- Gradle:
  - Compose material3 **1.3+**. It is needed for `PullToRefreshBox`.
  - `androidx.compose.material:material-icons-core`. material3 no longer includes the icons, so add this yourself.
  - Coil 3 `coil-compose` + `coil-network-okhttp`. Avatar uses them to load images from URLs.
  - `buildFeatures { buildConfig = true }`. It is needed for `BuildConfig.DEBUG`.
- `Dimens.kt` tokens: `spacingXSmall` (4.dp), `spacingSmall` (8.dp), `spacingMedium` (16.dp), `spacingLarge` (24.dp), `iconSmall` (18.dp), `buttonHeight` (48.dp), `avatarSmall` (32.dp), `avatarMedium` (48.dp), `avatarLarge` (72.dp), `illustrationSize` (96.dp)
- Strings in `values/strings.xml` **and every other locale**: `txt_retry`, `txt_error_details`, `cd_back`, `error_no_internet`, `error_timeout`, `error_unauthorized`, `error_server`, `error_generic`, `txt_search`, `cd_clear_search`, `txt_no_results`, `txt_confirm`, `txt_cancel`, `txt_show`, `txt_hide`, `error_field_required`, `error_invalid_email`, `error_invalid_phone`, `error_too_short` (`%1$d`), `error_too_long` (`%1$d`)
- `core/util/Validator.kt` and `InputLimits.kt` from `shared/core-utils.md`. `AppOutlinedTextField` uses them.

## AppScreen.kt

**What it does for every screen (no per-screen code needed):**
- Edge-to-edge `Scaffold` (`WindowInsets.safeDrawing`) with `Dimens.spacingMedium` padding around the content.
- When the user taps outside a field, it clears focus and hides the keyboard.
- `loading: LoadingState` has three fields: `isLoading`, `display` and `blockInput`. `display` is one of:
  - `LoadingDisplay.Line` (default): a thin progress line under the top bar. The content stays visible.
  - `LoadingDisplay.Skeleton`: no line. The screen draws its own shimmer skeleton (for the first load).
  - `LoadingDisplay.ReplaceContent`: a centered `LoadingIndicator` in place of the body.
- While loading, **all touches on the content are blocked** (`blockInput = true` is the default).
  - `AppTopBar`'s actions are also locked, through `LocalScreenInputBlocked`.
  - Back, the bottom bar, the FAB and the snackbar's Retry stay usable.
- `empty: EmptyState?`: a non-null value means "nothing to show". `EmptyView` then replaces the body.
  - **If an Error snackbar is also set, `ErrorView` (with its Retry) replaces the body instead, and no snackbar is shown.**
  - This way a failed first load never looks like "No items".
- `refresh: RefreshState?`: **required, with no default.**
  - `RefreshState(isRefreshing, onRefresh)` wraps the body in `PullToRefreshBox`.
  - `null` means the screen has no pull-to-refresh.
- `snackbarMessage: AppSnackbar?`: the screen's single snackbar, used for Error, Info, Success and Warning.

```kotlin
/** How AppScreen shows that it's loading. */
enum class LoadingDisplay {
    /** Thin progress line under the top bar; content stays visible. */
    Line,

    /** No line — the screen draws its own shimmer skeleton (first load); input is still blocked. */
    Skeleton,

    /** Swaps the body for a centered LoadingIndicator. */
    ReplaceContent,
}

/** Everything AppScreen needs to know about loading. Built in the Screen from the State's isLoading — display is a UI choice. */
data class LoadingState(
    val isLoading: Boolean = false,
    val display: LoadingDisplay = LoadingDisplay.Line,
    /** False keeps the content tappable while loading (e.g. a live search list). */
    val blockInput: Boolean = true,
)

/** Pull-to-refresh; a null `refresh` means the screen has none. Built in the Screen, never in ViewModel state (holds a lambda). */
data class RefreshState(
    val isRefreshing: Boolean,
    val onRefresh: () -> Unit,
)

/** True while the enclosing AppScreen is loading with input blocked — AppTopBar reads it to lock its actions (Back stays usable). */
internal val LocalScreenInputBlocked = compositionLocalOf { false }

/** Swallows every pointer event (taps, drags, scrolls) on this node's bounds. */
internal fun Modifier.blockPointerInput(): Modifier = pointerInput(Unit) {
    awaitPointerEventScope {
        while (true) {
            awaitPointerEvent(PointerEventPass.Initial).changes.forEach { it.consume() }
        }
    }
}

@OptIn(ExperimentalMaterial3Api::class)
@Composable
fun AppScreen(
    modifier: Modifier = Modifier,
    topBar: (@Composable () -> Unit)? = null,
    floatingActionButton: @Composable () -> Unit = {},
    bottomBar: @Composable () -> Unit = {},
    loading: LoadingState = LoadingState(),
    empty: EmptyState? = null,
    snackbarMessage: AppSnackbar? = null,
    refresh: RefreshState?, // no default — every screen decides: RefreshState(...) or null
    containerColor: Color = MaterialTheme.colorScheme.background,
    content: @Composable BoxScope.() -> Unit,
) {
    val focusManager = LocalFocusManager.current
    val keyboardController = LocalSoftwareKeyboardController.current
    val shownLoading = rememberMinimumLoading(loading.isLoading)
    val isInputBlocked = shownLoading && loading.blockInput
    // Nothing to show + an error → the error becomes the body (with Retry) instead of a snackbar over "No items".
    val bodyError = snackbarMessage?.takeIf { it.type == AppMessageType.Error && empty != null }

    CompositionLocalProvider(LocalScreenInputBlocked provides isInputBlocked) {
        Box(
            modifier = modifier
                .fillMaxSize()
                .pointerInput(Unit) {
                    detectTapGestures(onTap = {
                        focusManager.clearFocus(force = true)
                        keyboardController?.hide()
                    })
                },
        ) {
            Scaffold(
                modifier = Modifier.fillMaxSize(),
                topBar = topBar ?: {},
                floatingActionButton = floatingActionButton,
                bottomBar = bottomBar,
                containerColor = containerColor,
                contentWindowInsets = WindowInsets.safeDrawing,
            ) { innerPadding ->
                val contentModifier = Modifier
                    .padding(innerPadding)
                    .consumeWindowInsets(innerPadding)
                    .padding(Dimens.spacingMedium)
                    .fillMaxSize()

                @Composable
                fun BoxScope.ScreenBody() {
                    when {
                        shownLoading && loading.display == LoadingDisplay.ReplaceContent -> LoadingIndicator()
                        // Held back until the minimum loading window ends, so a fast response doesn't cut a loader short.
                        bodyError != null && !shownLoading -> ErrorView(
                            message = bodyError.text,
                            actionLabel = bodyError.actionLabel,
                            onAction = bodyError.onAction,
                        )
                        empty != null && !shownLoading -> EmptyView(title = empty.title, message = empty.message, action = empty.action)
                        else -> content()
                    }
                }

                if (refresh != null) {
                    PullToRefreshBox(isRefreshing = refresh.isRefreshing, onRefresh = refresh.onRefresh, modifier = contentModifier) {
                        ScreenBody()
                    }
                } else {
                    Box(modifier = contentModifier) { ScreenBody() }
                }

                if (isInputBlocked) {
                    // Sized to the content slot only (between top and bottom bars) — swallows touches on the content, nothing else.
                    Box(modifier = Modifier.padding(innerPadding).fillMaxSize().blockPointerInput())
                }

                // Line mode only — Skeleton draws its own placeholder, ReplaceContent its spinner.
                if (loading.display == LoadingDisplay.Line) {
                    // Inner padding only (not the content padding) so the line spans edge to edge under the top bar.
                    Box(modifier = Modifier.padding(innerPadding).fillMaxWidth()) {
                        AnimatedVisibility(visible = shownLoading, enter = fadeIn(), exit = fadeOut()) {
                            LinearProgressIndicator(
                                modifier = Modifier.fillMaxWidth().height(Dimens.spacingXSmall),
                                color = MaterialTheme.colorScheme.primary,
                                trackColor = MaterialTheme.colorScheme.primaryContainer,
                                strokeCap = StrokeCap.Round,
                            )
                        }
                    }
                }
            }

            AppSnackbarHost(
                message = snackbarMessage.takeIf { bodyError == null },
                modifier = Modifier
                    .align(Alignment.BottomCenter)
                    .imePadding()
                    .padding(Dimens.spacingMedium),
            )
        }
    }
}
```

**`rememberMinimumLoading` (public, same file)**: once any loading UI is shown, this keeps it on screen for at least `MIN_LOADING_VISIBLE_MS`.
This stops a loader from flashing briefly when the response is fast.
It only affects the UI. The data is never delayed.
```kotlin
const val MIN_LOADING_VISIBLE_MS = 1_000L

@Composable
fun rememberMinimumLoading(isLoading: Boolean, minVisibleMs: Long = MIN_LOADING_VISIBLE_MS): Boolean {
    var shown by remember { mutableStateOf(isLoading) }
    var shownSinceMs by remember { mutableLongStateOf(if (isLoading) SystemClock.uptimeMillis() else 0L) }
    LaunchedEffect(isLoading) {
        if (isLoading) {
            if (!shown) {
                shown = true
                shownSinceMs = SystemClock.uptimeMillis()
            }
        } else if (shown) {
            val remainingMs = minVisibleMs - (SystemClock.uptimeMillis() - shownSinceMs)
            if (remainingMs > 0) delay(remainingMs)
            shown = false
        }
    }
    // `|| isLoading` so the first frame of a load already shows it, before the effect runs.
    return shown || isLoading
}
```
Some screens draw their **own first-load skeleton**. Such a screen shows the skeleton **instead of** the data. It never shows the skeleton above the data.
```kotlin
val isInitialLoad = rememberMinimumLoading(state.isLoading && state.items.isEmpty())
AppScreen(
    loading = LoadingState(
        isLoading = state.isLoading,
        display = if (isInitialLoad) LoadingDisplay.Skeleton else LoadingDisplay.Line,
    ),
    refresh = ...,
) {
    if (isInitialLoad) ListSkeleton() else ItemList(state.items)
}
```

**`AppSnackbarHost` (private, same file)**: draws an `AppSnackbar`. It must keep all of these behaviours. Each one fixes a real bug.
- Colours:
  - One `when` maps each `AppMessageType` to an accent colour. This is the only place to change the palette.
  - The container colour is a 10% tint of the accent, `compositeOver(colorScheme.surface)`. This keeps it opaque over the content.
- It remembers the `last` non-null message. After `message` becomes null, the exit animation still has something to draw.
- Auto-dismiss:
  - Flow: `LaunchedEffect(isVisible, attempt, message?.text, message?.type)` → `delay(duration.millis)` → `onDismiss()`.
  - `Indefinite` (`millis == null`) never auto-dismisses.
  - `attempt` goes up by one on every action tap. So when a retry fails with the **same** text, the message still gets a new full display time.
  - Read `onDismiss` through `rememberUpdatedState`.
- Swipe-dismiss:
  - Use `SwipeToDismissBox` and react to `snapshotFlow { dismissState.currentValue }`.
  - Do **not** use `confirmValueChange`/`targetValue`. If you dismiss in the middle of a swipe, the two animations fight, and a thin strip of the snackbar stays stuck on screen.
  - Call `dismissState.reset()` when a new message shows.
- Text uses `MaterialTheme.typography.bodyMedium`, at most 2 lines, with an ellipsis.
- `if (BuildConfig.DEBUG && details != null)`: show a "Details" button (`R.string.txt_error_details`). It opens a bottom sheet. The sheet shows the details in a `SelectionContainer` with vertical scroll.
- `actionLabel` + `onAction` → a `TextButton`. On tap, it first increases `attempt`, then calls `onAction`.

## AppSnackbar.kt

```kotlin
enum class AppMessageType { Info, Success, Warning, Error }

/** Auto-dismiss time; Indefinite waits for a swipe or the action. */
enum class AppMessageDuration(val millis: Long?) {
    Short(3_000L),
    Long(6_000L),
    Indefinite(null),
}

/**
 * One message for AppScreen's snackbar. Built in the Composable layer, never kept in ViewModel state
 * (it holds lambdas) — the ViewModel reports what happened (an error in State, or a one-shot event),
 * the screen turns it into an AppSnackbar.
 */
data class AppSnackbar(
    val text: String,
    val type: AppMessageType = AppMessageType.Info,
    val duration: AppMessageDuration = AppMessageDuration.Short,
    val actionLabel: String? = null,
    val onAction: (() -> Unit)? = null,
    val details: String? = null,          // debug-only "Details" sheet: stack trace / raw response
    val onDismiss: () -> Unit = {},
)

/** Error message (Error, Long, Retry when onRetry is set) from loose strings; null text → null. */
@Composable
fun errorSnackbarMessage(
    text: String?,
    details: String? = null,
    onRetry: (() -> Unit)? = null,
    onDismiss: () -> Unit = {},
): AppSnackbar? {
    val retryLabel = stringResource(R.string.txt_retry)
    return text?.let {
        AppSnackbar(
            text = it,
            type = AppMessageType.Error,
            duration = AppMessageDuration.Long,
            actionLabel = onRetry?.let { retryLabel },
            onAction = onRetry,
            details = details,
            onDismiss = onDismiss,
        )
    }
}
```
Put the architecture's `toErrorMessage()` converter (see the last section) in this same file.

## AppTopBar.kt

```kotlin
private const val DISABLED_ALPHA = 0.38f

@OptIn(ExperimentalMaterial3Api::class)
@Composable
fun AppTopBar(
    title: String,
    modifier: Modifier = Modifier,
    onBack: (() -> Unit)? = null,
    actions: @Composable RowScope.() -> Unit = {},
) {
    val inputBlocked = LocalScreenInputBlocked.current
    TopAppBar(
        modifier = modifier,
        title = { Text(text = title, style = MaterialTheme.typography.titleLarge, maxLines = 1, overflow = TextOverflow.Ellipsis) },
        navigationIcon = {
            if (onBack != null) {
                IconButton(onClick = onBack) {
                    Icon(Icons.AutoMirrored.Filled.ArrowBack, contentDescription = stringResource(R.string.cd_back))
                }
            }
        },
        actions = {
            // Back stays usable; the actions lock while AppScreen blocks input.
            Row(
                modifier = if (inputBlocked) Modifier.alpha(DISABLED_ALPHA).blockPointerInput() else Modifier,
                verticalAlignment = Alignment.CenterVertically,
                content = actions,
            )
        },
    )
}
```

## AppButton.kt

```kotlin
enum class AppButtonStyle { Primary, Secondary, Text }

@Composable
fun AppButton(
    text: String,
    onClick: () -> Unit,
    modifier: Modifier = Modifier,
    style: AppButtonStyle = AppButtonStyle.Primary,
    enabled: Boolean = true,
    leadingIcon: ImageVector? = null,
) {
    val content: @Composable RowScope.() -> Unit = {
        if (leadingIcon != null) {
            Icon(leadingIcon, contentDescription = null, modifier = Modifier.size(Dimens.iconSmall))
            Spacer(Modifier.width(Dimens.spacingSmall))
        }
        Text(text = text, style = MaterialTheme.typography.labelLarge)
    }
    val buttonModifier = modifier.heightIn(min = Dimens.buttonHeight)
    when (style) {
        AppButtonStyle.Primary -> Button(onClick = onClick, modifier = buttonModifier, enabled = enabled, content = content)
        AppButtonStyle.Secondary -> OutlinedButton(onClick = onClick, modifier = buttonModifier, enabled = enabled, content = content)
        AppButtonStyle.Text -> TextButton(onClick = onClick, modifier = buttonModifier, enabled = enabled, content = content)
    }
}
```
`enabled` is for form validity. For example, "Submit" stays disabled until the form is valid.
Never use it for loading. AppScreen already blocks input while loading.

## AppChip.kt

```kotlin
@Composable
fun AppChip(
    label: String,
    selected: Boolean,
    onClick: () -> Unit,
    modifier: Modifier = Modifier,
    enabled: Boolean = true,
    leadingIcon: ImageVector? = null,
) {
    FilterChip(
        selected = selected,
        onClick = onClick,
        modifier = modifier,
        enabled = enabled,
        label = { Text(text = label, style = MaterialTheme.typography.labelLarge) },
        leadingIcon = leadingIcon?.let { icon ->
            { Icon(icon, contentDescription = null, modifier = Modifier.size(FilterChipDefaults.IconSize)) }
        },
    )
}
```
The chip's own colours show the selected state. The label keeps the same typography token.

## Avatar.kt

```kotlin
@Composable
fun Avatar(
    imageUrl: String?,
    name: String,
    modifier: Modifier = Modifier,
    size: Dp = Dimens.avatarMedium,
) {
    val initials = name.trim().split(Regex("\\s+")).filter { it.isNotEmpty() }.take(2)
        .joinToString("") { it.first().uppercase() }
    Box(
        modifier = modifier.size(size).clip(CircleShape).background(MaterialTheme.colorScheme.primaryContainer),
        contentAlignment = Alignment.Center,
    ) {
        // Initials sit underneath: visible while the image loads, when it fails, or when there is no URL.
        Text(text = initials, style = MaterialTheme.typography.titleMedium, color = MaterialTheme.colorScheme.onPrimaryContainer)
        if (!imageUrl.isNullOrBlank()) {
            AsyncImage(
                model = imageUrl,
                contentDescription = name,
                contentScale = ContentScale.Crop,
                modifier = Modifier.matchParentSize(),
            )
        }
    }
}
```

## LoadingIndicator.kt, ErrorView.kt, EmptyView.kt

```kotlin
@Composable
fun LoadingIndicator(modifier: Modifier = Modifier) {
    Box(modifier = modifier.fillMaxSize(), contentAlignment = Alignment.Center) {
        CircularProgressIndicator()
    }
}

@Composable
fun ErrorView(
    message: String,
    modifier: Modifier = Modifier,
    actionLabel: String? = null,
    onAction: (() -> Unit)? = null,
) {
    Column(
        modifier = modifier.fillMaxSize(),
        verticalArrangement = Arrangement.spacedBy(Dimens.spacingSmall, Alignment.CenterVertically),
        horizontalAlignment = Alignment.CenterHorizontally,
    ) {
        Icon(
            Icons.Outlined.Warning,
            contentDescription = null,
            modifier = Modifier.size(Dimens.illustrationSize),
            tint = MaterialTheme.colorScheme.error,
        )
        Text(text = message, style = MaterialTheme.typography.bodyMedium, textAlign = TextAlign.Center)
        if (actionLabel != null && onAction != null) {
            AppButton(text = actionLabel, onClick = onAction, style = AppButtonStyle.Secondary)
        }
    }
}

/** Pass as `empty = EmptyState(...).takeIf { nothingToShow }` — null means "not empty". */
data class EmptyState(
    val message: String? = null,
    val title: String? = null,
    val action: (@Composable () -> Unit)? = null,
)

@Composable
fun EmptyView(
    modifier: Modifier = Modifier,
    title: String? = null,
    message: String? = null,
    illustration: ImageVector = Icons.Outlined.Info,
    illustrationSize: Dp = Dimens.illustrationSize,
    action: (@Composable () -> Unit)? = null,
) {
    Column(
        modifier = modifier.fillMaxSize(),
        verticalArrangement = Arrangement.spacedBy(Dimens.spacingSmall, Alignment.CenterVertically),
        horizontalAlignment = Alignment.CenterHorizontally,
    ) {
        Icon(illustration, contentDescription = null, modifier = Modifier.size(illustrationSize), tint = MaterialTheme.colorScheme.onSurfaceVariant)
        title?.let { Text(text = it, style = MaterialTheme.typography.titleSmall, textAlign = TextAlign.Center) }
        message?.let { Text(text = it, style = MaterialTheme.typography.bodySmall, textAlign = TextAlign.Center) }
        action?.invoke()
    }
}
```

## AppOutlinedTextField.kt

```kotlin
@Composable
fun AppOutlinedTextField(
    value: String,
    onValueChange: (String) -> Unit,
    label: String,
    modifier: Modifier = Modifier,
    error: String? = null,
    maxLength: Int? = null,
    singleLine: Boolean = true,
    enabled: Boolean = true,
    isPassword: Boolean = false,
    keyboardOptions: KeyboardOptions = KeyboardOptions.Default,
    keyboardActions: KeyboardActions = KeyboardActions.Default,
    leadingIcon: ImageVector? = null,
) {
    var passwordVisible by rememberSaveable { mutableStateOf(false) }
    OutlinedTextField(
        value = value,
        onValueChange = { if (maxLength == null || it.length <= maxLength) onValueChange(it) },
        modifier = modifier.fillMaxWidth(),
        label = { Text(text = label) },
        isError = error != null,
        supportingText = error?.let { { Text(text = it) } },
        singleLine = singleLine,
        enabled = enabled,
        textStyle = MaterialTheme.typography.bodyLarge,
        leadingIcon = leadingIcon?.let { icon -> { Icon(icon, contentDescription = null) } },
        visualTransformation = if (isPassword && !passwordVisible) PasswordVisualTransformation() else VisualTransformation.None,
        trailingIcon = if (isPassword) {
            {
                TextButton(onClick = { passwordVisible = !passwordVisible }) {
                    Text(text = stringResource(if (passwordVisible) R.string.txt_hide else R.string.txt_show))
                }
            }
        } else {
            null
        },
        keyboardOptions = if (isPassword) keyboardOptions.copy(keyboardType = KeyboardType.Password) else keyboardOptions,
        keyboardActions = keyboardActions,
    )
}

/** User-facing text for a field error — called at render time: `error = state.emailError?.toMessage()`. */
@Composable
fun ValidationError.toMessage(): String = when (this) {
    ValidationError.Required -> stringResource(R.string.error_field_required)
    ValidationError.InvalidEmail -> stringResource(R.string.error_invalid_email)
    ValidationError.InvalidPhone -> stringResource(R.string.error_invalid_phone)
    is ValidationError.TooShort -> stringResource(R.string.error_too_short, min)
    is ValidationError.TooLong -> stringResource(R.string.error_too_long, max)
}
```

## SearchTextField.kt

```kotlin
@Composable
fun SearchTextField(
    query: String,
    onQueryChange: (String) -> Unit,
    modifier: Modifier = Modifier,
    placeholder: String = stringResource(R.string.txt_search),
) {
    OutlinedTextField(
        value = query,
        onValueChange = onQueryChange,
        modifier = modifier.fillMaxWidth(),
        singleLine = true,
        textStyle = MaterialTheme.typography.bodyLarge,
        shape = MaterialTheme.shapes.extraLarge,
        placeholder = { Text(text = placeholder) },
        leadingIcon = { Icon(Icons.Filled.Search, contentDescription = null) },
        trailingIcon = if (query.isNotEmpty()) {
            { IconButton(onClick = { onQueryChange("") }) { Icon(Icons.Filled.Clear, contentDescription = stringResource(R.string.cd_clear_search)) } }
        } else {
            null
        },
        keyboardOptions = KeyboardOptions(imeAction = ImeAction.Search),
    )
}
```
- Server search: the ViewModel debounces the query (300ms) before it calls the API.
- Search in a list that is already in memory: filter in the ViewModel, or use `SearchableBottomSheet`.

## AppBottomSheet.kt

```kotlin
@OptIn(ExperimentalMaterial3Api::class)
@Composable
fun AppBottomSheet(
    onDismiss: () -> Unit,
    modifier: Modifier = Modifier,
    title: String? = null,
    skipPartiallyExpanded: Boolean = true,
    content: @Composable ColumnScope.() -> Unit,
) {
    val sheetState = rememberModalBottomSheetState(skipPartiallyExpanded = skipPartiallyExpanded)
    ModalBottomSheet(onDismissRequest = onDismiss, sheetState = sheetState, modifier = modifier) {
        Column(
            modifier = Modifier
                .fillMaxWidth()
                .navigationBarsPadding()
                .imePadding()
                .padding(start = Dimens.spacingMedium, end = Dimens.spacingMedium, bottom = Dimens.spacingMedium),
            verticalArrangement = Arrangement.spacedBy(Dimens.spacingSmall),
        ) {
            title?.let { Text(text = it, style = MaterialTheme.typography.titleMedium) }
            content()
        }
    }
}
```

## SearchableBottomSheet.kt

```kotlin
@Composable
fun <T> SearchableBottomSheet(
    title: String,
    items: List<T>,
    itemLabel: (T) -> String,
    onItemSelected: (T) -> Unit,
    onDismiss: () -> Unit,
    modifier: Modifier = Modifier,
    itemKey: ((T) -> Any)? = null,
) {
    var query by rememberSaveable { mutableStateOf("") }
    // Display-only filtering of a list the screen already holds — not business logic.
    val visibleItems = remember(items, query) {
        if (query.isBlank()) items else items.filter { itemLabel(it).contains(query.trim(), ignoreCase = true) }
    }
    AppBottomSheet(onDismiss = onDismiss, title = title, modifier = modifier) {
        SearchTextField(query = query, onQueryChange = { query = it })
        if (visibleItems.isEmpty()) {
            Text(
                text = stringResource(R.string.txt_no_results),
                style = MaterialTheme.typography.bodyMedium,
                modifier = Modifier.padding(vertical = Dimens.spacingLarge),
            )
        } else {
            LazyColumn {
                items(visibleItems, key = itemKey) { item ->
                    Text(
                        text = itemLabel(item),
                        style = MaterialTheme.typography.bodyLarge,
                        modifier = Modifier
                            .fillMaxWidth()
                            .clickable { onItemSelected(item) }
                            .padding(vertical = Dimens.spacingMedium),
                    )
                }
            }
        }
    }
}
```

## Shimmer.kt

```kotlin
private const val SHIMMER_MIN_ALPHA = 0.04f
private const val SHIMMER_MAX_ALPHA = 0.12f
private const val SHIMMER_DURATION_MS = 900

@Composable
fun ShimmerBox(modifier: Modifier = Modifier, shape: Shape = MaterialTheme.shapes.small) {
    val transition = rememberInfiniteTransition(label = "shimmer")
    val alpha by transition.animateFloat(
        initialValue = SHIMMER_MIN_ALPHA,
        targetValue = SHIMMER_MAX_ALPHA,
        animationSpec = infiniteRepeatable(tween(SHIMMER_DURATION_MS), RepeatMode.Reverse),
        label = "shimmerAlpha",
    )
    Box(modifier = modifier.clip(shape).background(MaterialTheme.colorScheme.onSurface.copy(alpha = alpha)))
}

/** Generic first-load placeholder for list screens; a feature with a distinctive card builds its own from ShimmerBox. */
@Composable
fun ListSkeleton(modifier: Modifier = Modifier, rows: Int = 6) {
    Column(modifier = modifier.fillMaxWidth(), verticalArrangement = Arrangement.spacedBy(Dimens.spacingMedium)) {
        repeat(rows) {
            Row(verticalAlignment = Alignment.CenterVertically, horizontalArrangement = Arrangement.spacedBy(Dimens.spacingMedium)) {
                ShimmerBox(modifier = Modifier.size(Dimens.avatarMedium), shape = CircleShape)
                Column(modifier = Modifier.weight(1f), verticalArrangement = Arrangement.spacedBy(Dimens.spacingSmall)) {
                    ShimmerBox(modifier = Modifier.fillMaxWidth(0.7f).height(Dimens.spacingMedium))
                    ShimmerBox(modifier = Modifier.fillMaxWidth(0.4f).height(Dimens.spacingMedium))
                }
            }
        }
    }
}
```

## AppConfirmDialog.kt

```kotlin
@Composable
fun AppConfirmDialog(
    title: String,
    message: String,
    onConfirm: () -> Unit,
    onDismiss: () -> Unit,
    confirmLabel: String = stringResource(R.string.txt_confirm),
    dismissLabel: String = stringResource(R.string.txt_cancel),
    isDestructive: Boolean = false,
) {
    AlertDialog(
        onDismissRequest = onDismiss,
        title = { Text(text = title, style = MaterialTheme.typography.titleMedium) },
        text = { Text(text = message, style = MaterialTheme.typography.bodyMedium) },
        confirmButton = {
            TextButton(
                onClick = onConfirm,
                colors = if (isDestructive) {
                    ButtonDefaults.textButtonColors(contentColor = MaterialTheme.colorScheme.error)
                } else {
                    ButtonDefaults.textButtonColors()
                },
            ) { Text(text = confirmLabel) }
        },
        dismissButton = { TextButton(onClick = onDismiss) { Text(text = dismissLabel) } },
    )
}
```
Keep "is this dialog or sheet showing?" in a State flag (`showDeleteDialog: Boolean`). Change it through an Intent / ViewModel function.
Never use an Effect for it. An Effect is lost on rotation, so the dialog would disappear.

## Screen usage — the shape every feature screen follows

```kotlin
AppScreen(
    topBar = { AppTopBar(title = stringResource(R.string.title_orders), onBack = onBack) },
    loading = LoadingState(isLoading = state.isLoading),
    snackbarMessage = state.error.toErrorMessage(onRetry = onRetry, onDismiss = onErrorDismissed),
    // "Nothing to show" — also after a failed first load, where AppScreen then shows ErrorView + Retry instead.
    empty = EmptyState(message = stringResource(R.string.txt_no_orders))
        .takeIf { !state.isLoading && state.orders.isEmpty() },
    // Server-data screen → pull-to-refresh. A form/auth/picker/static screen writes `refresh = null`.
    refresh = RefreshState(isRefreshing = state.isRefreshing, onRefresh = onRefresh),
) {
    OrderList(orders = state.orders, onClick = onOrderClick)
}
```
`onRetry` / `onErrorDismissed` / `onRefresh` call the ViewModel in the way the architecture uses:
- MVI: `processIntent(...)`.
- MVVM: a public function.

In MVI + Clean, the State holds `status: UiStatus(isLoading, failure)`. So use `state.status.isLoading` and `state.status.toErrorMessage(...)`.

## Error → snackbar converter (one per architecture, in `AppSnackbar.kt`)

**MVI + Clean**: converts `UiStatus` / `Failure`. These come from `core/error/`, which stays free of UI code and string resources.
```kotlin
@Composable
fun UiStatus.toErrorMessage(onRetry: (() -> Unit)? = null, onDismiss: () -> Unit = {}): AppSnackbar? =
    failure.toErrorMessage(onRetry = onRetry, onDismiss = onDismiss)

@Composable
fun Failure?.toErrorMessage(onRetry: (() -> Unit)? = null, onDismiss: () -> Unit = {}): AppSnackbar? =
    errorSnackbarMessage(text = this?.toMessage(), details = this?.toDebugDetails(), onRetry = onRetry, onDismiss = onDismiss)

/** User-facing one-liner — string resources, so it is translated like every other text. */
@Composable
fun Failure.toMessage(): String = when (this) {
    is Failure.NoInternet -> stringResource(R.string.error_no_internet)
    is Failure.Timeout -> stringResource(R.string.error_timeout)
    is Failure.Unauthorized -> message ?: stringResource(R.string.error_unauthorized)
    is Failure.ServerError -> message ?: stringResource(R.string.error_server)
    is Failure.Unknown -> stringResource(R.string.error_generic)
}
```

**MVI, MVVM, MVVM + Clean**: converts `NetworkResult.Error`.
```kotlin
@Composable
fun NetworkResult.Error?.toErrorMessage(onRetry: (() -> Unit)? = null, onDismiss: () -> Unit = {}): AppSnackbar? {
    if (this == null) return null
    val text = message ?: stringResource(
        when {
            cause is SocketTimeoutException -> R.string.error_timeout // before IOException — it is one
            cause is IOException -> R.string.error_no_internet
            code == 401 -> R.string.error_unauthorized
            code != null -> R.string.error_server
            else -> R.string.error_generic
        },
    )
    val details = cause?.stackTraceToString() ?: code?.let { "HTTP $it" }
    return errorSnackbarMessage(text = text, details = details, onRetry = onRetry, onDismiss = onDismiss)
}
```
