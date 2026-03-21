# Firebase iOS Integration Reference

## Setup

Add Firebase SPM packages: Auth, Firestore, Storage, Functions, Analytics, Crashlytics, Messaging, Performance, Remote Config, App Check.

```swift
// AppNameApp.swift init()
FirebaseApp.configure()
```

Add `GoogleService-Info.plist` to the main target (gitignore it).

## Firestore Manager

Single service class for all Firestore operations:

```swift
final class FirestoreManager {
    static let shared = FirestoreManager()
    private let db = Firestore.firestore()

    // MARK: - Users
    func getUser(uid: String) async throws -> UserProfile? {
        let snapshot = try await db.collection("users")
            .whereField("firebaseUID", isEqualTo: uid)
            .limit(to: 1)
            .getDocuments()
        guard let doc = snapshot.documents.first else { return nil }
        return try doc.data(as: UserProfile.self)
    }

    func updateUser(docID: String, data: [String: Any]) async throws {
        try await db.collection("users").document(docID).updateData(data)
    }

    // MARK: - Content (any collection)
    func fetchCollection<T: Decodable>(_ path: String, as type: T.Type) async throws -> [T] {
        let snapshot = try await db.collection(path)
            .order(by: "createdAt", descending: true)
            .getDocuments()
        return snapshot.documents.compactMap { try? $0.data(as: type) }
    }

    // MARK: - Realtime listener
    func listen<T: Decodable>(to path: String, as type: T.Type, onChange: @escaping ([T]) -> Void) -> ListenerRegistration {
        db.collection(path).addSnapshotListener { snapshot, error in
            guard let docs = snapshot?.documents else { return }
            let items = docs.compactMap { try? $0.data(as: type) }
            onChange(items)
        }
    }
}
```

### Cache-First Pattern

For instant UI, query cache first then attach a realtime listener:

```swift
// Immediate cache hit
db.collection("users").whereField("uid", isEqualTo: uid).limit(to: 1)
    .getDocuments(source: .cache) { snap, _ in
        updateUI(from: snap)
    }

// Then realtime
listener = db.collection("users").whereField("uid", isEqualTo: uid).limit(to: 1)
    .addSnapshotListener(includeMetadataChanges: true) { snap, _ in
        updateUI(from: snap)
    }
```

## Authentication

### Email / Password

```swift
// Sign up
let result = try await Auth.auth().createUser(withEmail: email, password: password)
try await result.user.sendEmailVerification()

// Sign in
try await Auth.auth().signIn(withEmail: email, password: password)

// Password reset
try await Auth.auth().sendPasswordReset(withEmail: email)

// Auth state listener
Auth.auth().addStateDidChangeListener { _, user in
    if let user {
        // Signed in — bind services
        TrialManager.shared.bind(to: user.uid)
        StoreKitManager.shared.start()
    } else {
        // Signed out — unbind
        TrialManager.shared.unbind()
    }
}
```

### Google Sign-In

```swift
guard let clientID = FirebaseApp.app()?.options.clientID else { return }
let config = GIDConfiguration(clientID: clientID)
GIDSignIn.sharedInstance.configuration = config

guard let rootVC = UIApplication.shared.connectedScenes
    .compactMap({ $0 as? UIWindowScene }).first?.windows.first?.rootViewController else { return }

let result = try await GIDSignIn.sharedInstance.signIn(withPresenting: rootVC)
let credential = GoogleAuthProvider.credential(
    withIDToken: result.user.idToken!.tokenString,
    accessToken: result.user.accessToken.tokenString
)
try await Auth.auth().signIn(with: credential)
```

### Apple Sign-In

Requires the "Sign in with Apple" capability in Xcode.

