---
name: android-app-builder
description: >-
  Build production-ready Android apps with Kotlin, Jetpack Compose, and MVVM
  architecture. Covers Firebase backend (Auth, Firestore, Storage, Analytics,
  Crashlytics, Messaging, Remote Config), OpenAI/ChatGPT integration with
  vision API via OkHttp, Google Play Billing (subscriptions with trial
  management), in-app review prompts, in-app updates, Health Connect,
  WorkManager for scheduled notifications, App Widgets, live notification
  trackers via foreground service, onboarding flows, offline caching,
  Material 3 theming with dark mode and dynamic color, localization,
  biometric/pattern lock, deep linking, CameraX, and Lottie animations.
  Use when the user asks to build an Android app, Kotlin app, or any Jetpack
  Compose project with Firebase, AI, subscriptions, or similar features.
---

# Android App Builder

Build production-ready Android apps with Kotlin, Jetpack Compose, Material 3, and a services layer.

## Architecture

**MVVM + Services** — no DI framework (no Hilt/Koin/Dagger). ViewModels extend `AndroidViewModel` or use Compose state. Services are singletons accessed directly.

```
app/src/main/java/com/yourpackage/appname/
├── MainActivity.kt                   # Main activity, nav, splash, connectivity
├── AppNameApp.kt                     # Application class, Remote Config init
├── components/                       # Reusable UI components
│   ├── InAppReviewPrompt.kt          # Play Core review
│   ├── PatternLock.kt                # Pattern lock UI
│   ├── BiometricAuthActivity.kt      # Biometric unlock
│   ├── CameraPreview.kt             # CameraX preview
│   ├── GettingStartedCard.kt        # Onboarding checklist
│   └── Cards/                        # Dashboard cards
├── models/                           # Data models + ViewModels
│   ├── SomeViewModel.kt
│   └── SomeModel.kt
├── screens/                          # Full-screen Composables
│   ├── DashboardView.kt
│   ├── OnboardingView.kt            # Paged onboarding + OnboardingActivity
│   ├── SplashScreenView.kt
│   ├── SignInView.kt
│   ├── SignUpView.kt
│   ├── SubscriptionsView.kt
│   ├── ChatView.kt                   # AI chat
│   └── ...EntityViews.kt
├── scanners/                          # CameraX scan views
│   ├── SomeScannerView.kt
│   └── ScanResultSheet.kt
├── services/                          # Singletons + managers
│   ├── ChatGPTService.kt             # OpenAI chat (OkHttp)
│   ├── VisionGPTService.kt           # OpenAI vision
│   ├── FirestoreManager.kt           # All Firestore ops
│   ├── GooglePlayBillingManager.kt   # Play Billing
│   ├── TrialManager.kt              # Trial state
│   ├── AIConsentManager.kt          # AI consent gate
│   ├── FirebaseMessagingService.kt   # FCM
│   ├── FirebaseRemoteConfig.kt      # Remote Config
│   ├── NetworkMonitor.kt            # Connectivity
│   ├── HealthConnectService.kt      # Health Connect
│   ├── PlayInAppUpdateManager.kt    # In-app updates
│   ├── PushNotificationLinkActivity.kt # Deep link from push
│   ├── ReminderScheduler.kt         # WorkManager schedulers
│   ├── TrialDripScheduler.kt
│   ├── WinBackScheduler.kt
│   └── liveactivity/
│       ├── LiveActivityManager.kt
│       ├── LiveActivityService.kt     # Foreground service
│       └── LiveActivityNotificationRenderer.kt
├── widgets/                           # App Widgets
│   ├── SomeWidgetProvider.kt
│   ├── WidgetDataUpdater.kt
│   └── WidgetRingRenderer.kt
├── ui/theme/
│   ├── Color.kt
│   ├── Theme.kt                      # Material 3 + dynamic color
│   └── Type.kt
└── res/
    ├── layout/                        # Widget layouts (XML)
    ├── raw/                           # Lottie JSON
    ├── values/strings.xml
    ├── values-night/colors.xml
    ├── values-{locale}/strings.xml
    └── xml/
        ├── shortcuts.xml
        └── *_widget_info.xml
```

## Dependencies (Version Catalog)

Use `gradle/libs.versions.toml`:

```toml
[versions]
agp = "9.1.0"
kotlin = "2.2.10"
compose-bom = "2025.05.01"
firebase-bom = "33.14.0"
billing = "8.3.0"
okhttp = "5.3.2"
coil = "3.2.0"
lottie = "6.4.0"
camerax = "1.5.0"

[libraries]
# Compose
compose-bom = { module = "androidx.compose:compose-bom", version.ref = "compose-bom" }
compose-ui = { module = "androidx.compose.ui:ui" }
compose-material3 = { module = "androidx.compose.material3:material3" }
compose-material-icons = { module = "androidx.compose.material:material-icons-extended" }

# Firebase
firebase-bom = { module = "com.google.firebase:firebase-bom", version.ref = "firebase-bom" }
firebase-auth = { module = "com.google.firebase:firebase-auth" }
firebase-firestore = { module = "com.google.firebase:firebase-firestore" }
firebase-storage = { module = "com.google.firebase:firebase-storage" }
firebase-analytics = { module = "com.google.firebase:firebase-analytics" }
firebase-crashlytics = { module = "com.google.firebase:firebase-crashlytics" }
firebase-messaging = { module = "com.google.firebase:firebase-messaging" }
firebase-config = { module = "com.google.firebase:firebase-config" }

# Play
billing = { module = "com.android.billingclient:billing-ktx", version.ref = "billing" }
review = { module = "com.google.android.play:review-ktx", version = "2.0.2" }
app-update = { module = "com.google.android.play:app-update-ktx", version = "2.1.0" }

# Network
okhttp = { module = "com.squareup.okhttp3:okhttp", version.ref = "okhttp" }
gson = { module = "com.google.code.gson:gson", version = "2.13.2" }

# Image
coil-compose = { module = "io.coil-kt.coil3:coil-compose", version.ref = "coil" }

# Animation
lottie-compose = { module = "com.airbnb.android:lottie-compose", version.ref = "lottie" }

# Camera
camera-core = { module = "androidx.camera:camera-core", version.ref = "camerax" }
camera-lifecycle = { module = "androidx.camera:camera-lifecycle", version.ref = "camerax" }
camera-view = { module = "androidx.camera:camera-view", version.ref = "camerax" }

# Other
work-runtime = { module = "androidx.work:work-runtime-ktx", version = "2.11.1" }
biometric = { module = "androidx.biometric:biometric", version = "1.4.0-alpha02" }
navigation-compose = { module = "androidx.navigation:navigation-compose", version = "2.9.0" }
lifecycle-viewmodel-compose = { module = "androidx.lifecycle:lifecycle-viewmodel-compose" }
splashscreen = { module = "androidx.core:core-splashscreen", version = "1.2.0-alpha02" }
```

No Retrofit — use OkHttp + Gson directly for API calls.

## Application Class

```kotlin
class AppNameApp : Application() {
    override fun onCreate() {
        super.onCreate()
        AppRemoteConfig.init(this)
    }
}
```

## Main Activity

```kotlin
class MainActivity : ComponentActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        val splashScreen = installSplashScreen()
        enableEdgeToEdge()
        super.onCreate(savedInstanceState)

        setContent {
            AppTheme {
                val navController = rememberNavController()
                val networkMonitor = remember { NetworkMonitor(this) }
                val isOnline by networkMonitor.isConnected.collectAsState()

                Box {
                    MainNavHost(navController)
                    if (!isOnline) {
                        OfflineBanner(modifier = Modifier.align(Alignment.TopCenter))
                    }
                }
            }
        }
    }
}
```

## Navigation

```kotlin
@Composable
fun MainNavHost(navController: NavHostController) {
    NavHost(navController, startDestination = "dashboard") {
        composable("dashboard") { DashboardView(navController) }
        composable("profile") { UserProfileView(navController) }
        composable("chat") { ChatView() }
        composable("subscriptions") { SubscriptionsView() }
        composable("notifications") { NotificationsView() }
    }
}
```

Activities for complex screens (scanners, forms):

```kotlin
val intent = Intent(context, SomeScannerActivity::class.java)
context.startActivity(intent)
```

## Onboarding

Paged with `HorizontalPager` + Lottie:

