# Pitfalls Research: Android App Security & Quality Improvements

**Domain:** Android security and quality retrofitting
**Researched:** 2026-01-29
**Confidence:** HIGH (Official Android docs + verified community sources)

## Executive Summary

Retrofitting security and quality improvements to existing Android apps presents unique challenges. The most dangerous pitfalls involve **accidentally shipping debug configurations to production** and **assuming BuildConfig obfuscation equals security**. Based on ARVIO's current antipatterns, this research focuses on four critical areas: secrets migration, error handling adoption, static analysis rollout, and test retrofitting.

---

## Critical Pitfalls

Mistakes that cause security breaches, rewrites, or major production issues.

### Pitfall 1: BuildConfig Secrets Are Not Actually Secret

**What goes wrong:** Developers migrate API keys from Constants.kt to BuildConfig and believe they're now "secure." Keys are still easily extractable via APK decompilation.

**Why it happens:** Misunderstanding what BuildConfig provides. BuildConfig is for **hiding secrets from version control**, not from reverse engineering.

**Consequences:**
- False sense of security
- API keys still fully recoverable via `apkanalyzer` or JADX
- Potential API abuse, billing fraud, data breaches
- Legal liabilities from exposed credentials

**Prevention:**
1. **Understand the threat model:** BuildConfig protects against GitHub scraping, not APK reverse engineering
2. **Add API restrictions:** Configure Google Cloud/Firebase keys to only work from your package name/SHA-256 fingerprint
3. **Use NDK obfuscation for high-value keys:** XOR obfuscation + native binaries make extraction harder (see `hidden-secrets-gradle-plugin`)
4. **Rotate keys regularly:** Set calendar reminders (e.g., quarterly)
5. **Backend proxy pattern:** For highest security, never ship keys - proxy sensitive API calls through your own backend

**Detection:**
- Run `apkanalyzer dex packages <your.apk>` and search for your API keys
- Check if keys work when extracted and used from curl/Postman

**Phase mapping:** Secrets Migration phase (Phase 1-2)

