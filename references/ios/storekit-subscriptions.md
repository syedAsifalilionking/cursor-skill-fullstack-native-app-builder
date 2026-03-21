# StoreKit 2 Subscriptions Reference

Patterns for implementing auto-renewable subscriptions with StoreKit 2 and Firestore sync.

## Subscription IDs

Define product IDs in a central file:

```swift
enum SubscriptionIDs {
    static let monthly = "com.yourapp.premium.monthly"
    static let yearly = "com.yourapp.premium.yearly"
    static let all = [monthly, yearly]
    static let groupID = "XXXXXXXX" // App Store Connect subscription group
}
```

## StoreKitManager

```swift
@MainActor
final class StoreKitManager: ObservableObject {
    static let shared = StoreKitManager()

    @Published var products: [Product] = []
    @Published var productsByID: [String: Product] = [:]
    @Published var purchasedProductIDs: Set<String> = []
    @Published var isSubscribed: Bool
    @Published var didFinishInitialCheck = false

    private var transactionListener: Task<Void, Error>?

    init() {
        // Warm-start from UserDefaults to avoid paywall flash
        isSubscribed = UserDefaults.standard.bool(forKey: "isSubscribed")
    }

    func start() {
        transactionListener?.cancel()
        transactionListener = listenForTransactions()
        Task {
            await loadProducts()
            await refreshEntitlements()
        }
    }

    // MARK: - Load Products

    func loadProducts() async {
        do {
            let storeProducts = try await Product.products(for: SubscriptionIDs.all)
            products = storeProducts.sorted { $0.price < $1.price }
            productsByID = Dictionary(uniqueKeysWithValues: storeProducts.map { ($0.id, $0) })
        } catch {
            print("Failed to load products: \(error)")
        }
    }

    // MARK: - Purchase

    func purchase(_ product: Product) async throws -> Transaction? {
        let result = try await product.purchase()

        switch result {
        case .success(let verification):
            let transaction = try checkVerified(verification)
            await transaction.finish()
            await refreshEntitlements()
            return transaction

        case .userCancelled:
            return nil

        case .pending:
            return nil

        @unknown default:
            return nil
        }
    }

    // MARK: - Restore

    func restore() async throws {
        try await AppStore.sync()
        await refreshEntitlements()
    }

    // MARK: - Entitlements

    func refreshEntitlements() async {
        var validIDs: Set<String> = []

        for await result in Transaction.currentEntitlements {
            if let transaction = try? checkVerified(result) {
                validIDs.insert(transaction.productID)
            }
        }

        // Cold-start retry: if warm-start says subscribed but StoreKit returns empty
        if validIDs.isEmpty && isSubscribed && !didFinishInitialCheck {
            try? await Task.sleep(nanoseconds: 2_000_000_000)
            for await result in Transaction.currentEntitlements {
                if let transaction = try? checkVerified(result) {
                    validIDs.insert(transaction.productID)
                }
            }
        }

        purchasedProductIDs = validIDs
        let subscribed = !validIDs.isEmpty
        isSubscribed = subscribed
        UserDefaults.standard.set(subscribed, forKey: "isSubscribed")
        didFinishInitialCheck = true

        await syncToFirestore()
    }

    // MARK: - Transaction Listener

    private func listenForTransactions() -> Task<Void, Error> {
        Task.detached {
            for await result in Transaction.updates {
                if let transaction = try? self.checkVerified(result) {
                    await transaction.finish()
                    await self.refreshEntitlements()
                }
            }
        }
    }

    // MARK: - Verification

    private func checkVerified<T>(_ result: VerificationResult<T>) throws -> T {
        switch result {
        case .unverified(_, let error): throw error
        case .verified(let safe): return safe
        }
    }
}
```

## Firestore Subscription Sync

Sync subscription state to user's Firestore document:

```swift
extension StoreKitManager {
    func syncToFirestore() async {
        guard let uid = Auth.auth().currentUser?.uid else { return }
        guard let docID = await FirestoreManager.shared.getUserDocID(uid: uid) else { return }

        var snapshot: [String: Any] = [
            "subscriptionStatus": isSubscribed ? "active" : "expired",
            "updatedAt": FieldValue.serverTimestamp()
        ]

        // Get detailed status from subscription group
        if let statuses = try? await Product.SubscriptionInfo.status(for: SubscriptionIDs.groupID) {
            if let status = statuses.first {
                let renewalInfo = try? status.renewalInfo.payloadValue
                let transaction = try? status.transaction.payloadValue

                snapshot["subscriptionInterval"] = transaction?.productID.contains("yearly") == true ? "yearly" : "monthly"
                snapshot["willAutoRenew"] = renewalInfo?.willAutoRenew ?? false
                snapshot["expirationDate"] = transaction?.expirationDate?.timeIntervalSince1970

                switch status.state {
                case .subscribed: snapshot["subscriptionState"] = "subscribed"
                case .inGracePeriod: snapshot["subscriptionState"] = "grace_period"
                case .inBillingRetryPeriod: snapshot["subscriptionState"] = "billing_retry"
                case .revoked: snapshot["subscriptionState"] = "revoked"
                case .expired: snapshot["subscriptionState"] = "expired"
                default: snapshot["subscriptionState"] = "unknown"
                }
            }
        }

        try? await Firestore.firestore().collection("users").document(docID).updateData(snapshot)
    }
}
```

