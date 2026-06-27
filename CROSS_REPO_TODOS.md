# Cross-Repo TODOs

Shared handoff log for all SaviPets repos.

---

## PENDING

### [2026-06-25] Firestore indexes required for AI sitter scoring
- Triggered by: src/services/bookings/BookingAssignmentService.ts — selectBestSitter()
- Source repo: savipets-web-admin
- Target repo: savipets-backend
- File: firestore.indexes.json
- Change: Add four composite indexes required by selectBestSitter queries:
  1. publicProfiles: (bizId ASC, role ASC)
  2. serviceBookings: (sitterId ASC, bizId ASC)  — availability check
  3. serviceBookings: (sitterId ASC, bizId ASC, status ASC, petIds ARRAY)  — pet history + breed experience
  4. serviceBookings: (sitterId ASC, bizId ASC, status ASC, serviceType ASC)  — service type experience
  Without these, Firestore will throw "query requires an index" errors at runtime.
- Deploy command: firebase deploy --only firestore:indexes
- Status: PENDING

### [2026-06-26] iOS BUG — Owner schedule shows 0 bookings; Smart Insights never appear
- Triggered by: bug report (screenshots, owner account)
- Source repo: savipets-backend (diagnosis)
- Target repo: savipets-ios
- Priority: HIGH — functional regression, core owner flow broken
- Status: DONE — fixed 2026-06-26; Combine subscription added to PetOwnerScheduleViewModel.init()
- Root cause: `PetOwnerScheduleViewModel` holds `ServiceBookingDataService.shared` as a plain
  `let` property. Firestore snapshots update `userBookings` on the service, but the ViewModel
  never publishes a change — no `objectWillChange.send()`, no `@Published` mirror.
  SwiftUI re-renders only when the ViewModel's own `@Published` properties fire.
  Two symptoms from one cause:
  1. Calendar and booking list stay empty — computed props (`bookingsInMonth`,
     `enhancedFilteredBookings`) read `serviceBookings.userBookings` but re-render never fires.
  2. Smart Insights stay empty — `generateBookingInsights()` runs in `.task` before
     bookings arrive, never re-runs when data loads.
- Fix: `SaviPets/Dashboards/Owner/Schedule/ViewModels/PetOwnerScheduleViewModel.swift`
  Change:
  ```swift
  // existing:
  init() {}
  ```
  To:
  ```swift
  private var cancellables = Set<AnyCancellable>()

  init() {
      serviceBookings.$userBookings
          .receive(on: DispatchQueue.main)
          .dropFirst()
          .sink { [weak self] _ in
              self?.objectWillChange.send()
              self?.generateBookingInsights()
          }
          .store(in: &cancellables)
  }
  ```
  `Combine` is already imported. `dropFirst()` skips the initial empty `[]` on subscription.
  Each subsequent update triggers view re-render AND regenerates Smart Insights.
- Deploy: No backend deploy needed. iOS-only client fix.
- Verification: Log in as an owner with existing bookings → tap Schedule tab →
  calendar shows dots and booking count > 0 without needing to switch tabs.
  Smart Insights ("Tomorrow's Visits", "Pending Approvals") appear automatically.

### [2026-06-25] CRITICAL — Android: migrate sitter data lookups from users/ to publicProfiles/
- Triggered by: firestore.rules commit 67bc1b4 (removed petSitter read exception from users/{uid})
- Source repo: savipets-backend
- Target repo: savipets-android
- Priority: CRITICAL — owners on Android see "User {uid}" instead of sitter name/photo on every booking card and schedule tab. Actively broken in production.
- Status: PENDING
- Root cause: Android still reads `users/{sitterId}` directly for sitter display data.
  `isOwner(sitterUid)` = false for petOwner, not admin → PERMISSION_DENIED.
  iOS fixed in commit d62aeb6e5. Android has 4 remaining call sites:
  1. `SitterDataFetcher` — queries `users/` for sitter list
  2. `BookingDetailViewModel` — reads `users/{sitterId}` for booking detail view
  3. `ChatViewModelDisplayName` (×2 call sites) — reads `users/{uid}` for display name
  4. `RecurringBookingService` — reads `users/{sitterId}` for sitter metadata
