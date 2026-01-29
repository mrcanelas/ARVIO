# Architecture Research: Secrets & Error Handling Integration

**Domain:** Android TV App with MVVM + Hilt
**Focus:** Secrets Management & Error Handling Integration
**Researched:** 2026-01-29
**Confidence:** HIGH

## Executive Summary

This research addresses how secrets management and error handling integrate with existing MVVM + Hilt architecture in Android apps. The recommended approach uses Google's Secrets Gradle Plugin for build-time secret injection and sealed class-based Result patterns for type-safe error propagation through the repository-ViewModel-UI chain.

## Secrets Management Integration

### Recommended Approach: Secrets Gradle Plugin

**What:** Google's official Secrets Gradle Plugin reads secrets from `local.properties` (excluded from VCS) and exposes them as `BuildConfig` constants and manifest variables at build time.

**Why:** Provides build-time secret injection without runtime overhead, keeps secrets out of VCS, and integrates seamlessly with existing Hilt DI structure.

### Architecture Integration Points

```
Build Time:
local.properties → Gradle Plugin → BuildConfig.FIELD_NAME
                                  → AndroidManifest.xml ${VARIABLE}

Runtime:
BuildConfig.API_KEY → @Provides ApiKeyProvider → Retrofit/OkHttp (via Hilt)
```

#### Component Responsibilities

| Component | Responsibility | Secret Access |
|-----------|---------------|---------------|
| **Gradle Plugin** | Read `local.properties`, generate BuildConfig | Build-time only |
| **BuildConfig** | Store compiled secrets as static constants | Build artifact |
| **Hilt Module (AppModule)** | Provide secrets to DI graph | `BuildConfig.API_KEY` |
| **OkHttp/Retrofit** | Use secrets in network requests | Injected via constructor |

### Implementation Pattern

**Step 1: Configure Secrets Gradle Plugin**

Root `build.gradle.kts`:
```kotlin
buildscript {
    dependencies {
        classpath("com.google.android.libraries.mapsplatform.secrets-gradle-plugin:secrets-gradle-plugin:2.0.1")
    }
}
```

App-level `build.gradle.kts`:
```kotlin
plugins {
    id("com.google.android.libraries.mapsplatform.secrets-gradle-plugin")
}
```

**Step 2: Store Secrets in local.properties**

`local.properties` (gitignored):
```properties
TMDB_API_KEY=your_actual_api_key_here
BACKEND_BASE_URL=https://api.production.com
```

**Step 3: Inject via Hilt**

```kotlin
@Module
@InstallIn(SingletonComponent::class)
object NetworkModule {

    @Provides
    @Singleton
    fun provideApiKey(): String = BuildConfig.TMDB_API_KEY

    @Provides
    @Singleton
    fun provideOkHttpClient(
        apiKey: String,
        networkStateInterceptor: NetworkStateInterceptor
    ): OkHttpClient {
        return OkHttpClient.Builder()
            .addInterceptor { chain ->
                val request = chain.request().newBuilder()
                    .addHeader("Authorization", "Bearer $apiKey")
                    .build()
                chain.proceed(request)
            }
            .addNetworkInterceptor(networkStateInterceptor)
            .build()
    }
}
```

### Build Order Implications

1. **Secrets MUST exist before Gradle sync** - Missing `local.properties` causes build failure
2. **Provide fallback for CI/CD** - Use `local.defaults.properties` with safe default values
3. **Secrets available immediately after build** - No runtime initialization needed
4. **Rebuilds required for secret changes** - Changing `local.properties` requires rebuilding

### Security Limitations

**CRITICAL:** Secrets Gradle Plugin protects against VCS leaks but NOT reverse engineering. API keys compiled into APK are recoverable via decompilation.

**Mitigation strategies:**
- Add API key restrictions in provider console (package name, SHA-1)
- Use backend proxy for sensitive operations
- Implement certificate pinning for critical APIs
- Use Android Keystore for user credentials (separate from API keys)

## Error Handling Flow

### Recommended Pattern: Sealed Class Result<T>

**What:** Type-safe wrapper representing operation states (Loading, Success, Error) propagated through repository → ViewModel → UI layers using StateFlow.

**Why:** Eliminates null returns, makes error cases explicit in type system, provides compile-time safety for state handling.

### Error Propagation Architecture

```
API Call → Retrofit/OkHttp
    ↓ (try-catch in Repository)
Repository → Result<T> sealed class
    ↓ (map/transform)
ViewModel → UiState<T> StateFlow
    ↓ (collectAsState)
UI Layer → when(state) { ... }
```

