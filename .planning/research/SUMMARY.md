# Project Research Summary

**Project:** ARVIO Android TV Streaming App
**Domain:** Android TV / Streaming Media / Production Quality Retrofit
**Researched:** 2026-01-29
**Confidence:** HIGH

## Executive Summary

ARVIO is an Android TV streaming app built with Kotlin + Jetpack Compose for TV that requires comprehensive production quality fixes (P0-P3) before public release. Research across four domains reveals a clear path: the app currently has critical security vulnerabilities (hardcoded API keys, SSL validation disabled in debug potentially leaking to release), significant observability gaps (399+ production log statements, no crash reporting, silent API failures), and no testing infrastructure beyond one utility test.

The recommended approach prioritizes security fixes first (secrets management, SSL configuration), followed by error handling infrastructure (sealed class Result pattern, network state management), then observability (logging cleanup, Crashlytics), and finally testing. This sequencing addresses dependencies correctly: you cannot test error handling that does not exist, and you cannot safely ship without first securing secrets and establishing crash reporting. The architecture research confirms ARVIO's existing MVVM + Hilt foundation is sound and can integrate the recommended patterns without major refactoring.

Key risks include the false sense of security from BuildConfig (keys are still extractable from APK), the 500+ expected Detekt violations when static analysis is introduced (use baseline strategy), and the temptation to write tests for untestable code (refactor ViewModels first). The 2-3 week timeline for Phases 1-4 is achievable if the team resists scope creep and accepts that some improvements (offline caching, advanced features) belong in v2.

## Key Findings

### Recommended Stack

The 2025/2026 Android security and testing ecosystem has evolved significantly. `androidx.security.crypto` (EncryptedSharedPreferences) is deprecated as of June 2025 due to strict mode violations and keyset corruption issues. The modern approach uses Android Keystore + DataStore + Tink for runtime encryption, with Google Secrets Gradle Plugin for build-time secret injection.

**Core technologies:**
- **Google Secrets Gradle Plugin 2.0.1:** Keep API keys out of version control, inject via BuildConfig
- **Android Keystore + DataStore 1.2.0:** Hardware-backed encryption for sensitive runtime data (replaces EncryptedSharedPreferences)
- **Detekt 1.23.8 + detekt-formatting:** Unified static analysis and ktlint formatting in single tool
- **R8 with proguard-android-optimize.txt:** Mandatory for release builds, 30% startup improvement (Disney+ case study)
- **JUnit 4 + MockK 1.13.8 + Turbine 1.2.1:** Kotlin-native testing stack for ViewModels and Flows
- **Firebase Crashlytics:** Production crash reporting with AI insights and BigQuery export

### Expected Features

**Must have (table stakes):**
- Meaningful error messages (no generic "Error occurred", include error codes for support)
- Network connectivity checks before all API calls
- Retry mechanisms with exponential backoff (10s start, 32-64s max, jitter for thundering herd)
- Loading indicators within 0.1s of user action
- Production-safe logging (strip debug/verbose/info via R8)
- Crash reporting with custom context (user_id hashed, content_id, playback_state)
- TalkBack accessibility support with contentDescription on all interactive elements
- D-pad navigation with proper focus management (no dead ends)

**Should have (competitive):**
- Analytics event tracking (content_view, playback_start, error_occurred)
- Performance monitoring (startup time, API latency)
- Remote configuration (feature flags for kill switches)
- Watch Next integration for Android TV home screen
- Skeleton screens for content loading

**Defer (v2+):**
- Offline caching (high complexity, 7-10 days)
- Picture-in-Picture support
- Voice search integration
- Multi-device playback sync
- Predictive pre-loading

### Architecture Approach

ARVIO's existing MVVM + Hilt architecture provides a solid foundation. The recommended integration adds three key patterns: (1) Secrets Gradle Plugin for build-time injection into BuildConfig, with Hilt modules providing secrets to the DI graph; (2) Sealed class `Result<T>` pattern for type-safe error propagation through Repository -> ViewModel -> UI layers via StateFlow; (3) NetworkMonitor + Network Interceptor for centralized connectivity detection with automatic error wrapping.

