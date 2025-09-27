# Actenda – App Store Privacy Disclosure Guide

## Executive Summary / TL;DR

**What to mark in App Store Connect:**
- ✅ **Purchases** → Collected, Linked to user (App Functionality)
- ✅ **Diagnostics** → Crash Data → Collected, Not linked to user (App Functionality, Analytics)
- ✅ **Diagnostics** → Performance Data → Collected, Not linked to user (App Functionality, Analytics)  
- ✅ **Usage Data** → Product Interaction → Collected, Not linked to user (Analytics)
- ❌ **All other categories** → Not collected (including Location, Contact Info, User Content, etc.)
- ❌ **Tracking** → No

**Why it's "Not linked to user" for diagnostics:**
- Privacy hardening implemented: `sendDefaultPii: false`, no `setUser()` calls, Replay masking enabled
- Sentry collects crash/performance data and user interaction breadcrumbs anonymously
- RevenueCat handles purchases (linked to App Store account as expected)

**Total: 4 data types collected, 1 linked to user, 3 not linked.**

---

This document maps Apple's App Store privacy categories to what the app currently collects, how it's used, whether it's linked to the user, and recommended changes to minimize disclosures. It's based on the current code in this repo.

Key sources in code:
- Sentry initialization: `App.tsx` (Sentry.init with `sendDefaultPii: false`, Replay with masking enabled, performance enabled)
- Crash reporting helper: `src/utils/sentry.ts` (no `Sentry.setUser` calls - removed for privacy)
- Crash provider: `src/components/CrashProvider.tsx` (calls `reportError` without `userId`)
- General Sentry usage: `src/contexts/AppContext.tsx`, `src/services/calendarService.ts` (breadcrumbs, setContext, spans)
- Purchases (RevenueCat): `src/services/subscriptionService.ts` (no custom `appUserID` in production)
- Calendar integration: `src/services/calendarService.ts` (device calendar as source of truth; no location data collected)

Important interpretation notes
- “Collected” means data is transmitted off device to you or a third party (e.g., Sentry, RevenueCat), not just stored locally.
- “Linked to the user” means data is associated with a user identity (e.g., email, account ID) or consistently linkable to them across sessions. Avoiding explicit identity (no `setUser`, no email) generally allows “Not linked” for diagnostics/analytics.
- This guide reflects the production app; dev-only behaviors (e.g., `appUserID` in `__DEV__`) don’t impact App Store disclosure.

## Summary (what we collect today)

- Diagnostics: Crash Data (Yes), Performance Data (Yes via `tracesSampleRate`), Other diagnostic (breadcrumbs/context) – collected.
- Usage Data: Product Interaction (Yes – breadcrumbs describing user actions).
- Purchases: Yes (via RevenueCat entitlements/receipts; Apple IAP backend). Payment info itself is not collected by the app.
- Identifiers: A service-specific anonymous user identifier exists in RevenueCat (RC anonymous ID) and Sentry may create its own event/device identifiers. We do not set our own `User ID` in production, and do not collect Device Advertising ID.
- Contact, Location, Health/Fitness, Financial Info (beyond purchases), Contacts, User Content (photos/videos/audio), Surroundings, Body data: Not collected by the app.

If we keep Sentry's `sendDefaultPii: false` and Replay with masking enabled, diagnostics data can be marked as "Not linked to the user" since we don't set any personal identifiers (no email, no `setUser`, no IP address, masked screen content).

## Apple categories – current vs. recommended

For each category: Current (what the code does now) → App Store Connect selection → Recommendation (to minimize disclosure risk/complexity).

### Contact Info
- Name: Not collected.
- Email Address: Not collected in-app. Note: Sentry Feedback integration is included in `Sentry.init` but not used in UI; if you ever surface a feedback form that collects email, this becomes “Collected.”
- Phone Number: Not collected.
- Physical Address: Not collected.
- Other User Contact Info: Not collected.

App Store: No for all above.