```kotlin
@Composable
fun OnboardingScreen(onComplete: () -> Unit) {
    val pagerState = rememberPagerState(pageCount = { 5 })
    val scope = rememberCoroutineScope()

    Box(Modifier.fillMaxSize()) {
        HorizontalPager(state = pagerState) { page ->
            OnboardingPage(page)
        }
        Row(Modifier.align(Alignment.BottomCenter).padding(16.dp)) {
            TextButton(onClick = onComplete) { Text("Skip") }
            Spacer(Modifier.weight(1f))
            Button(onClick = {
                if (pagerState.currentPage < 4) scope.launch { pagerState.animateScrollToPage(pagerState.currentPage + 1) }
                else onComplete()
            }) {
                Icon(if (pagerState.currentPage < 4) Icons.Default.ArrowForward else Icons.Default.Check, null)
            }
        }
    }
}
```

## In-App Review Prompt

```kotlin
object ReviewManagerHelper {
    fun requestInAppReviewIfEligible(activity: Activity) {
        val prefs = activity.getSharedPreferences("review_prefs", Context.MODE_PRIVATE)
        val currentVersion = activity.packageManager
            .getPackageInfo(activity.packageName, 0).versionName
        if (prefs.getString("prompted_version", "") == currentVersion) return

        val manager = ReviewManagerFactory.create(activity)
        manager.requestReviewFlow().addOnSuccessListener { reviewInfo ->
            manager.launchReviewFlow(activity, reviewInfo).addOnCompleteListener {
                prefs.edit().putString("prompted_version", currentVersion).apply()
            }
        }
    }

    fun openPlayStoreListing(context: Context) {
        val uri = Uri.parse("market://details?id=${context.packageName}")
        val intent = Intent(Intent.ACTION_VIEW, uri).apply {
            addFlags(Intent.FLAG_ACTIVITY_NEW_TASK)
        }
        try { context.startActivity(intent) }
        catch (_: ActivityNotFoundException) {
            context.startActivity(Intent(Intent.ACTION_VIEW,
                Uri.parse("https://play.google.com/store/apps/details?id=${context.packageName}")))
        }
    }
}
```

**Triggers:** first content saved, onboarding completed, streak milestones.

## In-App Updates

```kotlin
class PlayInAppUpdateManager(private val activity: ComponentActivity) {
    private val appUpdateManager = AppUpdateManagerFactory.create(activity)
    private var launcher: ActivityResultLauncher<IntentSenderRequest>? = null

    fun registerLauncher() {
        launcher = activity.registerForActivityResult(
            ActivityResultContracts.StartIntentSenderForResult()
        ) { result ->
            if (result.resultCode == Activity.RESULT_OK) { /* updated */ }
        }
    }

    fun checkForUpdate() {
        appUpdateManager.appUpdateInfo.addOnSuccessListener { info ->
            if (info.updateAvailability() == UpdateAvailability.UPDATE_AVAILABLE) {
                val stale = info.clientVersionStalenessDays() ?: 0
                val priority = info.updatePriority()
                if (stale >= 5 || priority >= 4) startImmediate(info)
                else startFlexible(info)
            }
        }
    }

    private fun startImmediate(info: AppUpdateInfo) {
        appUpdateManager.startUpdateFlowForResult(info, launcher!!,
            AppUpdateOptions.newBuilder(AppUpdateType.IMMEDIATE).build())
    }

    private fun startFlexible(info: AppUpdateInfo) {
        appUpdateManager.startUpdateFlowForResult(info, launcher!!,
            AppUpdateOptions.newBuilder(AppUpdateType.FLEXIBLE).build())
        appUpdateManager.registerListener { state ->
            if (state.installStatus() == InstallStatus.DOWNLOADED) {
                appUpdateManager.completeUpdate()
            }
        }
    }
}
```

## Network Monitor

```kotlin
class NetworkMonitor(context: Context) {
    private val connectivityManager = context.getSystemService(ConnectivityManager::class.java)
    private val _isConnected = MutableStateFlow(current())
    val isConnected: StateFlow<Boolean> = _isConnected.asStateFlow()

    init {
        connectivityManager.registerDefaultNetworkCallback(object : ConnectivityManager.NetworkCallback() {
            override fun onAvailable(network: Network) { _isConnected.value = true }
            override fun onLost(network: Network) { _isConnected.value = false }
            override fun onCapabilitiesChanged(network: Network, capabilities: NetworkCapabilities) {
                _isConnected.value = capabilities.hasCapability(NetworkCapabilities.NET_CAPABILITY_VALIDATED)
            }
        })
    }

    private fun current(): Boolean {
        val capabilities = connectivityManager.getNetworkCapabilities(connectivityManager.activeNetwork)
        return capabilities?.hasCapability(NetworkCapabilities.NET_CAPABILITY_INTERNET) == true
                && capabilities.hasCapability(NetworkCapabilities.NET_CAPABILITY_VALIDATED)
    }
}
```