**Major components:**
1. **NetworkModule (Hilt):** Provides OkHttpClient with NetworkStateInterceptor, injects API keys from BuildConfig
2. **Result<T> + AppException:** Sealed classes for type-safe error handling, replaces null returns
3. **UiState<T>:** ViewModel state wrapper with Loading/Success/Error/Idle states, includes retry action
4. **NetworkMonitor:** ConnectivityManager wrapper, exposes `isConnected()` and `Flow<Boolean>` for proactive UI updates

### Critical Pitfalls

1. **BuildConfig secrets are not actually secret:** Keys compiled into APK are recoverable via decompilation. Must add API restrictions (package name/SHA-256 fingerprint) on provider consoles. Consider NDK obfuscation for high-value keys.

2. **SSL validation disabled in debug leaking to release:** Custom X509TrustManager implementations are dangerous. Use NetworkSecurityConfig.xml with debug-overrides instead. Google Play scans for unsafe TrustManager.

3. **PII in production logs (399+ statements):** 6% of top 100 Android apps leak PII to logs. Add R8 rules to strip debug/verbose/info logs. Override toString() on sensitive objects. CTS tests check for this.

4. **JWT expiration check returns false when missing exp claim:** This allows tokens without expiration to never expire. Reject tokens missing required claims (exp, iss, aud).

5. **Massive Detekt violation count on first run:** Expect 500-1000+ violations. Use `./gradlew detektBaseline` to generate baseline file. Enable security rules only initially, defer style rules.

## Implications for Roadmap

Based on research, suggested phase structure:

### Phase 1: Critical Security (Secrets Migration)

**Rationale:** Hardcoded API keys in Constants.kt is the highest-risk vulnerability. Must fix before any public release. SSL validation issues compound this risk.

**Delivers:** API keys removed from source control, BuildConfig injection working, SSL configuration corrected, CI/CD secrets configured

**Addresses:** P0 security vulnerability (API key exposure), SSL validation leak risk

**Avoids:** BuildConfig false security pitfall (by adding API restrictions on provider consoles)

**Time estimate:** 1-2 days

### Phase 2: Error Handling Foundation

**Rationale:** Silent null returns on API failures prevent proper debugging and user communication. Must establish error handling infrastructure before building features on top.

**Delivers:** Result<T> sealed class, AppException hierarchy, NetworkMonitor, NetworkStateInterceptor, error mapping utilities

**Uses:** Sealed classes, StateFlow, OkHttp interceptors

**Implements:** Repository error wrapping, ViewModel state transformation

**Avoids:** JWT validation pitfall (fix missing exp claim handling), global error mapper pitfall (client-specific handlers)

**Time estimate:** 3-4 days

### Phase 3: Repository Migration

**Rationale:** Incremental migration of existing repositories to use Result<T> pattern. Breaking changes require ViewModel/UI updates, so migrate one feature at a time.

**Delivers:** All repositories return Result<T>, ViewModels expose StateFlow<UiState<T>>, UI observes with repeatOnLifecycle

**Addresses:** Silent API failures, graceful degradation, meaningful error messages

**Avoids:** Testing untestable architecture pitfall (creates testable foundation)

**Time estimate:** 5-7 days (depends on repository count)

### Phase 4: Observability (Logging + Crashlytics)

**Rationale:** Cannot ship to production without crash visibility. Logging cleanup required for security compliance (PII in logs).

**Delivers:** R8 rules stripping debug logs, Firebase Crashlytics integrated, custom crash context (user_id, content_id, playback_state), non-fatal error tracking

**Uses:** R8 proguard rules, Firebase Crashlytics SDK

**Addresses:** 399+ log statement issue, no crash reporting gap, compliance requirements

**Avoids:** PII logging pitfall (R8 strips, toString() overrides sanitize)

**Time estimate:** 2-3 days

### Phase 5: Static Analysis Setup

**Rationale:** Establish code quality baseline before adding tests. Prevents new violations while allowing gradual cleanup of existing issues.

