# Firebase Android Integration Reference

## Setup

Add `google-services.json` to `app/` (gitignore it). Apply plugins in `build.gradle.kts`:

```kotlin
plugins {
    id("com.google.gms.google-services")
    id("com.google.firebase.crashlytics")
}
```

## Remote Config

```kotlin
object AppRemoteConfig {
    private lateinit var remoteConfig: FirebaseRemoteConfig
    var openAIAPIKey: String = ""
        private set

    fun init(context: Context) {
        remoteConfig = Firebase.remoteConfig
        remoteConfig.setDefaultsAsync(mapOf(
            "openai_api_key" to "",
            "feature_x_enabled" to false
        ))
        refresh()
    }

    fun refresh() {
        remoteConfig.fetchAndActivate().addOnCompleteListener {
            openAIAPIKey = remoteConfig.getString("openai_api_key")
        }
    }

    suspend fun suspendRefreshOnce(): String {
        if (openAIAPIKey.isNotEmpty()) return openAIAPIKey
        remoteConfig.fetchAndActivate().await()
        openAIAPIKey = remoteConfig.getString("openai_api_key")
        return openAIAPIKey
    }
}
```

## Firestore Manager

```kotlin
object FirestoreManager {
    private val db = Firebase.firestore

    suspend fun <T> fetchCollection(path: String, transform: (DocumentSnapshot) -> T): List<T> {
        val snapshot = db.collection(path).orderBy("createdAt", Query.Direction.DESCENDING).get().await()
        return snapshot.documents.mapNotNull { doc -> runCatching { transform(doc) }.getOrNull() }
    }

    suspend fun getUserDoc(uid: String): DocumentSnapshot? {
        val snapshot = db.collection("users")
            .whereEqualTo("firebaseUID", uid)
            .limit(1).get().await()
        return snapshot.documents.firstOrNull()
    }

    fun listenToCollection(path: String, onChange: (QuerySnapshot) -> Unit): ListenerRegistration {
        return db.collection(path).addSnapshotListener { snap, _ ->
            snap?.let { onChange(it) }
        }
    }

    suspend fun updateDocument(path: String, docId: String, data: Map<String, Any>) {
        db.collection(path).document(docId).update(data).await()
    }
}
```

### Cache-First Pattern

```kotlin
fun bindWithCacheFirst(uid: String, onUpdate: (DocumentSnapshot) -> Unit) {
    // Immediate cache hit
    db.collection("users").whereEqualTo("firebaseUID", uid).limit(1)
        .get(Source.CACHE)
        .addOnSuccessListener { snap -> snap.documents.firstOrNull()?.let(onUpdate) }

    // Then realtime
    listener = db.collection("users").whereEqualTo("firebaseUID", uid).limit(1)
        .addSnapshotListener { snap, _ ->
            snap?.documents?.firstOrNull()?.let(onUpdate)
        }
}
```

## Authentication

### Email / Password

```kotlin
Firebase.auth.createUserWithEmailAndPassword(email, password)
    .addOnSuccessListener { result ->
        result.user?.sendEmailVerification()
    }

Firebase.auth.signInWithEmailAndPassword(email, password)
    .addOnSuccessListener { /* signed in */ }
    .addOnFailureListener { e -> /* handle error */ }
```

### Google Sign-In

```kotlin
val gso = GoogleSignInOptions.Builder(GoogleSignInOptions.DEFAULT_SIGN_IN)
    .requestIdToken(getString(R.string.default_web_client_id))
    .requestEmail()
    .build()
val googleSignInClient = GoogleSignIn.getClient(context, gso)

// Launch sign-in intent, then in result:
val account = GoogleSignIn.getSignedInAccountFromIntent(data).result
val credential = GoogleAuthProvider.getCredential(account.idToken, null)
Firebase.auth.signInWithCredential(credential).addOnSuccessListener { /* done */ }
```

### Facebook Sign-In

Add the dependency in `build.gradle.kts`:

```kotlin
implementation("com.facebook.android:facebook-login:18.0.3")
```

Add to `res/values/strings.xml`:

```xml
<string name="facebook_app_id">YOUR_APP_ID</string>
<string name="fb_login_protocol_scheme">fbYOUR_APP_ID</string>
<string name="facebook_client_token">YOUR_CLIENT_TOKEN</string>
```

Add to `AndroidManifest.xml` inside `<application>`:

```xml
<meta-data android:name="com.facebook.sdk.ApplicationId"
    android:value="@string/facebook_app_id" />
<meta-data android:name="com.facebook.sdk.ClientToken"
    android:value="@string/facebook_client_token" />

<activity android:name="com.facebook.FacebookActivity"
    android:configChanges="keyboard|keyboardHidden|screenLayout|screenSize|orientation" />
<activity android:name="com.facebook.CustomTabActivity" android:exported="true">
    <intent-filter>
        <action android:name="android.intent.action.VIEW" />
        <category android:name="android.intent.category.DEFAULT" />
        <category android:name="android.intent.category.BROWSABLE" />
        <data android:scheme="@string/fb_login_protocol_scheme" />
    </intent-filter>
</activity>
```

```kotlin
class FacebookSignInHelper(private val activity: ComponentActivity) {
    private val callbackManager = CallbackManager.Factory.create()

    fun signIn() {
        LoginManager.getInstance().registerCallback(callbackManager,
            object : FacebookCallback<LoginResult> {
                override fun onSuccess(result: LoginResult) {
                    val credential = FacebookAuthProvider.getCredential(result.accessToken.token)
                    Firebase.auth.signInWithCredential(credential)
                        .addOnSuccessListener { /* signed in */ }
                        .addOnFailureListener { e -> /* handle error */ }
                }
                override fun onCancel() { /* user cancelled */ }
                override fun onError(error: FacebookException) { /* handle error */ }
            }
        )
        LoginManager.getInstance().logInWithReadPermissions(activity, listOf("email", "public_profile"))
    }

    fun onActivityResult(requestCode: Int, resultCode: Int, data: Intent?) {
        callbackManager.onActivityResult(requestCode, resultCode, data)
    }
}
```