- Fix: Replace all 4 with `db.collection("publicProfiles").document(sitterId).get()`.
  For the sitter list query: `db.collection("publicProfiles").whereEqualTo("bizId", "savipets-default").whereEqualTo("role", "petSitter")` — both constraints required for the list rule.
- Reference: iOS commit d62aeb6e5 — complete working implementation.
- Deploy: No deploy needed; client-side change only.

### [2026-06-25] NOTIFICATIONS: Backend must send FCM push on chat messages
- Status: COMPLETE (2026-06-26)
- Fix: Rewrote `functions/src/features/chat/triggers/onChatMessageCreated.ts`. Added `pushToRecipients()` that reads `conversations/{id}.participants`, filters out sender, fetches each recipient's `fcmToken` from `users/{uid}`, sends direct `admin.messaging().send()`. Web admin silently skipped (no token). Fetches `senderDoc` + `conversationDoc` in parallel. Admin replies push to all but don't increment `unreadCount`.
- Deploy: `firebase deploy --only functions:onChatMessageCreated`

### [2026-06-25] NOTIFICATIONS: Fix wasSent() query — missing Firestore composite index
- Status: COMPLETE (2026-06-26)
- Fix: Index was already present in `firestore.indexes.json` (4-field composite on notificationLogs: visitId + ownerId + eventType + status). Confirmed via grep. Just needs: `firebase deploy --only firestore:indexes`.
- Note: Field names differ from the TODO description (ownerId, not recipientId) — confirmed against the actual `wasSent()` implementation.

### [2026-06-25] NOTIFICATIONS: Implement mid-visit check-in push notifications
- Status: COMPLETE (2026-06-26)
- Fix: Created `functions/src/features/visits/triggers/midVisitCheckIn.ts` — `onSchedule` every 15 min. Queries `visits` where `status == "on_adventure"`, checks elapsed time since `timeline.checkIn.timestamp`, sends direct FCM push at 30-min and 60-min milestones. Idempotent via `notificationLogs`. Exported from `functions/src/index.ts`.
- Deploy: `firebase deploy --only functions:midVisitCheckIn`

### [2026-06-25] LOW — Role alias cleanup (target: 2 weeks post-deploy)
- Source repo: savipets-backend
- Target repo: savipets-backend
- Priority: LOW
- Status: PENDING
- Description: Clean up legacy 'sitter' role alias after migration is complete.
  - Query: `db.collection('users').where('role', '==', 'sitter').get()`
  - Metric (2026-06-25): 0 legacy sitter docs remaining.
  - When count = 0:
    - Remove 'sitter' from `hasAnyRole` in `firestore.rules` pets
    - Remove 'sitter' from `.where('role', 'in', [...])` arrays in `profileCompletionNotifications.ts`
    - Keep `setPlatformClaims` normalization permanently

### [2026-06-23] REGRESSION: Chat name resolution broken for non-sitter UIDs
- Source repo: savipets-backend
- Target repo: savipets-ios, savipets-android
- Priority: HIGH — mobile `MessageDisplayNameService.swift` and `ChatViewModelDisplayName.kt` use `publicProfiles/{uid}` to resolve names for ALL conversation participants (owners, admins, sitters). After publicProfiles became sitter-only, non-sitter UIDs have no publicProfile doc. Result: owners in support chat show as "User {uid}" to sitters/admins on mobile.
- Status: COMPLETE (2026-06-25). iOS writes participantNames + seeds name cache (MessageListenerManager.swift:73). Android reads participantNames (MessageListenerManager.kt). Web-admin now reads participantNames in all 4 service mapping sites and resolves names in getConversationTitle() with UID stub fallback.
- Files changed (Android):
  - `app/src/main/java/com/savipets/app/domain/models/chat/Conversation.kt` — added `participantNames: Map<String,String>` field, parse in `fromDocument()`
  - `app/src/main/java/com/savipets/app/data/firebase/chat/MessageListenerManager.kt` — seed `userNameCache` from `participantNames` on conversation snapshot (both regular + admin listeners), before `ensureDisplayNameListener` fires