**Delivers:** Detekt with ktlint formatting, Android Lint configuration, baseline files, pre-commit hooks (optional)

**Uses:** Detekt 1.23.8, detekt-formatting plugin

**Avoids:** Massive violation count pitfall (baseline strategy), breaking CI too early pitfall (informational mode first)

**Time estimate:** 2-3 days

### Phase 6: Testing Infrastructure

**Rationale:** With testable architecture (Result<T>, ViewModels), can now add meaningful tests. Start with ViewModels (easiest to test with MockK + Turbine).

**Delivers:** Test dependencies added, ViewModel unit tests, Repository unit tests, test coverage reporting with JaCoCo

**Uses:** JUnit 4, MockK, Turbine, kotlinx-coroutines-test, Robolectric

**Addresses:** Only one utility test gap, ViewModel testing, Repository testing

**Avoids:** Testing untestable architecture pitfall (Phase 3 created testable foundation), actual network calls pitfall (MockK mocks all dependencies)

**Time estimate:** 5-7 days

### Phase 7: Build Hardening (R8 + Signing)

**Rationale:** Production release requires optimized builds with proper signing. R8 configuration affects APK size and startup performance.

**Delivers:** R8 with proguard-android-optimize.txt, ARVIO-specific ProGuard rules, release signing config, mapping file storage strategy

**Addresses:** Performance optimization, APK size reduction, release preparation

**Time estimate:** 2-3 days

### Phase Ordering Rationale

- **Security first (Phase 1):** Cannot responsibly continue development with hardcoded API keys exposed. SSL fix prevents potential MITM attacks.
- **Error handling before testing (Phases 2-3 before 6):** Cannot test error handling that does not exist. Result<T> pattern provides testable boundaries.
- **Logging cleanup before release (Phase 4):** Compliance requirement for production. PII in logs risks Google Play rejection.
- **Static analysis as foundation (Phase 5):** Prevents accumulation of new violations while establishing quality standards.
- **Testing after architecture (Phase 6):** With clean error handling and testable ViewModels, tests are meaningful and maintainable.
- **Build hardening last (Phase 7):** R8 rules depend on understanding which classes need preservation (informed by earlier phases).

### Research Flags

Phases likely needing deeper research during planning:
- **Phase 3 (Repository Migration):** ARVIO-specific repository structure may require custom patterns. Review existing code before migration.
- **Phase 7 (Build Hardening):** ProGuard rules for Supabase SDK, ExoPlayer, and Compose need validation. Test release build thoroughly.

Phases with standard patterns (skip research-phase):
- **Phase 1 (Secrets Migration):** Well-documented with Google Secrets Gradle Plugin. Straightforward implementation.
- **Phase 4 (Observability):** Firebase Crashlytics has excellent documentation. R8 log stripping is standard pattern.
- **Phase 5 (Static Analysis):** Detekt setup is well-documented. Use official configuration examples.

## Confidence Assessment

| Area | Confidence | Notes |
|------|------------|-------|
| Stack | HIGH | Official Android docs, Google blog posts, verified library versions |
| Features | HIGH | Official TV quality guidelines, Firebase docs, established patterns |
| Architecture | HIGH | Official OkHttp docs, StateFlow best practices, tested patterns |
| Pitfalls | HIGH | Official Android security docs, CTS requirements, community consensus |

**Overall confidence:** HIGH

All research is backed by official Android documentation (developer.android.com), Google official blog posts (android-developers.googleblog.com), and verified library documentation. The 2025/2026 deprecations (EncryptedSharedPreferences, proguard-android.txt) are confirmed via official release notes.

### Gaps to Address

- **Supabase-specific security patterns:** No research found on Supabase + Android security integration. Validate during Phase 2 implementation.
- **OAuth flow security:** Google OAuth pitfalls not deeply researched. May need investigation if auth issues arise.
- **ProGuard/R8 edge cases:** Specific rules for Result<T> sealed classes need validation during Phase 7.
- **Android 14+ security changes:** New restrictions may affect ARVIO if targeting latest SDK. Review during Phase 1.
- **Gradle build cache issues:** Potential pitfalls when moving secrets between build types. Test thoroughly.

