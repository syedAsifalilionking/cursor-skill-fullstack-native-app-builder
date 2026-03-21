# Widgets & Live Notification Trackers Reference

Patterns for App Widgets and foreground-service-based live notification trackers on Android.

## Widget Data Bridge

SharedPreferences bridge between main app and widgets:

```kotlin
object WidgetDataUpdater {
    private const val PREFS_NAME = "widget_data"

    private fun prefs(context: Context) =
        context.getSharedPreferences(PREFS_NAME, Context.MODE_PRIVATE)

    // Write from main app
    fun updateProgress(context: Context, current: Int, goal: Int) {
        prefs(context).edit()
            .putInt("progress_current", current)
            .putInt("progress_goal", goal)
            .apply()
        refreshWidget(context, ProgressWidgetProvider::class.java)
        LiveActivityManager.updateProgress(context, current, goal)
    }

    fun updateStreak(context: Context, current: Int, longest: Int) {
        prefs(context).edit()
            .putInt("streak_current", current)
            .putInt("streak_longest", longest)
            .apply()
        refreshWidget(context, StreakWidgetProvider::class.java)
    }

    // Read from widget
    fun getProgressCurrent(context: Context) = prefs(context).getInt("progress_current", 0)
    fun getProgressGoal(context: Context) = prefs(context).getInt("progress_goal", 100)
    fun getStreakCurrent(context: Context) = prefs(context).getInt("streak_current", 0)
    fun getStreakLongest(context: Context) = prefs(context).getInt("streak_longest", 0)

    // Force widget refresh
    private fun refreshWidget(context: Context, providerClass: Class<out AppWidgetProvider>) {
        val manager = AppWidgetManager.getInstance(context)
        val ids = manager.getAppWidgetIds(ComponentName(context, providerClass))
        val intent = Intent(AppWidgetManager.ACTION_APPWIDGET_UPDATE).apply {
            putExtra(AppWidgetManager.EXTRA_APPWIDGET_IDS, ids)
            component = ComponentName(context, providerClass)
        }
        context.sendBroadcast(intent)
    }
}
```

## Widget Ring Renderer

Custom canvas drawing for circular progress in widgets:

```kotlin
object WidgetRingRenderer {
    fun renderRing(context: Context, sizeDp: Int, progress: Float, color: Int): Bitmap {
        val density = context.resources.displayMetrics.density
        val sizePx = (sizeDp * density).toInt()
        val bitmap = Bitmap.createBitmap(sizePx, sizePx, Bitmap.Config.ARGB_8888)
        val canvas = Canvas(bitmap)
        val strokeWidth = 8f * density
        val padding = strokeWidth / 2

        // Background ring
        val bgPaint = Paint(Paint.ANTI_ALIAS_FLAG).apply {
            this.color = Color.parseColor("#E0E0E0")
            style = Paint.Style.STROKE
            this.strokeWidth = strokeWidth
        }
        val rect = RectF(padding, padding, sizePx - padding, sizePx - padding)
        canvas.drawArc(rect, 0f, 360f, false, bgPaint)

        // Progress ring
        val progressPaint = Paint(Paint.ANTI_ALIAS_FLAG).apply {
            this.color = color
            style = Paint.Style.STROKE
            this.strokeWidth = strokeWidth
            strokeCap = Paint.Cap.ROUND
        }
        canvas.drawArc(rect, -90f, 360f * progress.coerceIn(0f, 1f), false, progressPaint)

        return bitmap
    }
}
```

## App Widget Provider

