# Google Play Billing Reference

Patterns for implementing auto-renewable subscriptions with Play Billing Library v8 and Firestore sync.

## Dependencies

```kotlin
implementation("com.android.billingclient:billing-ktx:8.3.0")
```

## Product IDs

```kotlin
object SubscriptionIDs {
    const val MONTHLY = "premium.monthly"
    const val YEARLY = "premium.yearly"
    val ALL = listOf(MONTHLY, YEARLY)
}
```

## Billing Manager

```kotlin
class GooglePlayBillingManager(application: Application) : AndroidViewModel(application) {

    data class PlanOfferUi(
        val productDetails: ProductDetails,
        val offerToken: String,
        val pricingPhase: ProductDetails.PricingPhase,
        val period: String // "P1M" or "P1Y"
    )

    sealed class BillingUiState {
        object Loading : BillingUiState()
        data class Ready(val plans: List<PlanOfferUi>) : BillingUiState()
        data class Subscribed(val productId: String) : BillingUiState()
        data class Error(val message: String) : BillingUiState()
    }

    private val _uiState = MutableStateFlow<BillingUiState>(BillingUiState.Loading)
    val uiState: StateFlow<BillingUiState> = _uiState.asStateFlow()

    private val _isSubscribed = MutableStateFlow(false)
    val isSubscribed: StateFlow<Boolean> = _isSubscribed.asStateFlow()

    private var billingClient: BillingClient? = null
    private val readyDeferred = CompletableDeferred<Unit>()
    private var productDetailsList: List<ProductDetails> = emptyList()

    private val purchasesUpdatedListener = PurchasesUpdatedListener { billingResult, purchases ->
        if (billingResult.responseCode == BillingClient.BillingResponseCode.OK && purchases != null) {
            viewModelScope.launch {
                for (purchase in purchases) {
                    handlePurchase(purchase)
                }
            }
        }
    }

    fun start() {
        billingClient = BillingClient.newBuilder(getApplication())
            .setListener(purchasesUpdatedListener)
            .enablePendingPurchases(PendingPurchasesParams.newBuilder().enableOneTimeProducts().build())
            .build()

        billingClient?.startConnection(object : BillingClientStateListener {
            override fun onBillingSetupFinished(result: BillingResult) {
                if (result.responseCode == BillingClient.BillingResponseCode.OK) {
                    readyDeferred.complete(Unit)
                    viewModelScope.launch {
                        queryProducts()
                        queryOwnedPurchases()
                    }
                }
            }
            override fun onBillingServiceDisconnected() {
                // Retry connection
            }
        })

        // Timeout after 12 seconds
        viewModelScope.launch {
            withTimeoutOrNull(12_000) { readyDeferred.await() }
        }
    }

    // MARK: - Query Products

    private suspend fun queryProducts() {
        readyDeferred.await()
        val params = QueryProductDetailsParams.newBuilder()
            .setProductList(SubscriptionIDs.ALL.map { id ->
                QueryProductDetailsParams.Product.newBuilder()
                    .setProductId(id)
                    .setProductType(BillingClient.ProductType.SUBS)
                    .build()
            })
            .build()

        val result = billingClient?.queryProductDetails(params) ?: return
        productDetailsList = result.productDetailsList ?: emptyList()

        val plans = productDetailsList.mapNotNull { details ->
            val offer = details.subscriptionOfferDetails?.firstOrNull() ?: return@mapNotNull null
            val pricing = offer.pricingPhases.pricingPhaseList.firstOrNull() ?: return@mapNotNull null
            PlanOfferUi(
                productDetails = details,
                offerToken = offer.offerToken,
                pricingPhase = pricing,
                period = pricing.billingPeriod // "P1M" or "P1Y"
            )
        }.sortedBy { it.pricingPhase.priceAmountMicros }

        _uiState.value = BillingUiState.Ready(plans)
    }

    // MARK: - Purchase

    fun purchase(activity: Activity, plan: PlanOfferUi) {
        val flowParams = BillingFlowParams.newBuilder()
            .setProductDetailsParamsList(listOf(
                BillingFlowParams.ProductDetailsParams.newBuilder()
                    .setProductDetails(plan.productDetails)
                    .setOfferToken(plan.offerToken)
                    .build()
            ))
            .build()

        billingClient?.launchBillingFlow(activity, flowParams)
    }

    // MARK: - Handle Purchase

    private suspend fun handlePurchase(purchase: Purchase) {
        if (purchase.purchaseState != Purchase.PurchaseState.PURCHASED) return

        // Acknowledge
        if (!purchase.isAcknowledged) {
            val params = AcknowledgePurchaseParams.newBuilder()
                .setPurchaseToken(purchase.purchaseToken)
                .build()
            billingClient?.acknowledgePurchase(params)
        }

        _isSubscribed.value = true
        syncToFirestore(purchase)
    }

    // MARK: - Query Owned

    private suspend fun queryOwnedPurchases() {
        readyDeferred.await()
        val params = QueryPurchasesParams.newBuilder()
            .setProductType(BillingClient.ProductType.SUBS)
            .build()

        val result = billingClient?.queryPurchasesAsync(params) ?: return
        val activePurchases = result.purchasesList.filter {
            it.purchaseState == Purchase.PurchaseState.PURCHASED
        }

        _isSubscribed.value = activePurchases.isNotEmpty()
        activePurchases.forEach { syncToFirestore(it) }

        if (activePurchases.isNotEmpty()) {
            _uiState.value = BillingUiState.Subscribed(activePurchases.first().products.first())
        }
    }

    // MARK: - Firestore Sync

    private fun syncToFirestore(purchase: Purchase) {
        val uid = Firebase.auth.currentUser?.uid ?: return
        Firebase.firestore.collection("users")
            .whereEqualTo("firebaseUID", uid).limit(1).get()
            .addOnSuccessListener { snap ->
                val docRef = snap.documents.firstOrNull()?.reference ?: return@addOnSuccessListener
                docRef.update(mapOf(
                    "subscriptionStatus" to "active",
                    "subscriptionProductId" to purchase.products.firstOrNull(),
                    "playPurchaseToken" to purchase.purchaseToken,
                    "subscriptionUpdatedAt" to FieldValue.serverTimestamp()
                ))
            }
    }

    override fun onCleared() {
        billingClient?.endConnection()
    }
}
```

