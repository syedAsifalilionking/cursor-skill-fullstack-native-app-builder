---
name: ios-app-builder
description: >-
  Build production-ready iOS apps with SwiftUI and MVVM architecture. Covers
  Firebase backend (Auth, Firestore, Storage, Analytics, Crashlytics, Messaging,
  Remote Config), OpenAI/ChatGPT integration with vision API, StoreKit 2
  subscriptions with trial management, TipKit, in-app review prompts,
  HealthKit, widgets, Live Activities, App Intents/Siri Shortcuts, push
  notifications with scheduled local reminders, onboarding flows, offline
  caching, dark mode, localization, pattern/biometric lock, and deep linking.
  Use when the user asks to build an iOS app, Swift app, iPhone app, or any
  SwiftUI project with Firebase, AI, subscriptions, or similar features.
---

# iOS App Builder

Build production-ready iOS apps with SwiftUI, MVVM, and a services layer.

## Architecture

**MVVM + Services** — no Combine pipelines or Redux. ViewModels are `@Observable` or `ObservableObject` classes. Services are singletons accessed directly.

```
AppName/
├── AppNameApp.swift              # @main entry, Firebase init, routing
├── ContentView.swift             # Tab bar, Plus menu, sheet coordination
├── MainView.swift                # Simplified tab wrapper
├── Info.plist
├── AppName.entitlements
├── GoogleService-Info.plist      # Firebase config (gitignored)
├── Components/                   # Reusable UI components
│   ├── ReviewPrompt.swift        # SKStoreReviewController wrapper
│   ├── Cards/                    # Dashboard cards
│   └── CameraPreview.swift
├── Models/                       # Data models + ViewModels
│   ├── SomeViewModel.swift
│   ├── SomeModel.swift
│   └── ML Model/                 # CoreML models if needed
├── Screens/                      # Full-screen views
│   ├── DashboardView.swift
│   ├── OnboardingView.swift
│   ├── SplashScreenView.swift
│   ├── SignInView.swift
│   ├── SignUpView.swift
│   ├── SubscriptionsView.swift
│   ├── ChatView.swift            # AI chat
│   └── ...EntityViews.swift
├── Scanners/                     # Camera/AI scan views
│   ├── SomeScannerView.swift
│   └── ScanResultSheet.swift
├── Services/                     # Singletons
│   ├── ChatGPTService.swift      # OpenAI chat
│   ├── VisionGPTService.swift    # OpenAI vision
│   ├── FirestoreManager.swift    # All Firestore ops
│   ├── StoreKitManager.swift     # StoreKit 2
│   ├── TrialManager.swift        # Trial state
│   ├── AIConsentManager.swift    # AI consent gate
│   ├── PushNotificationService.swift
│   ├── HealthKitManager.swift
│   ├── NetworkMonitor.swift      # NWPathMonitor
│   ├── LocalCacheManager.swift   # JSON file cache
│   ├── WidgetDataBridge.swift    # App Group shared data
│   ├── AppTips.swift             # TipKit tips
│   ├── NotificationSchedulers/   # Local notification schedulers
│   └── AppIntents/               # Siri Shortcuts
├── Resources/
│   └── *.json                    # Lottie animations
└── Extensions/                   # Swift extensions
```

### Widget Extension

```
AppNameWidgets/
├── AppNameWidgetBundle.swift
├── SomeWidget.swift
├── SomeLiveActivityWidget.swift
├── WidgetDataReader.swift
├── SomeActivityAttributes.swift
└── Info.plist
```

## Dependencies (SPM)

```
Firebase (Auth, Firestore, Storage, Functions, Analytics, Crashlytics, Messaging, Performance, Remote Config, App Check)
GoogleSignIn-iOS
Lottie
SDWebImage + SDWebImageSwiftUI
KeychainAccess
```

No CocoaPods — use Swift Package Manager exclusively.

## App Entry Point