## Trial Manager Integration

```swift
@MainActor
final class TrialManager: ObservableObject {
    static let shared = TrialManager()

    @Published var isTrialActive = false
    @Published var daysLeft = 0
    @Published var totalTrialDays = 7

    private var listener: ListenerRegistration?
    private var timer: AnyCancellable?

    var expiresAt: Date?
    var serverSaysPaid = false

    var hasAccess: Bool { isTrialActive || StoreKitManager.shared.isSubscribed }

    func bind(to uid: String) {
        let db = Firestore.firestore()

        // Cache-first
        db.collection("users").whereField("firebaseUID", isEqualTo: uid).limit(to: 1)
            .getDocuments(source: .cache) { [weak self] snap, _ in
                self?.processSnapshot(snap)
            }

        // Realtime
        listener = db.collection("users").whereField("firebaseUID", isEqualTo: uid).limit(to: 1)
            .addSnapshotListener(includeMetadataChanges: true) { [weak self] snap, _ in
                self?.processSnapshot(snap)
            }

        // Minute-by-minute updates for countdown
        timer = Timer.publish(every: 60, on: .main, in: .common).autoconnect()
            .sink { [weak self] _ in self?.recalculate() }
    }

    func unbind() {
        listener?.remove()
        listener = nil
        timer?.cancel()
    }

    private func processSnapshot(_ snap: QuerySnapshot?) {
        guard let doc = snap?.documents.first else { return }
        let data = doc.data()
        serverSaysPaid = (data["subscriptionStatus"] as? String) == "active"
        if let ts = data["trialExpiresAt"] as? Timestamp {
            expiresAt = ts.dateValue()
        }
        recalculate()
    }

    private func recalculate() {
        guard let expires = expiresAt else { isTrialActive = false; return }
        let remaining = Calendar.current.dateComponents([.day], from: Date(), to: expires).day ?? 0
        daysLeft = max(0, remaining)
        isTrialActive = expires > Date()
    }
}
```

## Paywall View

```swift
struct SubscriptionsView: View {
    @EnvironmentObject var storeKit: StoreKitManager
    @EnvironmentObject var trialManager: TrialManager
    @Environment(\.dismiss) var dismiss

    var body: some View {
        VStack(spacing: 20) {
            // Feature highlights
            FeatureList()

            // Trial banner
            if trialManager.isTrialActive {
                TrialBanner(daysLeft: trialManager.daysLeft)
            }

            // Product cards
            ForEach(storeKit.products, id: \.id) { product in
                ProductCard(product: product) {
                    Task {
                        try? await storeKit.purchase(product)
                        if storeKit.isSubscribed { dismiss() }
                    }
                }
            }

            // Restore
            Button("Restore Purchases") {
                Task { try? await storeKit.restore() }
            }
            .font(.footnote)

            // Terms links
            HStack {
                Link("Terms", destination: URL(string: "https://yourapp.com/terms")!)
                Text("·")
                Link("Privacy", destination: URL(string: "https://yourapp.com/privacy")!)
            }
            .font(.caption)
        }
    }
}
```

## Feature Gating

```swift
func checkAccess(for feature: Feature) -> AccessResult {
    if StoreKitManager.shared.isSubscribed { return .granted }
    if TrialManager.shared.isTrialActive { return .granted }
    if feature.hasMonthlyLimit {
        let used = getUsageCount(for: feature)
        if used < feature.freeLimit { return .granted }
        return .limitReached
    }
    return .subscriptionRequired
}

enum AccessResult {
    case granted
    case limitReached
    case subscriptionRequired
}
```

## Subscription Status Display

```swift
struct ManageSubscriptionView: View {
    @EnvironmentObject var storeKit: StoreKitManager

    var body: some View {
        List {
            Section("Current Plan") {
                if storeKit.isSubscribed {
                    Label("Premium Active", systemImage: "checkmark.seal.fill")
                } else {
                    Label("Free Plan", systemImage: "xmark.circle")
                }
            }

            Section {
                Button("Manage in App Store") {
                    Task {
                        guard let scene = UIApplication.shared.connectedScenes.first as? UIWindowScene else { return }
                        try? await AppStore.showManageSubscriptions(in: scene)
                    }
                }
            }
        }
    }
}
```
