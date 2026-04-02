# FurBaby Social - Security Assessment

**Date:** 2026-04-03

---

## Severity Summary

| Severity | Count |
|----------|-------|
| Critical | 1 |
| High | 3 |
| Medium | 4 |
| Low | 3 |
| Informational | 2 |

---

## Findings

### CRITICAL

#### SEC-01: Authentication Tokens Stored in Unencrypted Storage

**Location:** `src/shared/services/Effector/persist.ts`, `src/entities/session/model.ts`

**Description:** Access tokens and refresh tokens are persisted using `@react-native-async-storage/async-storage` via Effector's persistence layer with key prefix `furbaby:persist:`. AsyncStorage is an unencrypted key-value store backed by SQLite (Android) or plist files (iOS). On rooted/jailbroken devices, or via device backup extraction, tokens can be read in plaintext.

**Impact:** Complete account takeover if device is compromised or backup is accessed.

**Recommendation:** Migrate to `react-native-keychain` (iOS Keychain / Android Keystore) for all authentication credentials. Keep non-sensitive state in AsyncStorage.

```typescript
// Before (current)
createPersist(sessionModel.$tokens, 'tokens', sessionModel.authUser);

// After (recommended)
import * as Keychain from 'react-native-keychain';
// Store tokens in secure enclave
await Keychain.setGenericPassword('tokens', JSON.stringify(tokens));
```

---

### HIGH

#### SEC-02: WebSocket Connection Over Unencrypted Channel

**Location:** `env/.env.dev`, `env/.env.stage`, `env/.env.prod`

**Description:** All environments use `ws://` (unencrypted WebSocket) for the Centrifuge real-time connection:
```
WS_URL=ws://centrifugo.appelloproject.xyz:8000/connection/websocket
```
This transmits authentication tokens and real-time chat data in plaintext over the network.

**Impact:** Man-in-the-middle attacks can intercept tokens, chat messages, and user data on any network.

**Recommendation:** Upgrade to `wss://` (WebSocket Secure) with TLS. This requires server-side TLS certificate configuration on the Centrifugo instance.

---

#### SEC-03: Production Environment Uses Staging S3 Bucket

**Location:** `env/.env.prod`

**Description:**
```
IMAGE_AWS_BUCKET=furbaby-back-stage  # Should be furbaby-back-prod
```
Production is configured to use the staging S3 bucket for image storage. This means production user photos are stored in a staging environment.

**Impact:** Staging cleanup could delete production user data. Staging access policies may be more permissive than production.

**Recommendation:** Create and configure a dedicated `furbaby-back-prod` S3 bucket with appropriate IAM policies.

---

#### SEC-04: Firebase Credentials in Source Control

**Location:** `ios/GoogleService-Info.plist`

**Description:** Firebase configuration file containing API keys, project IDs, and GCM sender IDs is committed to the repository. While Firebase API keys are not secret by design (they're restricted by app bundle ID), best practice is to manage them outside source control.

**Impact:** Low direct risk (keys are scoped), but exposes project structure and identifiers.

**Recommendation:** Add to `.gitignore` and distribute via CI/CD secrets or a secure artifact system.

---

### MEDIUM

#### SEC-05: No Certificate Pinning

**Description:** The Axios HTTP client and Firebase SDK connections do not implement certificate pinning. The app trusts any CA-signed certificate, making it vulnerable to TLS interception on compromised networks.

**Recommendation:** Implement certificate pinning using `react-native-ssl-pinning` or TrustKit for critical API endpoints.

---

#### SEC-06: No Biometric/PIN Lock

**Description:** The app has no secondary authentication layer. Once tokens are present, the app grants full access without requiring biometric verification or PIN entry.

**Impact:** If a device is unlocked and accessed by another person, full account access is available.

**Recommendation:** Add optional biometric/PIN lock using `react-native-biometrics` or `expo-local-authentication`.

---

#### SEC-07: Cleartext Traffic Allowed (Android Debug)

**Location:** `android/app/src/debug/AndroidManifest.xml`

**Description:** `android:usesCleartextTraffic="true"` is set in the debug manifest. While limited to debug builds, this permits unencrypted HTTP connections.

**Impact:** Low - debug only. But ensure this flag is not present in release builds.

**Status:** Acceptable for development; verify release manifests.

---

#### SEC-08: No Request Rate Limiting (Client-Side)

**Description:** No debouncing or throttling on API calls for actions like login attempts, OTP sends, friend requests, or report submissions. Rate limiting should exist server-side, but client-side protection reduces abuse surface.

**Recommendation:** Add debouncing to sensitive actions (OTP send, login, report).

---

### LOW

#### SEC-09: Device Token Sent Over API

**Location:** `src/entities/user/model/deviceToken.model.ts`

**Description:** Firebase device tokens are sent to the backend API. Ensure the API endpoint requires authentication and tokens are stored securely server-side.

**Status:** Acceptable practice; verify backend handling.

---

#### SEC-10: No Jailbreak/Root Detection

**Description:** The app does not check for jailbroken iOS or rooted Android devices. Combined with SEC-01 (unencrypted token storage), this increases the attack surface.

**Recommendation:** Consider `jail-monkey` or similar libraries for detection, with appropriate user warnings.

---

#### SEC-11: Image URLs Predictable

**Description:** CloudFront image URLs follow a pattern based on S3 bucket names. While not directly exploitable (images are user-uploaded), enumeration may be possible.

**Status:** Low risk; consider signed URLs for private images.

---

### INFORMATIONAL

#### SEC-12: Privacy Manifest (iOS)
- `PrivacyInfo.xcprivacy` properly declares API usage categories
- No data collection declared (verify accuracy with actual data practices)
- Tracking disabled

#### SEC-13: Permissions Requested
- Camera, Microphone, Photo Library, Location, Push Notifications
- All have usage descriptions in Info.plist
- Permissions appear appropriate for app functionality
