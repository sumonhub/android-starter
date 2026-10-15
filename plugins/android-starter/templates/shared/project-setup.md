# Shared Project Setup (all architectures)

> Applies to every project, whatever architecture pattern was chosen. `/android-starter:init-project` applies every section below to the
> new project. It also runs the architecture template's own Gradle step. Use the latest stable version of each
> library/plugin named here, and add it to `gradle/libs.versions.toml`. Never hardcode a version in a build file.

## 1. Environments (build types)

There are three build types, each with its own base URL. This is the only place where a URL is defined:

```kotlin
android {
    buildTypes {
        debug {
            applicationIdSuffix = ".debug"
            buildConfigField("String", "BASE_URL", "\"https://api-dev.example.com/\"") // TODO: real dev URL
        }
        create("staging") {
            initWith(getByName("debug"))
            applicationIdSuffix = ".staging"
            matchingFallbacks += "debug"
            buildConfigField("String", "BASE_URL", "\"https://api-staging.example.com/\"") // TODO: real staging URL
        }
        release {
            buildConfigField("String", "BASE_URL", "\"https://api.example.com/\"") // TODO: real production URL
            signingConfig = signingConfigs.findByName("release") // null until keystore.properties exists — see 2.
        }
    }
}
```
- `core/network/ApiConfig.kt` reads `BuildConfig.BASE_URL`. It never declares a URL of its own
- This needs `buildFeatures { buildConfig = true }` (the architecture template's Gradle step already adds it)
- Base URLs are not secrets, so they live in the build file. Anything secret goes through 2.

## 2. Release signing — never committed

```kotlin
val keystoreProperties = Properties().apply {
    val file = rootProject.file("keystore.properties")
    if (file.exists()) file.inputStream().use(::load)
}

android {
    signingConfigs {
        if (keystoreProperties.isNotEmpty()) {
            create("release") {
                storeFile = rootProject.file(keystoreProperties.getProperty("storeFile"))
                storePassword = keystoreProperties.getProperty("storePassword")
                keyAlias = keystoreProperties.getProperty("keyAlias")
                keyPassword = keystoreProperties.getProperty("keyPassword")
            }
        }
    }
}
```
- Add to `.gitignore`: `keystore.properties`, `*.jks`, `*.keystore`
- Commit a `keystore.properties.example` with the four keys and empty values. This shows a new developer what to fill in
- CI writes secrets to `keystore.properties` at build time and signs with them. CI never signs from a committed file

## 3. R8 keep rules (release builds)

Gson reads DTO fields by reflection, so R8 must not rename or remove them. Add these rules to the app module's keep-rules file. It is `app/src/main/keepRules/rules.keep` on AGP 9+, and `proguard-rules.pro` on older AGP:
```
# Gson DTOs and the API envelope — fields are read by reflection
-keep class **.data.remote.dto.** { *; }
-keep class **.core.network.ApiResponse { *; }
```
- This works only because every DTO lives in a `data/remote/dto/` package. That location is a rule, not a suggestion
- Every DTO field also has `@SerializedName("json_name")`, so renaming a field never silently breaks parsing
- Before the first release, run `./gradlew assembleRelease`. Then smoke-test one API call on the release build

## 4. Splash screen

- Dependency: `androidx.core:core-splashscreen`
- Theme (`res/values/themes.xml`): add `Theme.App.Starting` with:
  - `parent="Theme.SplashScreen"`
  - `windowSplashScreenBackground`
  - `windowSplashScreenAnimatedIcon` (the launcher foreground)
  - `postSplashScreenTheme` pointing at the app theme

  Set it as the activity's theme in the manifest
- `MainActivity.onCreate`: call `installSplashScreen()` **before** `super.onCreate()`. The start destination may depend on async data (e.g. the saved session). In that case, keep the splash with `setKeepOnScreenCondition { !isReady }` until the data is known. Never use a fake delay
- No separate Compose splash route

## 5. Unit tests

- Dependencies (`testImplementation`): `junit`, `kotlinx-coroutines-test`, `app.cash.turbine:turbine`
- **Fakes, not mocks** — no mocking library. A fake is a small hand-written class that implements the real interface
- `app/src/test/java/<package>/core/testing/MainDispatcherRule.kt` replaces `Dispatchers.Main` with a test dispatcher:
```kotlin
class MainDispatcherRule(
    val dispatcher: TestDispatcher = UnconfinedTestDispatcher(),
) : TestWatcher() {
    override fun starting(description: Description) = Dispatchers.setMain(dispatcher)
    override fun finished(description: Description) = Dispatchers.resetMain()
}
```
- The test file location mirrors the source file: `app/src/test/java/<package>/feature/<name>/presentation/<Feature>ViewModelTest.kt`
- Test shape:
  - Use `runTest`
  - Wherever production code takes a named dispatcher, inject `mainDispatcherRule.dispatcher`
  - Assert State/Effect with Turbine:
```kotlin
@get:Rule val mainDispatcherRule = MainDispatcherRule()

@Test
fun `load shows the list on success`() = runTest {
    val viewModel = OrderListViewModel(/* real or fake dependencies, built on fakes */)
    viewModel.state.test {
        // assert the loading state, then the loaded state
        cancelAndIgnoreRemainingEvents()
    }
}
```
- Test names describe the behaviour, in backticks: `` `load shows error when the network fails` ``

## 6. ktlint

- Plugin: `org.jlleitschuh.gradle.ktlint`. In the root `build.gradle.kts`, use `apply false`. In the app module, apply it
- `.editorconfig` at the repo root:
```
root = true

[*.{kt,kts}]
indent_size = 4
max_line_length = 140
ktlint_code_style = android_studio
ktlint_function_naming_ignore_when_annotated_with = Composable
```
- `./gradlew ktlintCheck` must pass before a commit. `./gradlew ktlintFormat` fixes most issues

## 7. CI (GitHub Actions)

`.github/workflows/android.yml`:
```yaml
name: Android CI

on:
  push:
    branches: [main, develop]
  pull_request:

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-java@v4
        with:
          distribution: temurin
          java-version: 21
      - uses: gradle/actions/setup-gradle@v4
      - name: Make Gradle wrapper executable
        run: chmod +x gradlew
      - name: Lint, test and build
        run: ./gradlew ktlintCheck lintDebug testDebugUnitTest assembleDebug
```
- A failed (red) CI run blocks merging. Fix it. Never skip it