Recommendation: Keep it that way; avoid collecting emails in Sentry Feedback. If you add an in-app support form, disclose “Customer Support” and “Email Address.”

### Health & Fitness
- Health: Not collected.
- Fitness: Not collected.

App Store: No.

### Financial Info
- Payment Info: Not collected by the app. Apple handles payment, RevenueCat receives receipts from Apple; the app never sees card/bank info.
- Credit Info: Not collected.
- Other Financial Info: Not collected.

App Store: No.

### Location
- Precise Location: Not collected.
- Coarse Location: Not collected.

App Store: No.

### Sensitive Info
- Not collected.

App Store: No.

### Contacts
- Not collected.

App Store: No.

### User Content
- Emails or Text Messages: Not collected in-app. Users may email support externally; that’s outside the app’s collection scope.
- Photos or Videos: Not collected.
- Audio Data: Not collected.
- Gameplay Content / Other User Content: Not collected.
- Customer Support: Not collected in-app (no in-app form at present).

App Store: No for all.

Recommendation: If adding an in-app support form later, disclose “Customer Support” (Usage: App Functionality) and any contact fields you collect.

### Browsing History
- Not collected.

App Store: No.

### Search History
- Not collected.

App Store: No.

### Identifiers
- User ID: Not set by us in production. In dev, we may set `appUserID` for RevenueCat; in prod it’s undefined. RevenueCat creates an anonymous user ID; Sentry also assigns event/session identifiers. Apple typically treats service-specific app user IDs as “User ID.”
- Device ID: We do not collect advertising identifier (IDFA). Sentry does not collect IDFA by default. Device model/OS is diagnostic metadata, not a “Device ID.”

App Store (conservative):
- User ID: Yes (Linked to user: Yes). Purposes: App Functionality, Analytics. Recipients: RevenueCat (and Sentry for event-scoped IDs).
- Device ID: No.

Recommendation: Keep not setting our own `setUser` in Sentry. If you want to claim “Not Linked to You” for diagnostics, ensure you never associate identity (no email, no user IDs) and set `sendDefaultPii: false`.

### Purchases
- Collected via RevenueCat/StoreKit (entitlements, purchase state, expiration).

App Store: Yes (Linked to user: Yes). Purposes: App Functionality.

Notes: Even without a custom `appUserID`, the purchase history is associated with the user’s App Store account/RC anon ID, so Apple expects Purchases = Collected & Linked.

### Usage Data
- Product Interaction: Yes (Sentry breadcrumbs for user actions, e.g., toggling a target). Purpose: Analytics/App Functionality.
- Advertising Data: No.
- Other Usage Data: No.

App Store:
- Product Interaction: Yes. Linked to user: If you keep Sentry identity-free and disable default PII, select “Not Linked.” Otherwise, conservative is Linked = Yes.

Recommendation: Scrub breadcrumbs of any PII and set `sendDefaultPii: false` to confidently select “Not Linked.”

### Diagnostics
- Crash Data: Yes (Sentry).
- Performance Data: Yes (`tracesSampleRate: 0.05`).
- Other Diagnostic Data: Yes (Sentry breadcrumbs/context).

App Store:
- Diagnostics: Yes. Purposes: App Functionality, Analytics. Linked to user: Prefer “Not Linked” if identity-free; otherwise conservative is Linked = Yes.

Recommendation: Set `sendDefaultPii: false` and do not call `Sentry.setUser`; then select “Not Linked.”

### Surroundings (Environment Scanning)
- Not collected.

App Store: No.

### Body (Hands, Head)
- Not collected.

App Store: No.

### Other Data
- None beyond what’s detailed above.

App Store: No.

---

## Exact App Store Connect selections (current implementation)

With the privacy hardening now implemented:

- Data collection: Yes, we collect data.
- Purchases → Collected, Linked to user: Yes. Purpose: App Functionality.
- Diagnostics → Crash Data, Performance Data → Collected. Linked to user: No (Not linked to user). Purposes: App Functionality, Analytics.
- Usage Data → Product Interaction → Collected. Linked to user: No (Not linked to user). Purpose: Analytics.
- All other categories → Not collected.
- Tracking → No (we do not use data for cross-app tracking/ads).