### Component Boundaries

| Layer | Input | Output | Error Handling Responsibility |
|-------|-------|--------|------------------------------|
| **Retrofit/OkHttp** | API request | Response/Exception | Throw exceptions |
| **Repository** | API call | `Result<T>` | Catch exceptions, wrap in Result.Error |
| **ViewModel** | `Result<T>` | `StateFlow<UiState<T>>` | Transform Result to UiState, business logic |
| **UI (Activity/Fragment)** | `StateFlow<UiState<T>>` | Rendered UI | Display loading/success/error states |

### Sealed Class Definitions

**Repository Layer - Result<T>:**

```kotlin
sealed class Result<out T> {
    data class Success<T>(val data: T) : Result<T>()
    data class Error(val exception: AppException) : Result<Nothing>()
    object Loading : Result<Nothing>()
}

// Domain-specific error types
sealed class AppException : Exception() {
    data class NetworkException(val code: Int, override val message: String) : AppException()
    data class NoConnectivityException(override val message: String = "No internet connection") : AppException()
    data class ServerException(val code: Int, override val message: String) : AppException()
    data class UnauthorizedException(override val message: String = "Authentication failed") : AppException()
    data class NotFoundException(override val message: String = "Resource not found") : AppException()
    data class UnknownException(override val message: String = "An unknown error occurred") : AppException()
}
```

**ViewModel Layer - UiState<T>:**

```kotlin
sealed class UiState<out T> {
    object Idle : UiState<Nothing>()
    object Loading : UiState<Nothing>()
    data class Success<T>(val data: T) : UiState<T>()
    data class Error(val message: String, val retryAction: (() -> Unit)? = null) : UiState<Nothing>()
}
```

### Implementation Pattern

**Repository Layer:**

```kotlin
@Singleton
class StreamRepository @Inject constructor(
    private val api: StreamApi,
    private val networkMonitor: NetworkMonitor
) {
    suspend fun getStreams(): Result<List<Stream>> {
        return try {
            if (!networkMonitor.isConnected()) {
                return Result.Error(AppException.NoConnectivityException())
            }

            val response = api.getStreams()
            if (response.isSuccessful) {
                Result.Success(response.body() ?: emptyList())
            } else {
                Result.Error(mapHttpError(response.code(), response.message()))
            }
        } catch (e: IOException) {
            Result.Error(AppException.NetworkException(-1, "Network error: ${e.message}"))
        } catch (e: Exception) {
            Result.Error(AppException.UnknownException(e.message ?: "Unknown error"))
        }
    }

    private fun mapHttpError(code: Int, message: String): AppException {
        return when (code) {
            401 -> AppException.UnauthorizedException()
            404 -> AppException.NotFoundException()
            in 500..599 -> AppException.ServerException(code, message)
            else -> AppException.NetworkException(code, message)
        }
    }
}
```

**ViewModel Layer:**

```kotlin
@HiltViewModel
class StreamViewModel @Inject constructor(
    private val repository: StreamRepository
) : ViewModel() {

    private val _uiState = MutableStateFlow<UiState<List<Stream>>>(UiState.Idle)
    val uiState: StateFlow<UiState<List<Stream>>> = _uiState.asStateFlow()

    fun loadStreams() {
        viewModelScope.launch {
            _uiState.value = UiState.Loading

            when (val result = repository.getStreams()) {
                is Result.Success -> {
                    _uiState.value = UiState.Success(result.data)
                }
                is Result.Error -> {
                    _uiState.value = UiState.Error(
                        message = result.exception.message ?: "Unknown error",
                        retryAction = { loadStreams() }
                    )
                }
                is Result.Loading -> {
                    // Already set to loading
                }
            }
        }
    }
}
```

**UI Layer (Fragment/Activity):**

```kotlin
class StreamListFragment : Fragment() {
    private val viewModel: StreamViewModel by viewModels()

    override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
        super.onViewCreated(view, savedInstanceState)

        viewLifecycleOwner.lifecycleScope.launch {
            viewLifecycleOwner.repeatOnLifecycle(Lifecycle.State.STARTED) {
                viewModel.uiState.collect { state ->
                    when (state) {
                        is UiState.Idle -> {
                            // Initial state
                        }
                        is UiState.Loading -> {
                            showLoading()
                        }
                        is UiState.Success -> {
                            hideLoading()
                            displayStreams(state.data)
                        }
                        is UiState.Error -> {
                            hideLoading()
                            showError(state.message, state.retryAction)
                        }
                    }
                }
            }
        }

        viewModel.loadStreams()
    }
}
```