```swift
@main
struct AppNameApp: App {
    @UIApplicationDelegateAdaptor(PushNotificationService.self) var delegate
    @AppStorage("isFirstLaunch") private var isFirstLaunch = true
    @StateObject private var storeKit = StoreKitManager.shared
    @StateObject private var trialManager = TrialManager.shared
    @State private var route: Route = .undecided
    @State private var showSplashScreen = true

    enum Route { case undecided, onboarding, main }

    init() {
        FirebaseApp.configure()
        Tips.configure([.displayFrequency(.immediate)])
    }

    var body: some Scene {
        WindowGroup {
            Group {
                switch route {
                case .undecided:
                    Color.clear.onAppear { route = isFirstLaunch ? .onboarding : .main }
                case .onboarding:
                    OnboardingView(onComplete: {
                        isFirstLaunch = false
                        route = .main
                    })
                case .main:
                    ZStack {
                        ContentView()
                        if showSplashScreen { SplashScreenView(isPresented: $showSplashScreen) }
                    }
                }
            }
            .environmentObject(storeKit)
            .environmentObject(trialManager)
        }
    }
}
```

## Tab Bar + Plus Menu

Custom floating tab bar with a center "Plus" button that expands into action bubbles:

```swift
struct ContentView: View {
    @State private var selectedTab = 0
    @State private var showPlusMenu = false

    var body: some View {
        ZStack(alignment: .bottom) {
            TabView(selection: $selectedTab) {
                DashboardView().tag(0)
                SomeListView().tag(1)
                Color.clear.tag(2) // Plus placeholder
                NotificationsView().tag(3)
                MoreView().tag(4)
            }

            FloatingTabBar(selected: $selectedTab, onPlusTap: { showPlusMenu.toggle() })

            if showPlusMenu {
                PlusMenuOverlay(onDismiss: { showPlusMenu = false })
            }
        }
    }
}
```

## Onboarding Flow

Paged `TabView` with Lottie animations, light/dark variants:

```swift
struct OnboardingView: View {
    @State private var currentPage = 0
    let totalPages = 5
    var onComplete: () -> Void

    var body: some View {
        TabView(selection: $currentPage) {
            ForEach(0..<totalPages, id: \.self) { index in
                OnboardingPageView(page: index)
                    .tag(index)
            }
        }
        .tabViewStyle(.page(indexDisplayMode: .never))
        .overlay(alignment: .bottom) {
            HStack {
                Button("Skip") { onComplete() }
                Spacer()
                Button(action: { currentPage < totalPages - 1 ? (currentPage += 1) : onComplete() }) {
                    Image(systemName: currentPage < totalPages - 1 ? "arrow.right" : "checkmark")
                }
            }
            .padding()
        }
    }
}
```

Request push notification permission on the last onboarding page.

## Authentication

Firebase Auth with email/password, Google Sign-In, and Sign in with Apple:

```swift
// Email
Auth.auth().signIn(withEmail: email, password: password)
Auth.auth().createUser(withEmail: email, password: password)
Auth.auth().currentUser?.sendEmailVerification()

// Google
GIDSignIn.sharedInstance.signIn(withPresenting: rootVC) { result, error in
    let credential = GoogleAuthProvider.credential(
        withIDToken: result.user.idToken!.tokenString,
        accessToken: result.user.accessToken.tokenString
    )
    Auth.auth().signIn(with: credential)
}

// Apple — use ASAuthorizationAppleIDProvider in SignInWithAppleDelegates

// Auth state listener
Auth.auth().addStateDidChangeListener { _, user in
    if let user { /* signed in */ } else { /* signed out */ }
}
```

**Pattern / Biometric Lock:** Store credentials in Keychain (KeychainAccess), offer Face ID quick-login on return.

## In-App Review Prompts

See [ReviewPrompt pattern below]. Trigger on positive moments, limit once per app version.