- Files changed (web-admin):
  - `src/types/chat.ts` — added `participantNames?: Record<string, string>` to Conversation type
  - `chatHelpers.ts` — `getConversationTitle()` resolves via `participantNames[uid]` first, then user-lookup fallback, then `User {uid.slice(0,6)}`
  - `ConversationService.ts` (×2), `SupportInquiryService.ts` (×2), `chat.service.ts` (×1) — all 5 mapper sites pass `participantNames` through from Firestore doc
  - Note: `acceptInquiry` path doesn't write `participantNames`; falls back to user-lookup (admin has full users/ access)
- Chosen fix: Option B — `participantNames` map written at conversation creation (iOS), read on all clients. COMPLETE across iOS + Android + web-admin.


### [2026-06-19] Chat: allow sitters to read pet names from pets collection
- Source repo: savipets-ios
- Target repo: savipets-backend
- Priority: HIGH
- Status: COMPLETE (2026-06-25)
- Root cause: `'sitter'` was a legacy role alias for `'petSitter'`. `adminUsers.controller.ts` Zod schema accepted `'sitter'` but `validateRole` downstream rejected it — so no new `'sitter'` users can be created, but historical Firestore docs carry `role:'sitter'` and `setPlatformClaims` stamped them into JWT unchanged. Every code path that only checked `'petSitter'` silently broke for these users.
- Fixes applied:
  - `setPlatformClaims.ts` — normalizes `'sitter'` → `'petSitter'` before writing JWT AND writes canonical value back to Firestore (lazy migration on next login). `VALID_ROLES` array no longer includes `'sitter'`.
  - `bookings.controller.ts:59` — sitter role check now covers both values until Firestore docs converge.
  - `profileCompletionNotifications.ts:53,65` — both Firestore queries now include `'sitter'` in the `in` filter.
  - `adminUsers.controller.ts:12` — removed `'sitter'` from Zod enum (was dead code; `validateRole` already rejected it).
  - `firestore.rules` pets block — `hasAnyRole(['petSitter', 'sitter'])` as safety net during 60-min token TTL window.
  - `PLATFORM_ARCHITECTURE.md` — canonical role set updated; `'sitter'` documented as deprecated alias.

### [2026-06-22] Enable Firestore PITR + scheduled exports
- Source repo: savipets-backend
- Target repo: savipets-backend
- Priority: HIGH (BLOCKER before onboarding first business)
- Status: COMPLETE (2026-06-23 — Firebase Console. 7-day PITR + 98-day daily snapshots enabled.)
- Task: Enable Point-in-time recovery and set up scheduled exports to GCS bucket (daily export, 30-day retention).

### [2026-06-22] Create DISASTER_RECOVERY.md at workspace root
- Source repo: n/a
- Target repo: savipets-backend / workspace root
- Priority: MEDIUM (Should Fix before first external business)
- Status: COMPLETE (2026-06-23 — DISASTER_RECOVERY.md authored in savipets-backend/, commit f716544. 6 failure scenarios covered.)
- Task: Author DR runbook covering 4 scenarios:
  1. Firebase outage — what degrades, what still works, customer communication template
  2. Square outage — fallback (manual invoice, offline payment), how to resume
  3. Admin account compromised — revoke access steps, audit log review, password reset procedure
  4. Database restore — how to restore from PITR, steps to verify data integrity after restore

