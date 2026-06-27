# Focus Shield — Complete Android App Documentation

> Follow this document top to bottom. Every section is required. Skip nothing.

---

## Table of Contents

1. [Project Overview](#1-project-overview)
2. [Tech Stack & Dependencies](#2-tech-stack--dependencies)
3. [Project Structure](#3-project-structure)
4. [Android Permissions Setup](#4-android-permissions-setup)
5. [Database Design (Room)](#5-database-design-room)
6. [DataStore — User Preferences](#6-datastore--user-preferences)
7. [Accessibility Service — The Blocking Engine](#7-accessibility-service--the-blocking-engine)
8. [Overlay Service — Block Screen UI](#8-overlay-service--block-screen-ui)
9. [Foreground Service — Keep Alive](#9-foreground-service--keep-alive)
10. [WorkManager — Background Tasks](#10-workmanager--background-tasks)
11. [Dependency Injection (Hilt)](#11-dependency-injection-hilt)
12. [Repository Layer](#12-repository-layer)
13. [Use Cases (Domain Layer)](#13-use-cases-domain-layer)
14. [ViewModels](#14-viewmodels)
15. [UI Screens (Jetpack Compose)](#15-ui-screens-jetpack-compose)
16. [Navigation Setup](#16-navigation-setup)
17. [Permission Onboarding Flow](#17-permission-onboarding-flow)
18. [Battery Optimization Fix](#18-battery-optimization-fix)
19. [Strict Mode & PIN Protection](#19-strict-mode--pin-protection)
20. [Backend API (Node.js) — V2 Only](#20-backend-api-nodejs--v2-only)
21. [Build & Release](#21-build--release)
22. [Testing Checklist](#22-testing-checklist)
23. [Common Errors & Fixes](#23-common-errors--fixes)
24. [V1 vs V2 Feature Split](#24-v1-vs-v2-feature-split)

---

## 1. Project Overview

**App Name:** Focus Shield  
**Platform:** Android 8.0+ (API 26+)  
**Goal:** Block Instagram Reels, Facebook Reels, YouTube Shorts, TikTok, and Snapchat Spotlight using Android Accessibility Service and System Overlay.

### How Blocking Works (Core Logic)

```
User opens Instagram
       ↓
Accessibility Service fires onAccessibilityEvent()
       ↓
Service scans UI node tree for Reels ViewId
       ↓
Match found → Show full-screen Overlay via WindowManager
       ↓
User sees "Blocked" screen, cannot access Reels
       ↓
Block event logged to Room Database
```

This is the ONLY reliable method on Android. Do not use `UsageStatsManager` alone — it has 1-second polling delay which feels broken.

---

## 2. Tech Stack & Dependencies

### `build.gradle.kts` (Project level)

```kotlin
plugins {
    id("com.android.application") version "8.2.0" apply false
    id("com.android.library") version "8.2.0" apply false
    id("org.jetbrains.kotlin.android") version "1.9.22" apply false
    id("com.google.dagger.hilt.android") version "2.50" apply false
    id("com.google.devtools.ksp") version "1.9.22-1.0.17" apply false
}
```

### `build.gradle.kts` (App level) — complete

```kotlin
plugins {
    id("com.android.application")
    id("org.jetbrains.kotlin.android")
    id("com.google.dagger.hilt.android")
    id("com.google.devtools.ksp")
}

android {
    namespace = "com.focusshield.app"
    compileSdk = 34

    defaultConfig {
        applicationId = "com.focusshield.app"
        minSdk = 26
        targetSdk = 34
        versionCode = 1
        versionName = "1.0.0"
    }

    buildFeatures {
        compose = true
    }

    composeOptions {
        kotlinCompilerExtensionVersion = "1.5.8"
    }

    compileOptions {
        sourceCompatibility = JavaVersion.VERSION_17
        targetCompatibility = JavaVersion.VERSION_17
    }

    kotlinOptions {
        jvmTarget = "17"
    }
}

dependencies {
    // Jetpack Compose BOM
    val composeBom = platform("androidx.compose:compose-bom:2024.02.00")
    implementation(composeBom)
    implementation("androidx.compose.ui:ui")
    implementation("androidx.compose.material3:material3")
    implementation("androidx.compose.ui:ui-tooling-preview")
    implementation("androidx.activity:activity-compose:1.8.2")

    // Navigation
    implementation("androidx.navigation:navigation-compose:2.7.7")

    // ViewModel
    implementation("androidx.lifecycle:lifecycle-viewmodel-compose:2.7.0")
    implementation("androidx.lifecycle:lifecycle-runtime-compose:2.7.0")

    // Hilt
    implementation("com.google.dagger:hilt-android:2.50")
    ksp("com.google.dagger:hilt-compiler:2.50")
    implementation("androidx.hilt:hilt-navigation-compose:1.2.0")

    // Room
    implementation("androidx.room:room-runtime:2.6.1")
    implementation("androidx.room:room-ktx:2.6.1")
    ksp("androidx.room:room-compiler:2.6.1")

    // DataStore
    implementation("androidx.datastore:datastore-preferences:1.0.0")

    // WorkManager
    implementation("androidx.work:work-runtime-ktx:2.9.0")
    implementation("androidx.hilt:hilt-work:1.2.0")
    ksp("androidx.hilt:hilt-compiler:1.2.0")

    // Coroutines
    implementation("org.jetbrains.kotlinx:kotlinx-coroutines-android:1.7.3")

    // Retrofit (V2 backend)
    implementation("com.squareup.retrofit2:retrofit:2.9.0")
    implementation("com.squareup.retrofit2:converter-gson:2.9.0")
    implementation("com.squareup.okhttp3:logging-interceptor:4.12.0")

    // Coil
    implementation("io.coil-kt:coil-compose:2.6.0")

    // Splash Screen
    implementation("androidx.core:core-splashscreen:1.0.1")

    // Security (PIN hashing)
    implementation("androidx.security:security-crypto:1.1.0-alpha06")

    // Charts
    implementation("com.patrykandpatrick.vico:compose-m3:1.14.0")
}
```

---

## 3. Project Structure

```
app/src/main/java/com/focusshield/app/
│
├── MainActivity.kt
├── FocusShieldApp.kt                    ← Application class
│
├── di/
│   ├── AppModule.kt
│   ├── DatabaseModule.kt
│   └── NetworkModule.kt                 ← V2 only
│
├── data/
│   ├── local/
│   │   ├── db/
│   │   │   ├── AppDatabase.kt
│   │   │   ├── dao/
│   │   │   │   ├── BlockEventDao.kt
│   │   │   │   ├── AppSettingsDao.kt
│   │   │   │   ├── FocusSessionDao.kt
│   │   │   │   └── ScheduleDao.kt
│   │   │   └── entity/
│   │   │       ├── BlockEventEntity.kt
│   │   │       ├── BlockedAppEntity.kt
│   │   │       ├── FocusSessionEntity.kt
│   │   │       └── ScheduleEntity.kt
│   │   └── datastore/
│   │       └── UserPreferencesDataStore.kt
│   ├── remote/                          ← V2 only
│   │   ├── api/FocusShieldApi.kt
│   │   └── dto/
│   └── repository/
│       ├── BlockRepositoryImpl.kt
│       ├── StatsRepositoryImpl.kt
│       └── SettingsRepositoryImpl.kt
│
├── domain/
│   ├── model/
│   │   ├── BlockEvent.kt
│   │   ├── BlockedApp.kt
│   │   ├── FocusSession.kt
│   │   └── UserSettings.kt
│   ├── repository/
│   │   ├── BlockRepository.kt
│   │   ├── StatsRepository.kt
│   │   └── SettingsRepository.kt
│   └── usecase/
│       ├── block/
│       │   ├── ShouldBlockAppUseCase.kt
│       │   ├── LogBlockEventUseCase.kt
│       │   └── GetBlockedAppsUseCase.kt
│       ├── stats/
│       │   ├── GetDailyStatsUseCase.kt
│       │   └── GetWeeklyStatsUseCase.kt
│       └── settings/
│           ├── GetUserSettingsUseCase.kt
│           ├── UpdateBlockedAppsUseCase.kt
│           └── VerifyPinUseCase.kt
│
├── presentation/
│   ├── navigation/
│   │   ├── NavGraph.kt
│   │   └── Screen.kt
│   ├── screens/
│   │   ├── splash/SplashScreen.kt
│   │   ├── onboarding/
│   │   │   ├── OnboardingScreen.kt
│   │   │   └── PermissionSetupScreen.kt
│   │   ├── auth/
│   │   │   ├── LoginScreen.kt
│   │   │   └── RegisterScreen.kt
│   │   ├── home/
│   │   │   ├── HomeScreen.kt
│   │   │   └── HomeViewModel.kt
│   │   ├── apps/
│   │   │   ├── AppSelectionScreen.kt
│   │   │   └── AppSelectionViewModel.kt
│   │   ├── focus/
│   │   │   ├── FocusModeScreen.kt
│   │   │   └── FocusModeViewModel.kt
│   │   ├── analytics/
│   │   │   ├── AnalyticsScreen.kt
│   │   │   └── AnalyticsViewModel.kt
│   │   ├── schedule/
│   │   │   ├── ScheduleScreen.kt
│   │   │   └── ScheduleViewModel.kt
│   │   └── settings/
│   │       ├── SettingsScreen.kt
│   │       └── SettingsViewModel.kt
│   └── components/
│       ├── StatCard.kt
│       ├── AppItem.kt
│       ├── FocusTimer.kt
│       └── WeeklyChart.kt
│
├── services/
│   ├── FocusAccessibilityService.kt    ← CORE BLOCKING ENGINE
│   ├── OverlayService.kt
│   └── FocusForegroundService.kt
│
├── workers/
│   ├── DailyReportWorker.kt
│   └── StatsAggregationWorker.kt
│
└── utils/
    ├── AppBlockList.kt                  ← All blocked app package names & node IDs
    ├── PermissionHelper.kt
    ├── PinHasher.kt
    └── TimeFormatter.kt
```

---

## 4. Android Permissions Setup

### `AndroidManifest.xml` — complete

```xml
<?xml version="1.0" encoding="utf-8"?>
<manifest xmlns:android="http://schemas.android.com/apk/res/android">

    <!-- Core permissions -->
    <uses-permission android:name="android.permission.INTERNET" />
    <uses-permission android:name="android.permission.FOREGROUND_SERVICE" />
    <uses-permission android:name="android.permission.FOREGROUND_SERVICE_SPECIAL_USE" />
    <uses-permission android:name="android.permission.POST_NOTIFICATIONS" />
    <uses-permission android:name="android.permission.RECEIVE_BOOT_COMPLETED" />
    <uses-permission android:name="android.permission.REQUEST_IGNORE_BATTERY_OPTIMIZATIONS" />
    <uses-permission android:name="android.permission.SYSTEM_ALERT_WINDOW" />

    <!-- Usage Stats — user must grant manually in Settings -->
    <uses-permission
        android:name="android.permission.PACKAGE_USAGE_STATS"
        tools:ignore="ProtectedPermissions" />

    <application
        android:name=".FocusShieldApp"
        android:allowBackup="true"
        android:label="@string/app_name"
        android:theme="@style/Theme.FocusShield">

        <activity
            android:name=".MainActivity"
            android:exported="true">
            <intent-filter>
                <action android:name="android.intent.action.MAIN" />
                <category android:name="android.intent.category.LAUNCHER" />
            </intent-filter>
        </activity>

        <!-- CRITICAL: Accessibility Service declaration -->
        <service
            android:name=".services.FocusAccessibilityService"
            android:exported="true"
            android:label="Focus Shield"
            android:permission="android.permission.BIND_ACCESSIBILITY_SERVICE">
            <intent-filter>
                <action android:name="android.accessibilityservice.AccessibilityService" />
            </intent-filter>
            <meta-data
                android:name="android.accessibilityservice"
                android:resource="@xml/accessibility_service_config" />
        </service>

        <!-- Foreground Service -->
        <service
            android:name=".services.FocusForegroundService"
            android:exported="false"
            android:foregroundServiceType="specialUse" />

        <!-- Boot receiver — restart service after phone reboot -->
        <receiver
            android:name=".receivers.BootReceiver"
            android:exported="true">
            <intent-filter>
                <action android:name="android.intent.action.BOOT_COMPLETED" />
            </intent-filter>
        </receiver>

    </application>
</manifest>
```

### `res/xml/accessibility_service_config.xml`

```xml
<?xml version="1.0" encoding="utf-8"?>
<accessibility-service
    xmlns:android="http://schemas.android.com/apk/res/android"
    android:accessibilityEventTypes="typeWindowStateChanged|typeWindowContentChanged"
    android:accessibilityFeedbackType="feedbackGeneric"
    android:accessibilityFlags="flagDefault|flagRetrieveInteractiveWindows"
    android:canRetrieveWindowContent="true"
    android:description="@string/accessibility_service_description"
    android:notificationTimeout="100"
    android:packageNames="com.instagram.android,com.facebook.katana,com.google.android.youtube,com.zhiliaoapp.musically,com.snapchat.android"
    android:settingsActivity=".MainActivity" />
```

> **Note:** `packageNames` limits which apps trigger the service. This saves battery — the service only fires for these specific apps.

---

## 5. Database Design (Room)

### `BlockEventEntity.kt`

```kotlin
@Entity(tableName = "block_events")
data class BlockEventEntity(
    @PrimaryKey(autoGenerate = true) val id: Long = 0,
    val packageName: String,
    val appName: String,
    val blockedAt: Long = System.currentTimeMillis(),
    val contentType: String,  // "REELS", "SHORTS", "TIKTOK"
    val sessionId: String? = null
)
```

### `BlockedAppEntity.kt`

```kotlin
@Entity(tableName = "blocked_apps")
data class BlockedAppEntity(
    @PrimaryKey val packageName: String,
    val appName: String,
    val isEnabled: Boolean = true,
    val dailyLimitMinutes: Int = 0,   // 0 = no limit, just block reels
    val addedAt: Long = System.currentTimeMillis()
)
```

### `FocusSessionEntity.kt`

```kotlin
@Entity(tableName = "focus_sessions")
data class FocusSessionEntity(
    @PrimaryKey val id: String = UUID.randomUUID().toString(),
    val startTime: Long,
    val endTime: Long? = null,
    val plannedDurationMinutes: Int,
    val sessionType: String,  // "POMODORO", "DEEP_FOCUS", "CUSTOM"
    val isCompleted: Boolean = false
)
```

### `ScheduleEntity.kt`

```kotlin
@Entity(tableName = "schedules")
data class ScheduleEntity(
    @PrimaryKey(autoGenerate = true) val id: Long = 0,
    val name: String,              // "Study Hours", "Sleep Mode"
    val startHour: Int,
    val startMinute: Int,
    val endHour: Int,
    val endMinute: Int,
    val daysOfWeek: String,        // "1,2,3,4,5" = Mon-Fri (JSON array as String)
    val isEnabled: Boolean = true,
    val blockAllApps: Boolean = false
)
```

### `AppDatabase.kt`

```kotlin
@Database(
    entities = [
        BlockEventEntity::class,
        BlockedAppEntity::class,
        FocusSessionEntity::class,
        ScheduleEntity::class
    ],
    version = 1,
    exportSchema = true
)
abstract class AppDatabase : RoomDatabase() {
    abstract fun blockEventDao(): BlockEventDao
    abstract fun blockedAppDao(): BlockedAppDao
    abstract fun focusSessionDao(): FocusSessionDao
    abstract fun scheduleDao(): ScheduleDao
}
```

### `BlockEventDao.kt`

```kotlin
@Dao
interface BlockEventDao {
    @Insert
    suspend fun insert(event: BlockEventEntity)

    @Query("SELECT COUNT(*) FROM block_events WHERE date(blockedAt/1000, 'unixepoch') = date('now')")
    fun getTodayBlockCount(): Flow<Int>

    @Query("""
        SELECT COUNT(*) as count, appName, date(blockedAt/1000, 'unixepoch') as date
        FROM block_events
        WHERE blockedAt >= :startTime
        GROUP BY date, appName
        ORDER BY date DESC
    """)
    fun getWeeklyStats(startTime: Long): Flow<List<DailyStatResult>>

    @Query("SELECT * FROM block_events ORDER BY blockedAt DESC LIMIT 50")
    fun getRecentEvents(): Flow<List<BlockEventEntity>>
}
```

### `BlockedAppDao.kt`

```kotlin
@Dao
interface BlockedAppDao {
    @Query("SELECT * FROM blocked_apps WHERE isEnabled = 1")
    fun getEnabledApps(): Flow<List<BlockedAppEntity>>

    @Upsert
    suspend fun upsert(app: BlockedAppEntity)

    @Query("UPDATE blocked_apps SET isEnabled = :enabled WHERE packageName = :packageName")
    suspend fun setEnabled(packageName: String, enabled: Boolean)

    @Query("SELECT * FROM blocked_apps")
    fun getAllApps(): Flow<List<BlockedAppEntity>>
}
```

---

## 6. DataStore — User Preferences

### `UserPreferencesDataStore.kt`

```kotlin
private val Context.dataStore: DataStore<Preferences> by preferencesDataStore(name = "user_prefs")

class UserPreferencesDataStore @Inject constructor(
    @ApplicationContext private val context: Context
) {
    companion object {
        val IS_STRICT_MODE = booleanPreferencesKey("is_strict_mode")
        val PIN_HASH = stringPreferencesKey("pin_hash")
        val IS_ONBOARDING_DONE = booleanPreferencesKey("onboarding_done")
        val FOCUS_SCORE = intPreferencesKey("focus_score")
        val THEME_MODE = stringPreferencesKey("theme_mode")  // "LIGHT", "DARK", "SYSTEM"
        val IS_LOGGED_IN = booleanPreferencesKey("is_logged_in")
        val BLOCK_OVERLAY_MESSAGE = stringPreferencesKey("block_message")
        val DAILY_GOAL_MINUTES = intPreferencesKey("daily_goal_minutes")
    }

    val isStrictMode: Flow<Boolean> = context.dataStore.data
        .map { it[IS_STRICT_MODE] ?: false }

    val pinHash: Flow<String?> = context.dataStore.data
        .map { it[PIN_HASH] }

    val isOnboardingDone: Flow<Boolean> = context.dataStore.data
        .map { it[IS_ONBOARDING_DONE] ?: false }

    suspend fun setStrictMode(enabled: Boolean) {
        context.dataStore.edit { it[IS_STRICT_MODE] = enabled }
    }

    suspend fun setPin(rawPin: String) {
        val hash = PinHasher.hash(rawPin)
        context.dataStore.edit { it[PIN_HASH] = hash }
    }

    suspend fun setOnboardingDone() {
        context.dataStore.edit { it[IS_ONBOARDING_DONE] = true }
    }

    suspend fun setTheme(mode: String) {
        context.dataStore.edit { it[THEME_MODE] = mode }
    }
}
```

---

## 7. Accessibility Service — The Blocking Engine

This is the most critical file in the entire project. Read carefully.

### `utils/AppBlockList.kt`

```kotlin
object AppBlockList {

    // Package names of apps to monitor
    val MONITORED_PACKAGES = setOf(
        "com.instagram.android",
        "com.facebook.katana",
        "com.google.android.youtube",
        "com.zhiliaoapp.musically",       // TikTok
        "com.snapchat.android"
    )

    // ViewId resource names that identify Reels/Shorts UI
    // WARNING: These can change when apps update. Check periodically.
    val REELS_NODE_IDS = mapOf(
        "com.instagram.android" to listOf(
            "com.instagram.android:id/clips_viewer_view_pager",
            "com.instagram.android:id/reel_viewer_container",
            "com.instagram.android:id/clips_tab"
        ),
        "com.facebook.katana" to listOf(
            "com.facebook.katana:id/reels_video_container",
            "com.facebook.katana:id/short_video_container"
        ),
        "com.google.android.youtube" to listOf(
            "com.google.android.youtube:id/reel_watch_player",
            "com.google.android.youtube:id/shorts_pivot_item"
        ),
        "com.zhiliaoapp.musically" to listOf(
            "com.zhiliaoapp.musically:id/feed_item_video_layout"
        ),
        "com.snapchat.android" to listOf(
            "com.snapchat.android:id/spotlight_tabs_layout"
        )
    )

    // Content description fallback (if viewId changes, try matching these)
    val REELS_CONTENT_DESC_KEYWORDS = listOf(
        "reel", "short", "shorts", "reels", "spotlight", "clip"
    )
}
```

### `services/FocusAccessibilityService.kt`

```kotlin
@AndroidEntryPoint
class FocusAccessibilityService : AccessibilityService() {

    @Inject lateinit var blockRepository: BlockRepository
    @Inject lateinit var settingsRepository: SettingsRepository

    private val serviceScope = CoroutineScope(Dispatchers.IO + SupervisorJob())
    private var isOverlayShowing = false
    private var lastBlockedPackage = ""
    private var lastBlockTime = 0L
    private val BLOCK_COOLDOWN_MS = 500L  // Prevent rapid re-triggering

    override fun onServiceConnected() {
        super.onServiceConnected()
        val info = AccessibilityServiceInfo().apply {
            eventTypes = AccessibilityEvent.TYPE_WINDOW_STATE_CHANGED or
                    AccessibilityEvent.TYPE_WINDOW_CONTENT_CHANGED
            feedbackType = AccessibilityServiceInfo.FEEDBACK_GENERIC
            flags = AccessibilityServiceInfo.FLAG_RETRIEVE_INTERACTIVE_WINDOWS
            notificationTimeout = 100
            packageNames = AppBlockList.MONITORED_PACKAGES.toTypedArray()
        }
        serviceInfo = info
    }

    override fun onAccessibilityEvent(event: AccessibilityEvent) {
        val packageName = event.packageName?.toString() ?: return

        if (packageName !in AppBlockList.MONITORED_PACKAGES) return

        // Cooldown check — prevent spam
        val now = System.currentTimeMillis()
        if (packageName == lastBlockedPackage && (now - lastBlockTime) < BLOCK_COOLDOWN_MS) return

        serviceScope.launch {
            val blockedApps = settingsRepository.getEnabledBlockedPackages()
            if (packageName !in blockedApps) return@launch

            val shouldBlock = detectReelsContent(packageName)
            if (shouldBlock) {
                lastBlockedPackage = packageName
                lastBlockTime = now
                showBlockOverlay(packageName)
                logBlockEvent(packageName)
            }
        }
    }

    private fun detectReelsContent(packageName: String): Boolean {
        val rootNode = rootInActiveWindow ?: return false

        // Method 1: Check by ViewId (most reliable)
        val nodeIds = AppBlockList.REELS_NODE_IDS[packageName] ?: emptyList()
        for (nodeId in nodeIds) {
            val nodes = rootNode.findAccessibilityNodeInfosByViewId(nodeId)
            if (nodes.isNotEmpty()) {
                nodes.forEach { it.recycle() }
                return true
            }
        }

        // Method 2: Fallback — check content descriptions (less reliable but resilient to updates)
        return checkContentDescriptions(rootNode)
    }

    private fun checkContentDescriptions(node: AccessibilityNodeInfo): Boolean {
        val contentDesc = node.contentDescription?.toString()?.lowercase() ?: ""
        val text = node.text?.toString()?.lowercase() ?: ""

        for (keyword in AppBlockList.REELS_CONTENT_DESC_KEYWORDS) {
            if (contentDesc.contains(keyword) || text.contains(keyword)) {
                // Only match if it looks like a video feed (has children)
                if (node.childCount > 2) return true
            }
        }

        for (i in 0 until node.childCount) {
            val child = node.getChild(i) ?: continue
            if (checkContentDescriptions(child)) {
                child.recycle()
                return true
            }
            child.recycle()
        }
        return false
    }

    private fun showBlockOverlay(packageName: String) {
        if (isOverlayShowing) return
        isOverlayShowing = true

        val intent = Intent(this, OverlayService::class.java).apply {
            putExtra("PACKAGE_NAME", packageName)
            putExtra("APP_NAME", getAppName(packageName))
        }
        startService(intent)
    }

    fun hideOverlay() {
        isOverlayShowing = false
        val intent = Intent(this, OverlayService::class.java)
        intent.action = OverlayService.ACTION_HIDE
        startService(intent)
    }

    private fun logBlockEvent(packageName: String) {
        serviceScope.launch {
            blockRepository.logBlockEvent(
                packageName = packageName,
                appName = getAppName(packageName),
                contentType = getContentType(packageName)
            )
        }
    }

    private fun getAppName(packageName: String): String {
        return when (packageName) {
            "com.instagram.android" -> "Instagram"
            "com.facebook.katana" -> "Facebook"
            "com.google.android.youtube" -> "YouTube"
            "com.zhiliaoapp.musically" -> "TikTok"
            "com.snapchat.android" -> "Snapchat"
            else -> packageName
        }
    }

    private fun getContentType(packageName: String): String {
        return when (packageName) {
            "com.google.android.youtube" -> "SHORTS"
            "com.zhiliaoapp.musically" -> "TIKTOK"
            else -> "REELS"
        }
    }

    override fun onInterrupt() {
        serviceScope.cancel()
    }

    override fun onDestroy() {
        super.onDestroy()
        serviceScope.cancel()
    }
}
```

---

## 8. Overlay Service — Block Screen UI

### `services/OverlayService.kt`

```kotlin
class OverlayService : Service() {

    companion object {
        const val ACTION_HIDE = "ACTION_HIDE_OVERLAY"
    }

    private var windowManager: WindowManager? = null
    private var overlayView: View? = null

    override fun onBind(intent: Intent?): IBinder? = null

    override fun onStartCommand(intent: Intent?, flags: Int, startId: Int): Int {
        if (intent?.action == ACTION_HIDE) {
            removeOverlay()
            return START_NOT_STICKY
        }

        val packageName = intent?.getStringExtra("PACKAGE_NAME") ?: ""
        val appName = intent?.getStringExtra("APP_NAME") ?: "This app"
        showOverlay(appName, packageName)
        return START_STICKY
    }

    private fun showOverlay(appName: String, packageName: String) {
        if (overlayView != null) return
        if (!Settings.canDrawOverlays(this)) return

        windowManager = getSystemService(WINDOW_SERVICE) as WindowManager

        val params = WindowManager.LayoutParams(
            WindowManager.LayoutParams.MATCH_PARENT,
            WindowManager.LayoutParams.MATCH_PARENT,
            WindowManager.LayoutParams.TYPE_APPLICATION_OVERLAY,
            WindowManager.LayoutParams.FLAG_NOT_FOCUSABLE or
                    WindowManager.LayoutParams.FLAG_LAYOUT_IN_SCREEN,
            PixelFormat.TRANSLUCENT
        )

        // Build the block screen using ComposeView
        overlayView = ComposeView(this).apply {
            setViewTreeLifecycleOwner(MyLifecycleOwner())
            setViewTreeSavedStateRegistryOwner(MySavedStateRegistryOwner())
            setContent {
                BlockOverlayScreen(
                    appName = appName,
                    onGoBack = {
                        removeOverlay()
                        // Press back to go to home screen
                        val homeIntent = Intent(Intent.ACTION_MAIN).apply {
                            addCategory(Intent.CATEGORY_HOME)
                            flags = Intent.FLAG_ACTIVITY_NEW_TASK
                        }
                        startActivity(homeIntent)
                    }
                )
            }
        }

        windowManager?.addView(overlayView, params)
    }

    private fun removeOverlay() {
        overlayView?.let {
            windowManager?.removeView(it)
            overlayView = null
        }
    }

    override fun onDestroy() {
        super.onDestroy()
        removeOverlay()
    }
}
```

### `BlockOverlayScreen.kt` (Compose UI)

```kotlin
@Composable
fun BlockOverlayScreen(
    appName: String,
    onGoBack: () -> Unit
) {
    Box(
        modifier = Modifier
            .fillMaxSize()
            .background(Color(0xF0000000)),  // 94% transparent black
        contentAlignment = Alignment.Center
    ) {
        Column(
            horizontalAlignment = Alignment.CenterHorizontally,
            verticalArrangement = Arrangement.spacedBy(24.dp),
            modifier = Modifier.padding(32.dp)
        ) {
            // Block icon
            Text(
                text = "🚫",
                fontSize = 72.sp
            )

            Text(
                text = "Focus Mode Active",
                style = MaterialTheme.typography.headlineMedium,
                color = Color.White,
                fontWeight = FontWeight.Bold
            )

            Text(
                text = "$appName Reels is blocked.",
                style = MaterialTheme.typography.bodyLarge,
                color = Color.White.copy(alpha = 0.8f),
                textAlign = TextAlign.Center
            )

            Text(
                text = "You blocked this to stay focused.\nKeep going!",
                style = MaterialTheme.typography.bodyMedium,
                color = Color.White.copy(alpha = 0.6f),
                textAlign = TextAlign.Center
            )

            Spacer(modifier = Modifier.height(16.dp))

            Button(
                onClick = onGoBack,
                colors = ButtonDefaults.buttonColors(
                    containerColor = Color.White,
                    contentColor = Color.Black
                ),
                modifier = Modifier
                    .fillMaxWidth()
                    .height(56.dp)
            ) {
                Text("Go Back", fontWeight = FontWeight.SemiBold)
            }
        }
    }
}
```

---

## 9. Foreground Service — Keep Alive

Without this, Android kills the accessibility service in background.

### `services/FocusForegroundService.kt`

```kotlin
@AndroidEntryPoint
class FocusForegroundService : Service() {

    companion object {
        const val CHANNEL_ID = "focus_shield_channel"
        const val NOTIFICATION_ID = 1001
        const val ACTION_START = "ACTION_START"
        const val ACTION_STOP = "ACTION_STOP"

        fun startService(context: Context) {
            val intent = Intent(context, FocusForegroundService::class.java).apply {
                action = ACTION_START
            }
            context.startForegroundService(intent)
        }

        fun stopService(context: Context) {
            val intent = Intent(context, FocusForegroundService::class.java).apply {
                action = ACTION_STOP
            }
            context.startService(intent)
        }
    }

    override fun onStartCommand(intent: Intent?, flags: Int, startId: Int): Int {
        when (intent?.action) {
            ACTION_START -> {
                createNotificationChannel()
                startForeground(NOTIFICATION_ID, buildNotification())
            }
            ACTION_STOP -> {
                stopForeground(STOP_FOREGROUND_REMOVE)
                stopSelf()
            }
        }
        return START_STICKY  // Restart if killed
    }

    private fun buildNotification(): Notification {
        val pendingIntent = PendingIntent.getActivity(
            this, 0,
            Intent(this, MainActivity::class.java),
            PendingIntent.FLAG_IMMUTABLE
        )

        return NotificationCompat.Builder(this, CHANNEL_ID)
            .setSmallIcon(R.drawable.ic_shield)
            .setContentTitle("Focus Shield is active")
            .setContentText("Protecting you from distractions")
            .setContentIntent(pendingIntent)
            .setOngoing(true)
            .setSilent(true)
            .build()
    }

    private fun createNotificationChannel() {
        val channel = NotificationChannel(
            CHANNEL_ID,
            "Focus Shield Protection",
            NotificationManager.IMPORTANCE_LOW
        ).apply {
            description = "Shows when Focus Shield is actively blocking"
            setShowBadge(false)
        }
        val manager = getSystemService(NotificationManager::class.java)
        manager.createNotificationChannel(channel)
    }

    override fun onBind(intent: Intent?): IBinder? = null
}
```

### `receivers/BootReceiver.kt`

```kotlin
class BootReceiver : BroadcastReceiver() {
    override fun onReceive(context: Context, intent: Intent) {
        if (intent.action == Intent.ACTION_BOOT_COMPLETED) {
            FocusForegroundService.startService(context)
        }
    }
}
```

---

## 10. WorkManager — Background Tasks

### `workers/DailyReportWorker.kt`

```kotlin
@HiltWorker
class DailyReportWorker @AssistedInject constructor(
    @Assisted context: Context,
    @Assisted workerParams: WorkerParameters,
    private val statsRepository: StatsRepository
) : CoroutineWorker(context, workerParams) {

    override suspend fun doWork(): Result {
        return try {
            statsRepository.aggregateDailyStats()
            Result.success()
        } catch (e: Exception) {
            Result.retry()
        }
    }

    companion object {
        fun schedule(workManager: WorkManager) {
            val request = PeriodicWorkRequestBuilder<DailyReportWorker>(
                1, TimeUnit.DAYS
            )
                .setInitialDelay(calculateDelayUntilMidnight(), TimeUnit.MILLISECONDS)
                .setConstraints(
                    Constraints.Builder()
                        .setRequiredNetworkType(NetworkType.NOT_REQUIRED)
                        .build()
                )
                .build()

            workManager.enqueueUniquePeriodicWork(
                "daily_report",
                ExistingPeriodicWorkPolicy.UPDATE,
                request
            )
        }

        private fun calculateDelayUntilMidnight(): Long {
            val now = Calendar.getInstance()
            val midnight = Calendar.getInstance().apply {
                set(Calendar.HOUR_OF_DAY, 0)
                set(Calendar.MINUTE, 0)
                set(Calendar.SECOND, 0)
                add(Calendar.DAY_OF_MONTH, 1)
            }
            return midnight.timeInMillis - now.timeInMillis
        }
    }
}
```

---

## 11. Dependency Injection (Hilt)

### `FocusShieldApp.kt`

```kotlin
@HiltAndroidApp
class FocusShieldApp : Application() {

    override fun onCreate() {
        super.onCreate()
        // Schedule background workers
        DailyReportWorker.schedule(WorkManager.getInstance(this))
    }
}
```

### `di/DatabaseModule.kt`

```kotlin
@Module
@InstallIn(SingletonComponent::class)
object DatabaseModule {

    @Provides
    @Singleton
    fun provideDatabase(@ApplicationContext context: Context): AppDatabase {
        return Room.databaseBuilder(
            context,
            AppDatabase::class.java,
            "focus_shield_db"
        )
            .fallbackToDestructiveMigration()
            .build()
    }

    @Provides fun provideBlockEventDao(db: AppDatabase) = db.blockEventDao()
    @Provides fun provideBlockedAppDao(db: AppDatabase) = db.blockedAppDao()
    @Provides fun provideFocusSessionDao(db: AppDatabase) = db.focusSessionDao()
    @Provides fun provideScheduleDao(db: AppDatabase) = db.scheduleDao()
}
```

### `di/AppModule.kt`

```kotlin
@Module
@InstallIn(SingletonComponent::class)
object AppModule {

    @Provides
    @Singleton
    fun provideUserPreferencesDataStore(
        @ApplicationContext context: Context
    ): UserPreferencesDataStore = UserPreferencesDataStore(context)

    @Provides
    @Singleton
    fun provideBlockRepository(
        blockEventDao: BlockEventDao,
        blockedAppDao: BlockedAppDao
    ): BlockRepository = BlockRepositoryImpl(blockEventDao, blockedAppDao)

    @Provides
    @Singleton
    fun provideStatsRepository(
        blockEventDao: BlockEventDao,
        focusSessionDao: FocusSessionDao
    ): StatsRepository = StatsRepositoryImpl(blockEventDao, focusSessionDao)

    @Provides
    @Singleton
    fun provideSettingsRepository(
        blockedAppDao: BlockedAppDao,
        dataStore: UserPreferencesDataStore
    ): SettingsRepository = SettingsRepositoryImpl(blockedAppDao, dataStore)
}
```

---

## 12. Repository Layer

### `domain/repository/BlockRepository.kt`

```kotlin
interface BlockRepository {
    suspend fun logBlockEvent(packageName: String, appName: String, contentType: String)
    fun getTodayBlockCount(): Flow<Int>
    fun getRecentEvents(): Flow<List<BlockEvent>>
    fun getEnabledBlockedApps(): Flow<List<BlockedApp>>
    suspend fun getEnabledBlockedPackages(): Set<String>
    suspend fun addBlockedApp(app: BlockedApp)
    suspend fun setAppEnabled(packageName: String, enabled: Boolean)
}
```

### `data/repository/BlockRepositoryImpl.kt`

```kotlin
class BlockRepositoryImpl @Inject constructor(
    private val blockEventDao: BlockEventDao,
    private val blockedAppDao: BlockedAppDao
) : BlockRepository {

    override suspend fun logBlockEvent(
        packageName: String,
        appName: String,
        contentType: String
    ) {
        blockEventDao.insert(
            BlockEventEntity(
                packageName = packageName,
                appName = appName,
                contentType = contentType
            )
        )
    }

    override fun getTodayBlockCount(): Flow<Int> =
        blockEventDao.getTodayBlockCount()

    override fun getRecentEvents(): Flow<List<BlockEvent>> =
        blockEventDao.getRecentEvents().map { list ->
            list.map { it.toDomain() }
        }

    override fun getEnabledBlockedApps(): Flow<List<BlockedApp>> =
        blockedAppDao.getEnabledApps().map { list ->
            list.map { it.toDomain() }
        }

    override suspend fun getEnabledBlockedPackages(): Set<String> {
        return blockedAppDao.getEnabledPackageNames().toSet()
    }

    override suspend fun addBlockedApp(app: BlockedApp) {
        blockedAppDao.upsert(app.toEntity())
    }

    override suspend fun setAppEnabled(packageName: String, enabled: Boolean) {
        blockedAppDao.setEnabled(packageName, enabled)
    }
}
```

---

## 13. Use Cases (Domain Layer)

### `domain/usecase/block/ShouldBlockAppUseCase.kt`

```kotlin
class ShouldBlockAppUseCase @Inject constructor(
    private val blockRepository: BlockRepository,
    private val settingsRepository: SettingsRepository
) {
    suspend operator fun invoke(packageName: String): Boolean {
        val isStrictMode = settingsRepository.isStrictModeEnabled()
        val enabledPackages = blockRepository.getEnabledBlockedPackages()
        return packageName in enabledPackages || isStrictMode
    }
}
```

### `domain/usecase/stats/GetDailyStatsUseCase.kt`

```kotlin
class GetDailyStatsUseCase @Inject constructor(
    private val statsRepository: StatsRepository
) {
    operator fun invoke(): Flow<DailyStats> {
        return statsRepository.getDailyStats()
    }
}

data class DailyStats(
    val blockCount: Int,
    val focusMinutes: Int,
    val focusScore: Int,
    val streakDays: Int,
    val timeSavedMinutes: Int
)
```

### `domain/usecase/settings/VerifyPinUseCase.kt`

```kotlin
class VerifyPinUseCase @Inject constructor(
    private val settingsRepository: SettingsRepository
) {
    suspend operator fun invoke(enteredPin: String): Boolean {
        val storedHash = settingsRepository.getPinHash() ?: return false
        return PinHasher.verify(enteredPin, storedHash)
    }
}
```

---

## 14. ViewModels

### `presentation/screens/home/HomeViewModel.kt`

```kotlin
@HiltViewModel
class HomeViewModel @Inject constructor(
    private val getDailyStatsUseCase: GetDailyStatsUseCase,
    private val blockRepository: BlockRepository,
    private val settingsRepository: SettingsRepository
) : ViewModel() {

    private val _uiState = MutableStateFlow(HomeUiState())
    val uiState: StateFlow<HomeUiState> = _uiState.asStateFlow()

    init {
        loadDashboard()
    }

    private fun loadDashboard() {
        viewModelScope.launch {
            combine(
                getDailyStatsUseCase(),
                blockRepository.getTodayBlockCount(),
                settingsRepository.isStrictModeFlow()
            ) { stats, blockCount, isStrictMode ->
                HomeUiState(
                    blockCount = blockCount,
                    focusScore = stats.focusScore,
                    streakDays = stats.streakDays,
                    timeSavedMinutes = stats.timeSavedMinutes,
                    isStrictMode = isStrictMode,
                    isLoading = false
                )
            }.collect { state ->
                _uiState.value = state
            }
        }
    }

    fun toggleStrictMode(enabled: Boolean) {
        viewModelScope.launch {
            settingsRepository.setStrictMode(enabled)
        }
    }
}

data class HomeUiState(
    val blockCount: Int = 0,
    val focusScore: Int = 0,
    val streakDays: Int = 0,
    val timeSavedMinutes: Int = 0,
    val isStrictMode: Boolean = false,
    val isLoading: Boolean = true,
    val error: String? = null
)
```

### `presentation/screens/apps/AppSelectionViewModel.kt`

```kotlin
@HiltViewModel
class AppSelectionViewModel @Inject constructor(
    private val blockRepository: BlockRepository,
    private val updateBlockedAppsUseCase: UpdateBlockedAppsUseCase
) : ViewModel() {

    val blockedApps: StateFlow<List<BlockedApp>> = blockRepository
        .getEnabledBlockedApps()
        .stateIn(viewModelScope, SharingStarted.WhileSubscribed(5000), emptyList())

    fun toggleApp(packageName: String, enabled: Boolean) {
        viewModelScope.launch {
            blockRepository.setAppEnabled(packageName, enabled)
        }
    }

    fun addDefaultApps() {
        viewModelScope.launch {
            val defaultApps = listOf(
                BlockedApp("com.instagram.android", "Instagram", true),
                BlockedApp("com.facebook.katana", "Facebook", true),
                BlockedApp("com.google.android.youtube", "YouTube", true),
                BlockedApp("com.zhiliaoapp.musically", "TikTok", true),
                BlockedApp("com.snapchat.android", "Snapchat", false)
            )
            defaultApps.forEach { blockRepository.addBlockedApp(it) }
        }
    }
}
```

---

## 15. UI Screens (Jetpack Compose)

### `presentation/screens/home/HomeScreen.kt`

```kotlin
@Composable
fun HomeScreen(
    viewModel: HomeViewModel = hiltViewModel(),
    onNavigateToApps: () -> Unit,
    onNavigateToFocus: () -> Unit,
    onNavigateToAnalytics: () -> Unit
) {
    val uiState by viewModel.uiState.collectAsStateWithLifecycle()

    Scaffold(
        topBar = {
            TopAppBar(
                title = { Text("Focus Shield") },
                actions = {
                    IconButton(onClick = { /* settings */ }) {
                        Icon(Icons.Default.Settings, contentDescription = "Settings")
                    }
                }
            )
        }
    ) { padding ->
        LazyColumn(
            modifier = Modifier
                .fillMaxSize()
                .padding(padding)
                .padding(horizontal = 16.dp),
            verticalArrangement = Arrangement.spacedBy(16.dp)
        ) {
            // Today's stats
            item {
                Text(
                    text = "Today",
                    style = MaterialTheme.typography.titleLarge,
                    modifier = Modifier.padding(top = 16.dp)
                )
            }

            item {
                Row(
                    modifier = Modifier.fillMaxWidth(),
                    horizontalArrangement = Arrangement.spacedBy(12.dp)
                ) {
                    StatCard(
                        modifier = Modifier.weight(1f),
                        title = "Blocked",
                        value = "${uiState.blockCount}",
                        subtitle = "attempts",
                        icon = Icons.Default.Block
                    )
                    StatCard(
                        modifier = Modifier.weight(1f),
                        title = "Saved",
                        value = "${uiState.timeSavedMinutes}",
                        subtitle = "minutes",
                        icon = Icons.Default.Timer
                    )
                }
            }

            item {
                Row(
                    modifier = Modifier.fillMaxWidth(),
                    horizontalArrangement = Arrangement.spacedBy(12.dp)
                ) {
                    StatCard(
                        modifier = Modifier.weight(1f),
                        title = "Focus Score",
                        value = "${uiState.focusScore}",
                        subtitle = "out of 100",
                        icon = Icons.Default.Star
                    )
                    StatCard(
                        modifier = Modifier.weight(1f),
                        title = "Streak",
                        value = "${uiState.streakDays}",
                        subtitle = "days",
                        icon = Icons.Default.LocalFire
                    )
                }
            }

            // Strict Mode Toggle
            item {
                Card(modifier = Modifier.fillMaxWidth()) {
                    Row(
                        modifier = Modifier
                            .fillMaxWidth()
                            .padding(16.dp),
                        horizontalArrangement = Arrangement.SpaceBetween,
                        verticalAlignment = Alignment.CenterVertically
                    ) {
                        Column {
                            Text("Strict Mode", style = MaterialTheme.typography.titleMedium)
                            Text(
                                "Harder to disable blocking",
                                style = MaterialTheme.typography.bodySmall,
                                color = MaterialTheme.colorScheme.onSurfaceVariant
                            )
                        }
                        Switch(
                            checked = uiState.isStrictMode,
                            onCheckedChange = { viewModel.toggleStrictMode(it) }
                        )
                    }
                }
            }

            // Quick actions
            item {
                Text(
                    text = "Quick Actions",
                    style = MaterialTheme.typography.titleMedium
                )
            }

            item {
                Row(
                    modifier = Modifier.fillMaxWidth(),
                    horizontalArrangement = Arrangement.spacedBy(12.dp)
                ) {
                    OutlinedButton(
                        onClick = onNavigateToFocus,
                        modifier = Modifier.weight(1f)
                    ) {
                        Text("Start Focus")
                    }
                    OutlinedButton(
                        onClick = onNavigateToApps,
                        modifier = Modifier.weight(1f)
                    ) {
                        Text("Manage Apps")
                    }
                }
            }
        }
    }
}
```

### `presentation/components/StatCard.kt`

```kotlin
@Composable
fun StatCard(
    modifier: Modifier = Modifier,
    title: String,
    value: String,
    subtitle: String,
    icon: ImageVector
) {
    Card(modifier = modifier) {
        Column(
            modifier = Modifier.padding(16.dp),
            verticalArrangement = Arrangement.spacedBy(4.dp)
        ) {
            Icon(
                imageVector = icon,
                contentDescription = null,
                tint = MaterialTheme.colorScheme.primary,
                modifier = Modifier.size(20.dp)
            )
            Text(
                text = value,
                style = MaterialTheme.typography.headlineMedium,
                fontWeight = FontWeight.Bold
            )
            Text(
                text = title,
                style = MaterialTheme.typography.labelMedium,
                color = MaterialTheme.colorScheme.onSurfaceVariant
            )
            Text(
                text = subtitle,
                style = MaterialTheme.typography.bodySmall,
                color = MaterialTheme.colorScheme.onSurfaceVariant
            )
        }
    }
}
```

---

## 16. Navigation Setup

### `presentation/navigation/Screen.kt`

```kotlin
sealed class Screen(val route: String) {
    object Splash : Screen("splash")
    object Onboarding : Screen("onboarding")
    object PermissionSetup : Screen("permission_setup")
    object Login : Screen("login")
    object Register : Screen("register")
    object Home : Screen("home")
    object AppSelection : Screen("app_selection")
    object FocusMode : Screen("focus_mode")
    object Analytics : Screen("analytics")
    object Schedule : Screen("schedule")
    object Settings : Screen("settings")
    object Profile : Screen("profile")
}
```

### `presentation/navigation/NavGraph.kt`

```kotlin
@Composable
fun FocusShieldNavGraph(
    navController: NavHostController,
    startDestination: String
) {
    NavHost(
        navController = navController,
        startDestination = startDestination
    ) {
        composable(Screen.Splash.route) {
            SplashScreen(
                onNavigateToOnboarding = {
                    navController.navigate(Screen.Onboarding.route) {
                        popUpTo(Screen.Splash.route) { inclusive = true }
                    }
                },
                onNavigateToHome = {
                    navController.navigate(Screen.Home.route) {
                        popUpTo(Screen.Splash.route) { inclusive = true }
                    }
                }
            )
        }

        composable(Screen.Onboarding.route) {
            OnboardingScreen(
                onComplete = {
                    navController.navigate(Screen.PermissionSetup.route)
                }
            )
        }

        composable(Screen.PermissionSetup.route) {
            PermissionSetupScreen(
                onAllPermissionsGranted = {
                    navController.navigate(Screen.Home.route) {
                        popUpTo(Screen.Onboarding.route) { inclusive = true }
                    }
                }
            )
        }

        composable(Screen.Home.route) {
            HomeScreen(
                onNavigateToApps = { navController.navigate(Screen.AppSelection.route) },
                onNavigateToFocus = { navController.navigate(Screen.FocusMode.route) },
                onNavigateToAnalytics = { navController.navigate(Screen.Analytics.route) }
            )
        }

        composable(Screen.AppSelection.route) {
            AppSelectionScreen(
                onBack = { navController.popBackStack() }
            )
        }

        composable(Screen.FocusMode.route) {
            FocusModeScreen(
                onBack = { navController.popBackStack() }
            )
        }

        composable(Screen.Analytics.route) {
            AnalyticsScreen(
                onBack = { navController.popBackStack() }
            )
        }

        composable(Screen.Settings.route) {
            SettingsScreen(
                onBack = { navController.popBackStack() }
            )
        }
    }
}
```

### `MainActivity.kt`

```kotlin
@AndroidEntryPoint
class MainActivity : ComponentActivity() {

    @Inject lateinit var userPreferencesDataStore: UserPreferencesDataStore

    override fun onCreate(savedInstanceState: Bundle?) {
        installSplashScreen()
        super.onCreate(savedInstanceState)

        setContent {
            FocusShieldTheme {
                val navController = rememberNavController()
                val isOnboardingDone by userPreferencesDataStore.isOnboardingDone
                    .collectAsStateWithLifecycle(initialValue = false)

                val startDestination = if (isOnboardingDone) {
                    Screen.Home.route
                } else {
                    Screen.Onboarding.route
                }

                FocusShieldNavGraph(
                    navController = navController,
                    startDestination = startDestination
                )
            }
        }
    }
}
```

---

## 17. Permission Onboarding Flow

This is critical — users must grant 4 permissions before the app works.

### `presentation/screens/onboarding/PermissionSetupScreen.kt`

```kotlin
@Composable
fun PermissionSetupScreen(
    onAllPermissionsGranted: () -> Unit,
    viewModel: PermissionSetupViewModel = hiltViewModel()
) {
    val context = LocalContext.current
    val uiState by viewModel.uiState.collectAsStateWithLifecycle()

    val permissions = listOf(
        PermissionItem(
            title = "Accessibility Service",
            description = "Allows Focus Shield to detect and block Reels content",
            isGranted = uiState.hasAccessibility,
            onGrant = {
                val intent = Intent(Settings.ACTION_ACCESSIBILITY_SETTINGS)
                context.startActivity(intent)
            }
        ),
        PermissionItem(
            title = "Usage Access",
            description = "Allows tracking which apps you use and for how long",
            isGranted = uiState.hasUsageAccess,
            onGrant = {
                val intent = Intent(Settings.ACTION_USAGE_ACCESS_SETTINGS)
                context.startActivity(intent)
            }
        ),
        PermissionItem(
            title = "Display Over Apps",
            description = "Required to show the block screen over Instagram/YouTube",
            isGranted = uiState.hasOverlayPermission,
            onGrant = {
                val intent = Intent(
                    Settings.ACTION_MANAGE_OVERLAY_PERMISSION,
                    Uri.parse("package:${context.packageName}")
                )
                context.startActivity(intent)
            }
        ),
        PermissionItem(
            title = "Battery Optimization",
            description = "Prevents Android from stopping Focus Shield in background",
            isGranted = uiState.hasBatteryOptimization,
            onGrant = {
                val intent = Intent(Settings.ACTION_REQUEST_IGNORE_BATTERY_OPTIMIZATIONS).apply {
                    data = Uri.parse("package:${context.packageName}")
                }
                context.startActivity(intent)
            }
        )
    )

    // Check permissions when screen resumes (user comes back from Settings)
    val lifecycleOwner = LocalLifecycleOwner.current
    DisposableEffect(lifecycleOwner) {
        val observer = LifecycleEventObserver { _, event ->
            if (event == Lifecycle.Event.ON_RESUME) {
                viewModel.checkAllPermissions()
            }
        }
        lifecycleOwner.lifecycle.addObserver(observer)
        onDispose { lifecycleOwner.lifecycle.removeObserver(observer) }
    }

    LaunchedEffect(uiState.allGranted) {
        if (uiState.allGranted) onAllPermissionsGranted()
    }

    Column(
        modifier = Modifier
            .fillMaxSize()
            .padding(24.dp),
        verticalArrangement = Arrangement.spacedBy(16.dp)
    ) {
        Text(
            text = "Setup Required",
            style = MaterialTheme.typography.headlineMedium,
            fontWeight = FontWeight.Bold
        )
        Text(
            text = "Focus Shield needs these permissions to block Reels. Each one is essential.",
            style = MaterialTheme.typography.bodyMedium,
            color = MaterialTheme.colorScheme.onSurfaceVariant
        )

        Spacer(modifier = Modifier.height(8.dp))

        permissions.forEach { permission ->
            PermissionCard(permission = permission)
        }

        Spacer(modifier = Modifier.weight(1f))

        val allGranted = permissions.all { it.isGranted }
        Button(
            onClick = { if (allGranted) onAllPermissionsGranted() },
            enabled = allGranted,
            modifier = Modifier.fillMaxWidth()
        ) {
            Text(if (allGranted) "All Set! Continue" else "Grant All Permissions Above")
        }
    }
}

data class PermissionItem(
    val title: String,
    val description: String,
    val isGranted: Boolean,
    val onGrant: () -> Unit
)

@Composable
fun PermissionCard(permission: PermissionItem) {
    Card(
        modifier = Modifier.fillMaxWidth(),
        colors = CardDefaults.cardColors(
            containerColor = if (permission.isGranted)
                MaterialTheme.colorScheme.primaryContainer
            else
                MaterialTheme.colorScheme.surface
        )
    ) {
        Row(
            modifier = Modifier.padding(16.dp),
            horizontalArrangement = Arrangement.spacedBy(12.dp),
            verticalAlignment = Alignment.CenterVertically
        ) {
            Icon(
                imageVector = if (permission.isGranted)
                    Icons.Default.CheckCircle
                else
                    Icons.Default.RadioButtonUnchecked,
                contentDescription = null,
                tint = if (permission.isGranted)
                    MaterialTheme.colorScheme.primary
                else
                    MaterialTheme.colorScheme.outline
            )

            Column(modifier = Modifier.weight(1f)) {
                Text(permission.title, style = MaterialTheme.typography.titleSmall)
                Text(
                    permission.description,
                    style = MaterialTheme.typography.bodySmall,
                    color = MaterialTheme.colorScheme.onSurfaceVariant
                )
            }

            if (!permission.isGranted) {
                TextButton(onClick = permission.onGrant) {
                    Text("Grant")
                }
            }
        }
    }
}
```

### `PermissionSetupViewModel.kt`

```kotlin
@HiltViewModel
class PermissionSetupViewModel @Inject constructor(
    @ApplicationContext private val context: Context
) : ViewModel() {

    private val _uiState = MutableStateFlow(PermissionUiState())
    val uiState: StateFlow<PermissionUiState> = _uiState.asStateFlow()

    init {
        checkAllPermissions()
    }

    fun checkAllPermissions() {
        val hasAccessibility = isAccessibilityEnabled()
        val hasUsageAccess = isUsageAccessGranted()
        val hasOverlay = Settings.canDrawOverlays(context)
        val hasBattery = isIgnoringBatteryOptimizations()

        _uiState.value = PermissionUiState(
            hasAccessibility = hasAccessibility,
            hasUsageAccess = hasUsageAccess,
            hasOverlayPermission = hasOverlay,
            hasBatteryOptimization = hasBattery,
            allGranted = hasAccessibility && hasUsageAccess && hasOverlay && hasBattery
        )
    }

    private fun isAccessibilityEnabled(): Boolean {
        val service = "${context.packageName}/.services.FocusAccessibilityService"
        val enabled = Settings.Secure.getString(
            context.contentResolver,
            Settings.Secure.ENABLED_ACCESSIBILITY_SERVICES
        ) ?: return false
        return enabled.contains(service)
    }

    private fun isUsageAccessGranted(): Boolean {
        val appOps = context.getSystemService(Context.APP_OPS_SERVICE) as AppOpsManager
        val mode = appOps.checkOpNoThrow(
            AppOpsManager.OPSTR_GET_USAGE_STATS,
            android.os.Process.myUid(),
            context.packageName
        )
        return mode == AppOpsManager.MODE_ALLOWED
    }

    private fun isIgnoringBatteryOptimizations(): Boolean {
        val pm = context.getSystemService(Context.POWER_SERVICE) as PowerManager
        return pm.isIgnoringBatteryOptimizations(context.packageName)
    }
}

data class PermissionUiState(
    val hasAccessibility: Boolean = false,
    val hasUsageAccess: Boolean = false,
    val hasOverlayPermission: Boolean = false,
    val hasBatteryOptimization: Boolean = false,
    val allGranted: Boolean = false
)
```

---

## 18. Battery Optimization Fix

This is the most common reason the app stops working on real devices.

### What happens without this fix

Samsung, Xiaomi, OPPO, OnePlus — all kill background services aggressively. The Foreground Service gets killed, the Accessibility Service stops, and Reels are no longer blocked.

### Fix 1: Request battery optimization exemption (in manifest)

Already in AndroidManifest.xml above:
```xml
<uses-permission android:name="android.permission.REQUEST_IGNORE_BATTERY_OPTIMIZATIONS" />
```

### Fix 2: Detect and guide users (in PermissionSetupScreen — already shown above)

### Fix 3: Manufacturer-specific instructions

```kotlin
object BatteryOptimizationHelper {

    fun getManufacturerInstructions(): String? {
        return when (Build.MANUFACTURER.lowercase()) {
            "xiaomi", "redmi" -> "Settings → Apps → Manage Apps → Focus Shield → Battery Saver → No restrictions. Also enable AutoStart."
            "huawei", "honor" -> "Settings → Apps → Focus Shield → Battery → Enable 'Keep running after screen off'"
            "samsung" -> "Settings → Apps → Focus Shield → Battery → 'Unrestricted'. Also: Settings → Device Care → Battery → App Power Management → never sleeping apps → add Focus Shield"
            "oneplus" -> "Settings → Battery → Battery Optimization → Focus Shield → Don't optimize"
            "oppo", "realme" -> "Settings → App Management → Focus Shield → Battery Saver → No restrictions"
            else -> null
        }
    }
}
```

---

## 19. Strict Mode & PIN Protection

### `utils/PinHasher.kt`

```kotlin
object PinHasher {
    private const val SALT = "focus_shield_salt_v1"

    fun hash(pin: String): String {
        val input = "$SALT$pin"
        val digest = MessageDigest.getInstance("SHA-256")
        val bytes = digest.digest(input.toByteArray())
        return bytes.joinToString("") { "%02x".format(it) }
    }

    fun verify(enteredPin: String, storedHash: String): Boolean {
        return hash(enteredPin) == storedHash
    }
}
```

### Strict Mode Logic

When Strict Mode is on:
1. PIN required before disabling any blocked app
2. 5 second delay before overlay can be dismissed
3. Cannot disable Accessibility Service without PIN (show warning)

```kotlin
// In SettingsViewModel
fun disableStrictMode(enteredPin: String) {
    viewModelScope.launch {
        val isValid = verifyPinUseCase(enteredPin)
        if (isValid) {
            settingsRepository.setStrictMode(false)
            _uiState.value = _uiState.value.copy(strictModeDisabled = true)
        } else {
            _uiState.value = _uiState.value.copy(wrongPinError = true)
        }
    }
}
```

---

## 20. Backend API (Node.js) — V2 Only

> **Skip this for V1.** Build it only when local features are working perfectly.

### Project structure

```
backend/
├── src/
│   ├── routes/
│   │   ├── auth.js
│   │   ├── stats.js
│   │   └── settings.js
│   ├── models/
│   │   ├── User.js
│   │   ├── BlockEvent.js
│   │   └── UserSettings.js
│   ├── middleware/
│   │   └── auth.js
│   └── app.js
├── .env
└── package.json
```

### `package.json`

```json
{
  "name": "focus-shield-api",
  "version": "1.0.0",
  "scripts": { "start": "node src/app.js", "dev": "nodemon src/app.js" },
  "dependencies": {
    "express": "^4.18.2",
    "mongoose": "^8.0.0",
    "jsonwebtoken": "^9.0.0",
    "bcryptjs": "^2.4.3",
    "cors": "^2.8.5",
    "dotenv": "^16.0.0"
  }
}
```

### `src/app.js`

```javascript
require('dotenv').config();
const express = require('express');
const mongoose = require('mongoose');
const cors = require('cors');

const app = express();
app.use(cors());
app.use(express.json());

mongoose.connect(process.env.MONGODB_URI);

app.use('/api/auth', require('./routes/auth'));
app.use('/api/stats', require('./routes/stats'));
app.use('/api/settings', require('./routes/settings'));

app.listen(process.env.PORT || 3000);
```

### Android Retrofit Setup (V2)

```kotlin
// di/NetworkModule.kt
@Module
@InstallIn(SingletonComponent::class)
object NetworkModule {

    @Provides
    @Singleton
    fun provideOkHttpClient(): OkHttpClient {
        return OkHttpClient.Builder()
            .addInterceptor(HttpLoggingInterceptor().apply {
                level = HttpLoggingInterceptor.Level.BODY
            })
            .addInterceptor { chain ->
                // Add JWT token to every request
                val token = TokenManager.getToken()
                val request = chain.request().newBuilder()
                    .addHeader("Authorization", "Bearer $token")
                    .build()
                chain.proceed(request)
            }
            .build()
    }

    @Provides
    @Singleton
    fun provideRetrofit(okHttpClient: OkHttpClient): Retrofit {
        return Retrofit.Builder()
            .baseUrl(BuildConfig.BASE_URL)
            .client(okHttpClient)
            .addConverterFactory(GsonConverterFactory.create())
            .build()
    }

    @Provides
    @Singleton
    fun provideFocusShieldApi(retrofit: Retrofit): FocusShieldApi {
        return retrofit.create(FocusShieldApi::class.java)
    }
}
```

---

## 21. Build & Release

### `local.properties`
```
sdk.dir=/path/to/android/sdk
```

### `gradle.properties`
```
android.useAndroidX=true
kotlin.code.style=official
android.nonTransitiveRClass=true
org.gradle.jvmargs=-Xmx4g -XX:+UseParallelGC
```

### Release signing in `build.gradle.kts`

```kotlin
android {
    signingConfigs {
        create("release") {
            keyAlias = System.getenv("KEY_ALIAS")
            keyPassword = System.getenv("KEY_PASSWORD")
            storeFile = file(System.getenv("KEYSTORE_PATH") ?: "keystore.jks")
            storePassword = System.getenv("STORE_PASSWORD")
        }
    }

    buildTypes {
        release {
            isMinifyEnabled = true
            isShrinkResources = true
            proguardFiles(getDefaultProguardFile("proguard-android-optimize.txt"), "proguard-rules.pro")
            signingConfig = signingConfigs.getByName("release")
        }
        debug {
            isDebuggable = true
            applicationIdSuffix = ".debug"
        }
    }
}
```

### `proguard-rules.pro`

```
-keep class com.focusshield.app.data.local.db.entity.** { *; }
-keep class com.focusshield.app.domain.model.** { *; }
-keepclassmembers class * extends androidx.room.RoomDatabase { *; }
-keep @dagger.hilt.android.HiltAndroidApp class *
```

---

## 22. Testing Checklist

Run these tests on a real device — emulator cannot test Accessibility Service properly.

### Permission Tests
- [ ] App requests Accessibility permission on first launch
- [ ] App requests Usage Access permission
- [ ] App requests Overlay permission  
- [ ] App requests Battery Optimization exemption
- [ ] App correctly detects when each permission is granted
- [ ] App shows correct status after returning from Settings

### Blocking Tests
- [ ] Open Instagram → scroll to Reels tab → block screen appears
- [ ] Open YouTube → scroll to Shorts → block screen appears
- [ ] Open TikTok → block screen appears immediately
- [ ] Open Facebook → scroll to Reels → block screen appears
- [ ] Press "Go Back" on block screen → returns to home screen
- [ ] Block event is logged in database after each block
- [ ] Block count increments on home dashboard

### Service Persistence Tests
- [ ] Lock phone for 5 minutes → unlock → open Instagram Reels → still blocked
- [ ] Restart phone → open Instagram Reels → still blocked (BootReceiver)
- [ ] Force stop app → check if Foreground Service restarts (on supported devices)
- [ ] Low battery mode → check if blocking still works

### Strict Mode Tests
- [ ] Enable Strict Mode → try to disable it without PIN → should fail
- [ ] Enter correct PIN → Strict Mode disables
- [ ] Enter wrong PIN 3 times → show lockout message

### UI Tests
- [ ] Dashboard shows correct block count
- [ ] Dashboard shows correct streak days
- [ ] App selection screen shows all 5 apps
- [ ] Toggle app on/off → blocking updates within 2 seconds
- [ ] Analytics screen loads weekly chart
- [ ] Dark mode works on all screens

---

## 23. Common Errors & Fixes

### Error: Accessibility Service not detecting Reels after Instagram update

**Cause:** Instagram changed the viewId of Reels container.

**Fix:**
1. Install `Layout Inspector` in Android Studio
2. Open Instagram Reels on device
3. Connect with Layout Inspector → capture snapshot
4. Find the ViewId of the Reels scroll container
5. Update `AppBlockList.REELS_NODE_IDS` with new ID

### Error: `IllegalStateException: Not allowed to start service Intent` 

**Cause:** Trying to start Foreground Service when app is in background on Android 12+.

**Fix:** Use `WorkManager` instead for deferred starts, or call `startForegroundService()` only when app is in foreground.

### Error: Overlay not showing on some devices

**Cause:** `Settings.canDrawOverlays()` returns false even after permission granted.

**Fix:**
```kotlin
// Add delay after returning from settings (some devices need time)
viewModelScope.launch {
    delay(500)
    viewModel.checkAllPermissions()
}
```

### Error: `Room cannot verify the data integrity` crash

**Cause:** Database schema changed without migration.

**Fix:** Either implement proper migration or use `fallbackToDestructiveMigration()` during development.

### Error: Hilt injection fails in Service

**Cause:** Service not annotated with `@AndroidEntryPoint`.

**Fix:**
```kotlin
@AndroidEntryPoint  // ← Must add this
class FocusAccessibilityService : AccessibilityService() {
    @Inject lateinit var blockRepository: BlockRepository
```

### Error: `ComposeView` in Overlay crashes with lifecycle error

**Cause:** `ComposeView` needs a `ViewTreeLifecycleOwner`.

**Fix:**
```kotlin
class MyLifecycleOwner : LifecycleOwner, DefaultLifecycleObserver {
    private val lifecycleRegistry = LifecycleRegistry(this)
    override val lifecycle: Lifecycle get() = lifecycleRegistry

    fun start() { lifecycleRegistry.currentState = Lifecycle.State.STARTED }
    fun stop() { lifecycleRegistry.currentState = Lifecycle.State.DESTROYED }
}

// In OverlayService:
val lifecycleOwner = MyLifecycleOwner().also { it.start() }
composeView.setViewTreeLifecycleOwner(lifecycleOwner)
```

---

## 24. V1 vs V2 Feature Split

### Build V1 First (Complete these before anything else)

| Feature | Status |
|---------|--------|
| Permission onboarding flow | Required |
| Accessibility Service blocking engine | Required |
| Overlay block screen | Required |
| Foreground Service (keep alive) | Required |
| Boot receiver (restart after reboot) | Required |
| App selection (enable/disable per app) | Required |
| Home dashboard (block count, streak) | Required |
| Room database (block events, blocked apps) | Required |
| DataStore (user preferences) | Required |
| Battery optimization guide | Required |
| Strict Mode + PIN | Required |
| Dark mode support | Required |

### Build V2 After V1 is Working

| Feature | Notes |
|---------|-------|
| Backend (Node.js + MongoDB) | Only for sync/parent mode |
| Firebase Authentication | Replace local-only accounts |
| Weekly/monthly analytics charts | Nice to have |
| Parent Mode with remote sync | Needs backend |
| Achievements system | Gamification |
| AI scrolling detection | Advanced, needs ML model |
| Multi-device sync | Needs backend |
| Subscription/premium tier | After user base established |
| Snapchat Spotlight blocking | Optional |
| Pomodoro timer | Can add to FocusMode screen |

---

## Final Notes

1. **Test on real devices only.** Emulators cannot run Accessibility Service properly.

2. **Test on at least 3 manufacturers** — Samsung, Xiaomi, and one stock Android device. Battery optimization behavior is completely different between them.

3. **The node IDs in `AppBlockList.kt` will break when Instagram/YouTube update.** Plan to update these every 2–3 months. Consider fetching them from a remote config (Firebase Remote Config is free).

4. **Do not add the backend in V1.** Room + DataStore is enough for everything. Backend adds auth complexity, server costs, and API maintenance. Build it only when you have users who need multi-device sync.

5. **Never store PIN in plain text.** The `PinHasher` already handles this correctly.

6. **Always handle the case where Accessibility Service is disabled.** Show a banner on the home screen when protection is off.