### Auth State Listener

```kotlin
Firebase.auth.addAuthStateListener { auth ->
    val user = auth.currentUser
    if (user != null) {
        TrialManager.bind(user.uid)
    } else {
        TrialManager.unbind()
    }
}
```

### Localized Error Messages

```kotlin
fun userFacingAuthMessage(exception: Exception): String {
    return when ((exception as? FirebaseAuthException)?.errorCode) {
        "ERROR_WRONG_PASSWORD" -> context.getString(R.string.auth_wrong_password)
        "ERROR_USER_NOT_FOUND" -> context.getString(R.string.auth_user_not_found)
        "ERROR_EMAIL_ALREADY_IN_USE" -> context.getString(R.string.auth_email_in_use)
        else -> context.getString(R.string.auth_unknown_error)
    }
}
```

## FCM Push Notifications

```kotlin
class AppMessagingService : FirebaseMessagingService() {

    override fun onNewToken(token: String) {
        super.onNewToken(token)
        updateTokenInFirestore(token)
    }

    override fun onMessageReceived(message: RemoteMessage) {
        super.onMessageReceived(message)
        // Default notification display handled by system
        // Custom handling for data messages:
        message.data["link"]?.let { link ->
            // Open deep link
        }
    }

    private fun updateTokenInFirestore(token: String) {
        val uid = Firebase.auth.currentUser?.uid ?: return
        val oldToken = getSharedPreferences("fcm", MODE_PRIVATE).getString("token", null)

        // Delete old token doc
        if (oldToken != null && oldToken != token) {
            Firebase.firestore.collection("fcmTokens").document(oldToken).delete()
        }

        // Save new token
        Firebase.firestore.collection("fcmTokens").document(token).set(mapOf(
            "token" to token,
            "platform" to "android",
            "userId" to uid,
            "updatedAt" to FieldValue.serverTimestamp()
        ))

        getSharedPreferences("fcm", MODE_PRIVATE).edit().putString("token", token).apply()
    }
}
```

### AndroidManifest

```xml
<service android:name=".services.AppMessagingService"
    android:exported="false">
    <intent-filter>
        <action android:name="com.google.firebase.MESSAGING_EVENT" />
    </intent-filter>
</service>

<meta-data
    android:name="com.google.firebase.messaging.default_notification_icon"
    android:resource="@drawable/ic_notification" />
```

## WorkManager Scheduled Notifications

```kotlin
object ReminderScheduler {
    fun scheduleDailyReminder(context: Context) {
        val now = Calendar.getInstance()
        val target = Calendar.getInstance().apply {
            set(Calendar.HOUR_OF_DAY, 12)
            set(Calendar.MINUTE, 30)
            set(Calendar.SECOND, 0)
            if (before(now)) add(Calendar.DAY_OF_YEAR, 1)
        }
        val delay = target.timeInMillis - now.timeInMillis

        val request = OneTimeWorkRequestBuilder<ReminderWorker>()
            .setInitialDelay(delay, TimeUnit.MILLISECONDS)
            .addTag("daily_reminder")
            .build()

        WorkManager.getInstance(context)
            .enqueueUniqueWork("daily_reminder", ExistingWorkPolicy.REPLACE, request)
    }

    fun cancel(context: Context) {
        WorkManager.getInstance(context).cancelUniqueWork("daily_reminder")
    }
}

class ReminderWorker(context: Context, params: WorkerParameters) : Worker(context, params) {
    override fun doWork(): Result {
        val messages = applicationContext.resources.getStringArray(R.array.reminder_messages)
        val body = messages.random()

        val notification = NotificationCompat.Builder(applicationContext, "reminders")
            .setSmallIcon(R.drawable.ic_notification)
            .setContentTitle(applicationContext.getString(R.string.reminder_title))
            .setContentText(body)
            .setAutoCancel(true)
            .build()

        val manager = applicationContext.getSystemService(NotificationManager::class.java)
        manager.notify(1001, notification)

        // Reschedule for tomorrow
        ReminderScheduler.scheduleDailyReminder(applicationContext)
        return Result.success()
    }
}
```

### Notification Types

| Type | Scheduler | Timing |
|------|-----------|--------|
| Daily reminder | `ReminderScheduler` | Fixed time daily |
| Streak reminder | `StreakScheduler` | Evening |
| Trial drip | `TrialDripScheduler` | Days 1, 3, 5 of trial |
| Trial last day | `TrialLastDayScheduler` | Last trial day |
| Win-back | `WinBackScheduler` | 3, 7 days after churn |
| Verification | `VerificationScheduler` | Day after signup |

### Notification Channels

Create channels in `Application.onCreate()` or `MainActivity`:

```kotlin
if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.O) {
    val channel = NotificationChannel("reminders", "Reminders", NotificationManager.IMPORTANCE_DEFAULT)
    getSystemService(NotificationManager::class.java).createNotificationChannel(channel)
}
```

## Storage Uploads

```kotlin
suspend fun uploadImage(uri: Uri, path: String): String {
    val ref = Firebase.storage.reference.child(path)
    ref.putFile(uri).await()
    return ref.downloadUrl.await().toString()
}
```

## Analytics

```kotlin
Firebase.analytics.logEvent("item_scanned") { param("type", "camera") }
Firebase.analytics.logEvent("subscription_started") { param("plan", "monthly") }
```