```swift
final class ReviewManager {
    static let shared = ReviewManager()
    private let promptedVersionKey = "reviewPromptedVersion"

    func requestIfEligible() {
        let currentVersion = Bundle.main.infoDictionary?["CFBundleShortVersionString"] as? String ?? ""
        guard UserDefaults.standard.string(forKey: promptedVersionKey) != currentVersion else { return }

        guard let scene = UIApplication.shared.connectedScenes
            .first(where: { $0.activationState == .foregroundActive }) as? UIWindowScene else { return }

        UserDefaults.standard.set(currentVersion, forKey: promptedVersionKey)

        if #available(iOS 18.0, *) {
            AppStore.requestReview(in: scene)
        } else {
            SKStoreReviewController.requestReview(in: scene)
        }
    }
}

// Trigger on positive moments via NotificationCenter
struct ReviewOnPositiveMoment: ViewModifier {
    func body(content: Content) -> some View {
        content
            .onReceive(NotificationCenter.default.publisher(for: .streakReachedSeven)) { _ in
                DispatchQueue.main.asyncAfter(deadline: .now() + 1.5) {
                    ReviewManager.shared.requestIfEligible()
                }
            }
    }
}
```

**Positive moment triggers:** streak milestones, first content saved, onboarding completed, subscription purchased.

## TipKit Integration

```swift
// Define tips
struct ScanTip: Tip {
    @Parameter static var dashboardReady: Bool = false
    static let scanned = Event(id: "userScanned")

    var title: Text { Text("Scan Items") }
    var message: Text? { Text("Use the camera to scan and get instant AI analysis.") }
    var image: Image? { Image(systemName: "camera.viewfinder") }

    var rules: [Rule] {
        #Rule(Self.$dashboardReady) { $0 == true }
        #Rule(Self.scanned) { $0.donations.count == 0 }
    }
}

// Gate tips until dashboard is ready
struct TipGate {
    static func unlock() { ScanTip.dashboardReady = true }
}

// Conditional display (avoid showing during splash)
struct ConditionalTipModifier: ViewModifier { /* check tipsAllowed environment */ }
```

Configure in app init: `Tips.configure([.displayFrequency(.immediate)])`.

## Network Monitor + Offline Banner

```swift
final class NetworkMonitor: ObservableObject {
    static let shared = NetworkMonitor()
    @Published var isConnected = true
    private let monitor = NWPathMonitor()

    init() {
        monitor.pathUpdateHandler = { [weak self] path in
            DispatchQueue.main.async { self?.isConnected = path.status == .satisfied }
        }
        monitor.start(queue: DispatchQueue(label: "NetworkMonitor"))
    }
}

// Show banner in ContentView
if !networkMonitor.isConnected {
    OfflineBanner()
        .transition(.move(edge: .top))
}
```

## Local Cache Manager

JSON file cache in `Caches/` directory for offline support:

```swift
final class LocalCacheManager {
    static let shared = LocalCacheManager()
    private let cacheDir: URL

    init() {
        cacheDir = FileManager.default.urls(for: .cachesDirectory, in: .userDomainMask)[0]
            .appendingPathComponent("AppLocalCache")
        try? FileManager.default.createDirectory(at: cacheDir, withIntermediateDirectories: true)
    }

    func save<T: Encodable>(_ items: T, forKey key: String) {
        let url = cacheDir.appendingPathComponent("\(key).json")
        let encoder = JSONEncoder()
        encoder.dateEncodingStrategy = .iso8601
        guard let data = try? encoder.encode(items) else { return }
        try? data.write(to: url)
    }

    func load<T: Decodable>(_ type: T.Type, forKey key: String) -> T? {
        let url = cacheDir.appendingPathComponent("\(key).json")
        guard let data = try? Data(contentsOf: url) else { return nil }
        let decoder = JSONDecoder()
        decoder.dateDecodingStrategy = .iso8601
        return try? decoder.decode(type, from: data)
    }
}
```

## Trial Management