### Data Flow Direction

**Unidirectional Data Flow:**

```
User Action (UI) → ViewModel method call
    ↓
ViewModel → Repository call
    ↓
Repository → Network/Database
    ↓
Repository ← Response/Error
    ↓
ViewModel ← Result<T>
    ↓
ViewModel → Transform to UiState<T>
    ↓
UI ← Observe StateFlow<UiState<T>>
```

**Key principle:** Data flows down (state), events flow up (actions).

## Network State Management

### Architecture Decision: Network Interceptor Pattern

**Where to detect:** OkHttp Network Interceptor + Android ConnectivityManager
**How to propagate:** Throw custom exception in interceptor, caught by Repository, wrapped in Result.Error

### Component Responsibilities

| Component | Responsibility | Integration Point |
|-----------|---------------|-------------------|
| **NetworkMonitor** | Observe ConnectivityManager, expose isConnected() | Injected into Repository |
| **NetworkStateInterceptor** | Check connectivity before network call | Added to OkHttpClient via Hilt |
| **Repository** | Catch NoConnectivityException, wrap in Result.Error | try-catch block |
| **ViewModel** | Transform to user-friendly error message | Result → UiState mapping |
| **UI** | Display offline state, offer retry | Observe UiState.Error |

### Implementation Pattern

**NetworkMonitor (Injectable):**

```kotlin
interface NetworkMonitor {
    fun isConnected(): Boolean
    fun observeConnectivity(): Flow<Boolean>
}

@Singleton
class NetworkMonitorImpl @Inject constructor(
    @ApplicationContext private val context: Context
) : NetworkMonitor {

    private val connectivityManager = context.getSystemService(Context.CONNECTIVITY_SERVICE) as ConnectivityManager

    override fun isConnected(): Boolean {
        val network = connectivityManager.activeNetwork ?: return false
        val capabilities = connectivityManager.getNetworkCapabilities(network) ?: return false
        return capabilities.hasCapability(NetworkCapabilities.NET_CAPABILITY_INTERNET)
    }

    override fun observeConnectivity(): Flow<Boolean> = callbackFlow {
        val callback = object : ConnectivityManager.NetworkCallback() {
            override fun onAvailable(network: Network) {
                trySend(true)
            }

            override fun onLost(network: Network) {
                trySend(false)
            }
        }

        connectivityManager.registerDefaultNetworkCallback(callback)
        awaitClose { connectivityManager.unregisterNetworkCallback(callback) }
    }
}
```

**NetworkStateInterceptor:**

```kotlin
class NetworkStateInterceptor @Inject constructor(
    private val networkMonitor: NetworkMonitor
) : Interceptor {

    override fun intercept(chain: Interceptor.Chain): Response {
        if (!networkMonitor.isConnected()) {
            throw AppException.NoConnectivityException()
        }
        return chain.proceed(chain.request())
    }
}
```

**Hilt Integration:**

```kotlin
@Module
@InstallIn(SingletonComponent::class)
object NetworkModule {

    @Provides
    @Singleton
    fun provideNetworkMonitor(
        @ApplicationContext context: Context
    ): NetworkMonitor = NetworkMonitorImpl(context)

    @Provides
    @Singleton
    fun provideOkHttpClient(
        networkStateInterceptor: NetworkStateInterceptor
    ): OkHttpClient {
        return OkHttpClient.Builder()
            .addNetworkInterceptor(networkStateInterceptor) // Use network interceptor
            .connectTimeout(30, TimeUnit.SECONDS)
            .readTimeout(30, TimeUnit.SECONDS)
            .build()
    }
}
```

### Why Network Interceptor (Not Application Interceptor)?

**Network Interceptor advantages:**
- Observes data as transmitted over network
- Not invoked for cached responses (avoids false negatives)
- Access to Connection object
- Operates on intermediate responses (redirects, retries)

**Application Interceptor disadvantages:**
- Always invoked, even for cached responses
- Cannot access network-level information
- Higher in OkHttp stack (less visibility)

**Source:** Official OkHttp documentation recommends network interceptors for monitoring actual network behavior.

### Proactive Network State Handling

**In addition to interceptor**, expose connectivity state to ViewModels:

```kotlin
@HiltViewModel
class StreamViewModel @Inject constructor(
    private val repository: StreamRepository,
    private val networkMonitor: NetworkMonitor
) : ViewModel() {

    val isConnected: StateFlow<Boolean> = networkMonitor.observeConnectivity()
        .stateIn(viewModelScope, SharingStarted.WhileSubscribed(5000), true)

    // UI can observe and show offline banner
}
```

## Testing Architecture

### What to Mock, How to Structure

#### Unit Testing Strategy

| Layer | What to Test | What to Mock | Testing Library |
|-------|-------------|--------------|----------------|
| **Repository** | Error mapping, Result wrapping | Retrofit API interface | JUnit, MockK, Kotlin Coroutines Test |
| **ViewModel** | Business logic, state transformations | Repository | JUnit, MockK, Turbine (Flow testing) |
| **UI** | State rendering, user interactions | ViewModel | Espresso/Compose Test, Hilt Testing |

#### Repository Layer Testing

**What to test:**
- Success response maps to Result.Success
- HTTP errors map to correct AppException types
- Network exceptions caught and wrapped
- No connectivity throws NoConnectivityException

**Mocking strategy:**

```kotlin
class StreamRepositoryTest {

    @MockK
    private lateinit var api: StreamApi

    @MockK
    private lateinit var networkMonitor: NetworkMonitor

    private lateinit var repository: StreamRepository

    @Before
    fun setup() {
        MockKAnnotations.init(this)
        repository = StreamRepository(api, networkMonitor)
    }

    @Test
    fun `getStreams returns Success when API call succeeds`() = runTest {
        // Given
        val mockStreams = listOf(Stream(id = "1", title = "Test"))
        coEvery { networkMonitor.isConnected() } returns true
        coEvery { api.getStreams() } returns Response.success(mockStreams)

        // When
        val result = repository.getStreams()

        // Then
        assertTrue(result is Result.Success)
        assertEquals(mockStreams, (result as Result.Success).data)
    }

    @Test
    fun `getStreams returns Error when network unavailable`() = runTest {
        // Given
        coEvery { networkMonitor.isConnected() } returns false

        // When
        val result = repository.getStreams()

        // Then
        assertTrue(result is Result.Error)
        assertTrue((result as Result.Error).exception is AppException.NoConnectivityException)
    }

    @Test
    fun `getStreams maps 401 to UnauthorizedException`() = runTest {
        // Given
        coEvery { networkMonitor.isConnected() } returns true
        coEvery { api.getStreams() } returns Response.error(401, "".toResponseBody())

        // When
        val result = repository.getStreams()

        // Then
        assertTrue(result is Result.Error)
        assertTrue((result as Result.Error).exception is AppException.UnauthorizedException)
    }
}
```

#### ViewModel Layer Testing

**What to test:**
- Initial state is Idle
- Loading state emitted before repository call
- Success result transforms to UiState.Success
- Error result transforms to UiState.Error with retry

**Mocking strategy (Hilt NOT needed for unit tests):**

```kotlin
class StreamViewModelTest {

    @MockK
    private lateinit var repository: StreamRepository

    @MockK
    private lateinit var networkMonitor: NetworkMonitor

    private lateinit var viewModel: StreamViewModel

    @get:Rule
    val mainDispatcherRule = MainDispatcherRule()

    @Before
    fun setup() {
        MockKAnnotations.init(this)
        every { networkMonitor.observeConnectivity() } returns flowOf(true)
        viewModel = StreamViewModel(repository, networkMonitor)
    }

    @Test
    fun `loadStreams emits Loading then Success on repository success`() = runTest {
        // Given
        val mockStreams = listOf(Stream(id = "1", title = "Test"))
        coEvery { repository.getStreams() } returns Result.Success(mockStreams)

        // When
        viewModel.loadStreams()

        // Then
        viewModel.uiState.test {
            assertEquals(UiState.Idle, awaitItem()) // Initial
            assertEquals(UiState.Loading, awaitItem())
            val success = awaitItem()
            assertTrue(success is UiState.Success)
            assertEquals(mockStreams, (success as UiState.Success).data)
        }
    }

    @Test
    fun `loadStreams emits Error with retry action on repository error`() = runTest {
        // Given
        val error = AppException.NoConnectivityException()
        coEvery { repository.getStreams() } returns Result.Error(error)

        // When
        viewModel.loadStreams()

        // Then
        viewModel.uiState.test {
            skipItems(2) // Skip Idle and Loading
            val errorState = awaitItem()
            assertTrue(errorState is UiState.Error)
            assertNotNull((errorState as UiState.Error).retryAction)
        }
    }
}
```