## AI Consent Management

```kotlin
class AIConsentManager(private val context: Context) {
    private val prefs = context.getSharedPreferences("ai_consent", Context.MODE_PRIVATE)

    fun hasConsented(uid: String): Boolean = prefs.getBoolean("consent_$uid", false)

    fun grant(uid: String) {
        prefs.edit().putBoolean("consent_$uid", true).apply()
        syncToFirestore(uid, true)
    }

    fun revoke(uid: String) {
        prefs.edit().putBoolean("consent_$uid", false).apply()
        syncToFirestore(uid, false)
    }

    private fun syncToFirestore(uid: String, consented: Boolean) {
        FirebaseFirestore.getInstance().collection("users")
            .whereEqualTo("firebaseUID", uid).limit(1).get()
            .addOnSuccessListener { snap ->
                snap.documents.firstOrNull()?.reference?.update("aiDataSharingConsent", consented)
            }
    }
}
```

## Dark Mode + Dynamic Color

```kotlin
@Composable
fun AppTheme(
    darkTheme: Boolean = isSystemInDarkTheme(),
    dynamicColor: Boolean = true,
    content: @Composable () -> Unit
) {
    val colorScheme = when {
        dynamicColor && Build.VERSION.SDK_INT >= Build.VERSION_CODES.S ->
            if (darkTheme) dynamicDarkColorScheme(LocalContext.current)
            else dynamicLightColorScheme(LocalContext.current)
        darkTheme -> darkColorScheme()
        else -> lightColorScheme()
    }
    MaterialTheme(colorScheme = colorScheme, typography = Typography, content = content)
}
```

Lottie: load `animation_light.json` / `animation_dark.json` based on `isSystemInDarkTheme()`.

## Localization

Use `values-{locale}/strings.xml`. Supported locales configured in `build.gradle.kts`:

```kotlin
resourceConfigurations += listOf("en", "es", "de", "fr", "ru", "ja", "ko", "pt-rBR", "zh-rCN", "iw")
```

## Deep Linking

**Shortcuts** (`res/xml/shortcuts.xml`):

```xml
<shortcuts xmlns:android="http://schemas.android.com/apk/res/android">
    <shortcut android:shortcutId="scan"
        android:shortcutShortLabel="@string/shortcut_scan"
        android:icon="@drawable/ic_scan">
        <intent android:action="android.intent.action.VIEW"
            android:targetPackage="com.yourpackage.appname"
            android:targetClass="com.yourpackage.appname.scanners.ScannerActivity" />
    </shortcut>
</shortcuts>
```

**Push deep link:** `PushNotificationLinkActivity` reads `link` from intent extras and opens it.

## Biometric / Pattern Lock

```kotlin
val promptInfo = BiometricPrompt.PromptInfo.Builder()
    .setTitle("Unlock")
    .setNegativeButtonText("Use password")
    .setAllowedAuthenticators(BiometricManager.Authenticators.BIOMETRIC_STRONG)
    .build()

BiometricPrompt(activity, executor, callback).authenticate(promptInfo)
```

Pattern lock: custom Compose canvas drawing with gesture detection.

## ProGuard / R8

```proguard
-keep class com.yourpackage.appname.services.ChatGPTMessage { *; }
-keep class com.yourpackage.appname.services.OpenAIChatRequest { *; }
-keep class com.yourpackage.appname.services.OpenAIChatResponse { *; }
-keep class com.yourpackage.appname.services.OpenAIAPIError { *; }
-keepattributes Signature
-keep class com.google.gson.reflect.TypeToken { *; }
```

## Additional References

- **Firebase Android** (Auth, Firestore, FCM, Remote Config): [firebase-android.md](firebase-android.md)
- **OpenAI integration** (OkHttp, chat, vision, retries): [openai-android.md](openai-android.md)
- **Google Play Billing** (subscriptions, trial, Firestore sync): [play-billing.md](play-billing.md)
- **Widgets & live notifications** (AppWidgetProvider, foreground service): [widgets-livenotifications.md](widgets-livenotifications.md)
