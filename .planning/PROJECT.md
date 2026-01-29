# ARVIO - Android TV Streaming App

## What This Is

ARVIO is a native Android TV streaming application that aggregates content from multiple sources (TMDB metadata, Stremio addons, Real-Debrid) into a unified Netflix-style interface. Built with Kotlin and Jetpack Compose for TV, it provides D-pad navigation, ExoPlayer-based video playback with codec support for 4K/HDR/DTS/Atmos, and cross-device watch history sync via Trakt and Supabase.

## Core Value

Users can browse, discover, and stream content reliably on their TV with a smooth, crash-free experience.

## Requirements

### Validated

<!-- Existing capabilities in the codebase -->

- Video playback with ExoPlayer + FFmpeg decoder extension — existing
- Content discovery via TMDB API (trending, popular, search) — existing
- Authentication via Supabase (email/password + Google Sign-In) — existing
- Watch history and progress sync via Trakt.tv — existing
- Stream resolution via Stremio addons + Real-Debrid/TorBox — existing
- TV-optimized UI with D-pad/remote navigation — existing
- Watchlist management with cloud sync — existing
- Multi-audio track and subtitle support — existing
- Continue watching functionality — existing
- Netflix-style horizontal row browsing — existing

### Active

<!-- This milestone: Quality & Stability fixes -->

**P0 - Critical Security:**
- [ ] Move API keys from source code to secure BuildConfig
- [ ] Fix JWT expiration validation bug (treats missing exp as valid)
- [ ] Remove SSL certificate validation bypass in OkHttpProvider

**P1 - Logging & Secrets:**
- [ ] Add comprehensive .gitignore at project root
- [ ] Add ProGuard rules to strip Log statements in release builds
- [ ] Remove/wrap all sensitive data logging (API keys, emails, tokens)

**P2 - Reliability & Error Handling:**
- [ ] Replace silent API failures (null returns) with proper Result types
- [ ] Add pagination guard to prevent OOM in getAllPlaybackProgress
- [ ] Fix List<Any> type safety issue in StreamApi.kt
- [ ] Add network connectivity check before API calls
- [ ] Improve error messages (distinguish network/auth/server errors)

**P3 - Code Quality:**
- [ ] Extract magic numbers to Constants (buffer durations, timeouts, dimensions)
- [ ] Fix code duplication in StreamRepository (movie vs episode resolution)
- [ ] Extract hardcoded dp/sp values to theme constants
- [ ] Add detekt/ktlint configuration for code quality
- [ ] Make player auto-hide timeout configurable
- [ ] Add unit tests for critical business logic

### Out of Scope

- New features beyond current functionality — this milestone is stability-focused
- iOS/mobile version — TV-only for now
- Offline download support — requires significant architecture changes
- Multiple user profiles — deferred to future milestone
- Live TV / IPTV support — different product category

## Context

**Current State:**
- ~70 Kotlin source files across data/ui/navigation/util layers
- MVVM architecture with Hilt dependency injection
- Jetpack Compose for TV with custom Arctic Fuse 2 design system
- ExoPlayer 1.3.1 with Jellyfin FFmpeg extension for codec support
- Supabase for auth + cloud sync, Trakt for watch history
- Retrofit + OkHttp for networking, Coil for image loading

**Known Issues (from comprehensive analysis):**
- All API keys hardcoded in Constants.kt (TMDB, Trakt, Supabase, Google OAuth)
- SSL validation completely disabled in debug builds
- JWT expiration check returns false (valid) when exp field is missing
- 399+ log statements execute in production, some with sensitive data
- Silent null returns on API failures make debugging impossible
- No pagination limit on playback progress fetch (potential OOM)
- List<Any> type in StreamApi allows runtime ClassCastException
- Magic numbers scattered throughout PlayerScreen (2141 lines) and HomeScreen (1416 lines)
- Duplicate stream resolution logic for movies vs episodes
- No tests except one utility test file (ContinueWatchingSelectorTest.kt)
- No .gitignore at project root

**Target:**
Public release (Play Store or sideload distribution)

## Constraints

- **Platform**: Android TV only (API 26+, leanback required) — TV-first experience
- **Language**: Kotlin — existing codebase, no reason to change
- **UI Framework**: Jetpack Compose for TV — already adopted, alpha but functional
- **Testing**: Emulator + real Android TV device — must verify on actual hardware
- **Breaking Changes**: Allowed — clean slate to fix issues properly

## Key Decisions

| Decision | Rationale | Outcome |
|----------|-----------|---------|
| Fix all P0-P3 issues | Public release requires production quality | — Pending |
| Allow breaking changes | Enables proper fixes vs workarounds | — Pending |
| Keep existing architecture | MVVM+Hilt is solid, issues are implementation-level | — Pending |
| Use BuildConfig for secrets | Standard Android pattern, works with ProGuard | — Pending |

---
*Last updated: 2026-01-29 after project initialization*
