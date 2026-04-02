# FurBaby Social - Accessibility & Internationalization Report

**Date:** 2026-04-03

---

## Accessibility (a11y) Assessment

### Current State: NOT IMPLEMENTED

| Check | Result |
|-------|--------|
| `accessibilityLabel` attributes | 0 found |
| `accessibilityRole` attributes | 0 found |
| `accessibilityHint` attributes | 0 found |
| `accessibilityState` attributes | 0 found |
| `aria-*` attributes | 0 found |
| `role` attributes | 0 found |
| `testID` attributes | 0 found |
| Screen reader tested | No evidence |
| Minimum touch targets (48x48dp) | Not verified |
| Color contrast compliance | Not verified |
| Focus management | Not implemented |

### Impact

- **Legal risk:** Apps in many jurisdictions must meet WCAG 2.1 AA standards. Australia has the Disability Discrimination Act (DDA) which could apply.
- **User exclusion:** Visually impaired users cannot use any part of the app.
- **App Store risk:** Both Apple and Google are increasing accessibility review requirements.

### Priority Remediation

#### Phase 1: Critical Interactive Elements
Every touchable element needs at minimum:
```tsx
<TouchableOpacity
  accessibilityRole="button"
  accessibilityLabel="Like post"
  accessibilityState={{ selected: isLiked }}
>
```

**Components requiring immediate attention:**
1. All Button variants (`Button.tsx` - 173 lines, used everywhere)
2. Tab bar items (CustomTabBar)
3. Form inputs (Input, Select, PhoneInput, DatePicker, Checkbox, Radio)
4. Navigation headers and back buttons
5. Post interaction buttons (like, comment, share)
6. Chat message input and send button
7. Image viewers and galleries

#### Phase 2: Screen-Level
- Add heading hierarchy for screen readers
- Implement focus management on navigation transitions
- Add live regions for dynamic content (chat messages, notifications)
- Ensure modal/bottom sheet focus trapping

#### Phase 3: Verification
- Test with VoiceOver (iOS) and TalkBack (Android)
- Run automated a11y audit (react-native-a11y or axe-core)
- Verify color contrast ratios meet WCAG AA (4.5:1 for text)

---

## Internationalization (i18n) Assessment

### Current State: PARTIALLY IMPLEMENTED

| Check | Result |
|-------|--------|
| i18n framework installed | Yes - i18next + react-i18next |
| Translation files | Present (exact count TBD) |
| Locale detection | expo-localization |
| RTL support | Declared in AndroidManifest (`supportsRtl="true"`) |
| Date localization | dayjs (configurable) |
| Phone number localization | libphonenumber-js (AU default) |
| Currency formatting | Not found |

### Strengths
- i18next framework properly integrated
- expo-localization for device locale detection
- dayjs handles date formatting per locale
- Phone input supports international format via libphonenumber-js

### Concerns
1. **Hardcoded AU assumption** - Phone validation defaults to AU country code
2. **Email contact hardcoded** - `info@furbabysocial.com.au` in constants
3. **Date format hardcoded** - `DD.MM.YYYY` may not suit all locales
4. **Unknown translation coverage** - Need to audit for untranslated strings
5. **RTL layout testing** - Declared support but likely untested
6. **No pluralization verification** - i18next supports it but usage unclear

### Recommendations
1. Audit all user-visible strings for i18n key usage vs hardcoded text
2. Test RTL layouts if targeting Middle Eastern or Hebrew-speaking markets
3. Make date/time formats locale-aware rather than hardcoded
4. Add locale-specific validation rules (not just AU phone format)
