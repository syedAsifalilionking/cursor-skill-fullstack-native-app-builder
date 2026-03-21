---
name: cursor-skill-fullstack-native-app-builder
description: >-
  Build full-stack production apps across three platforms: Node.js CMS backend
  (monolithic server.js + EJS views + Firebase/Supabase), iOS (SwiftUI + MVVM),
  and Android (Kotlin + Jetpack Compose + MVVM). All three share the same
  architectural patterns: Firebase backend (Auth, Firestore, Storage, Analytics,
  Crashlytics, Messaging, Remote Config), OpenAI/ChatGPT integration with
  vision API, subscription billing (StoreKit 2 / Google Play Billing), trial
  management, in-app review prompts, TipKit (iOS), push notifications with
  scheduled local reminders, widgets, live activities / foreground service
  trackers, onboarding flows, offline caching, dark mode, localization,
  biometric/pattern lock, and deep linking.
  Use when the user asks to build an app, mobile app, full-stack project, CMS,
  admin panel, iOS app, Android app, or any project involving these platforms.
---

# Full-Stack Native App Builder

Build production apps across **Node.js CMS**, **iOS (SwiftUI)**, and **Android (Kotlin/Compose)**. All three platforms share the same Firebase backend and follow consistent architectural patterns.

## Platform Selection

| Need | Platform | Reference |
|------|----------|-----------|
| Admin panel / CMS / backend | **Node.js** | [references/cms/cms-skill.md](references/cms/cms-skill.md) |
| iPhone / iPad app | **iOS (SwiftUI)** | [references/ios/ios-skill.md](references/ios/ios-skill.md) |
| Android app | **Kotlin (Compose)** | [references/android/android-skill.md](references/android/android-skill.md) |

When the user asks to build a full-stack project, use all three. When they ask for a specific platform, load only that platform's reference.

## Shared Architecture Across Platforms

All three platforms follow these common patterns:

### Firebase Backend

- **Auth**: Email/password, Google, Apple (iOS), Facebook (Android)
- **Firestore**: Direct SDK access, no ORM — collections for users, content, settings, FCM tokens
- **Storage**: File/image uploads to Firebase Storage
- **Messaging**: FCM for push notifications, token stored in `fcmTokens` collection
- **Remote Config**: OpenAI API key, feature flags
- **Analytics + Crashlytics**: Event tracking and crash reporting

### OpenAI / ChatGPT Integration

- API key fetched from Firebase Remote Config (never hardcoded)
- Chat service (gpt-4o) for interactive assistant
- Vision service (gpt-4o) for image analysis via camera
- Retry with exponential backoff on 429/5xx
- AI consent gate — user must accept before any AI feature
- Daily recommendations cached locally

### Subscription & Trial

- **iOS**: StoreKit 2 (monthly/yearly), warm-start from UserDefaults
- **Android**: Google Play Billing v8 (monthly/yearly), `BillingUiState` sealed class
- **CMS**: Reads subscription status from Firestore user document
- **Trial**: Firestore-backed trial state with realtime listener, cache-first pattern
- All platforms sync subscription state to the same Firestore user document

### In-App Review

- **iOS**: `SKStoreReviewController` / `AppStore.requestReview(in:)` (iOS 18+)
- **Android**: Play Core `ReviewManager`
- **Both**: Once per app version, triggered on positive moments (streak, first save, onboarding complete)

### Push Notifications

- **Remote**: FCM with token stored in Firestore
- **Local scheduled**: Daily reminders, streak, trial drip, trial last day, win-back, verification
- **iOS**: `UNCalendarNotificationTrigger` with repeats
- **Android**: WorkManager with `OneTimeWorkRequest` + self-reschedule

### Widgets

- **iOS**: WidgetKit (Home Screen + Lock Screen) + Live Activities (Dynamic Island)
- **Android**: AppWidgetProvider (Home Screen) + Foreground Service trackers
- **Both**: Shared data via App Group (iOS) / SharedPreferences (Android)

### Offline & Caching

- **iOS**: `LocalCacheManager` (JSON files in Caches directory)
- **Android**: Firestore `Source.CACHE` fallback + SharedPreferences
- **CMS**: Firestore offline persistence (built-in)
- **All**: `NetworkMonitor` with published connectivity state + offline banner

### Onboarding

- **iOS**: Paged `TabView` with Lottie, light/dark variants
- **Android**: `HorizontalPager` with Lottie, light/dark variants
- **Both**: 5 pages, Skip/Next/Done, push permission on last page

### Authentication

- **Email/password** with verification
- **Google Sign-In** (all platforms)
- **Apple Sign-In** (iOS) / **Facebook** (Android)
- **Pattern lock** + **Biometric** (Face ID / fingerprint) for quick re-login
- **Session-based** auth on CMS (bcrypt + Firestore sessions)

