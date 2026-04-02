# FurBaby Social - CI/CD & DevOps Assessment

**Date:** 2026-04-03

---

## Pipeline Overview

| Platform | CI System | Trigger | Target |
|----------|-----------|---------|--------|
| Android | Drone CI | `build/*` branches, `android-build` tag | S3 APK upload |
| iOS Dev | Codemagic | `dev` branch, `ios-build` tag | TestFlight |
| iOS Stage | Codemagic | `stage` branch | TestFlight |
| iOS Prod | Codemagic | `main`/`master` branch | TestFlight |

---

## Strengths

1. **Multi-environment support** - Dev/Stage/Prod with per-environment `.env` files, iOS schemes, and Android flavors
2. **Automated iOS builds** - Codemagic handles code signing, provisioning, and TestFlight upload automatically
3. **Build numbering** - iOS builds auto-increment from latest TestFlight build number
4. **Slack notifications** - Both pipelines notify on success/failure
5. **Commit linting** - Conventional commits enforced via commitlint + devmoji
6. **Pre-commit hooks** - Type checking + lint-staged via Husky
7. **Release notes** - Auto-generated from git changelog for iOS builds

---

## Gaps & Issues

### CRITICAL: No Test Execution in CI

Neither pipeline runs tests. There is no quality gate before deployment.

```yaml
# Missing from both .drone.yml and codemagic.yaml:
- npm run test -- --ci --coverage
```

**Impact:** Broken code can be deployed to TestFlight/S3 without any automated verification.

---

### HIGH: Inconsistent CI Systems

| Aspect | Android (Drone) | iOS (Codemagic) |
|--------|-----------------|-----------------|
| Build trigger | `build/*` branches | Branch-based (dev/stage/main) |
| Environment | Docker container | Mac Mini M1 |
| Artifact storage | AWS S3 | TestFlight |
| Config format | YAML (.drone.yml) | YAML (codemagic.yaml) |

Using two different CI systems increases maintenance burden and creates inconsistency. Drone appears simpler/less maintained than the Codemagic setup.

**Recommendation:** Consider consolidating to Codemagic for both platforms, or adding Android builds to Codemagic.

---

### HIGH: No Staging/Production Android Builds

The Drone pipeline only builds the `dev` flavor. There are no CI pipelines for Android Stage or Production builds.

```yaml
# .drone.yml only runs:
npm run android:build-dev
```

**Impact:** Stage and Production Android builds must be done manually.

---

### MEDIUM: No Static Analysis in CI

Missing from pipelines:
- ESLint execution
- TypeScript type checking (`npm run check-types`)
- Bundle size analysis
- Dependency vulnerability scanning (`npm audit`)

Pre-commit hooks catch some of this locally, but bypassing hooks is trivial (`--no-verify`).

---

### MEDIUM: Secrets Management

| Secret | Drone | Codemagic |
|--------|-------|-----------|
| AWS credentials | Drone secrets | N/A |
| Slack tokens | Drone secrets | Codemagic env vars |
| App Store Connect | N/A | Codemagic integration |
| Android keystore | Not in CI | Gradle properties (local) |
| Firebase config | Committed to repo | Committed to repo |

**Concerns:**
- Android signing keystore is not in CI - release builds require local machine access
- Firebase config files are in source control
- No rotation policy documented

---

### MEDIUM: Version Management Inconsistency

| Location | Version |
|----------|---------|
| package.json | 0.0.2 |
| Android build.gradle | versionName: 2.5, versionCode: 8 |
| iOS | Auto-incremented from TestFlight |

Three different version sources create confusion. No single source of truth.

**Recommendation:** Use `package.json` version as the source of truth. Derive iOS/Android versions from it during CI builds.

---

### LOW: npm install --force

Codemagic uses `npm install --force` which bypasses peer dependency checks:
```yaml
- npm install --force
```

This can mask breaking dependency conflicts.

---

## Recommended CI/CD Improvements

### Quick Wins
1. Add `npm run check-types` and `npm run lint` to both pipelines
2. Add `npm audit --audit-level=high` for vulnerability scanning
3. Add test execution step (once tests exist)
4. Fix version management - single source of truth

### Next Steps
5. Add Android stage/prod builds to CI
6. Add bundle size tracking and PR comments
7. Implement branch protection rules requiring CI pass
8. Add Dependabot or Renovate for dependency updates

### Longer Horizon
9. Consolidate to single CI platform
10. Add E2E test execution on staging builds
11. Implement deployment approval gates for production
12. Add performance regression testing (startup time, memory)
13. Implement semantic versioning with automated changelog

---

## Pipeline Maturity Score

| Dimension | Score (1-5) | Notes |
|-----------|-------------|-------|
| Build Automation | 4 | Multi-env builds work well |
| Test Automation | 1 | No tests in CI |
| Deploy Automation | 4 | TestFlight/S3 automated |
| Quality Gates | 2 | Pre-commit only, no CI gates |
| Monitoring/Alerts | 3 | Slack notifications, Crashlytics |
| Security Scanning | 1 | No vulnerability scanning |
| **Overall** | **2.5/5** | **Builds well, validates poorly** |