### [2026-06-22] Verify Square payment flow end-to-end (manual test)
- Source repo: savipets-ios, savipets-web-admin
- Target repo: savipets-backend
- Priority: HIGH (BLOCKER before charging real money)
- Status: PLAYBOOK CREATED & BACKEND WEBHOOK GAP FIXED (Manual verification pending user run)
- Task: End-to-end test of booking service on iOS, paying with Square test card, generating invoice, and refunding from web admin. Confirm auditLog entries for invoice_created, invoice_paid, refund_processed. Playbook available in [square_payment_verification_guide.md](file:///Users/kimo/.gemini/antigravity-cli/brain/32df5073-3b04-4f05-999f-ce31ab92f5e4/square_payment_verification_guide.md).

### [2026-06-22] Set up monitoring alerts for 4 critical failure categories
- Source repo: n/a
- Target repo: savipets-backend
- Priority: MEDIUM (Should Fix before first external business)
- Status: COMPLETE (2026-06-23 — 4 alert policies live: error rate, p99, Firestore reads, auth failures. Duplicate entry exists in COMPLETED section.)
- Task: Configure Cloud Monitoring for:
  1. Booking creation failures (Cloud Function errors on createAdminBooking)
  2. Payment failures (Square webhook errors or invoice status stuck)
  3. Login failures > threshold (account lockout trigger anomalies)
  4. Cloud Function failures > threshold (any function error rate spike)
  Minimum viable: email alert to admin@savipets.com when any CF error rate > 5% in 5 minutes.

### [2026-06-24] MULTI-TENANT: Create businesses/{bizId} collection + rules for BusinessContext
- Source repo: savipets-web-admin
- Target repo: savipets-backend
- Priority: MEDIUM
- Status: COMPLETE (rule deployed 2026-06-20 in commit ca78b2f)
- Rule at firestore.rules:966 — `allow get: if request.auth.token.bizId == bizId || isAdmin()`
- Note: spec in this TODO differed from what was actually implemented; actual rule is correct.

### [2026-06-25] GPS consent flow missing on Android — visitTracking writes now blocked
- Triggered by: firestore.rules GPS-CONSENT gate (deployed this session)
- Source repo: savipets-backend
- Target repo: savipets-android
- Status: COMPLETE (2026-06-25)
- Files changed:
  - NEW: `app/src/main/java/com/savipets/app/features/sitter/consent/GPSConsentService.kt`
  - MOD: `features/visit/ui/VisitCardViewModel.kt` — consent gate in `startVisit()`; `grantGpsConsent()` / `denyGpsConsent()`
  - MOD: `features/visit/ui/VisitCard.kt` — GPS consent AlertDialog
- Notes: consentVersion "1.0" matches iOS. Deny path: visit starts but GPS skipped (Firestore rule blocks visitTracking). Runtime verification pending device test.

### [2026-06-25] Phase 7 P3 verification - Android code complete, runtime TBD
- Source repo: savipets-android
- Target repo: n/a (tracking item)
- Priority: MEDIUM
- Status: CODE COMPLETE (commits `9e313c6` + `39fa994`), RUNTIME PENDING (device/Firestore spot-check)
- Remaining: Spot-check new booking bizId in Firestore; verify reschedule and time-off doc creation from device; verify invoice FCM push.

### [2026-06-26] Add Firestore rules for scheduling collections
- Triggered by: Scheduling hub refactor (savipets-web-admin) — ShiftsView/TimeOffView/FeedingMedicationLog
  removed from render tree; their Firestore collections still lack security rules.
- Source repo: savipets-web-admin
- Target repo: savipets-backend
- File: firestore.rules
- Change: Add admin-only read/write rules for:
    staffShifts, timeOffRequests, blackoutPeriods,
    feedingMedicationLogs, staffBlockedTime
    (filter by bizId, isAdmin() guard)
  Also verify changeRequests compound index in firestore.indexes.json:
    (bizId ASC, status ASC, createdAt DESC)
- Deploy command: firebase deploy --only firestore:rules,firestore:indexes
- Status: PENDING

### [2026-06-26] SCHEMA — serviceType (machine key) + serviceName (display snapshot) split
- Triggered by: bug report (owner dashboard screenshot — "dog-walking" card, no service name).
  Root cause is schema conflation, NOT a render bug. SUPERSEDES the earlier "humanize the slug"
  framing — do not store display strings in `serviceType`.
- Source repo: savipets-web-admin (design + web-admin implementation DONE this session)
- Architecture doc (source of truth): savipets-web-admin/docs/architecture/SERVICE_NAMING.md
- Priority: HIGH — schema contract shared by all clients; analytics + receipts depend on it
- Design (load-bearing — read SERVICE_NAMING.md before implementing):
  - `serviceType` = machine key (slug, e.g. `dog-walking`). Stable forever. Drives logic,
    analytics grouping, sitter matching, reporting. NEVER a display string.
  - `serviceName` = display label SNAPSHOT (e.g. `"Potty Break - 15 min"`), frozen at booking
    creation by copying the catalog `Service.name`. Tenant may rename the catalog later; existing
    bookings keep their original name (a booking is a receipt). Analytics never groups on it.
  - Read contract: display `serviceName ?? humanize(serviceType)`. No migration needed — the
    fallback handles all legacy docs.
  - Write contract: every client writes BOTH fields on booking creation.
- Three existing doc classes (handled by the read fallback, no migration required for display):
  - Class 0 (new): has `serviceName`.
  - Class 1 (web-admin legacy): slug in `serviceType`, no `serviceName`.
  - Class 2 (iOS legacy): human LABEL wrongly in `serviceType` (`BookingModels.swift:96`,
    `BookingCRUDService.swift:43`), no `serviceName`. Dirty analytics — needs the backfill below.

- web-admin: DONE this session.
  - Types: `serviceName` added to Booking, FirestoreBookingDocument, FirestoreRecurringSeriesDocument,
    AdminBookingCreate, AdminRecurringBookingCreate.
  - Write: `BasicBookingFields.handleServiceChange` snapshots `match?.name || humanizeSlug(slug)` into
    a hidden form field; `useBookingFormSubmission` + `AdminBookingService` write `serviceName` to
    booking docs, recurring series, recurring bookings, and `visits/{id}`.
  - Read: new `resolveServiceName(booking)` helper (bookingHelpers.ts) drives every display site
    (table, drawer, daysheet, calendar, list, timeline, my-schedule, invoices summary, invoice modal).
  - Receipts: `CreateInvoiceModal` line-item description + booking-picker now use `resolveServiceName`.
  - Also (prior session): `createAdminBooking` writes `scheduledTime` ("h:mm a") — fixes missing time.

#### iOS — PENDING (Target: savipets-ios)
  1. WRITE — store the catalog SLUG in `serviceType` (not the human label) AND snapshot the catalog
     name into a new `serviceName` field at booking creation (`BookingCRUDService.swift:43`,
     catalog `BookingModels.swift:96`).
  2. READ — replace raw `Text(booking.serviceType)` (`BookingCardInline.swift:16`, and any other
     raw print sites) with `serviceName ?? humanize(serviceType)`. Idempotent resolver: pass through
     values that already look like labels (space/mixed case), humanize pure slugs (`^[a-z0-9-]+$`).
     Mirror web-admin `humanizeSlug()` (serviceDisplayUtils.ts:23).
  - NOT a bug (do not "fix"): pets. Web-admin stores pet NAMES (extractPetNames), iOS renders them as
    avatar initials. Empty avatars in the repro = admin didn't select pets.
  - Deploy: iOS-only client change.

#### Android — PENDING (Target: savipets-android)
  - Same dual-write (slug → `serviceType`, catalog name → `serviceName`) at booking creation
    (`FirebaseBookingRepository.createBooking`, `RecurringSeriesRepositoryImpl`).
  - Same resolver-read (`serviceName ?? humanize(serviceType)`) at every booking display site.
  - Deploy: Android-only client change.

#### Backend — PENDING (Target: savipets-backend)
  1. Document the split in `PLATFORM_ARCHITECTURE.md` + `.claude/rules/contracts.md` (serviceBookings,
     visits). Add a field-level rule for `serviceName` (immutable after create — it's a frozen snapshot).
  2. One-time backfill CF for Class-2 docs: for `serviceBookings` (and `visits`) where `serviceType`
     is NOT a slug (`^[a-z0-9-]+$`): set `serviceName = serviceType` (preserve the existing label as the
     frozen receipt), then set `serviceType = slugify(label)` mapped via the catalog. Idempotent; skip
     docs that already have `serviceName`. Dirty analytics is a sales problem — this is required, not
     optional.
  3. Square/invoice receipts: any backend line-item description built from `serviceType` must read
     `serviceName ?? humanize(serviceType)`. Mirrors the web-admin invoice fix.
  - Deploy: `firebase deploy --only functions:backfillServiceNames` (after provisioning), then rules.

- Verification: as an owner, view a web-admin-created booking → friendly service name + time render.
  iOS/Android-created bookings still render correctly (resolver pass-through). Analytics groups by slug
  with no fragmentation after backfill.

---

## COMPLETED

### [2026-06-25] Missing fonts not bundled in iOS app binary
- Target: `savipets-ios`
- Status: COMPLETE (2026-06-25)
- Details: Replaced DMSans.ttf with individual weight files (DMSans-Regular.ttf, DMSans-Medium.ttf, DMSans-SemiBold.ttf) under `SaviPets/Resources/Fonts/`, registered them in UIAppFonts of Info.plist, and updated DesignTypography.swift references to use correct PostScript names (DMSans9pt-Regular, DMSans9pt-Medium, DMSans9pt-SemiBold).

### [2026-06-25] MULTI-TENANT: Create businesses/savipets-default seed doc
- Target: `savipets-backend` (Firestore DB)
- Status: COMPLETE (2026-06-25)
- Details: Created Firestore document `businesses/savipets-default` with branding config (primaryColor: "#2D8B73") and required features.

### [2026-06-22] Verify conversations and notifications don't store legacy clientId
- Target: `savipets-backend`
- Status: COMPLETE (2026-06-25 audit)
- Result: Zero `clients/` references in support/chat code. All lookups use `users/{uid}`. Phase 7 pre-flight Check 3 already confirmed 17 conversation docs, 0 clientId fields, all participants are Auth UIDs. No code or data changes needed.

### [2026-06-23] GDPR-DEL: Mobile apps account self-deletion UI
- Target: `savipets-ios`, `savipets-android`
- Status: COMPLETE
- iOS: `SaviPets/Services/Auth/FirebaseAuthService.swift:150-157` (via `deleteAccountViaCF()`)
- Android: `app/src/main/java/com/savipets/app/features/owner/OwnerProfileViewModel.kt` (calls `deleteUserData` callable)

### [2026-06-23] SEC-CLAIMS: userMetadata/{uid} subscription for forced token refresh
- Target: `savipets-ios`, `savipets-android`
- Status: COMPLETE
- iOS: `SaviPets/Services/Auth/TokenRefreshManager.swift:1-100` (snapshot listener on `userMetadata/{uid}`)
- Android: `app/src/main/java/com/savipets/app/data/auth/TokenRefreshManagerImpl.kt:1-100` (snapshot listener on `userMetadata/{uid}`)
- Backend Trigger: `functions/src/features/users/triggers/setPlatformClaims.ts:88` (writes `claimsUpdatedAt`)

### [2026-06-22] IMPL-01: adminCloseVisit.ts wrong wrapper
- Target: `savipets-backend`
- Status: COMPLETE
- File/Line: `functions/src/features/visits/api/adminCloseVisit.ts:31` (replaced `withSafeTriggerHandling` with `withErrorHandling`; updated role check)

### [2026-06-22] BOOK-02: booking status field client write block
- Target: `savipets-backend`
- Status: COMPLETE
- File/Line: `firestore.rules:253-255` (added status block to immutable-fields list for client updates)

### [2026-06-22] GPS TTL / CCPA — visitTracking + locations TTL
- Target: `savipets-backend`
- Status: COMPLETE
- File/Line: `functions/src/features/visits/triggers/backfillVisitTrackingBookingId.ts:68-72` (adds `expireAt` field; backfill script ran)

### [2026-06-22] GDPR export endpoint — exportUserData
- Target: `savipets-backend`
- Status: COMPLETE
- File/Line: `functions/src/features/users/api/exportUserData.ts:1-100` (onCall CF)

### [2026-06-22] Client/User type merge in web-admin
- Target: `savipets-web-admin`
- Status: COMPLETE
- File/Line: Git History commits `bb01123` & `a867659` (Client type deleted; User type canonical; 80+ file imports resolved)

### [2026-06-22] Cloud Monitoring alerts setup
- Target: `savipets-backend` (Ops Console)
- Status: COMPLETE
- Details: 3 alert policies configured for auth failure spikes, Square webhooks, and function errors.

### [2026-06-22] Backfill bizId on users collection
- Target: `savipets-backend`
- Status: COMPLETE
- File/Line: `functions/src/features/users/createOwnerProfile.ts` and `functions/src/features/migrations/backfillBizId.ts` (15 users and 5 lastNames patched)

### [2026-06-21] Phase 0 — Freeze new clients writes in web-admin
- Target: `savipets-web-admin`
- Status: COMPLETE
- File/Line: `src/pages/Clients.tsx` (New Client button disabled; tooltips added)

### [2026-06-21] Phase 2 — Fix web-admin to query users instead of clients
- Target: `savipets-web-admin`
- Status: COMPLETE
- File/Line: `src/services/clients/ClientService.ts` and booking wizard (swapped queries to `users` collection)

### [2026-06-22] SEC-4 regression — missing users index
- Target: `savipets-backend`
- Status: COMPLETE
- File/Line: `firestore.indexes.json` (added users composite index bizId+role+lastName)

### [2026-06-22] SEC-4 regression — serviceBookings create rule blocks admin
- Target: `savipets-backend`
- Status: COMPLETE
- File/Line: `firestore.rules:253-255` (moved sitterId check inside non-admin branch)

### [2026-06-18] TTP Phase 0 — Backend CFs + rules + indexes for clients
- Target: `savipets-backend`
- Status: COMPLETE
- File/Line: `functions/src/features/clients/adminClients.ts` (createClient/updateClient/deactivateClient deployed)

### [2026-06-21] SECURITY: Sitter discovery must read publicProfiles
- Target: `savipets-ios`, `savipets-android`, `savipets-backend`
- Status: COMPLETE
- iOS: `SitterDataService.swift`, `GuestBrowsingViewModel.swift`, `MessageDisplayNameService.swift`
- Android: `GuestBrowsingViewModel.kt`, `ChatViewModelDisplayName.kt`
- Backend Rule: `firestore.rules` (users read rule tightened; publicProfiles read rule added)

### [2026-06-23] BLOCKER: publicProfiles backfill
- Target: `savipets-backend`
- Status: COMPLETE
- Details: Sitter/owner/admin public profiles backfilled (67 orphaned non-sitter docs deleted, 3 sitter docs kept)

### [2026-06-18] CRITICAL SECURITY: OAuth sign-in bypasses checks
- Target: `savipets-ios`, `savipets-backend`
- Status: COMPLETE
- iOS: `Services/Auth/AuthenticationFlowService.swift` and OAuth sign-in handlers updated to route through processSuccessfulAuthentication
- Backend CF: `functions/src/features/users/triggers/onUserSignup.ts` (OAuth sign-up guard)

### [2026-06-17] Backend P-08 first-time-client discount deployment
- Target: `savipets-backend`
- Status: COMPLETE
- File/Line: `functions/src/features/payments/square/handlers/invoice.ts` (first-time discount server check deployed)

### [2026-06-17] P0 SECURITY: Cross-user data leak + auth hardening
- Target: `savipets-ios`
- Status: COMPLETE
- File/Line: `SaviPets/Services/ServiceBookingDataService.swift:71` (clears stale lists before listener starts; SignOutHelper resets singletons; Google sign-out session cleanup)

### [2026-06-17] Visit notifications — iOS foreground push + sitter bell badge
- Target: `savipets-ios`
- Status: COMPLETE
- File/Line: `SaviPets/SaviPetsApp.swift:23` (UNUserNotificationCenterDelegate added) and `SitterScheduleTab.swift:15` (bell overlay badge)

### [2026-06-17] P0 DATA INTEGRITY: iOS approveBooking() clientId guard
- Target: `savipets-ios`
- Status: COMPLETE
- File/Line: `SaviPets/Features/Booking/Services/BookingCRUDService.swift:147` (guards empty clientId with Auth current UID fallback)

### [2026-06-17] P0 DATA INTEGRITY: Backend onServiceBookingWrite clientId check
- Target: `savipets-backend`
- Status: COMPLETE
- File/Line: `functions/src/features/bookings/triggers/onServiceBookingWrite.ts:131` (logger.error + return null on empty clientId)

### [2026-06-17] P0 DATA INTEGRITY: One-time backfill for visits with clientId:''
- Target: `savipets-backend`
- Status: COMPLETE
- Details: Backfill script `backfillVisitClientIds.ts` ran (65 scanned, 7 skipped, 58 okay)

### [2026-06-18] TTP Phase 1 — Services catalog backend CFs + rules
- Target: `savipets-backend`
- Status: COMPLETE
- File/Line: `functions/src/features/services/adminServices.ts` (createService/updateService/archiveService/reorderServices)

### [2026-06-19] TTP Phase 6 — Client Portal backend CF + rules
- Target: `savipets-backend`
- Status: COMPLETE
- File/Line: `functions/src/features/clients/adminClients.ts` (`createClientPortalUser` CF + rules for invoices/credits read)

### [2026-06-22] React Hooks violations pages crash fix
- Target: `savipets-web-admin`
- Status: COMPLETE
- Details: Verified hooks are declared above early returns in all 5 pages.

### [2026-06-22] Wire ClientDetailDrawer portal-enable to createOwnerProfile CF
- Target: `savipets-web-admin`
- Status: COMPLETE
- File/Line: `src/features/clients/components/ClientDetailDrawer.tsx` (onClick now passes full client payload to createOwnerProfile)

### [2026-06-22] SEC-4 rules — deploy privilege escalation rules
- Target: `savipets-backend`
- Status: COMPLETE
- File/Line: `firestore.rules:78-82` (privilege rules deployed)

### [2026-06-22] Fix setPlatformClaims.ts resurrection bug
- Target: `savipets-backend`
- Status: COMPLETE
- File/Line: `functions/src/features/users/triggers/setPlatformClaims.ts` (strips legacy clientId claim before writing)

### [2026-06-22] crm_tags delete rule
- Target: `savipets-backend`
- Status: COMPLETE
- File/Line: `firestore.rules` (removed `|| request.auth.uid == userId` from crm_tags write permission)

### [2026-06-22] serviceFeedback clientId audit
- Target: `savipets-backend` (Audit only)
- Status: COMPLETE
- Details: Verified all 17 docs had valid UIDs; 0 were legacy client IDs.

### [2026-06-22] Audit legacy CF admin boolean checks
- Target: `savipets-backend`
- Status: COMPLETE
- File/Line: `functions/src/features/users/triggers/accountLockout.ts` (removed legacy boolean checks)

### [2026-06-25] Phase 7 backend CFs missing
- Target: `savipets-backend`
- Status: COMPLETE
- Details: Created triggers `onChangeRequestCreated.ts` and `onTimeOffRequestCreated.ts` to notify admins when reschedules or time-off are submitted from mobile devices.

### [2026-06-25] BUG-1: Stale pet names in booking detail
- Triggered by: iOS `4035386fe` session analysis — BookingParser.swift:25 + BookingDetailInfoCard.swift:163
- Source repo: savipets-ios
- Target repo: savipets-backend
- Change: Add Firestore trigger on `pets/{petId}` writes. When `name` changes, query `serviceBookings` where `clientId == pets/{petId}.ownerId` AND `status in [pending, scheduled, approved]` AND `pets` array-contains the OLD name, then batch-update `pets` array to replace old name with new name. Completed and cancelled bookings should retain the historical name.
- Deploy command: `firebase deploy --only functions:onPetNameChanged`
- Status: PENDING

### [2026-06-26] DONE — Ant Design static message API replaced with App.useApp() across web-admin
- Triggered by: P1 issue — static message API bypasses ConfigProvider tokens (dark mode shows white popups)
- Source repo: savipets-web-admin
- Target repo: none (self-contained)
- Files changed: 61 hook + component files migrated from `import { message } from 'antd'` to `const { message } = App.useApp()`
- Exceptions: `src/features/crm/utils/errorHandler.ts` (module-level utility, called from 20+ service files, not a hook)
- Test update: `useBookingActions.test.tsx` antd mock updated to `App: { useApp: () => ({ message: mockMessage }) }`
- Status: DONE ✅

### [2026-06-25] BUG-2: AdminBookingService.ts omits sitterName at booking creation
- Triggered by: iOS `4035386fe` session analysis — iOS fix patches the symptom; backend is root cause
- Source repo: savipets-ios
- Target repo: savipets-web-admin
- Status: DONE (2026-06-26) — code inspection confirmed getSitterName() already called at lines 58/242, sitterName written at lines 71/110/254/343. Cross-repo TODO was stale. Also fixed: recurring booking `price` field was `String(basePrice)` instead of number.
