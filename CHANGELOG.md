# Changelog

All notable changes to this skill are documented here.

## [0.4.0] - 2026-03-21

### Added
- CI/CD reference file (references/cicd.md) covering all three platforms:
  - Playwright E2E testing for CMS with config, example tests, and GitHub Actions workflow
  - Firebase Hosting deploy workflow
  - Fastlane setup for iOS (tests, TestFlight, App Store submission) with GitHub Actions
  - Fastlane setup for Android (tests, Play Store internal/production) with GitHub Actions
  - Required GitHub Secrets reference table
  - Branch strategy guidance
- Referenced [Artl13/playwright-automation-test-package](https://github.com/Artl13/playwright-automation-test-package) as automation baseline

## [0.3.0] - 2026-03-21

### Added
- Apple Sign-In with full `ASAuthorizationController` flow (firebase-ios.md)
- Crashlytics usage examples: set user ID, custom keys, breadcrumb logs, non-fatal error recording (firebase-android.md)
- Example prompts section in README showing how to trigger the skill in Cursor
- CHANGELOG

## [0.2.0] - 2026-03-21

### Added
- Facebook Sign-In for Android with `LoginManager`, `CallbackManager`, manifest config (firebase-android.md)
- Supabase Edge Functions: Deno handler skeleton, CLI commands, Stripe webhook example (supabase.md)
- CMS subscription status reading: dashboard stats, user detail with trial info, targeted push via FCM (firebase.md)
- Subscription and trial fields on user schema: `subscriptionStatus`, `subscriptionPlan`, `trialExpiresAt` (content-modeling.md)
- `fcmTokens` collection in CMS Firestore schema (firebase.md)
- HealthKit integration for iOS: authorization, read steps/calories, write samples, background delivery (ios-skill.md)
- Health Connect integration for Android: permissions, read/write records (android-skill.md)
- CameraX scanning for Android: camera preview composable, image analysis, photo capture for AI vision (android-skill.md)
- RBAC middleware for CMS: `requireRole()`, route examples, author-only filtering, EJS conditionals (cms-skill.md)

## [0.1.0] - 2026-03-21

### Added
- Initial release with full skill and reference files
- Node.js CMS platform: architecture, server.js, EJS views, Firebase and Supabase backends, content modeling
- iOS platform: SwiftUI + MVVM, Firebase, OpenAI, StoreKit 2, widgets, Live Activities, TipKit
- Android platform: Kotlin + Compose, Firebase, OpenAI, Play Billing, widgets, foreground service trackers
- Shared patterns: Firebase backend, subscription/trial, push notifications, offline caching, dark mode, localization
- MIT License, .gitignore, GitHub topics