#### UI Layer Testing (with Hilt)

**What to test:**
- Loading spinner shows during Loading state
- Success data renders correctly
- Error message displays with retry button
- Retry button triggers ViewModel action

**Hilt testing strategy:**

```kotlin
@HiltAndroidTest
@UninstallModules(NetworkModule::class) // Replace with test module
class StreamListFragmentTest {

    @get:Rule(order = 0)
    val hiltRule = HiltAndroidRule(this)

    @get:Rule(order = 1)
    val activityRule = ActivityScenarioRule(MainActivity::class.java)

    @BindValue
    @JvmField
    val mockRepository: StreamRepository = mockk(relaxed = true)

    @Before
    fun setup() {
        hiltRule.inject()
    }

    @Test
    fun showsLoadingIndicatorWhenStateIsLoading() {
        // Given
        val viewModel = StreamViewModel(mockRepository, mockNetworkMonitor)
        // Set ViewModel state to Loading

        // When
        launchFragmentInContainer<StreamListFragment>()

        // Then
        onView(withId(R.id.loadingIndicator)).check(matches(isDisplayed()))
    }

    @Test
    fun displaysErrorMessageAndRetryButton() {
        // Given - ViewModel with Error state

        // Then
        onView(withText("No internet connection")).check(matches(isDisplayed()))
        onView(withId(R.id.retryButton)).check(matches(isDisplayed()))

        // When
        onView(withId(R.id.retryButton)).perform(click())

        // Then
        verify { mockRepository.getStreams() } // Retry called
    }
}
```

### Testing Tools

| Tool | Purpose | Use Case |
|------|---------|----------|
| **JUnit 4/5** | Test runner | All test types |
| **MockK** | Kotlin-friendly mocking | Mock repositories, APIs |
| **Kotlin Coroutines Test** | Coroutine testing (runTest, TestDispatcher) | Repository, ViewModel |
| **Turbine** | Flow testing library | Testing StateFlow emissions |
| **Hilt Testing** | DI testing (@HiltAndroidTest, @BindValue) | UI/integration tests |
| **Espresso** | UI testing for Views | Fragment/Activity testing |
| **Compose Test** | UI testing for Compose | If using Jetpack Compose |

### Hilt Testing Best Practices

1. **Unit tests don't need Hilt** - Directly instantiate with mocks (faster)
2. **Use @BindValue for simple replacements** - Avoid creating test modules
3. **Use @TestInstallIn for test-wide replacements** - Better build times than @UninstallModules
4. **Test network errors by mocking NetworkMonitor** - More reliable than mocking ConnectivityManager

## Architecture Decision Summary

### Secrets Management

**Decision:** Secrets Gradle Plugin + Hilt injection
**Rationale:** Build-time injection, VCS safety, seamless DI integration, zero runtime overhead
**Tradeoff:** Requires rebuild for changes, secrets still in APK (add API restrictions)

### Error Handling

**Decision:** Sealed class Result<T> pattern with StateFlow
**Rationale:** Type-safe, compile-time checks, clear error boundaries, reactive UI updates
**Tradeoff:** More boilerplate than nullable types, requires when() exhaustiveness

### Network State

**Decision:** NetworkMonitor + Network Interceptor
**Rationale:** Centralized detection, automatic propagation, observability for proactive UI
**Tradeoff:** Requires Context injection for ConnectivityManager, API 21+ features preferred

### Testing Strategy

**Decision:** Layer-specific mocking (no Hilt for unit tests)
**Rationale:** Fast unit tests, Hilt only for integration tests, clear boundaries
**Tradeoff:** More test setup code, but better isolation and speed

## Migration Path from Current ARVIO State

Given ARVIO currently has:
- API keys in Constants.kt
- Null returns on errors
- MVVM with Hilt
- Repository pattern established

### Phase 1: Secrets Management (Low Risk)

1. Add Secrets Gradle Plugin (1 hour)
2. Move API keys from Constants.kt to local.properties (30 min)
3. Update Hilt modules to use BuildConfig (1 hour)
4. Test build and runtime (1 hour)

**Total:** ~3.5 hours, zero breaking changes

### Phase 2: Error Handling Foundation (Medium Risk)

1. Define Result<T> and AppException sealed classes (1 hour)
2. Add NetworkMonitor interface and implementation (2 hours)
3. Create NetworkStateInterceptor (1 hour)
4. Update Hilt NetworkModule to provide interceptor (30 min)
5. Write unit tests for new components (2 hours)