```swift
@MainActor
final class TrialManager: ObservableObject {
    static let shared = TrialManager()
    @Published var isTrialActive = false
    @Published var daysLeft = 0
    private var listener: ListenerRegistration?

    func bind(to uid: String) {
        // Cache-first for instant UI
        db.collection("users").whereField("uid", isEqualTo: uid).limit(to: 1)
          .getDocuments(source: .cache) { [weak self] snap, _ in
              self?.updateState(from: snap)
          }
        // Then listen for realtime
        listener = db.collection("users").whereField("uid", isEqualTo: uid).limit(to: 1)
          .addSnapshotListener { [weak self] snap, _ in
              self?.updateState(from: snap)
          }
    }

    func unbind() { listener?.remove(); listener = nil }
}
```

## Push Notifications

For push notification setup and local notification scheduling, see [firebase-ios.md](firebase-ios.md).

## Scheduled Local Notifications

Pattern for daily reminders with localized random messages:

```swift
struct ReminderScheduler {
    static func scheduleDailyReminder() {
        let center = UNUserNotificationCenter.current()
        center.getNotificationSettings { settings in
            guard settings.authorizationStatus == .authorized else { return }
            center.removePendingNotificationRequests(withIdentifiers: ["daily-reminder"])

            let content = UNMutableNotificationContent()
            content.title = NSLocalizedString("reminder_title", comment: "")
            content.body = localizedMessages.randomElement() ?? ""
            content.sound = .default
            content.categoryIdentifier = "DAILY_REMINDER"

            var dateComponents = DateComponents()
            dateComponents.hour = 12
            dateComponents.minute = 30
            let trigger = UNCalendarNotificationTrigger(dateMatching: dateComponents, repeats: true)

            center.add(UNNotificationRequest(identifier: "daily-reminder", content: content, trigger: trigger))
        }
    }

    static func cancel() {
        UNUserNotificationCenter.current().removePendingNotificationRequests(withIdentifiers: ["daily-reminder"])
    }
}
```

## AI Consent Management

Gate AI features behind user consent:

```swift
final class AIConsentManager: ObservableObject {
    static let shared = AIConsentManager()
    @Published var hasConsented: Bool

    init() { hasConsented = UserDefaults.standard.bool(forKey: "aiConsent") }

    func grant() {
        hasConsented = true
        UserDefaults.standard.set(true, forKey: "aiConsent")
        syncToFirestore(consented: true)
    }

    func revoke() {
        hasConsented = false
        UserDefaults.standard.set(false, forKey: "aiConsent")
        syncToFirestore(consented: false)
    }
}
```

Show `AIConsentView` sheet before any AI feature if `!hasConsented`.

## Dark Mode

Use `@Environment(\.colorScheme)` and provide light/dark Lottie animation variants:

```swift
let animationName = colorScheme == .dark ? "animation_dark" : "animation_light"
```

CSS-style color definitions:

```swift
extension Color {
    init(hex: String) { /* parse hex */ }
    static let appBackground = Color("AppBackground")  // Asset catalog
    static let cardBackground = Color("CardBackground")
}
```

## Localization

Use `Localizable.xcstrings` (Xcode 15+). Support multiple languages. Use `String(localized:)` or `NSLocalizedString`.

## Deep Linking

```swift
// URL scheme: appname://action
// Info.plist: CFBundleURLSchemes = ["appname"]

.onOpenURL { url in
    guard url.scheme == "appname" else { return }
    switch url.host {
    case "someaction": handleAction()
    default: break
    }
}
```

## App Intents / Siri Shortcuts

```swift
struct LogSomethingIntent: AppIntent {
    static var title: LocalizedStringResource = "Log Something"
    static var description = IntentDescription("Quickly log an item")

    func perform() async throws -> some IntentResult {
        // Perform action
        return .result()
    }
}

struct AppShortcuts: AppShortcutsProvider {
    static var appShortcuts: [AppShortcut] {
        AppShortcut(intent: LogSomethingIntent(), phrases: ["Log in \(.applicationName)"])
    }
}
```

