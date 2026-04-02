# FurBaby Social - Testing Strategy Gap Analysis

**Date:** 2026-04-03

---

## Current State

| Metric | Value |
|--------|-------|
| Test files | 0 |
| Test coverage | 0% |
| Test framework configured | Yes (Jest + react-native preset) |
| Testing libraries installed | Jest only |
| Test IDs on components | None |
| E2E framework | None |
| CI test step | None |

**Verdict: No testing has been implemented.** The Jest config exists as scaffolding only.

---

## Risk Exposure

Without tests, every deployment is a manual QA exercise. The following critical paths have zero automated verification:

### Critical (Business Impact: High)
1. **Authentication flow** - Login → OTP → Token storage → Session management → Token refresh → Logout
2. **Token refresh mechanism** - Silent 401 → refresh → retry pattern in API layer
3. **Payment/Service transactions** - Service marketplace interactions
4. **Chat messaging** - Firebase Firestore read/write, real-time sync, Centrifuge WebSocket
5. **Push notification handling** - FCM token registration, foreground/background handlers

### High (User Impact)
6. **Form validation** - Zod schemas for phone, profile, pet, post, comment
7. **Image upload pipeline** - Pick → crop → resize → upload to S3
8. **Navigation guards** - Auth vs Session navigator switching
9. **Pagination/Infinite scroll** - InfiniteList component, cursor management
10. **Friend management** - Request/accept/block/unblock flows

### Medium (Quality Impact)
11. **Effector state models** - 35+ models with events, effects, computed stores
12. **API error handling** - Field errors, string errors, status codes, toast display
13. **Mention parsing** - Complex regex for @user/@pet mentions in posts
14. **Date/phone formatting** - dayjs and libphonenumber-js transformations
15. **i18n translations** - Internationalization key resolution

---

## Recommended Testing Strategy

### Phase 1: Foundation

**Install required packages:**
```bash
npm install --save-dev @testing-library/react-native @testing-library/jest-native
npm install --save-dev msw                    # API mocking
npm install --save-dev @faker-js/faker        # Test data generation
```

**Priority tests to write:**

#### Unit Tests (Target: 60+ tests)
- [ ] Zod validation schemas (`auth.ts`, `pet.ts`, `comment.ts`) - 15 tests
- [ ] API service error handling (`Api/index.ts`) - 10 tests
- [ ] Session model (token storage, auth state computation) - 10 tests
- [ ] Utility functions (date formatting, phone parsing, mention utils) - 15 tests
- [ ] createApiFactory behavior - 5 tests
- [ ] Effector model factories (event → store update cycles) - 10 tests

#### Component Tests (Target: 30+ tests)
- [ ] Form components (Input, Select, PhoneInput, DatePicker) - 12 tests
- [ ] Button states (loading, disabled, variants) - 5 tests
- [ ] InfiniteList (loading, empty, error states) - 5 tests
- [ ] ConfirmModal / ConfirmSheet behavior - 4 tests
- [ ] Avatar / Image fallback handling - 4 tests

### Phase 2: Integration

**Install additional packages:**
```bash
npm install --save-dev detox                  # E2E testing
```

#### Integration Tests (Target: 20+ tests)
- [ ] Auth flow: Login → Verify → SetupAccount (with MSW mocks) - 5 tests
- [ ] Post creation: BuildPost → API call → Feed refresh - 3 tests
- [ ] Pet CRUD: Create → Edit → Delete with state updates - 4 tests
- [ ] Profile edit: Form fill → Validation → API submission - 3 tests
- [ ] Navigation: Auth guard, tab switching, deep screen navigation - 5 tests

### Phase 3: E2E & Automation

#### E2E Tests (Detox - Target: 10 critical flows)
- [ ] Full login → home feed journey
- [ ] Create post with image
- [ ] Create pet profile
- [ ] Chat conversation flow
- [ ] Profile edit and save
- [ ] Friend request send/accept
- [ ] Service browsing
- [ ] Notification tap → navigation
- [ ] Logout flow
- [ ] Deep link handling

### Phase 4: CI Integration

Add to CI pipelines:
```yaml
# Add to .drone.yml and codemagic.yaml
- npm run test -- --coverage --ci
- npm run test:e2e  # Detox for E2E
```

Set coverage thresholds in `jest.config.js`:
```javascript
module.exports = {
  preset: 'react-native',
  coverageThreshold: {
    global: {
      branches: 60,
      functions: 60,
      lines: 60,
      statements: 60,
    },
  },
  collectCoverageFrom: [
    'src/**/*.{ts,tsx}',
    '!src/**/*.d.ts',
    '!src/**/types.ts',
    '!src/resources/**',
  ],
  setupFilesAfterSetup: ['./jest.setup.js'],
};
```

---

## Testing Infrastructure Gaps

| Gap | Impact |
|-----|--------|
| No `testID` props on components | Blocks E2E and component testing |
| No MSW or API mocking setup | Cannot test API integration |
| No test data factories | Tests will be brittle with inline data |
| No CI test execution | Tests don't gate deployments |
| No coverage reporting | Cannot track progress |
| No snapshot testing | UI regression risk |
| ESLint ignores test files | Test code quality ungoverned |

---

## Coverage Targets

| Milestone | Target Coverage | Focus |
|-----------|----------------|-------|
| Initial | 30% | Validation, API layer, session model |
| Core | 60% | All models, shared components, critical screens |
| Comprehensive | 80% | Full component coverage + E2E suite |
