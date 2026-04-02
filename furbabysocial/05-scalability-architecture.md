# FurBaby Social - Scalability & Architecture Review

**Date:** 2026-04-03

---

## Architecture Overview

```
src/
├── core/              # App initialization, notification container
├── entities/          # Domain models (user, pet, post, chat, etc.)
│   ├── app/           # Global app state
│   ├── auth/          # Authentication
│   ├── chat/          # Chat/messaging
│   ├── comment/       # Post comments
│   ├── mention/       # @mention system
│   ├── notification/  # Push notifications
│   ├── pet/           # Pet profiles
│   ├── post/          # Social posts
│   ├── service/       # Pet services marketplace
│   ├── session/       # Auth session tokens
│   └── user/          # User profiles
├── features/          # Cross-cutting features
├── screens/           # Screen components (30 total)
├── shared/            # Reusable infrastructure
│   ├── components/    # UI components (53+)
│   ├── config/        # Constants, environment
│   ├── contexts/      # React contexts (ChatFb)
│   ├── hooks/         # Custom hooks
│   ├── services/      # API, Navigation, Effector, i18n
│   ├── theme/         # Restyle theme system
│   └── utils/         # Validation, helpers
└── resources/         # Fonts, icons, images
```

**Pattern:** Feature-Sliced Design (adapted). Clean separation of entities, features, and shared infrastructure.

---

## Scalability Assessment

### 1. State Management Scalability

**Current:** Effector with modelFactory pattern across 35+ models.

| Aspect | Status | Risk |
|--------|--------|------|
| Store isolation | Good - scoped stores | Low |
| Memory management | No store cleanup on logout | Medium |
| State persistence | AsyncStorage (sync bottleneck at scale) | Medium |
| Computed state | Proper use of `.map()` for derived values | Low |
| Side effects | Well-structured with `sample()` and `guard()` | Low |

**Concern:** No evidence of store reset/cleanup on user logout beyond token clearing. Stale user data from previous sessions may persist in entity stores.

**Recommendation:** Implement a `resetAllStores` event triggered on logout that clears all user-specific stores.

---

### 2. Network Layer Scalability

| Aspect | Status | Risk |
|--------|--------|------|
| Request deduplication | Not implemented | Medium |
| Response caching | Not implemented | Medium |
| Retry logic | Only on token refresh (401) | Medium |
| Offline support | No offline queue | High |
| Pagination | Cursor-based via InfiniteList | Low |
| WebSocket reconnection | Centrifuge has built-in reconnect | Low |

**Concerns:**
- No request deduplication - rapid navigation could trigger duplicate API calls
- No offline queue - failed mutations are silently lost
- No response caching strategy (stale-while-revalidate, etc.)
- Single API version (`/api/v1/`) with no fallback

**Recommendations:**
1. Implement request deduplication in the API factory
2. Add an offline mutation queue with retry-on-reconnect
3. Consider React Query or SWR patterns for cache management
4. Plan API versioning strategy

---

### 3. Rendering Performance

| Aspect | Status | Risk |
|--------|--------|------|
| Memoization | Good - 148 useMemo/useCallback instances | Low |
| List rendering | FlashList for chat, InfiniteList for feeds | Low |
| Code splitting | **Not implemented** | Medium |
| Lazy loading | **Not implemented** | Medium |
| Image optimization | FastImage with CloudFront CDN | Low |
| Animation | Reanimated 3 (native driver) | Low |

**Concerns:**
- No `React.lazy()` or `Suspense` boundaries - entire app loads at startup
- No screen-level code splitting
- ChatFbProvider (557 lines) renders context for entire session - heavy initialization

**Recommendations:**
1. Implement lazy loading for tab screens
2. Add Suspense boundaries with skeleton placeholders
3. Split ChatFbProvider into smaller, lazily-initialized contexts
4. Profile startup time and implement deferred initialization

---

### 4. Data Layer Scalability

| Aspect | Status | Risk |
|--------|--------|------|
| Pagination | Implemented in InfiniteList | Low |
| Data normalization | Not implemented | Medium |
| Optimistic updates | Not implemented | Medium |
| Real-time sync | Centrifuge + Firestore | Low |
| Image upload | Resize + S3 upload pipeline | Low |

**Concerns:**
- No data normalization (e.g., a user entity appears in posts, comments, friends - all as separate copies)
- No optimistic updates for likes, follows, or comments
- Post/pet creation requires server round-trip before UI update

**Recommendations:**
1. Consider a normalized entity cache for users/pets appearing in multiple contexts
2. Implement optimistic updates for high-frequency actions (likes, comments)

---

### 5. Build & Bundle Scalability

| Aspect | Status | Risk |
|--------|--------|------|
| Bundle size | Unknown (no analysis tool) | Medium |
| Tree shaking | Metro bundler (limited) | Medium |
| Hermes engine | Enabled (Android) | Low |
| New Architecture | Enabled (Android) | Low |
| Module resolution | Path aliases (~/) | Low |

**Recommendations:**
1. Add `react-native-bundle-visualizer` to analyze bundle composition
2. Audit lodash imports (should use `lodash/specific-function` not full import)
3. Verify tree shaking effectiveness for large dependencies

---

### 6. Feature Scalability Concerns

#### Chat System
- Firebase Firestore + Centrifuge is a dual real-time system - adds complexity
- ChatFbProvider manages all chat state in a single context (557 lines)
- No message pagination beyond initial load limit of 10
- No message search capability

#### Social Feed
- Feed types (all, friends, public) are separate API calls with separate stores
- No feed caching between tab switches
- No background feed refresh

#### Pet Services Marketplace
- Static service browsing only
- No search/filter infrastructure for scaling to many services
- No service booking/payment flow visible

---

## Scaling Readiness Score

| Dimension | Score (1-5) | Notes |
|-----------|-------------|-------|
| State Management | 4 | Good patterns, needs cleanup on logout |
| Network Resilience | 2 | No offline, caching, or dedup |
| Rendering Performance | 3 | Good memoization, no code splitting |
| Data Architecture | 3 | Pagination works, no normalization |
| Build Optimization | 3 | Hermes enabled, no bundle analysis |
| Feature Extensibility | 4 | Clean entity separation allows easy additions |
| **Overall** | **3.2/5** | **Solid for current scale; needs work for growth** |