**Total:** ~6.5 hours, still compatible with existing code

### Phase 3: Repository Migration (Breaking Changes)

1. Change one Repository method signature to return Result<T> (1 hour)
2. Update corresponding ViewModel to handle Result (1 hour)
3. Update UI to handle UiState (2 hours)
4. Test end-to-end (1 hour)
5. Repeat for remaining repositories (multiply by N)

**Total per repository:** ~5 hours, breaking changes require ViewModel/UI updates

### Phase 4: Complete Migration

- All repositories return Result<T>
- All ViewModels expose StateFlow<UiState<T>>
- All UI observes StateFlow with repeatOnLifecycle
- Delete old null-check logic

**Recommendation:** Migrate incrementally, one feature at a time, to minimize risk.

## Sources

### Secrets Management
- [Secrets Gradle Plugin - Google Developers](https://developers.google.com/maps/documentation/android-sdk/secrets-gradle-plugin) - Official documentation (HIGH confidence)
- [GitHub - google/secrets-gradle-plugin](https://github.com/google/secrets-gradle-plugin) - Official repository (HIGH confidence)
- [Keeping Your Android Project's Secrets Secret](https://medium.com/@geocohn/keeping-your-android-projects-secrets-secret-393b8855765d) - Medium (MEDIUM confidence)

### Error Handling with Sealed Classes
- [Mastering Error Handling in Android in Clean Architecture with Sealed Classes](https://medium.com/@u3f.r72/mastering-error-handling-in-android-in-clean-architecture-with-sealed-classes-c965cf6a65b6) - Medium (MEDIUM confidence)
- [Android MVVM architecture with clean error handling](https://chathurangashan.com/android-mvvm-architecture-with-clean-error-handling/) - Tutorial (MEDIUM confidence)
- [Error Handling for the new Retrofit 2.6.0+ with Repository Pattern](https://medium.com/@akshaymore.ac/error-handling-for-the-new-retrofit-2-6-0-with-repository-pattern-2984b1f3eff5) - Medium (MEDIUM confidence)

### Network State Management
- [Interceptors - OkHttp](https://square.github.io/okhttp/features/interceptors/) - Official documentation (HIGH confidence)
- [Using Retrofit Interceptors to check network connection](https://dev.to/theplebdev/using-retrofit-interceptors-to-check-network-connection-in-android-and-testing-it-1kl1) - DEV (MEDIUM confidence)
- [Android: Intercept on no internet connection](https://medium.com/mobile-app-development-publication/android-intercept-on-no-internet-connection-acb91d305357) - Medium (MEDIUM confidence)

### StateFlow Best Practices
- [StateFlow and SharedFlow - Android Developers](https://developer.android.com/kotlin/flow/stateflow-and-sharedflow) - Official documentation (HIGH confidence)
- [Best Practices for Managing State in Android Apps with Kotlin Coroutines and Flow](https://hazaleroglu.medium.com/best-practices-for-managing-state-in-android-apps-with-kotlin-coroutines-and-flow-f1b26218101a) - Medium (MEDIUM confidence)
- [Simplifying Complex Android App Logic with Effective State Management](https://dev.to/lucy1/simplifying-complex-android-app-logic-with-effective-state-management-5d4a) - DEV (MEDIUM confidence)

### Hilt Testing
- [Hilt testing guide - Android Developers](https://developer.android.com/training/dependency-injection/hilt-testing) - Official documentation (HIGH confidence)
- [A Complete Guide to MVVM and ViewModel Testing in Android: Hilt, JUnit, and Mockito Explained](https://medium.com/@deepak.patidark93/a-complete-guide-to-mvvm-and-viewmodel-testing-in-android-hilt-junit-and-mockito-explained-df54324b8dca) - Medium (MEDIUM confidence)
- [Testing an Android MVVM App with Hilt](https://medium.com/@hamanhduy/test-your-android-mvvm-app-with-hilt-2ad583e49f74) - Medium (MEDIUM confidence)

### Repository Pattern
- [Repository Pattern in Android - The Startup](https://medium.com/swlh/repository-pattern-in-android-c31d0268118c) - Medium (MEDIUM confidence)
- [Incorporating the Repository Pattern into a Real-World Android](https://medium.com/@siarhei.krupenich/incorporating-the-repository-pattern-into-a-real-world-android-app-739f2fee1460) - Medium (MEDIUM confidence)