## Preferred App Store Connect selections (after light hardening)

If you apply the recommendations below, you can simplify your label:

- In `App.tsx` Sentry.init:
  - Set `sendDefaultPii: false`
  - Consider removing `mobileReplayIntegration()` or set sample rate to 0
  - Keep `tracesSampleRate` as needed; if you set it to 0, you can drop “Performance Data”
- In `src/utils/sentry.ts`:
  - Remove or permanently disable `Sentry.setUser(...)`
- Ensure breadcrumbs contain no PII (names, emails, freeform user content)

Then disclose:
- Data collection: Yes, we collect data.
- Purchases → Collected, Linked to user: Yes. Purpose: App Functionality.
- Diagnostics → Crash Data (and Performance Data if traces enabled) → Collected, Not linked to user. Purposes: App Functionality, Analytics.
- Usage Data → Product Interaction → Collected, Not linked to user. Purpose: Analytics.
- All other categories → Not collected.
- Tracking → No.

## Implementation status (✅ completed)

**Privacy hardening has been implemented**:

1) **App.tsx – Sentry config**
   - ✅ Set `sendDefaultPii: false` (no IP address collection)
   - ✅ Enabled Replay masking: `maskAllText: true, maskAllImages: true` (no PII in screen recordings)
   - ✅ Added breadcrumb scrubbing to remove emails and long text
   - ✅ Kept `tracesSampleRate: 0.05` for performance monitoring

2) **src/utils/sentry.ts – identity**
   - ✅ Removed `Sentry.setUser(...)` call entirely
   - ✅ Added comment explaining privacy rationale

3) **Breadcrumb hygiene** 
   - ✅ Added `beforeBreadcrumb` filter to strip email patterns and long strings
   - ✅ High-level breadcrumbs only, no freeform user content

**Result**: Diagnostics and Usage Data can now be marked as "Not linked to user" in App Store Connect.

## Optional future enhancement

4) **In-app Privacy toggle** (not yet implemented)
   - Add a Settings switch: "Share anonymous crash reports." 
   - If disabled, don't init Sentry (or set sample rates to 0)
   - This would allow marking App Store Connect collection as "Optional"
```ts
Sentry.init({
  // ...
  sendDefaultPii: false,
  // Optional: disable Replay if not essential
  integrations: [
    // Remove mobileReplayIntegration or set sample rates to 0
  ],
  tracesSampleRate: 0.05, // set 0 to remove Performance Data collection
  beforeSend: (event) => { /* scrub extras, stack depth, etc. */ return event; }
});
```

2) src/utils/sentry.ts – identity
```ts
// Remove this block to avoid linking diagnostics to a user
// if (context?.userId) {
//   Sentry.setUser({ id: context.userId });
// }
```

3) Breadcrumb hygiene
- Keep breadcrumbs high-level; avoid including freeform user text or PII.

4) Optional: In-app Privacy toggle
- Add a Settings switch: “Share anonymous crash reports.” If disabled, don’t init Sentry (or set sample rates to 0). Then mark App Store Connect collection as “Optional.”

## Notes on third parties

- Sentry: Receives diagnostics (crash, performance, breadcrumbs). With `sendDefaultPii: false` and no `setUser`, data can be treated as Not Linked to the user.
- RevenueCat: Receives purchase/entitlement data (Linked to user). App does not receive payment card/bank data.
- Apple Calendar (EventKit): We create calendar events on-device; no off-device collection by the app. iCloud syncing is user-controlled at OS level.

## Health research study exception

Not applicable. The app does not collect health research data governed by IRB/ethics review.

---

**Status: Privacy hardening complete**. The app now collects diagnostics and usage data as "Not Linked to User," keeping Replay functionality while simplifying App Store privacy disclosures.