## Trial Manager

```kotlin
class TrialManager {
    data class TrialState(
        val isLoading: Boolean = true,
        val isInTrial: Boolean = false,
        val hasExpired: Boolean = false,
        val daysLeft: Int = 0,
        val subscriptionStatus: String = ""
    )

    private val _state = MutableStateFlow(TrialState())
    val state: StateFlow<TrialState> = _state.asStateFlow()

    val hasAccess: Boolean get() = _state.value.isInTrial || billingManager.isSubscribed.value

    private var listener: ListenerRegistration? = null

    fun bind(uid: String) {
        val db = Firebase.firestore

        // Cache-first
        db.collection("users").whereEqualTo("firebaseUID", uid).limit(1)
            .get(Source.CACHE)
            .addOnSuccessListener { processSnapshot(it) }

        // Realtime
        listener = db.collection("users").whereEqualTo("firebaseUID", uid).limit(1)
            .addSnapshotListener { snap, _ -> snap?.let { processSnapshot(it) } }
    }

    fun unbind() { listener?.remove(); listener = null }

    private fun processSnapshot(snap: QuerySnapshot) {
        val doc = snap.documents.firstOrNull() ?: return
        val data = doc.data ?: return

        val status = data["subscriptionStatus"] as? String ?: ""
        val isPaid = status in listOf("active", "paid", "premium")
        val trialEndsAt = (data["trialExpiresAt"] as? com.google.firebase.Timestamp)?.toDate()
            ?: (data["trialEndsAt"] as? com.google.firebase.Timestamp)?.toDate()

        val now = Date()
        val isInTrial = !isPaid && trialEndsAt != null && now.before(trialEndsAt)
        val daysLeft = if (trialEndsAt != null) {
            val diff = trialEndsAt.time - now.time
            maxOf(0, (diff / (1000 * 60 * 60 * 24)).toInt())
        } else 0

        _state.value = TrialState(
            isLoading = false,
            isInTrial = isInTrial,
            hasExpired = !isPaid && !isInTrial,
            daysLeft = daysLeft,
            subscriptionStatus = status
        )

        // Write expired status if trial ended
        if (!isPaid && !isInTrial && status != "expired") {
            doc.reference.update("subscriptionStatus", "expired")
        }
    }
}
```

## Paywall View

```kotlin
@Composable
fun SubscriptionsView(
    billingManager: GooglePlayBillingManager,
    onDismiss: () -> Unit
) {
    val uiState by billingManager.uiState.collectAsState()
    val context = LocalContext.current

    Column(modifier = Modifier.fillMaxSize().padding(16.dp)) {
        // Feature highlights
        FeatureList()

        when (val state = uiState) {
            is GooglePlayBillingManager.BillingUiState.Loading ->
                CircularProgressIndicator(Modifier.align(Alignment.CenterHorizontally))

            is GooglePlayBillingManager.BillingUiState.Ready -> {
                state.plans.forEach { plan ->
                    PlanCard(plan) {
                        billingManager.purchase(context as Activity, plan)
                    }
                }
            }

            is GooglePlayBillingManager.BillingUiState.Subscribed -> {
                Text("Premium Active", style = MaterialTheme.typography.headlineMedium)
                onDismiss()
            }

            is GooglePlayBillingManager.BillingUiState.Error -> {
                Text(state.message, color = MaterialTheme.colorScheme.error)
            }
        }

        Spacer(Modifier.weight(1f))

        // Restore
        TextButton(onClick = { billingManager.viewModelScope.launch { billingManager.queryOwnedPurchases() } }) {
            Text("Restore Purchases")
        }

        // Terms
        Row(horizontalArrangement = Arrangement.Center, modifier = Modifier.fillMaxWidth()) {
            TextButton(onClick = { /* open terms URL */ }) { Text("Terms", style = MaterialTheme.typography.labelSmall) }
            Text(" · ", style = MaterialTheme.typography.labelSmall)
            TextButton(onClick = { /* open privacy URL */ }) { Text("Privacy", style = MaterialTheme.typography.labelSmall) }
        }
    }
}
```

## Feature Gating

```kotlin
fun checkAccess(billingManager: GooglePlayBillingManager, trialManager: TrialManager): AccessResult {
    if (billingManager.isSubscribed.value) return AccessResult.GRANTED
    if (trialManager.state.value.isInTrial) return AccessResult.GRANTED
    return AccessResult.SUBSCRIPTION_REQUIRED
}

enum class AccessResult {
    GRANTED, SUBSCRIPTION_REQUIRED, LIMIT_REACHED
}
```

## Manage Subscription

```kotlin
// Open Play Store subscription management
val intent = Intent(Intent.ACTION_VIEW, Uri.parse(
    "https://play.google.com/store/account/subscriptions"
))
context.startActivity(intent)
```