```kotlin
class ProgressWidgetProvider : AppWidgetProvider() {

    override fun onUpdate(context: Context, appWidgetManager: AppWidgetManager, appWidgetIds: IntArray) {
        for (widgetId in appWidgetIds) {
            updateWidget(context, appWidgetManager, widgetId)
        }
    }

    override fun onReceive(context: Context, intent: Intent) {
        super.onReceive(context, intent)
        if (intent.action == ACTION_QUICK_ADD) {
            handleQuickAdd(context)
            // Refresh all widgets
            val manager = AppWidgetManager.getInstance(context)
            val ids = manager.getAppWidgetIds(ComponentName(context, ProgressWidgetProvider::class.java))
            ids.forEach { updateWidget(context, manager, it) }
        }
    }

    private fun updateWidget(context: Context, manager: AppWidgetManager, widgetId: Int) {
        val current = WidgetDataUpdater.getProgressCurrent(context)
        val goal = WidgetDataUpdater.getProgressGoal(context)
        val progress = if (goal > 0) current.toFloat() / goal else 0f

        // Choose layout based on widget size
        val options = manager.getAppWidgetOptions(widgetId)
        val minWidth = options.getInt(AppWidgetManager.OPTION_APPWIDGET_MIN_WIDTH, 0)
        val layoutId = if (minWidth >= 200) R.layout.widget_medium else R.layout.widget_small

        val views = RemoteViews(context.packageName, layoutId).apply {
            setTextViewText(R.id.tv_value, "$current")
            setTextViewText(R.id.tv_goal, "/ $goal")
            setImageViewBitmap(R.id.iv_ring, WidgetRingRenderer.renderRing(context, 80, progress, Color.BLUE))

            // Quick-add button
            val addIntent = Intent(context, ProgressWidgetProvider::class.java).apply {
                action = ACTION_QUICK_ADD
            }
            setOnClickPendingIntent(R.id.btn_add, PendingIntent.getBroadcast(
                context, 0, addIntent, PendingIntent.FLAG_UPDATE_CURRENT or PendingIntent.FLAG_IMMUTABLE
            ))

            // Tap to open app
            val openIntent = Intent(context, MainActivity::class.java)
            setOnClickPendingIntent(R.id.widget_root, PendingIntent.getActivity(
                context, 0, openIntent, PendingIntent.FLAG_UPDATE_CURRENT or PendingIntent.FLAG_IMMUTABLE
            ))
        }

        manager.updateAppWidget(widgetId, views)
    }

    private fun handleQuickAdd(context: Context) {
        val current = WidgetDataUpdater.getProgressCurrent(context) + 1
        WidgetDataUpdater.updateProgress(context, current, WidgetDataUpdater.getProgressGoal(context))
        // Optionally sync to Firestore
    }

    companion object {
        const val ACTION_QUICK_ADD = "com.yourpackage.ACTION_QUICK_ADD"
    }
}
```

## Widget Layout (XML)

```xml
<!-- res/layout/widget_small.xml -->
<RelativeLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:id="@+id/widget_root"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:padding="12dp"
    android:background="@drawable/widget_bg">

    <ImageView
        android:id="@+id/iv_ring"
        android:layout_width="64dp"
        android:layout_height="64dp"
        android:layout_centerHorizontal="true" />

    <TextView
        android:id="@+id/tv_value"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:layout_below="@id/iv_ring"
        android:layout_centerHorizontal="true"
        android:textSize="16sp"
        android:textStyle="bold" />

    <TextView
        android:id="@+id/tv_goal"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:layout_below="@id/tv_value"
        android:layout_centerHorizontal="true"
        android:textSize="12sp"
        android:textColor="#888" />

    <ImageButton
        android:id="@+id/btn_add"
        android:layout_width="32dp"
        android:layout_height="32dp"
        android:layout_alignParentEnd="true"
        android:layout_alignParentBottom="true"
        android:src="@drawable/ic_add"
        android:background="?attr/selectableItemBackgroundBorderless" />
</RelativeLayout>
```

## Widget Info (XML)

```xml
<!-- res/xml/progress_widget_info.xml -->
<appwidget-provider xmlns:android="http://schemas.android.com/apk/res/android"
    android:minWidth="110dp"
    android:minHeight="110dp"
    android:targetCellWidth="2"
    android:targetCellHeight="2"
    android:updatePeriodMillis="1800000"
    android:initialLayout="@layout/widget_small"
    android:resizeMode="horizontal|vertical"
    android:widgetCategory="home_screen"
    android:previewImage="@drawable/widget_preview" />
```

## AndroidManifest for Widget

```xml
<receiver android:name=".widgets.ProgressWidgetProvider" android:exported="true">
    <intent-filter>
        <action android:name="android.appwidget.action.APPWIDGET_UPDATE" />
        <action android:name="com.yourpackage.ACTION_QUICK_ADD" />
    </intent-filter>
    <meta-data
        android:name="android.appwidget.provider"
        android:resource="@xml/progress_widget_info" />
</receiver>
```

