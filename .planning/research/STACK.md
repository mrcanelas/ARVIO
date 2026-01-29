# Stack Research: Android TV Security & Quality

**Project:** ARVIO Android TV Streaming App
**Researched:** 2026-01-29
**Focus:** Production-ready security, testing, and code quality stack

---

## Executive Summary

The 2025/2026 Android security and testing ecosystem has undergone significant shifts:

1. **androidx.security.crypto is DEPRECATED** - Direct Android Keystore + DataStore + Tink is now the standard
2. **R8 is mandatory** - ProGuard is legacy; use `proguard-android-optimize.txt` for 30% better performance
3. **Detekt wraps ktlint** - Single tool for both static analysis and formatting
4. **Compose testing is semantic-first** - Focus on user interactions, not implementation details

**Overall Confidence:** HIGH (verified via official Android docs, Google blog posts, and Context7)

---

## Secrets Management

### Current Situation: CRITICAL SECURITY ISSUE
ARVIO currently has API keys hardcoded in `Constants.kt` - this is extractable via APK decompilation.

### Recommended Stack

| Technology | Version | Purpose | Confidence |
|------------|---------|---------|------------|
| **Google Secrets Gradle Plugin** | 2.0.1 | Keep secrets out of version control | HIGH |
| **Android Keystore** | Platform API | Hardware-backed key storage | HIGH |
| **Jetpack DataStore Preferences** | 1.2.0 | Encrypted settings storage (replaces SharedPreferences) | HIGH |
| **Tink Crypto Library** | 1.18.0+ | Modern encryption (Google Security maintained) | MEDIUM |

### Architecture: Multi-Layered Defense

```
Layer 1: Build-Time (Secrets Gradle Plugin)
├─ API keys in local.properties (gitignored)
├─ Exposed to BuildConfig at build time
└─ Still extractable from APK → Need Layer 2

Layer 2: Runtime Protection (Android Keystore)
├─ Generate encryption keys in hardware-backed Keystore
├─ Keys never leave secure enclave
└─ Use to encrypt sensitive data at rest

Layer 3: Network Security
├─ Move high-value secrets to backend
├─ Use short-lived tokens from server
└─ Rotate credentials regularly
```

### Implementation Priority

**PHASE 1: Immediate (Block APK decompilation)**
1. **Secrets Gradle Plugin** - Move all API keys from `Constants.kt` to `local.properties`
   ```gradle
   // In build.gradle.kts
   plugins {
       id("com.google.android.libraries.mapsplatform.secrets-gradle-plugin") version "2.0.1"
   }
   ```

   ```properties
   # In local.properties (gitignored)
   SUPABASE_URL=your_supabase_url
   SUPABASE_ANON_KEY=your_anon_key
   TRAKT_CLIENT_ID=your_trakt_id
   TMDB_API_KEY=your_tmdb_key
   ```

   **Limitation:** Keys still in APK binary, but not in source control.

**PHASE 2: Production Hardening (Encrypt secrets at rest)**
2. **Android Keystore + DataStore** - Encrypt tokens/session data
   ```kotlin
   // Generate key in Keystore
   val keyGenParameterSpec = KeyGenParameterSpec.Builder(
       "arvio_master_key",
       KeyProperties.PURPOSE_ENCRYPT or KeyProperties.PURPOSE_DECRYPT
   ).apply {
       setBlockModes(KeyProperties.BLOCK_MODE_GCM)
       setEncryptionPaddings(KeyProperties.ENCRYPTION_PADDING_NONE)
       setKeySize(256)
       setUserAuthenticationRequired(false) // True for biometric-gated secrets
   }.build()

   val keyGenerator = KeyGenerator.getInstance(
       KeyProperties.KEY_ALGORITHM_AES,
       "AndroidKeyStore"
   )
   keyGenerator.init(keyGenParameterSpec)
   keyGenerator.generateKey()
   ```

3. **Jetpack DataStore** - Store encrypted preferences (replaces SharedPreferences)
   ```gradle
   implementation("androidx.datastore:datastore-preferences:1.2.0")
   ```

**PHASE 3: Best Practice (Server-side secrets)**
4. **Backend API Gateway** - Move Trakt/TMDB keys to backend proxy
   - Client requests content → Backend queries APIs → Response to client
   - Client never holds API keys
   - Implement rate limiting server-side