**Confidence:** HIGH - Verified with [official Android security guidance](https://developer.android.com/privacy-and-security/security-config), [ProAndroidDev article](https://proandroiddev.com/gradle-properties-buildconfig-and-secrets-management-the-right-way-8b1b161aaefd), and [Google Secrets Gradle Plugin docs](https://developers.google.com/maps/documentation/android-sdk/secrets-gradle-plugin)

### Pitfall 2: SSL Validation Disabled in Debug Leaking to Release

**What goes wrong:** Custom `X509TrustManager` that disables SSL validation for local testing accidentally remains active in production builds. ARVIO currently has "SSL validation disabled in debug builds."

**Why it happens:**
- StackOverflow solutions that bypass SSL without proper `BuildConfig.DEBUG` guards
- Forgetting to remove test code before release
- Using `if (BuildConfig.DEBUG)` in code instead of proper NetworkSecurityConfig

**Consequences:**
- **Man-in-the-Middle attacks possible** - attackers can intercept all HTTPS traffic
- **Google Play rejection** - Google scans for unsafe TrustManager implementations
- Complete loss of transport security
- PII, credentials, session tokens exposed to network attackers

**Prevention:**
1. **NEVER use custom X509TrustManager implementations** - Android's official guidance is explicit
2. **Use NetworkSecurityConfig.xml instead:**
   ```xml
   <network-security-config>
       <debug-overrides>
           <trust-anchors>
               <certificates src="@raw/debug_cas"/>
           </trust-anchors>
       </debug-overrides>
   </network-security-config>
   ```
3. **Debug-overrides only active when `android:debuggable="true"`** - app stores reject debuggable apps, so this is safe
4. **Automated detection:** Add lint checks for `X509TrustManager` implementations in your codebase

**Detection:**
- Search codebase for `X509TrustManager`, `checkServerTrusted`, `HostnameVerifier`
- Run `grep -r "trustAllCerts\|ALLOW_ALL_HOSTNAME_VERIFIER" .`
- Inspect decompiled release APK to verify no unsafe trust managers

**Phase mapping:** Secrets Migration phase (clean up alongside API key migration)

**Confidence:** HIGH - [Official Android docs](https://developer.android.com/privacy-and-security/risks/unsafe-trustmanager) state explicitly to use NetworkSecurityConfig

### Pitfall 3: Logging PII to Logcat in Production

**What goes wrong:** 399+ log statements include sensitive data (API responses, user IDs, tokens, etc.) that persist in production builds. Any app with `READ_LOGS` permission can access these.

**Why it happens:**
- Debug logging added during development and forgotten
- Developers unaware that logcat persists in production
- Assuming log statements are "internal only"
- Not using R8 rules to strip logs

**Consequences:**
- **CTS test failures** - Android's Compatibility Test Suite checks for PII in logs
- **Privacy violations** - Research shows 6% of top 100 Android apps leak PII to logs
- **Regulatory issues** - GDPR, CCPA violations
- **Google Play rejection** - Apps must not log PII during normal operation

**Prevention:**
1. **Strip logs with R8 in release builds** - Add to `proguard-rules.pro`:
   ```proguard
   -assumenosideeffects class android.util.Log {
       public static int v(...);
       public static int d(...);
       public static int i(...);
   }
   ```
2. **Sanitize sensitive objects** - Override `toString()` to return redacted values:
   ```kotlin
   data class Credential<T>(val data: String) {
       override fun toString() = "Credential XX"
   }
   ```
3. **Use data masking** - `XXXX-XXXX-XXXX-1313` instead of full credit card numbers
4. **Tokenization** - Log reference tokens instead of actual secrets
5. **Only log compile-time constants** - Avoid logging variables that might contain user data

**Detection:**
- Run `adb logcat` while using release build - should see no verbose/debug/info logs
- Search codebase for `Log.v`, `Log.d`, `Log.i` with variable parameters
- Use automated tooling to detect PII patterns in log statements

**Phase mapping:** Logging cleanup phase (likely Phase 2-3 after core security fixes)

**Confidence:** HIGH - [Official Android Log Info Disclosure docs](https://developer.android.com/privacy-and-security/risks/log-info-disclosure) with specific CTS test requirements

### Pitfall 4: JWT Expiration Check Returns False When Missing

**What goes wrong:** ARVIO's JWT validation returns `false` when `exp` claim is missing, **allowing expired or malformed tokens** to be accepted.

**Why it happens:**
- Defensive programming instinct to return `false` (seems "safe") instead of throwing
- Misunderstanding JWT security model - missing `exp` should be treated as invalid
- Confusion about what "false" means in context (false = valid vs false = invalid)

**Consequences:**
- **Security bypass** - tokens without expiration never expire
- **Unauthorized access** - revoked/expired tokens may still work
- Inability to invalidate compromised tokens

**Prevention:**
1. **Reject tokens missing required claims** - `exp` is required for security:
   ```kotlin
   fun isJWTValid(decodedJWT: DecodedJWT): Boolean {
       val expiresAt = decodedJWT.getExpiresAt()
           ?: return false  // Missing exp = invalid
       return expiresAt.after(Date())
   }
   ```
2. **Validate all critical claims:**
   - `iss` (issuer) - matches expected issuer
   - `aud` (audience) - intended for your app
   - `exp` (expiration) - not expired
   - `nbf` (not before) - valid time window
3. **Use short-lived tokens** - 15 minutes for access tokens, longer for refresh tokens
4. **Implement token blacklist** - Redis-backed invalidation for forced logout
5. **Validate signature** - always verify token hasn't been tampered with

**Detection:**
- Unit tests with tokens missing `exp` claim - should fail validation
- Review all JWT validation logic for missing claim handling
- Check if `DecodedJWT.getExpiresAt()` is null-checked properly

**Phase mapping:** Error handling phase (when adopting Result<T> pattern)

**Confidence:** MEDIUM - Multiple community sources agree on [JWT best practices](https://curity.io/resources/learn/jwt-best-practices/), [Baeldung JWT validation](https://www.baeldung.com/java-jwt-check-expiry-no-exception), though not Android-specific official docs

---

## Moderate Pitfalls

Mistakes that cause delays, technical debt, or require rework.

### Pitfall 5: Forgetting String Quoting in buildConfigField

**What goes wrong:** `buildConfigField("String", "API_KEY", apiKey)` generates invalid Kotlin code because the value needs nested quotes.

**Why it happens:** `buildConfigField` generates raw Kotlin code, so string values need quote characters as part of the value itself.

**Prevention:**
```gradle
// Wrong - generates: val API_KEY = abc123
buildConfigField("String", "API_KEY", apiKey)

// Correct - generates: val API_KEY = "abc123"
buildConfigField("String", "API_KEY", "\"${apiKey}\"")
```

**Detection:** Build fails with syntax error in generated BuildConfig.kt

**Phase mapping:** Secrets Migration phase

**Confidence:** HIGH - [Google Secrets Gradle Plugin docs](https://developers.google.com/maps/documentation/android-sdk/secrets-gradle-plugin)

### Pitfall 6: Missing CI/CD Secrets Configuration

**What goes wrong:** CI builds fail because `secrets.properties` is gitignored and not available on CI runners.

**Why it happens:** Local development uses `secrets.properties` file, but CI needs secrets injected differently.

**Prevention:**
1. **Use CI environment variables:** GitHub Actions secrets, GitLab CI/CD variables, etc.
2. **Read from environment in build.gradle:**
   ```gradle
   val tmdbApiKey = System.getenv("TMDB_API_KEY") ?:
                    project.findProperty("TMDB_API_KEY") as String?
   ```
3. **Provide fallback values:** Use `local.defaults.properties` with dummy keys for CI
4. **Document in README:** List all required secrets for CI configuration

**Detection:** CI pipeline fails with "property not found" errors

**Phase mapping:** Secrets Migration phase

**Confidence:** MEDIUM - [ProAndroidDev article](https://proandroiddev.com/gradle-properties-buildconfig-and-secrets-management-the-right-way-8b1b161aaefd)

### Pitfall 7: AGP 8.0+ BuildConfig Disabled by Default

**What goes wrong:** After upgrading Android Gradle Plugin to 8.0+, `BuildConfig` class is not generated, breaking compilation.

**Why it happens:** AGP 8.0 changed defaults - BuildConfig generation is now opt-in.

**Prevention:**
Add to `gradle.properties`:
```properties
android.defaults.buildfeatures.buildconfig=true
```

Or in module `build.gradle`:
```gradle
android {
    buildFeatures {
        buildConfig = true
    }
}
```

**Detection:** Compilation error "Unresolved reference: BuildConfig"

**Phase mapping:** Secrets Migration phase (pre-requisite check)

**Confidence:** MEDIUM - [Multiple Android dev sources](https://proandroiddev.com/gradle-properties-buildconfig-and-secrets-management-the-right-way-8b1b161aaefd)

### Pitfall 8: Silent Null Returns on API Failures (Error Handling)

**What goes wrong:** ARVIO currently has "silent null returns on API failures" - errors are swallowed without notification to user or logging.

**Why it happens:**
- Defensive programming gone wrong (returning null seems "safe")
- Lack of structured error handling pattern
- Fear of app crashes from uncaught exceptions

**Prevention:**
1. **Adopt Result<T> pattern with sealed classes:**
   ```kotlin
   sealed class Result<out T> {
       data class Success<T>(val data: T) : Result<T>()
       data class Error(val exception: Exception) : Result<Nothing>()
   }
   ```
2. **Use CallAdapter to wrap Retrofit responses:**
   ```kotlin
   interface ApiService {
       @GET("movies")
       suspend fun getMovies(): Result<List<Movie>>
   }
   ```
3. **Handle all error cases explicitly:**
   - HTTP errors (400, 500 series)
   - Network failures (no internet)
   - Parsing errors (malformed JSON)
   - Timeout errors
4. **Never catch and ignore exceptions** - at minimum, log them

**Detection:**
- Search for `catch` blocks with empty bodies or only `return null`
- Review API call sites - do they handle failure cases?
- Check if users see errors or just blank screens

**Phase mapping:** Error Handling phase

**Confidence:** MEDIUM - [Retrofit + Kotlin Result pattern](https://medium.com/canopas/retrofit-effective-error-handling-with-kotlin-coroutine-and-result-api-405217e9a73d)

### Pitfall 9: Global Error Mapper Across Multiple Clients

**What goes wrong:** Creating a single error converter used by all Retrofit clients (TMDB, Trakt, Supabase, Google OAuth) causes incorrect error interpretation.

**Why it happens:** DRY principle applied incorrectly - seems efficient to have one error handler.

**Prevention:**
1. **Each Retrofit client should map its own errors:**
   ```kotlin
   // TMDBClient has its own error types
   sealed class TMDBError : Error

   // TraktClient has different error types
   sealed class TraktError : Error
   ```
2. **Avoid global error converters** - each API has unique error codes/structures
3. **Client-specific error handling** - TMDB 429 (rate limit) ≠ OAuth 401 (unauthorized)

**Detection:**
- Check if single `ErrorConverter` is used across multiple API clients
- Review if error handling distinguishes between different API sources

**Phase mapping:** Error Handling phase

**Confidence:** MEDIUM - [Tim Malseed's Retrofit error handling](https://timusus.medium.com/network-error-handling-on-android-with-retrofit-kotlin-draft-6614f58fa58d)

### Pitfall 10: Massive Initial Detekt/Ktlint Violation Count

**What goes wrong:** Adding detekt to existing codebase reports 500-1000+ violations, causing paralysis about how to proceed.

**Why it happens:** Codebase wasn't written with linting rules in mind, so violations accumulated.

**Prevention:**
1. **Use baseline file to suppress existing issues:**
   ```gradle
   detekt {
       baseline = file("detekt-baseline.xml")
   }
   ```
   Run: `./gradlew detektBaseline` to generate
2. **Baseline prevents new violations** - existing issues ignored, new ones caught
3. **Gradually fix existing issues** - chip away at baseline over time
4. **Choose rules carefully** - don't enable everything, focus on:
   - Security rules (always enable)
   - Critical bugs (NullPointerException risks)
   - Defer style rules for later
5. **Block Engineering's lesson:** Took 2 weeks to evaluate which rules made sense for their project

**Detection:**
- Run `./gradlew detekt` and see violation count
- If >100 violations, you need a baseline strategy

**Phase mapping:** Static Analysis phase

**Confidence:** HIGH - [Official detekt docs](https://detekt.dev/) and [Block Engineering migration](https://engineering.block.xyz/blog/adopting-ktfmt-and-detekt)

### Pitfall 11: Detekt Breaking CI Before Team is Ready

**What goes wrong:** Detekt added to CI pipeline immediately, causing all PRs to fail due to violations.

**Why it happens:** Eagerness to enforce quality standards before team is trained.

**Prevention:**
1. **Informational phase first** - detekt runs in CI but doesn't fail builds
2. **Use `ignoreFailures = true` initially:**
   ```gradle
   detekt {
       ignoreFailures = true
   }
   ```
3. **Generate reports** - HTML/Markdown reports for team to review
4. **Train team** - ensure everyone understands rules before enforcement
5. **Gradual enforcement** - start with critical rules, add more over time
6. **Set date for enforcement** - "detekt will start failing builds on [date]"

**Detection:**
- Check if `ignoreFailures` is set
- Review CI config - does detekt failure block merge?

**Phase mapping:** Static Analysis phase (Phase 2-3 of rollout)

**Confidence:** MEDIUM - [ArcTouch static analysis guide](https://arctouch.com/blog/static-analysis-ktlint-detekt)

### Pitfall 12: Testing Untestable Architecture

**What goes wrong:** Attempting to write unit tests for code that directly depends on Android framework (Activity, Context, Fragment) without refactoring first.

**Why it happens:** Desire to add tests immediately without addressing underlying architecture problems.

**Prevention:**
1. **Accept that architecture needs refactoring** - legacy code is often untestable as-is
2. **Extract business logic from Android components:**
   ```kotlin
   // Before: Untestable
   class MovieActivity : Activity() {
       fun loadMovies() {
           val movies = api.getMovies() // Direct API call
           recyclerView.adapter = MoviesAdapter(movies)
       }
   }

   // After: Testable
   class MovieViewModel(private val repository: MovieRepository) {
       fun loadMovies(): Flow<Result<List<Movie>>> {
           return repository.getMovies()
       }
   }
   ```
3. **Start with ViewModels** - if using MVVM, these are easiest to test
4. **Use dependency injection** - Hilt/Koin for swappable dependencies
5. **Don't write E2E tests first** - they're slow, brittle, and require UI infrastructure

**Detection:**
- Count how many classes directly extend Activity/Fragment
- Check if business logic is mixed with UI code
- Review if ViewModels/repositories exist

**Phase mapping:** Testing phase (requires architecture refactoring first)

**Confidence:** MEDIUM - [Working with Android Legacy Code](https://medium.com/android-testing-daily/working-effectively-with-android-legacy-code-b71414f195d6), [Kodeco Testing Legacy Apps](https://www.kodeco.com/32839187-testing-legacy-apps-on-android)

### Pitfall 13: Forcing Unit Tests on Code Not Designed for Testing

**What goes wrong:** Writing tests that check implementation details rather than behavior, making refactoring impossible.

**Why it happens:** Trying to achieve high coverage percentage on existing code without refactoring.

**Prevention:**
1. **Start with high-level tests** - test behavior, not implementation:
   ```kotlin
   // Bad - tests implementation
   @Test
   fun `verify fetchMovies calls api_getMovies`() {
       viewModel.fetchMovies()
       verify(api).getMovies()
   }

   // Good - tests behavior
   @Test
   fun `loading movies updates UI state to success`() {
       val movies = listOf(Movie("Inception"))
       coEvery { repository.getMovies() } returns Result.Success(movies)

       viewModel.fetchMovies()

       assertEquals(UiState.Success(movies), viewModel.uiState.value)
   }
   ```
2. **Use Approval Testing** - capture existing behavior before refactoring
3. **Add tests for new features first** - establish pattern before retrofitting
4. **Accept that some code is better replaced than tested**
5. **Test coverage is not a quality metric** - 80% coverage of bad tests = bad codebase

**Detection:**
- Tests break when internal implementation changes
- Tests use lots of mocking without clear behavior verification
- Test names describe "how" instead of "what"

**Phase mapping:** Testing phase

**Confidence:** MEDIUM - [Best way to start testing untested code](https://understandlegacycode.com/blog/best-way-to-start-testing-untested-code/)

### Pitfall 14: Actual Network Calls in Unit Tests

**What goes wrong:** Tests make real HTTP calls to TMDB/Trakt/Supabase APIs, causing slow, flaky tests that fail when internet is unavailable.

**Why it happens:** Not understanding difference between unit tests (fast, isolated) and integration tests (slow, external dependencies).

**Prevention:**
1. **Mock all network calls:**
   ```kotlin
   @Test
   fun `test movie loading`() {
       val mockMovies = listOf(Movie("Inception"))
       coEvery { apiService.getMovies() } returns mockMovies

       val result = repository.getMovies()

       assertEquals(mockMovies, result)
   }
   ```
2. **Use MockWebServer for integration tests:**
   ```kotlin
   val mockWebServer = MockWebServer()
   mockWebServer.enqueue(MockResponse().setBody(movieJson))
   ```
3. **Keep unit tests < 100ms** - if slower, you're probably hitting network/disk
4. **Integration tests run separately** - different test source set

**Detection:**
- Run tests with airplane mode - do they fail?
- Time your test suite - unit tests should be < 10s total

**Phase mapping:** Testing phase

**Confidence:** HIGH - [JUnit tests common pitfalls](https://javanexus.com/blog/common-pitfalls-junit-tests-android)

---

## Minor Pitfalls

Mistakes that cause annoyance but are easily fixable.

### Pitfall 15: Not Handling Asynchronous Code in Tests

**What goes wrong:** Tests for RxJava/Coroutines fail with NullPointerException or don't wait for async operations to complete.

**Prevention:**
1. **For RxJava:** Use `TestScheduler` or `.blockingGet()`
   ```kotlin
   @Test
   fun `test async operation`() {
       val result = repository.getMovies().blockingGet()
       assertEquals(expected, result)
   }
   ```
2. **For Coroutines:** Use `runTest` from kotlinx-coroutines-test
   ```kotlin
   @Test
   fun `test suspend function`() = runTest {
       val result = repository.getMovies()
       assertEquals(expected, result)
   }
   ```
3. **Set test dispatchers:**
   ```kotlin
   @Before
   fun setup() {
       Dispatchers.setMain(StandardTestDispatcher())
   }
   ```

**Confidence:** MEDIUM - [RxJava testing guide](https://www.ericthecoder.com/2019/09/16/unit-testing-android-with-rxjava-and-retrofit/)

### Pitfall 16: Missing @RunWith and @Mock Annotations

**What goes wrong:** Tests fail with "JUnit version 3.8 or later expected" or mocks don't work.

**Prevention:**
```kotlin
@RunWith(MockitoJUnitRunner::class)
class MovieRepositoryTest {
    @Mock
    lateinit var apiService: ApiService

    @Before
    fun setup() {
        MockitoAnnotations.openMocks(this)
    }
}
```

**Confidence:** MEDIUM - [JUnit tests common pitfalls](https://javanexus.com/blog/common-pitfalls-junit-tests-android)

---

## Android TV Specific Pitfalls

### Pitfall 17: Android TV Apps Lack Malware Detection

**What goes wrong:** Android TV apps have weaker security posture than mobile apps. Research shows "static identifiers for tracking persist despite Google's new policies" and "insecure coding practices among developers."

**Why it happens:**
- Android TV development is less mature than mobile
- Fewer security scanning tools target TV apps
- Developers port mobile apps without TV-specific security review

**Prevention:**
1. **Apply all mobile security best practices to TV apps**
2. **Review tracking/advertising libraries** - many persist static identifiers
3. **Audit third-party dependencies** - TV apps often use obsolete communication APIs
4. **Test on actual TV devices** - emulator doesn't catch all issues
5. **Enable Google Play Protect** (on by default, but verify)

**Detection:**
- Run static analysis tools (detekt, SonarQube)
- Check for deprecated APIs in dependencies
- Review network traffic with HTTPS proxy

**Phase mapping:** All phases (security mindset throughout)

**Confidence:** MEDIUM - [IEEE research on Android TV security](https://ieeexplore.ieee.org/document/9679824/), [MDPI malware detection study](https://www.mdpi.com/2076-3417/15/5/2802)

### Pitfall 18: Android TV Account Security Loophole

**What goes wrong:** Historical Android TV OS security loophole allowed unauthorized account access (fixed in 2024, but verify your min SDK).

**Prevention:**
1. **Update to latest Android TV OS security patch** (2026-01-05 or later)
2. **Verify user session validity** - don't trust cached credentials
3. **Implement re-authentication for sensitive operations**

**Confidence:** LOW - Single source ([9to5Google article](https://9to5google.com/2024/04/25/google-android-tv-os-security-loophole/)) about past vulnerability

---

## Phase-Specific Warnings

| Phase Topic | Likely Pitfall | Mitigation |
|-------------|---------------|------------|
| Secrets Migration | BuildConfig false security | Add API restrictions, consider NDK obfuscation for high-value keys |
| Secrets Migration | SSL validation disabled leaking to release | Use NetworkSecurityConfig with debug-overrides |
| Secrets Migration | CI/CD secrets missing | Configure environment variables in CI |
| Error Handling | Silent nulls continuing | Adopt Result<T> pattern, fail loudly |
| Error Handling | JWT missing exp accepted | Reject tokens missing required claims |
| Error Handling | Global error mapper | Client-specific error handling |
| Static Analysis | 500+ violations paralyzing team | Use detekt baseline file |
| Static Analysis | Breaking CI too early | Start informational, gradual enforcement |
| Testing | Testing untestable architecture | Refactor for testability first (ViewModels, DI) |
| Testing | Unit tests hitting network | Mock all external dependencies |
| Testing | Forcing high coverage on legacy code | Start with Approval Testing, test new features |
| Logging Cleanup | PII in production logs | R8 rules to strip debug/info logs, sanitize sensitive objects |
| All Phases | Android TV-specific security gaps | Apply mobile security standards + TV-specific audits |

---

## Recommendations for ARVIO

Based on ARVIO's specific antipatterns, prioritize:

### Phase 1: Secrets Migration (HIGH RISK)
1. **Fix SSL validation** - Use NetworkSecurityConfig, remove custom TrustManager
2. **Migrate API keys to BuildConfig** - But add API restrictions on backend
3. **Document that BuildConfig ≠ security** - Set team expectations
4. **Configure CI secrets** - Prevent build failures

### Phase 2: Error Handling (MEDIUM RISK)
1. **Fix JWT expiration check** - Reject missing `exp` claims
2. **Adopt Result<T> pattern** - Replace silent nulls
3. **Client-specific error handling** - Don't use global mapper

### Phase 3: Logging Cleanup (COMPLIANCE RISK)
1. **Add R8 rules** - Strip debug/info logs from release
2. **Sanitize sensitive objects** - Override toString()
3. **Audit 399+ log statements** - Remove PII

### Phase 4: Static Analysis (QUALITY IMPROVEMENT)
1. **Generate detekt baseline** - Don't let violations block progress
2. **Enable security rules only first** - Other rules can wait
3. **Informational mode in CI** - Train team before enforcement

### Phase 5: Testing (LONG-TERM INVESTMENT)
1. **Refactor for testability first** - Extract ViewModels/repositories
2. **Add tests for new features** - Establish pattern
3. **Use Approval Testing for legacy code** - Capture behavior before refactoring

---

## Sources

### Official Android Documentation (HIGH confidence)
- [Log Info Disclosure](https://developer.android.com/privacy-and-security/risks/log-info-disclosure)
- [Unsafe X509TrustManager](https://developer.android.com/privacy-and-security/risks/unsafe-trustmanager)
- [Network Security Configuration](https://developer.android.com/privacy-and-security/security-config)
- [Android Security Bulletin January 2026](https://source.android.com/docs/security/bulletin/2026/2026-01-01)

### Google Official Tools (HIGH confidence)
- [Secrets Gradle Plugin](https://developers.google.com/maps/documentation/android-sdk/secrets-gradle-plugin)

### Detekt/Static Analysis (HIGH confidence)
- [Detekt Official Documentation](https://detekt.dev/)
- [Block Engineering: Adopting Ktfmt and Detekt](https://engineering.block.xyz/blog/adopting-ktfmt-and-detekt)

### Community Best Practices (MEDIUM confidence)
- [ProAndroidDev: Gradle Properties, BuildConfig, and Secrets Management](https://proandroiddev.com/gradle-properties-buildconfig-and-secrets-management-the-right-way-8b1b161aaefd)
- [Canopas: Retrofit Error Handling with Kotlin Coroutine and Result API](https://medium.com/canopas/retrofit-effective-error-handling-with-kotlin-coroutine-and-result-api-405217e9a73d)
- [Medium: Enforcing Code Quality in Android with Detekt and Ktlint](https://medium.com/@mohamad.alemicode/enforcing-code-quality-in-android-with-detekt-and-ktlint-a-practical-guide-907b57d047ec)
- [ArcTouch: Static Analysis with ktlint and detekt](https://arctouch.com/blog/static-analysis-ktlint-detekt)
- [Curity: JWT Security Best Practices](https://curity.io/resources/learn/jwt-best-practices/)
- [Medium: Handling JWT Token Expiration in Android](https://medium.com/@prakash_ranjan/handling-jwt-token-expiration-and-re-authentication-in-android-kotlin-441838e5ce0a)

### Legacy Testing (MEDIUM confidence)
- [Medium: Working Effectively with Android Legacy Code](https://medium.com/android-testing-daily/working-effectively-with-android-legacy-code-b71414f195d6)
- [Kodeco: Testing Legacy Apps on Android](https://www.kodeco.com/32839187-testing-legacy-apps-on-android)
- [UnderstandLegacyCode: Best Way to Start Testing Untested Code](https://understandlegacycode.com/blog/best-way-to-start-testing-untested-code/)
- [Java Tech Blog: Common Pitfalls in Running JUnit Tests for Android](https://javanexus.com/blog/common-pitfalls-junit-tests-android)

### Android TV Security (MEDIUM-LOW confidence)
- [IEEE: A First Look at Security Risks of Android TV Apps](https://ieeexplore.ieee.org/document/9679824/)
- [MDPI: Is Malware Detection Needed for Android TV?](https://www.mdpi.com/2076-3417/15/5/2802)
- [9to5Google: Android TV OS Security Loophole Fixed](https://9to5google.com/2024/04/25/google-android-tv-os-security-loophole/)

### Build Configuration (LOW-MEDIUM confidence)
- [Android Developers: BuildType API Reference](https://developer.android.com/reference/tools/gradle-api/8.3/null/com/android/build/api/dsl/BuildType)
- [Android Developers: Configure Build Variants](https://developer.android.com/build/build-variants)

---

## Confidence Summary

| Category | Confidence | Rationale |
|----------|-----------|-----------|
| Secrets/SSL Security | HIGH | Official Android docs + Google Secrets Gradle Plugin |
| PII Logging | HIGH | Official Android docs with CTS requirements |
| JWT Validation | MEDIUM | Community consensus, no Android-specific official docs |
| Static Analysis Rollout | HIGH | Official detekt docs + Block Engineering case study |
| Testing Legacy Code | MEDIUM | Multiple sources agree, but not Android-official |
| Android TV Specific | MEDIUM-LOW | Research papers + single article sources |

---

## What Might I Have Missing?

1. **Gradle build cache issues** - Potential pitfalls when moving secrets between build types
2. **ProGuard/R8 edge cases** - Specific rules needed for Result<T> sealed classes
3. **Retrofit version-specific issues** - CallAdapter compatibility across versions
4. **Android 14+ security changes** - New restrictions that might affect ARVIO
5. **Supabase-specific security patterns** - ARVIO uses Supabase, but no research found on Supabase + Android security
6. **OAuth flow security** - Google OAuth pitfalls not deeply researched

These gaps should be investigated during phase-specific research as needed.