## Live Notification Tracker (Foreground Service)

Persistent notification that acts like iOS Live Activities:

### Manager

```kotlin
object LiveActivityManager {
    private const val PREFS = "live_activity"

    fun isActive(context: Context, type: String): Boolean {
        return context.getSharedPreferences(PREFS, Context.MODE_PRIVATE)
            .getBoolean("${type}_active", false)
    }

    fun start(context: Context, type: String) {
        context.getSharedPreferences(PREFS, Context.MODE_PRIVATE)
            .edit().putBoolean("${type}_active", true).apply()

        val intent = Intent(context, LiveActivityService::class.java).apply {
            action = "ACTION_START"
            putExtra("type", type)
        }
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.O) {
            context.startForegroundService(intent)
        } else {
            context.startService(intent)
        }
    }

    fun update(context: Context, type: String, current: Int, goal: Int) {
        val intent = Intent(context, LiveActivityService::class.java).apply {
            action = "ACTION_UPDATE"
            putExtra("type", type)
            putExtra("current", current)
            putExtra("goal", goal)
        }
        context.startService(intent)
    }

    fun stop(context: Context, type: String) {
        context.getSharedPreferences(PREFS, Context.MODE_PRIVATE)
            .edit().putBoolean("${type}_active", false).apply()

        val intent = Intent(context, LiveActivityService::class.java).apply {
            action = "ACTION_STOP"
            putExtra("type", type)
        }
        context.startService(intent)
    }
}
```

### Foreground Service

```kotlin
class LiveActivityService : Service() {
    override fun onStartCommand(intent: Intent?, flags: Int, startId: Int): Int {
        val type = intent?.getStringExtra("type") ?: return START_NOT_STICKY

        when (intent.action) {
            "ACTION_START", "ACTION_UPDATE" -> {
                val current = intent.getIntExtra("current", 0)
                val goal = intent.getIntExtra("goal", 100)
                val notification = buildNotification(type, current, goal)
                startForeground(notificationId(type), notification)
            }
            "ACTION_STOP" -> {
                stopForeground(STOP_FOREGROUND_REMOVE)
                stopSelf()
            }
        }
        return START_STICKY
    }

    private fun buildNotification(type: String, current: Int, goal: Int): Notification {
        val progress = if (goal > 0) (current * 100 / goal) else 0

        val openIntent = PendingIntent.getActivity(
            this, 0, Intent(this, MainActivity::class.java),
            PendingIntent.FLAG_UPDATE_CURRENT or PendingIntent.FLAG_IMMUTABLE
        )

        return NotificationCompat.Builder(this, "live_tracker")
            .setSmallIcon(R.drawable.ic_notification)
            .setContentTitle("$type: $current / $goal")
            .setContentText("$progress% complete")
            .setProgress(100, progress, false)
            .setOngoing(true)
            .setContentIntent(openIntent)
            .setCategory(NotificationCompat.CATEGORY_PROGRESS)
            .build()
    }

    private fun notificationId(type: String): Int = type.hashCode().and(0xFFFF)

    override fun onBind(intent: Intent?) = null
}
```

### AndroidManifest for Foreground Service

```xml
<service android:name=".services.liveactivity.LiveActivityService"
    android:foregroundServiceType="specialUse"
    android:exported="false" />
```

### Notification Channel

```kotlin
if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.O) {
    val channel = NotificationChannel("live_tracker", "Live Tracker",
        NotificationManager.IMPORTANCE_LOW).apply {
        setShowBadge(false)
    }
    getSystemService(NotificationManager::class.java).createNotificationChannel(channel)
}
```

## Permissions

```xml
<uses-permission android:name="android.permission.POST_NOTIFICATIONS" />
<uses-permission android:name="android.permission.FOREGROUND_SERVICE" />
<uses-permission android:name="android.permission.FOREGROUND_SERVICE_SPECIAL_USE" />
```

Runtime permission for notifications (API 33+):

```kotlin
if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.TIRAMISU) {
    val launcher = rememberLauncherForActivityResult(
        ActivityResultContracts.RequestPermission()
    ) { granted -> /* handle */ }
    launcher.launch(Manifest.permission.POST_NOTIFICATIONS)
}
```
