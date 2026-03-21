# CI/CD & Automated Testing Reference

GitHub Actions workflows, Fastlane automation, Playwright E2E testing, and deployment pipelines for all three platforms.

Playwright test automation reference: [Artl13/playwright-automation-test-package](https://github.com/Artl13/playwright-automation-test-package)

## CMS: Playwright E2E Testing

### Setup

```bash
npm install -D @playwright/test @types/node
npx playwright install --with-deps
```

### Playwright Config

```typescript
// playwright.config.ts
import { defineConfig, devices } from '@playwright/test';

export default defineConfig({
    testDir: './tests',
    fullyParallel: true,
    forbidOnly: !!process.env.CI,
    retries: process.env.CI ? 2 : 0,
    workers: process.env.CI ? 1 : undefined,
    reporter: [
        ['html', { outputFolder: 'playwright-report', open: 'never' }],
        ['json', { outputFile: 'playwright-report/playwright-report.json' }]
    ],
    use: {
        baseURL: process.env.BASE_URL || 'http://localhost:3000',
        trace: 'on-first-retry',
    },
    projects: [
        { name: 'chromium', use: { ...devices['Desktop Chrome'] } },
        { name: 'firefox', use: { ...devices['Desktop Firefox'] } },
        { name: 'webkit', use: { ...devices['Desktop Safari'] } },
    ],
    webServer: {
        command: 'npm start',
        url: 'http://localhost:3000',
        reuseExistingServer: !process.env.CI,
    },
});
```

### Example CMS Tests

```typescript
// tests/auth.spec.ts
import { test, expect } from '@playwright/test';

test('login page loads', async ({ page }) => {
    await page.goto('/login');
    await expect(page.locator('input[name="email"]')).toBeVisible();
    await expect(page.locator('input[name="password"]')).toBeVisible();
});

test('login with valid credentials', async ({ page }) => {
    await page.goto('/login');
    await page.fill('input[name="email"]', process.env.TEST_ADMIN_EMAIL!);
    await page.fill('input[name="password"]', process.env.TEST_ADMIN_PASSWORD!);
    await page.click('button[type="submit"]');
    await expect(page).toHaveURL('/dashboard');
});

test('dashboard shows stats', async ({ page }) => {
    // Assumes authenticated via storageState
    await page.goto('/dashboard');
    await expect(page.locator('.stat-card')).toHaveCount(3);
});

// tests/crud.spec.ts
test('create and delete an article', async ({ page }) => {
    await page.goto('/articles/add');
    await page.fill('input[name="title"]', 'Test Article');
    await page.fill('textarea[name="content"]', 'Test content');
    await page.click('button[type="submit"]');
    await expect(page).toHaveURL('/articles');
    await expect(page.locator('text=Test Article')).toBeVisible();

    // Cleanup
    await page.locator('text=Test Article').locator('..').locator('.delete-btn').click();
    await page.locator('.confirm-delete').click();
});
```

### GitHub Actions: Playwright

```yaml
# .github/workflows/playwright.yml
name: Playwright Tests

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]
  schedule:
    - cron: '0 11 * * 1-4'  # Mon-Thu at 11:00 UTC
  workflow_dispatch:

jobs:
  test:
    timeout-minutes: 60
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: lts/*

      - name: Install dependencies
        run: npm ci

      - name: Install Playwright Browsers
        run: npx playwright install --with-deps

      - name: Run Playwright tests
        run: npx playwright test
        env:
          BASE_URL: ${{ secrets.BASE_URL }}
          TEST_ADMIN_EMAIL: ${{ secrets.TEST_ADMIN_EMAIL }}
          TEST_ADMIN_PASSWORD: ${{ secrets.TEST_ADMIN_PASSWORD }}

      - name: Upload Playwright Report
        uses: actions/upload-artifact@v4
        if: ${{ !cancelled() }}
        with:
          name: playwright-report
          path: playwright-report/
          retention-days: 30
```

## CMS: Firebase Deploy

```yaml
# .github/workflows/deploy-cms.yml
name: Deploy CMS

on:
  push:
    branches: [main]
    paths:
      - 'server.js'
      - 'views/**'
      - 'public/**'
      - 'package.json'

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: lts/*

      - run: npm ci

      - name: Deploy to Firebase
        uses: FirebaseExtended/action-hosting-deploy@v0
        with:
          repoToken: ${{ secrets.GITHUB_TOKEN }}
          firebaseServiceAccount: ${{ secrets.FIREBASE_SERVICE_ACCOUNT }}
          channelId: live
          projectId: ${{ secrets.FIREBASE_PROJECT_ID }}
```

## iOS: Fastlane + GitHub Actions

### Fastlane Setup

```bash
# In the iOS project root
gem install fastlane
fastlane init
```

### Fastfile

```ruby
# fastlane/Fastfile
default_platform(:ios)

platform :ios do
  desc "Run unit tests"
  lane :test do
    run_tests(
      scheme: "AppName",
      devices: ["iPhone 16"],
      clean: true
    )
  end

  desc "Build and upload to TestFlight"
  lane :beta do
    setup_ci if ENV['CI']

    match(type: "appstore", readonly: is_ci)

    increment_build_number(
      build_number: ENV["GITHUB_RUN_NUMBER"] || latest_testflight_build_number + 1
    )

    build_app(
      scheme: "AppName",
      export_method: "app-store",
      output_directory: "./build"
    )

    upload_to_testflight(
      skip_waiting_for_build_processing: true
    )
  end

  desc "Submit to App Store review"
  lane :release do
    deliver(
      skip_metadata: false,
      skip_screenshots: true,
      submit_for_review: true,
      automatic_release: false,
      force: true
    )
  end
end
```

### Matchfile (Code Signing)

```ruby
# fastlane/Matchfile
git_url("https://github.com/YourOrg/certificates.git")
storage_mode("git")
type("appstore")
app_identifier("com.yourpackage.appname")
```

### GitHub Actions: iOS

```yaml
# .github/workflows/ios.yml
name: iOS Build & TestFlight

on:
  push:
    branches: [main]
    paths:
      - 'AppName/**'
      - '*.xcodeproj/**'

jobs:
  build:
    runs-on: macos-15
    steps:
      - uses: actions/checkout@v4

      - name: Select Xcode
        run: sudo xcode-select -s /Applications/Xcode_16.3.app

      - name: Install Fastlane
        run: gem install fastlane

      - name: Run tests
        run: fastlane test

      - name: Build and upload to TestFlight
        run: fastlane beta
        env:
          MATCH_PASSWORD: ${{ secrets.MATCH_PASSWORD }}
          MATCH_GIT_BASIC_AUTHORIZATION: ${{ secrets.MATCH_GIT_AUTH }}
          APP_STORE_CONNECT_API_KEY_KEY_ID: ${{ secrets.ASC_KEY_ID }}
          APP_STORE_CONNECT_API_KEY_ISSUER_ID: ${{ secrets.ASC_ISSUER_ID }}
          APP_STORE_CONNECT_API_KEY_KEY: ${{ secrets.ASC_PRIVATE_KEY }}
```

## Android: Fastlane + GitHub Actions

### Fastfile

```ruby
# fastlane/Fastfile
default_platform(:android)

platform :android do
  desc "Run unit tests"
  lane :test do
    gradle(task: "test")
  end

  desc "Build debug APK"
  lane :debug_build do
    gradle(task: "assembleDebug")
  end

  desc "Build and upload to Play Store internal track"
  lane :beta do
    gradle(
      task: "bundle",
      build_type: "Release",
      properties: {
        "android.injected.signing.store.file" => ENV["KEYSTORE_PATH"],
        "android.injected.signing.store.password" => ENV["KEYSTORE_PASSWORD"],
        "android.injected.signing.key.alias" => ENV["KEY_ALIAS"],
        "android.injected.signing.key.password" => ENV["KEY_PASSWORD"]
      }
    )

    upload_to_play_store(
      track: "internal",
      aab: "app/build/outputs/bundle/release/app-release.aab",
      skip_upload_metadata: true,
      skip_upload_images: true,
      skip_upload_screenshots: true
    )
  end

  desc "Promote internal to production"
  lane :release do
    upload_to_play_store(
      track: "internal",
      track_promote_to: "production",
      skip_upload_changelogs: false
    )
  end
end
```

### GitHub Actions: Android

```yaml
# .github/workflows/android.yml
name: Android Build & Play Store

on:
  push:
    branches: [main]
    paths:
      - 'app/**'
      - 'build.gradle.kts'
      - 'gradle/**'

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-java@v4
        with:
          distribution: 'temurin'
          java-version: '21'

      - name: Setup Gradle
        uses: gradle/actions/setup-gradle@v4

      - name: Run tests
        run: ./gradlew test

      - name: Decode keystore
        run: echo "${{ secrets.KEYSTORE_BASE64 }}" | base64 -d > app/keystore.jks

      - name: Build and upload to Play Store
        run: bundle exec fastlane beta
        env:
          KEYSTORE_PATH: app/keystore.jks
          KEYSTORE_PASSWORD: ${{ secrets.KEYSTORE_PASSWORD }}
          KEY_ALIAS: ${{ secrets.KEY_ALIAS }}
          KEY_PASSWORD: ${{ secrets.KEY_PASSWORD }}
          SUPPLY_JSON_KEY_DATA: ${{ secrets.PLAY_STORE_SERVICE_ACCOUNT }}
```

## Required GitHub Secrets

| Secret | Platform | Purpose |
|--------|----------|---------|
| `FIREBASE_SERVICE_ACCOUNT` | CMS | Firebase deploy service account JSON |
| `FIREBASE_PROJECT_ID` | CMS | Firebase project ID |
| `BASE_URL` | CMS | Playwright test target URL |
| `TEST_ADMIN_EMAIL` | CMS | Test admin credentials |
| `TEST_ADMIN_PASSWORD` | CMS | Test admin credentials |
| `MATCH_PASSWORD` | iOS | Fastlane Match encryption password |
| `MATCH_GIT_AUTH` | iOS | Base64-encoded `user:token` for certificates repo |
| `ASC_KEY_ID` | iOS | App Store Connect API key ID |
| `ASC_ISSUER_ID` | iOS | App Store Connect API issuer ID |
| `ASC_PRIVATE_KEY` | iOS | App Store Connect API private key (.p8 contents) |
| `KEYSTORE_BASE64` | Android | Base64-encoded release keystore |
| `KEYSTORE_PASSWORD` | Android | Keystore password |
| `KEY_ALIAS` | Android | Signing key alias |
| `KEY_PASSWORD` | Android | Signing key password |
| `PLAY_STORE_SERVICE_ACCOUNT` | Android | Google Play service account JSON |

## Branch Strategy

```
main ─────────────────────────────────► production
  │
  ├── feature/auth-flow ──► PR ──► merge
  ├── feature/billing ────► PR ──► merge
  └── fix/crash-on-scan ──► PR ──► merge
```

- **`main`**: Always deployable. Pushes trigger CI and auto-deploy.
- **Feature branches**: Created from `main`, merged via PR with required checks.
- **Playwright tests**: Run on every PR to `main` and on a daily schedule.
- **Mobile builds**: Triggered on push to `main` when platform files change (path filtering).