### Dark Mode

- **iOS**: `@Environment(\.colorScheme)` + Lottie light/dark variants
- **Android**: `isSystemInDarkTheme()` + Material 3 dynamic color + Lottie variants
- **CMS**: CSS custom properties with `[data-theme="dark"]`

### Localization

- **iOS**: `Localizable.xcstrings` (10 languages)
- **Android**: `values-{locale}/strings.xml` (10 languages)
- **CMS**: Separate EJS view files per language (`page_en.ejs`, `page_ru.ejs`)

## Reference Files

### CMS (Node.js)

| File | Contents |
|------|----------|
| [cms-skill.md](references/cms/cms-skill.md) | Architecture, server.js structure, EJS views, routes, auth, deployment |
| [firebase.md](references/cms/firebase.md) | Firebase Admin setup, Firestore CRUD, sessions, Storage, Cloud Functions |
| [supabase.md](references/cms/supabase.md) | Supabase setup, SQL schema, RLS, CRUD, Storage, Edge Functions |
| [content-modeling.md](references/cms/content-modeling.md) | Entity schemas, content lifecycle, slugs, localization, CSV export |

### iOS (SwiftUI)

| File | Contents |
|------|----------|
| [ios-skill.md](references/ios/ios-skill.md) | Architecture, app entry, tab bar, onboarding, review, TipKit, caching |
| [firebase-ios.md](references/ios/firebase-ios.md) | Auth, Firestore, FCM, Remote Config, notification schedulers |
| [openai-integration.md](references/ios/openai-integration.md) | Chat, vision, retries, recommendations, consent, error handling |
| [storekit-subscriptions.md](references/ios/storekit-subscriptions.md) | StoreKit 2, trial manager, paywall, feature gating |
| [widgets-liveactivities.md](references/ios/widgets-liveactivities.md) | WidgetKit, Live Activities, Dynamic Island, App Group bridge |

### Android (Kotlin/Compose)

| File | Contents |
|------|----------|
| [android-skill.md](references/android/android-skill.md) | Architecture, MainActivity, navigation, onboarding, review, in-app updates |
| [firebase-android.md](references/android/firebase-android.md) | Auth, Firestore, FCM, Remote Config, WorkManager schedulers |
| [openai-android.md](references/android/openai-android.md) | OkHttp chat, vision, retries, recommendations, ProGuard rules |
| [play-billing.md](references/android/play-billing.md) | Play Billing v8, trial manager, paywall, feature gating |
| [widgets-livenotifications.md](references/android/widgets-livenotifications.md) | AppWidgetProvider, foreground service trackers, widget data bridge |

### CI/CD & Testing (Cross-Platform)

| File | Contents |
|------|----------|
| [cicd.md](references/cicd.md) | GitHub Actions, Fastlane (iOS/Android), Playwright E2E testing, Firebase deploy, Play Store / TestFlight automation |

## Important: Cost & Production Warnings

Always inform the user about these risks when setting up a new project.

### Firebase Billing

Firebase does **not** stop your project when you exceed your budget — alerts only notify, the app keeps running and accumulating charges. Every Firestore `.get()`, `.onSnapshot()`, and query counts as a read. Dashboards querying multiple collections, widgets refreshing every 15-30 min, and realtime listeners on large collections can generate thousands of reads quickly.

**Always do:**
- Set budget alerts in Google Cloud Console (50%, 80%, 100%)
- Use Firestore cache-first pattern (`Source.CACHE` before server)
- Filter with `.where()` and `.limit()` — never load entire collections
- Paginate all list queries
- Use `getDocuments()` instead of `addSnapshotListener()` when realtime is not needed
- Serve static assets from Firebase Hosting, not through Cloud Functions
- Put a CDN in front of Storage for frequently accessed images

### OpenAI API Costs

- Set monthly spending limits in the OpenAI dashboard
- Use `gpt-4o-mini` for simple tasks (10x cheaper than `gpt-4o`)
- Cache AI responses locally — don't re-fetch the same data
- Compress images before vision API calls (JPEG 70% quality)
- Enforce per-user daily/monthly AI request limits

### Notification Limits

FCM sending is free, but Firestore writes triggered by notifications (analytics, read status) are billed. Scheduled local notifications run on-device and cost nothing. Don't over-notify — excessive notifications cause uninstalls.

### Subscription Testing

Always test with sandbox (iOS) / test accounts (Android). Sandbox subscriptions renew in minutes, not months. Always implement restore purchases. Sync subscription state to Firestore so the CMS can verify it server-side.