## HealthKit Integration

### Entitlements & Info.plist

Enable the HealthKit capability in Xcode (Signing & Capabilities). Add usage descriptions to `Info.plist`:

```xml
<key>NSHealthShareUsageDescription</key>
<string>We read your step count and activity data to track your progress.</string>
<key>NSHealthUpdateUsageDescription</key>
<string>We save your logged activities to Apple Health.</string>
```

### HealthKit Manager

```swift
import HealthKit

final class HealthKitManager: ObservableObject {
    static let shared = HealthKitManager()
    private let store = HKHealthStore()
    @Published var isAuthorized = false

    var isAvailable: Bool { HKHealthStore.isHealthDataAvailable() }

    func requestAuthorization() async throws {
        let readTypes: Set<HKObjectType> = [
            HKQuantityType(.stepCount),
            HKQuantityType(.activeEnergyBurned),
            HKQuantityType(.distanceWalkingRunning)
        ]
        let writeTypes: Set<HKSampleType> = [
            HKQuantityType(.activeEnergyBurned)
        ]

        try await store.requestAuthorization(toShare: writeTypes, read: readTypes)
        await MainActor.run { isAuthorized = true }
    }

    func fetchSteps(for date: Date) async throws -> Double {
        let start = Calendar.current.startOfDay(for: date)
        let end = Calendar.current.date(byAdding: .day, value: 1, to: start)!
        let predicate = HKQuery.predicateForSamples(withStart: start, end: end)

        let stepType = HKQuantityType(.stepCount)
        let descriptor = HKStatisticsQueryDescriptor(
            predicate: .init(quantityType: stepType, predicate: predicate),
            options: .cumulativeSum
        )
        let result = try await descriptor.result(for: store)
        return result?.sumQuantity()?.doubleValue(for: .count()) ?? 0
    }

    func fetchActiveEnergy(for date: Date) async throws -> Double {
        let start = Calendar.current.startOfDay(for: date)
        let end = Calendar.current.date(byAdding: .day, value: 1, to: start)!
        let predicate = HKQuery.predicateForSamples(withStart: start, end: end)

        let energyType = HKQuantityType(.activeEnergyBurned)
        let descriptor = HKStatisticsQueryDescriptor(
            predicate: .init(quantityType: energyType, predicate: predicate),
            options: .cumulativeSum
        )
        let result = try await descriptor.result(for: store)
        return result?.sumQuantity()?.doubleValue(for: .kilocalorie()) ?? 0
    }

    func writeActiveEnergy(calories: Double, start: Date, end: Date) async throws {
        let quantity = HKQuantity(unit: .kilocalorie(), doubleValue: calories)
        let sample = HKQuantitySample(
            type: HKQuantityType(.activeEnergyBurned),
            quantity: quantity,
            start: start,
            end: end
        )
        try await store.save(sample)
    }
}
```

### Background Delivery

Enable background delivery so the app receives updates even when not in the foreground:

```swift
func enableBackgroundDelivery() {
    let stepType = HKQuantityType(.stepCount)
    store.enableBackgroundDelivery(for: stepType, frequency: .hourly) { success, error in
        if let error { print("Background delivery failed: \(error)") }
    }
}
```

Register an observer query in `AppDelegate` or app init to handle background updates.

## Additional References

- **Firebase iOS setup** (Auth, Firestore, Messaging, Remote Config): [firebase-ios.md](firebase-ios.md)
- **OpenAI / ChatGPT integration** (chat, vision, structured output): [openai-integration.md](openai-integration.md)
- **StoreKit 2 subscriptions** (products, purchase, entitlements, Firestore sync): [storekit-subscriptions.md](storekit-subscriptions.md)
- **Widgets & Live Activities**: [widgets-liveactivities.md](widgets-liveactivities.md)