## Sources

### Primary (HIGH confidence)
- [Android Keystore System](https://developer.android.com/privacy-and-security/keystore) - Secrets storage
- [Jetpack DataStore](https://developer.android.com/topic/libraries/architecture/datastore) - Encrypted preferences
- [Google Secrets Gradle Plugin](https://github.com/google/secrets-gradle-plugin) - Build-time secrets
- [R8 Shrinking Blog Post (Nov 2025)](https://android-developers.googleblog.com/2025/11/use-r8-to-shrink-optimize-and-fast.html) - Performance improvements
- [TV App Quality Guidelines](https://developer.android.com/docs/quality-guidelines/tv-app-quality) - Production requirements
- [Log Info Disclosure Risk](https://developer.android.com/privacy-and-security/risks/log-info-disclosure) - PII logging
- [Unsafe X509TrustManager](https://developer.android.com/privacy-and-security/risks/unsafe-trustmanager) - SSL validation
- [Detekt Official Docs](https://detekt.dev/) - Static analysis
- [Firebase Crashlytics](https://firebase.google.com/docs/crashlytics) - Crash reporting
- [OkHttp Interceptors](https://square.github.io/okhttp/features/interceptors/) - Network interceptor patterns

### Secondary (MEDIUM confidence)
- [Goodbye EncryptedSharedPreferences: 2026 Migration Guide](https://proandroiddev.com/goodbye-encryptedsharedpreferences-a-2026-migration-guide-4b819b4a537a)
- [Block Engineering: Adopting Ktfmt and Detekt](https://engineering.block.xyz/blog/adopting-ktfmt-and-detekt)
- [Retrofit Error Handling with Kotlin Result API](https://medium.com/canopas/retrofit-effective-error-handling-with-kotlin-coroutine-and-result-api-405217e9a73d)
- [Curity: JWT Security Best Practices](https://curity.io/resources/learn/jwt-best-practices/)
- [Working Effectively with Android Legacy Code](https://medium.com/android-testing-daily/working-effectively-with-android-legacy-code-b71414f195d6)

### Tertiary (LOW confidence)
- [IEEE: Android TV Security Risks](https://ieeexplore.ieee.org/document/9679824/) - TV-specific security
- [9to5Google: Android TV Security Loophole](https://9to5google.com/2024/04/25/google-android-tv-os-security-loophole/) - Historical vulnerability

---

## Critical Decisions Required

Before proceeding to roadmap creation, the following decisions need resolution:

1. **API key restriction strategy:** Which provider consoles need API restrictions configured? (Google Cloud, Firebase, TMDB, Trakt)
2. **Backend proxy consideration:** Is backend infrastructure available for proxying high-value API calls? (v2 vs now)
3. **Baseline tolerance:** What Detekt rules to enable initially? (Recommend: security rules only, style rules deferred)
4. **Test coverage target:** What percentage is realistic for initial release? (Recommend: focus on ViewModel tests, not coverage %)
5. **Crashlytics vs alternatives:** Firebase Crashlytics is recommended, but does team have existing crash reporting preference?

## Tradeoffs Identified

| Decision | Option A | Option B | Recommendation |
|----------|----------|----------|----------------|
| Secret storage | BuildConfig only | BuildConfig + NDK obfuscation | BuildConfig + API restrictions (simpler, adequate for streaming app) |
| Error handling | kotlin.Result | Custom sealed class | Custom sealed class (more explicit, includes Loading state) |
| Static analysis | ktlint standalone | Detekt with formatting | Detekt (single tool, better Android support) |
| Test approach | High coverage on legacy | Behavior tests on new code | Behavior tests + approval testing on legacy |
| Log stripping | ProGuard rules | Timber with custom tree | ProGuard rules (R8 handles at compile time, more reliable) |

---

*Research completed: 2026-01-29*
*Ready for roadmap: yes*