```swift
import AuthenticationServices
import CryptoKit

final class AppleSignInHelper: NSObject, ASAuthorizationControllerDelegate,
                                ASAuthorizationControllerPresentationContextProviding {

    private var currentNonce: String?
    var onComplete: ((Result<AuthDataResult, Error>) -> Void)?

    func startSignInFlow() {
        let nonce = randomNonceString()
        currentNonce = nonce

        let provider = ASAuthorizationAppleIDProvider()
        let request = provider.createRequest()
        request.requestedScopes = [.fullName, .email]
        request.nonce = sha256(nonce)

        let controller = ASAuthorizationController(authorizationRequests: [request])
        controller.delegate = self
        controller.presentationContextProvider = self
        controller.performRequests()
    }

    func presentationAnchor(for controller: ASAuthorizationController) -> ASPresentationAnchor {
        UIApplication.shared.connectedScenes
            .compactMap { $0 as? UIWindowScene }
            .first?.windows.first { $0.isKeyWindow } ?? UIWindow()
    }

    func authorizationController(controller: ASAuthorizationController,
                                  didCompleteWithAuthorization authorization: ASAuthorization) {
        guard let appleIDCredential = authorization.credential as? ASAuthorizationAppleIDCredential,
              let nonce = currentNonce,
              let appleIDToken = appleIDCredential.identityToken,
              let idTokenString = String(data: appleIDToken, encoding: .utf8) else { return }

        let credential = OAuthProvider.appleCredential(
            withIDToken: idTokenString,
            rawNonce: nonce,
            fullName: appleIDCredential.fullName
        )

        Task {
            do {
                let result = try await Auth.auth().signIn(with: credential)
                onComplete?(.success(result))
            } catch {
                onComplete?(.failure(error))
            }
        }
    }

    func authorizationController(controller: ASAuthorizationController, didCompleteWithError error: Error) {
        onComplete?(.failure(error))
    }

    private func randomNonceString(length: Int = 32) -> String {
        var randomBytes = [UInt8](repeating: 0, count: length)
        _ = SecRandomCopyBytes(kSecRandomDefault, randomBytes.count, &randomBytes)
        let charset: [Character] = Array("0123456789ABCDEFGHIJKLMNOPQRSTUVXYZabcdefghijklmnopqrstuvwxyz-._")
        return String(randomBytes.map { charset[Int($0) % charset.count] })
    }

    private func sha256(_ input: String) -> String {
        let data = Data(input.utf8)
        let hash = SHA256.hash(data: data)
        return hash.compactMap { String(format: "%02x", $0) }.joined()
    }
}
```

SwiftUI button integration:

```swift
SignInWithAppleButton(.signIn) { request in
    request.requestedScopes = [.fullName, .email]
}
onCompletion: { result in
    // Use AppleSignInHelper for Firebase credential relay
}
.frame(height: 50)
.signInWithAppleButtonStyle(colorScheme == .dark ? .white : .black)
```

### Localized Auth Errors

```swift
func userFacingAuthMessage(for error: Error) -> String {
    let code = AuthErrorCode(_nsError: error as NSError).code
    switch code {
    case .wrongPassword: return String(localized: "auth_wrong_password")
    case .userNotFound: return String(localized: "auth_user_not_found")
    case .emailAlreadyInUse: return String(localized: "auth_email_in_use")
    case .networkError: return String(localized: "auth_network_error")
    default: return String(localized: "auth_unknown_error")
    }
}
```

## Push Notifications (FCM)

### Delegate Setup