### Why NOT EncryptedSharedPreferences?

**Status:** DEPRECATED as of June 2025 (androidx.security.crypto:1.1.0-beta01)

**Reasons for deprecation:**
- Strict mode violations (synchronous I/O on main thread)
- Keyset corruption on certain OEM devices
- Not coroutine-friendly
- Maintenance ended by Google

**Migration:** Use DataStore + Tink + Android Keystore directly.

### Alternatives Considered

| Approach | Pros | Cons | Verdict |
|----------|------|------|---------|
| **Hardcoded secrets** | Easy | CRITICAL security vulnerability | Never use |
| **NDK obfuscation** (Hidden Secrets Plugin) | Higher reverse-engineering cost | Still extractable; adds complexity | Not worth it for streaming app |
| **Backend proxy for all APIs** | Most secure | Requires backend infrastructure | Future consideration for v2.0 |
| **Secrets Gradle Plugin** | Industry standard; hides from git | Keys still in APK | Use as baseline |

### Key References
- [Android Keystore Official Docs](https://developer.android.com/privacy-and-security/keystore) (HIGH confidence)
- [Secrets Gradle Plugin](https://github.com/google/secrets-gradle-plugin) (HIGH confidence)
- [DataStore Documentation](https://developer.android.com/topic/libraries/architecture/datastore) (HIGH confidence)
- [Goodbye EncryptedSharedPreferences: 2026 Migration Guide](https://proandroiddev.com/goodbye-encryptedsharedpreferences-a-2026-migration-guide-4b819b4a537a) (MEDIUM confidence)
- [Securing Secrets in Android](https://medium.com/@vaibhav.shakya786/securing-secrets-in-android-from-api-keys-to-production-grade-defense-a2c8dc46948f) (MEDIUM confidence)

---

## Testing Frameworks

### Current Situation
ARVIO has only one utility test. No UI tests, integration tests, or ViewModels tests.

### Recommended Testing Stack

| Framework | Version | Purpose | Test Type | Confidence |
|-----------|---------|---------|-----------|------------|
| **JUnit 4** | 4.13.2 | Test runner foundation | Unit | HIGH |
| **MockK** | 1.13.8 | Kotlin-native mocking | Unit | HIGH |
| **Turbine** | 1.2.1 | Flow/StateFlow testing | Unit | HIGH |
| **kotlinx-coroutines-test** | 1.9.0 | Coroutine testing utilities | Unit | HIGH |
| **Compose UI Test JUnit4** | 1.7.6 | Declarative Compose UI testing | UI (instrumented) | HIGH |
| **Espresso Core** | 3.6.1 | UI testing (for XML views if any) | UI (instrumented) | HIGH |
| **Robolectric** | 4.16 | JVM-based Android tests (10x faster) | Unit (simulated Android) | HIGH |

### Testing Pyramid for ARVIO

```
         /\
        /  \  E2E Tests (Manual QA on Android TV device)
       /----\
      / UI   \  Compose UI Tests (20% of tests)
     /--------\
    / Integration \  ViewModel + Repository Tests (30%)
   /--------------\
  /   Unit Tests   \  Pure Kotlin logic tests (50%)
 /------------------\
```

### Unit Testing Setup

**Dependencies:**
```gradle
dependencies {
    // Unit testing
    testImplementation("junit:junit:4.13.2")
    testImplementation("io.mockk:mockk:1.13.8")
    testImplementation("app.cash.turbine:turbine:1.2.1")
    testImplementation("org.jetbrains.kotlinx:kotlinx-coroutines-test:1.9.0")

    // Robolectric for Android framework testing on JVM
    testImplementation("org.robolectric:robolectric:4.16")
}
```

**What to test:**
- ViewModels (state changes, error handling)
- Repositories (data source coordination)
- Use cases (business logic)
- Utility functions (existing `StringExtensionsTest` expanded)

**Example ViewModel Test with MockK + Turbine:**
```kotlin
@Test
fun `fetchMovies success updates state`() = runTest {
    // Arrange
    val mockRepo = mockk<MovieRepository>()
    coEvery { mockRepo.getPopularMovies() } returns flowOf(
        Result.success(listOf(movie1, movie2))
    )
    val viewModel = MoviesViewModel(mockRepo)

    // Act & Assert
    viewModel.uiState.test {
        assertEquals(UiState.Loading, awaitItem())
        assertEquals(UiState.Success(listOf(movie1, movie2)), awaitItem())
        cancelAndIgnoreRemainingEvents()
    }
}
```

### UI Testing Setup (Jetpack Compose)

**Dependencies:**
```gradle
dependencies {
    // Compose UI testing
    androidTestImplementation("androidx.compose.ui:ui-test-junit4:1.7.6")
    debugImplementation("androidx.compose.ui:ui-test-manifest:1.7.6")
}
```

**Best Practices:**
1. **Semantic testing** - Test what users see/do, not implementation
2. **Use test tags** - `Modifier.testTag("movieCard")` for stable node lookup
3. **Disable animations** - Use `composeTestRule.mainClock` for deterministic tests
4. **Fail fast** - Call `assertExists()` before actions

**Example Compose Test:**
```kotlin
@Test
fun movieDetailsScreen_showsCorrectTitle() {
    composeTestRule.setContent {
        MovieDetailsScreen(movie = testMovie)
    }

    composeTestRule
        .onNodeWithTag("movieTitle")
        .assertIsDisplayed()
        .assertTextEquals("Inception")
}
```

### Android TV-Specific Testing Considerations

**D-Pad Navigation Testing:**
```kotlin
@Test
fun homeScreen_dpadNavigationWorks() {
    composeTestRule
        .onNodeWithTag("movieList")
        .performKeyInput { pressKey(Key.DirectionDown) }

    composeTestRule
        .onNodeWithTag("selectedMovie")
        .assertIsFocused()
}
```

**Leanback Components (if using XML views):**
```gradle
androidTestImplementation("androidx.leanback:leanback:1.2.0-alpha04")
androidTestImplementation("androidx.test.espresso:espresso-core:3.6.1")
```

### CI/CD Integration

**Recommended GitHub Actions workflow:**
```yaml
- name: Run Unit Tests
  run: ./gradlew test

- name: Run Instrumented Tests
  uses: reactivecircus/android-emulator-runner@v2
  with:
    api-level: 29
    target: android-tv
    script: ./gradlew connectedAndroidTest
```

### Testing Coverage Tools

```gradle
android {
    buildTypes {
        debug {
            enableUnitTestCoverage = true
            enableAndroidTestCoverage = true
        }
    }
}
```

**Run coverage report:**
```bash
./gradlew jacocoTestReport
# Report: build/reports/jacoco/jacocoTestReport/html/index.html
```

### Key References
- [Android Testing Frameworks 2025](https://www.globalapptesting.com/blog/android-testing-frameworks) (MEDIUM confidence)
- [Jetpack Compose Testing Official Guide](https://developer.android.com/develop/ui/compose/testing) (HIGH confidence)
- [MockK Documentation](https://mockk.io/) (HIGH confidence)
- [Turbine GitHub](https://github.com/cashapp/turbine) (HIGH confidence)
- [Robolectric Official Docs](https://robolectric.org/) (HIGH confidence)

---

## Static Analysis & Code Quality

### Recommended Stack

| Tool | Version | Purpose | Confidence |
|------|---------|---------|------------|
| **Detekt** | 1.23.8 | Kotlin static analysis + ktlint wrapper | HIGH |
| **Android Lint** | Built-in AGP | Platform-specific checks | HIGH |
| **ktlint** (via Detekt) | 1.8.0 | Code formatting | HIGH |

### Why Detekt Over Standalone ktlint?

**Problem:** Standalone ktlint Gradle plugins are poorly maintained and don't support Android well.

**Solution:** Detekt's `detekt-formatting` module wraps ktlint and provides unified configuration.

**Benefits:**
- Single config file (`detekt.yml`) instead of `.editorconfig` + `detekt.yml`
- Active maintenance (latest release: Feb 2025)
- Comprehensive static analysis beyond formatting
- Built-in report generation (HTML, XML, SARIF, Markdown)

### Setup

**1. Add Detekt Plugin:**
```kotlin
// In build.gradle.kts (project level)
plugins {
    id("io.gitlab.arturbosch.detekt") version "1.23.8"
}

// In build.gradle.kts (app level)
plugins {
    id("io.gitlab.arturbosch.detekt")
}

dependencies {
    detektPlugins("io.gitlab.arturbosch.detekt:detekt-formatting:1.23.8")
}

detekt {
    toolVersion = "1.23.8"
    config.setFrom(file("$rootDir/config/detekt/detekt.yml"))
    buildUponDefaultConfig = true
    autoCorrect = true

    source.setFrom(
        "src/main/kotlin",
        "src/main/java"
    )
}

tasks.withType<Detekt>().configureEach {
    jvmTarget = "17" // Match your project's JVM target

    reports {
        html.required.set(true)
        xml.required.set(true)
        sarif.required.set(true) // For GitHub Code Scanning
        md.required.set(true)
    }
}
```

**2. Create Detekt Configuration (`config/detekt/detekt.yml`):**
```yaml
build:
  maxIssues: 0 # Fail build on any issue

formatting:
  active: true
  android: true # Use Android Studio code style
  autoCorrect: true

complexity:
  TooManyFunctions:
    thresholdInFiles: 15
    thresholdInClasses: 15
  LongMethod:
    threshold: 60

naming:
  FunctionNaming:
    functionPattern: '[a-z][a-zA-Z0-9]*'
  VariableNaming:
    variablePattern: '[a-z][a-zA-Z0-9]*'

style:
  MagicNumber:
    ignoreNumbers: ['-1', '0', '1', '2']
  MaxLineLength:
    maxLineLength: 120
```

**3. Configure Android Lint:**
```kotlin
// In build.gradle.kts (app level)
android {
    lint {
        // Fail build on errors
        abortOnError = true
        checkReleaseBuilds = true

        // Treat warnings as errors for critical checks
        warningsAsErrors = true

        // Disable noisy warnings
        disable += setOf(
            "ObsoleteLintCustomCheck",
            "GradleDependency"
        )

        // Enable additional checks
        checkAllWarnings = true

        // Baseline for existing issues
        baseline = file("lint-baseline.xml")

        // Reports
        htmlReport = true
        htmlOutput = file("$buildDir/reports/lint-results.html")
        xmlReport = true
        xmlOutput = file("$buildDir/reports/lint-results.xml")
        sarifReport = true // For GitHub Code Scanning
    }
}
```

### Running Static Analysis

**Local development:**
```bash
# Run Detekt with auto-fix
./gradlew detekt --auto-correct

# Run Android Lint
./gradlew lint

# Run both
./gradlew detekt lint
```

**CI/CD:**
```yaml
- name: Run Static Analysis
  run: |
    ./gradlew detekt
    ./gradlew lint

- name: Upload SARIF to GitHub
  uses: github/codeql-action/upload-sarif@v2
  with:
    sarif_file: app/build/reports/detekt/detekt.sarif
```

### Pre-commit Hook (Optional)

Create `.git/hooks/pre-commit`:
```bash
#!/bin/sh
echo "Running Detekt..."
./gradlew detekt --auto-correct

if [ $? -ne 0 ]; then
  echo "Detekt failed. Fix issues before committing."
  exit 1
fi
```

### Baseline Strategy

**Problem:** Inheriting legacy code with hundreds of warnings.

**Solution:** Create baseline to focus on new issues only.

```bash
# Generate baseline (one-time)
./gradlew detekt --baseline

# This creates detekt-baseline.xml with all existing issues
# Future runs only report NEW issues
```

### Key References
- [Detekt Official Docs](https://detekt.dev/) (HIGH confidence)
- [Detekt GitHub Releases](https://github.com/detekt/detekt/releases) (HIGH confidence)
- [Android Lint Configuration Guide](https://developer.android.com/studio/write/lint) (HIGH confidence)
- [Enforcing Code Quality with Detekt and Ktlint](https://medium.com/@mohamad.alemicode/enforcing-code-quality-in-android-with-detekt-and-ktlint-a-practical-guide-907b57d047ec) (MEDIUM confidence)

---

## Build Security (ProGuard/R8)

### Critical Finding: Use R8, Not ProGuard

**Status:** R8 is the default code shrinker since Android Gradle Plugin 3.4.0. ProGuard is legacy.

**Performance Impact:**
- Disney+ saw **30% faster startup** and **25% fewer ANRs** after enabling R8 optimizations
- Reddit achieved **40% faster startup** with full R8 optimization

### Recommended Configuration

**1. Enable R8 with Full Optimization:**

```kotlin
// In build.gradle.kts (app level)
android {
    buildTypes {
        release {
            isMinifyEnabled = true
            isShrinkResources = true

            proguardFiles(
                getDefaultProguardFile("proguard-android-optimize.txt"), // NOT proguard-android.txt
                "proguard-rules.pro"
            )
        }
    }
}
```

**CRITICAL:** Use `proguard-android-optimize.txt`, NOT `proguard-android.txt`.
- `proguard-android.txt` = No optimizations (outdated, removed in AGP 9.0)
- `proguard-android-optimize.txt` = Full R8 optimizations enabled

**2. ARVIO-Specific ProGuard Rules (`proguard-rules.pro`):**

```proguard
# ---------------------------
# ARVIO ProGuard Rules
# ---------------------------

# Retrofit (Networking)
-keepattributes Signature, InnerClasses, EnclosingMethod
-keepattributes RuntimeVisibleAnnotations, RuntimeVisibleParameterAnnotations
-keepattributes AnnotationDefault
-keepclassmembers,allowshrinking,allowobfuscation interface * {
    @retrofit2.http.* <methods>;
}
-dontwarn javax.annotation.**
-dontwarn org.codehaus.mojo.animal_sniffer.*

# OkHttp
-dontwarn okhttp3.**
-dontwarn okio.**
-dontwarn javax.annotation.**

# Gson (if using for JSON parsing)
-keepattributes Signature
-keepattributes *Annotation*
-keep class com.google.gson.** { *; }

# Keep data models (DO NOT obfuscate - used for JSON parsing)
-keep class com.arvio.data.model.** { *; }
-keep class com.arvio.domain.model.** { *; }

# Kotlin Serialization (if used instead of Gson)
-keepattributes *Annotation*, InnerClasses
-dontnote kotlinx.serialization.SerializationKt
-keep,includedescriptorclasses class com.arvio.**$$serializer { *; }
-keepclassmembers class com.arvio.** {
    *** Companion;
}
-keepclasseswithmembers class com.arvio.** {
    kotlinx.serialization.KSerializer serializer(...);
}

# Hilt (Dependency Injection)
-keepnames @dagger.hilt.android.lifecycle.HiltViewModel class * extends androidx.lifecycle.ViewModel

# ExoPlayer
-dontwarn com.google.android.exoplayer2.**
-keep class com.google.android.exoplayer2.** { *; }

# Compose (usually handled automatically)
-keep class androidx.compose.** { *; }
-keep class kotlin.Metadata { *; }

# Coroutines
-keepnames class kotlinx.coroutines.internal.MainDispatcherFactory {}
-keepnames class kotlinx.coroutines.CoroutineExceptionHandler {}
-keep class kotlinx.coroutines.** { *; }

# Supabase SDK (if obfuscation causes issues)
-keep class io.github.jan.supabase.** { *; }

# Keep BuildConfig
-keep class com.arvio.BuildConfig { *; }

# Keep R class
-keep class **.R$* { *; }

# ---------------------------
# Debugging Rules (REMOVE BEFORE RELEASE)
# ---------------------------
# Uncomment these ONLY when debugging R8 issues

# -dontoptimize
# -dontobfuscate
# -printconfiguration r8-config.txt
# -whyareyoukeeping class com.arvio.data.model.Movie
```

**3. Debugging R8 Issues:**

If the release build crashes or behaves incorrectly:

```proguard
# Add to proguard-rules.pro temporarily
-printconfiguration build/outputs/mapping/configuration.txt
-whyareyoukeeping class com.arvio.problematic.Class
```

Run build and check `build/outputs/mapping/configuration.txt` to see final rules.

**4. Mapping File Management:**

R8 generates `mapping.txt` to de-obfuscate stack traces.

```kotlin
// In build.gradle.kts
android {
    buildTypes {
        release {
            // Store mapping files for each release
            multiDexEnabled = true
            proguardFiles(...)
        }
    }
}
```

**Store mapping files:**
- **Local:** `app/build/outputs/mapping/release/mapping.txt`
- **Play Console:** Auto-uploaded if using Play App Signing
- **Version Control:** DO NOT commit `mapping.txt` (too large). Use CI/CD artifacts.

**De-obfuscate crashes:**
```bash
# Using retrace
retrace.sh mapping.txt stacktrace.txt
```

**Android Studio integration:** AGP 9.0+ auto-deobfuscates logcat for R8 builds.

### Common R8 Anti-Patterns (DO NOT DO THIS)

| Anti-Pattern | Why It's Bad | Impact |
|--------------|--------------|--------|
| `-dontoptimize` in release | Disables all optimizations | 30-40% performance loss |
| `-dontshrink` | Keeps unused code | APK size bloat |
| `-dontobfuscate` | Security risk | Easy reverse engineering |
| `-keep class * { *; }` | Defeats entire purpose of R8 | No benefits at all |

**Only use these flags temporarily for debugging, then remove.**

### APK Size Optimization

```kotlin
android {
    buildTypes {
        release {
            isShrinkResources = true // Remove unused resources
            isMinifyEnabled = true
        }
    }

    // Enable resource shrinking per language
    bundle {
        language {
            enableSplit = true
        }
        density {
            enableSplit = true
        }
        abi {
            enableSplit = true
        }
    }
}
```

**Analyze APK size:**
```bash
./gradlew assembleRelease
# Android Studio > Build > Analyze APK > app-release.aab
```

### App Signing Best Practices

**1. Use Play App Signing:**
- Google manages app signing key
- You keep upload key
- Upload key can be reset if compromised

**2. Keystore Security:**
```bash
# Generate keystore with strong security
keytool -genkey -v \
  -keystore arvio-release.keystore \
  -alias arvio-key \
  -keyalg RSA \
  -keysize 2048 \
  -validity 10000 \
  -storetype PKCS12
```

**3. Store Credentials Securely:**
```kotlin
// In gradle.properties (gitignored)
ARVIO_STORE_FILE=../keystores/arvio-release.keystore
ARVIO_STORE_PASSWORD=***
ARVIO_KEY_ALIAS=arvio-key
ARVIO_KEY_PASSWORD=***
```

```kotlin
// In build.gradle.kts
android {
    signingConfigs {
        create("release") {
            storeFile = file(project.property("ARVIO_STORE_FILE") as String)
            storePassword = project.property("ARVIO_STORE_PASSWORD") as String
            keyAlias = project.property("ARVIO_KEY_ALIAS") as String
            keyPassword = project.property("ARVIO_KEY_PASSWORD") as String
        }
    }

    buildTypes {
        release {
            signingConfig = signingConfigs.getByName("release")
        }
    }
}
```

**4. Backup Keystore:**
- Store in password manager (1Password, Bitwarden)
- Store in secure cloud (encrypted AWS S3, Azure Key Vault)
- **NEVER commit to Git**

**5. CI/CD Signing:**
```yaml
# GitHub Actions example
- name: Decode Keystore
  env:
    KEYSTORE_BASE64: ${{ secrets.KEYSTORE_BASE64 }}
  run: echo $KEYSTORE_BASE64 | base64 -d > arvio-release.keystore

- name: Build Release APK
  env:
    ARVIO_STORE_PASSWORD: ${{ secrets.STORE_PASSWORD }}
    ARVIO_KEY_PASSWORD: ${{ secrets.KEY_PASSWORD }}
  run: ./gradlew assembleRelease
```

### Key References
- [R8 Official Blog Post (Nov 2025)](https://android-developers.googleblog.com/2025/11/use-r8-to-shrink-optimize-and-fast.html) (HIGH confidence)
- [R8 Configuration Troubleshooting](https://android-developers.googleblog.com/2025/11/configure-and-troubleshoot-r8-keep-rules.html) (HIGH confidence)
- [Ultimate ProGuard/R8 Rules (2025 Edition)](https://medium.com/@lakshitagangola123/the-ultimate-proguard-r8-rules-for-modern-android-apps-2025-edition-aa78e0939193) (MEDIUM confidence)
- [Android App Signing Guide](https://developer.android.com/studio/publish/app-signing) (HIGH confidence)

---

## Alternatives Considered

| Category | Recommended | Alternative | Why Not Alternative |
|----------|-------------|-------------|---------------------|
| **Secrets** | Secrets Gradle Plugin + Keystore | NDK obfuscation (Hidden Secrets Plugin) | Complexity doesn't justify benefit for streaming app |
| **Secrets Storage** | DataStore + Tink | EncryptedSharedPreferences | Deprecated June 2025; strict mode violations |
| **Testing: Mocking** | MockK | Mockito | Mockito is Java-first; doesn't support Kotlin features well |
| **Testing: Flows** | Turbine | Manual Flow collection | Turbine handles high-frequency emissions better |
| **Testing: Android Components** | Robolectric | Emulator for all tests | Robolectric is 10x faster for unit tests |
| **Static Analysis** | Detekt + ktlint wrapper | Standalone ktlint plugin | Standalone plugins are unmaintained |
| **Code Shrinking** | R8 with optimize | ProGuard | ProGuard is legacy; R8 is 30-40% faster |

---

## Installation Guide

### 1. Add Dependencies to `build.gradle.kts` (app level)

```kotlin
plugins {
    id("com.android.application")
    kotlin("android")
    id("com.google.android.libraries.mapsplatform.secrets-gradle-plugin") version "2.0.1"
    id("io.gitlab.arturbosch.detekt") version "1.23.8"
    id("com.google.dagger.hilt.android")
    kotlin("kapt")
}

android {
    compileSdk = 35

    defaultConfig {
        minSdk = 21
        targetSdk = 35
        versionCode = 1
        versionName = "1.0.0"

        testInstrumentationRunner = "androidx.test.runner.AndroidJUnitRunner"
    }

    buildTypes {
        release {
            isMinifyEnabled = true
            isShrinkResources = true
            proguardFiles(
                getDefaultProguardFile("proguard-android-optimize.txt"),
                "proguard-rules.pro"
            )
            signingConfig = signingConfigs.getByName("release")
        }
    }

    lint {
        abortOnError = true
        checkReleaseBuilds = true
        htmlReport = true
        baseline = file("lint-baseline.xml")
    }

    buildFeatures {
        compose = true
        buildConfig = true
    }

    composeOptions {
        kotlinCompilerExtensionVersion = "1.5.8"
    }
}

dependencies {
    // Existing dependencies...
    // (Kotlin, Compose, Hilt, Retrofit, ExoPlayer, etc.)

    // Secrets Management
    implementation("androidx.datastore:datastore-preferences:1.2.0")
    implementation("com.google.crypto.tink:tink-android:1.18.0")

    // Testing: Unit
    testImplementation("junit:junit:4.13.2")
    testImplementation("io.mockk:mockk:1.13.8")
    testImplementation("app.cash.turbine:turbine:1.2.1")
    testImplementation("org.jetbrains.kotlinx:kotlinx-coroutines-test:1.9.0")
    testImplementation("org.robolectric:robolectric:4.16")

    // Testing: Instrumented (UI)
    androidTestImplementation("androidx.compose.ui:ui-test-junit4:1.7.6")
    debugImplementation("androidx.compose.ui:ui-test-manifest:1.7.6")
    androidTestImplementation("androidx.test.espresso:espresso-core:3.6.1")

    // Static Analysis
    detektPlugins("io.gitlab.arturbosch.detekt:detekt-formatting:1.23.8")
}

detekt {
    toolVersion = "1.23.8"
    config.setFrom(file("$rootDir/config/detekt/detekt.yml"))
    buildUponDefaultConfig = true
    autoCorrect = true
}
```

### 2. Create `.gitignore` Additions

```gitignore
# Secrets
local.properties
*.keystore
*.jks
gradle.properties

# Reports
build/reports/
lint-baseline.xml

# R8 mapping files (too large for git; store in CI artifacts)
mapping.txt
```

### 3. Setup Commands

```bash
# Initialize project structure
mkdir -p config/detekt
mkdir -p keystores

# Generate keystore (one-time)
keytool -genkey -v \
  -keystore keystores/arvio-release.keystore \
  -alias arvio-key \
  -keyalg RSA \
  -keysize 2048 \
  -validity 10000

# Move secrets to local.properties
echo "SUPABASE_URL=your_url" >> local.properties
echo "SUPABASE_ANON_KEY=your_key" >> local.properties
echo "TRAKT_CLIENT_ID=your_id" >> local.properties
echo "TMDB_API_KEY=your_key" >> local.properties

# Generate lint baseline
./gradlew lint --continue
cp app/build/reports/lint-baseline.xml app/

# Generate detekt baseline
./gradlew detekt --baseline

# Run all checks
./gradlew detekt lint test
```

---

## Confidence Assessment

| Area | Confidence | Reason |
|------|------------|--------|
| Secrets Management | HIGH | Official Android docs + Google blog posts; deprecation verified |
| Testing Frameworks | HIGH | Official Jetpack docs + verified library versions on Maven |
| Static Analysis | HIGH | Detekt official releases + Android Lint documentation |
| Build Security | HIGH | Google Android Developers blog (Nov 2025) + official AGP docs |

---

## Sources

### Official Android Documentation (HIGH Confidence)
- [Android Keystore System](https://developer.android.com/privacy-and-security/keystore)
- [Jetpack Security Deprecation](https://developer.android.com/jetpack/androidx/releases/security)
- [Jetpack DataStore](https://developer.android.com/topic/libraries/architecture/datastore)
- [Jetpack Compose Testing](https://developer.android.com/develop/ui/compose/testing)
- [Android Lint Configuration](https://developer.android.com/studio/write/lint)
- [Android App Signing](https://developer.android.com/studio/publish/app-signing)

### Google Official Blog Posts (HIGH Confidence)
- [Use R8 to Shrink, Optimize, and Fast-Track Your App (Nov 2025)](https://android-developers.googleblog.com/2025/11/use-r8-to-shrink-optimize-and-fast.html)
- [Configure and Troubleshoot R8 Keep Rules (Nov 2025)](https://android-developers.googleblog.com/2025/11/configure-and-troubleshoot-r8-keep-rules.html)

### Library Documentation (HIGH Confidence)
- [Google Secrets Gradle Plugin](https://github.com/google/secrets-gradle-plugin)
- [Detekt Official Docs](https://detekt.dev/)
- [Detekt GitHub Releases](https://github.com/detekt/detekt/releases)
- [MockK Documentation](https://mockk.io/)
- [Turbine GitHub](https://github.com/cashapp/turbine)
- [Robolectric Official Site](https://robolectric.org/)

### Community Resources (MEDIUM Confidence)
- [Goodbye EncryptedSharedPreferences: A 2026 Migration Guide](https://proandroiddev.com/goodbye-encryptedsharedpreferences-a-2026-migration-guide-4b819b4a537a)
- [Securing Secrets in Android: From API Keys to Production-Grade Defense](https://medium.com/@vaibhav.shakya786/securing-secrets-in-android-from-api-keys-to-production-grade-defense-a2c8dc46948f)
- [6 Android Testing Frameworks in 2025](https://www.globalapptesting.com/blog/android-testing-frameworks)
- [Enforcing Code Quality in Android with Detekt and Ktlint](https://medium.com/@mohamad.alemicode/enforcing-code-quality-in-android-with-detekt-and-ktlint-a-practical-guide-907b57d047ec)
- [Ultimate ProGuard/R8 Rules for Modern Android Apps (2025 Edition)](https://medium.com/@lakshitagangola123/the-ultimate-proguard-r8-rules-for-modern-android-apps-2025-edition-aa78e0939193)

---

## Next Steps for Roadmap Creation

Based on this research, the recommended phase structure for the security & quality milestone:

1. **Phase 1: Immediate Security (Secrets Management)** - Block 1-2 days
   - Add Secrets Gradle Plugin
   - Move API keys from Constants.kt to local.properties
   - Add .gitignore entries
   - Verify BuildConfig generation

2. **Phase 2: Testing Infrastructure** - Block 1 week
   - Add all testing dependencies
   - Write ViewModel unit tests with MockK + Turbine
   - Write repository tests
   - Write utility function tests
   - Setup test coverage reporting

3. **Phase 3: Static Analysis** - Block 2-3 days
   - Add Detekt with ktlint formatting
   - Configure Android Lint
   - Generate baselines for existing issues
   - Fix critical issues flagged
   - Setup pre-commit hooks

4. **Phase 4: Build Hardening (R8 + Signing)** - Block 2-3 days
   - Configure R8 with proguard-android-optimize.txt
   - Write ARVIO-specific ProGuard rules
   - Setup release signing config
   - Test release build thoroughly
   - Document mapping file storage strategy

5. **Phase 5: Production Secrets (Optional)** - Future consideration
   - Implement Android Keystore encryption
   - Migrate to DataStore for preferences
   - Setup Tink for encryption
   - (Defer backend proxy to v2.0)

**Total Estimated Time:** 2-3 weeks for Phases 1-4

**Research Confidence:** HIGH - All recommendations are production-ready and verified with official sources.