```swift
final class PushNotificationService: NSObject, UIApplicationDelegate, UNUserNotificationCenterDelegate, MessagingDelegate {

    func application(_ application: UIApplication, didFinishLaunchingWithOptions _: [UIApplication.LaunchOptionsKey: Any]?) -> Bool {
        UNUserNotificationCenter.current().delegate = self
        Messaging.messaging().delegate = self
        configureNotificationCategories()
        return true
    }

    func configureNotificationCategories() {
        let categories: Set<UNNotificationCategory> = [
            UNNotificationCategory(identifier: "DAILY_REMINDER", actions: [], intentIdentifiers: []),
            UNNotificationCategory(identifier: "STREAK_REMINDER", actions: [], intentIdentifiers: []),
            UNNotificationCategory(identifier: "TRIAL_DRIP", actions: [], intentIdentifiers: []),
            UNNotificationCategory(identifier: "WIN_BACK", actions: [], intentIdentifiers: []),
        ]
        UNUserNotificationCenter.current().setNotificationCategories(categories)
    }

    // Request permission
    func requestPermission() {
        UNUserNotificationCenter.current().requestAuthorization(options: [.alert, .badge, .sound]) { granted, _ in
            if granted { DispatchQueue.main.async { UIApplication.shared.registerForRemoteNotifications() } }
        }
    }

    // FCM token
    func messaging(_ messaging: Messaging, didReceiveRegistrationToken fcmToken: String?) {
        guard let token = fcmToken, let uid = Auth.auth().currentUser?.uid else { return }
        Firestore.firestore().collection("fcmTokens").document(uid).setData([
            "token": token,
            "platform": "ios",
            "updatedAt": FieldValue.serverTimestamp()
        ], merge: true)
    }

    // Handle notification tap
    func userNotificationCenter(_ center: UNUserNotificationCenter, didReceive response: UNNotificationResponse) async {
        let userInfo = response.notification.request.content.userInfo
        if let link = userInfo["link"] as? String, let url = URL(string: link) {
            await UIApplication.shared.open(url)
        }
    }

    // Foreground display
    func userNotificationCenter(_ center: UNUserNotificationCenter, willPresent notification: UNNotification) async -> UNNotificationPresentationOptions {
        [.banner, .sound, .badge]
    }
}
```

### Local Notification Schedulers

Create separate scheduler structs per notification type:

```swift
struct StreakReminderScheduler {
    static func schedule() {
        let center = UNUserNotificationCenter.current()
        center.getNotificationSettings { settings in
            guard settings.authorizationStatus == .authorized else { return }
            center.removePendingNotificationRequests(withIdentifiers: ["streak-reminder"])

            let content = UNMutableNotificationContent()
            content.title = String(localized: "streak_reminder_title")
            content.body = String(localized: "streak_reminder_body")
            content.sound = .default
            content.categoryIdentifier = "STREAK_REMINDER"

            var components = DateComponents()
            components.hour = 20
            components.minute = 0
            let trigger = UNCalendarNotificationTrigger(dateMatching: components, repeats: true)
            center.add(UNNotificationRequest(identifier: "streak-reminder", content: content, trigger: trigger))
        }
    }

    static func cancel() {
        UNUserNotificationCenter.current().removePendingNotificationRequests(withIdentifiers: ["streak-reminder"])
    }
}
```

### Notification Types to Implement

| Type | Timing | Purpose |
|------|--------|---------|
| Daily reminder | Fixed time | Core engagement |
| Streak reminder | Evening | Maintain streaks |
| Trial drip | Days 1, 3, 5 of trial | Conversion |
| Trial last day | Last day of trial | Urgency |
| Win-back | 3, 7, 14 days after churn | Re-engagement |
| Verification | Day after signup | Email verification |

## Remote Config

Fetch feature flags and API keys from Firebase Remote Config:

```swift
final class AppRemoteConfig {
    static let shared = AppRemoteConfig()
    private let remoteConfig = RemoteConfig.remoteConfig()

    var openAIAPIKey: String { remoteConfig["openai_api_key"].stringValue ?? "" }
    var featureEnabled: Bool { remoteConfig["feature_x_enabled"].boolValue }

    func fetch() async {
        let settings = RemoteConfigSettings()
        settings.minimumFetchInterval = 3600
        remoteConfig.configSettings = settings
        try? await remoteConfig.fetchAndActivate()
    }
}
```

## Analytics Events

```swift
Analytics.logEvent("item_scanned", parameters: ["type": "barcode"])
Analytics.logEvent("subscription_started", parameters: ["plan": "monthly"])
Analytics.logEvent("ai_chat_sent", parameters: ["message_count": count])
```

## App Check

```swift
// AppNameApp init()
let providerFactory = AppCheckDebugProviderFactory() // debug
AppCheck.setAppCheckProviderFactory(providerFactory)
```

## Storage Uploads

```swift
func uploadImage(_ image: UIImage, path: String) async throws -> URL {
    guard let data = image.jpegData(compressionQuality: 0.8) else { throw UploadError.invalidImage }
    let ref = Storage.storage().reference().child(path)
    _ = try await ref.putDataAsync(data)
    return try await ref.downloadURL()
}
```
