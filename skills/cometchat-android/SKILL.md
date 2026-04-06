---
name: cometchat-android
description: Integrate CometChat Android UI Kit v5 into a native Android app (Kotlin or Java, Jetpack Compose or XML Views). Handles Gradle DSL detection (Groovy vs Kotlin DSL), dependency configuration, AndroidManifest permissions, CometChat initialization, and UI component setup. Use for Conversation List + Messages, One-to-One/Group Chat, or Tab-Based Chat in Android. Do not use for Flutter, React Native, or web frameworks.
---

## HARD RULES

```
- GRADLE DSL PRE-CHECK IS MANDATORY — read file extension before ANY build.gradle changes:
  * .gradle → Groovy DSL (quotes for strings, no = for properties, apply plugin:)
  * .gradle.kts → Kotlin DSL (= for assignments, parentheses for functions, id() for plugins)
  NEVER apply Groovy syntax to .kts or vice versa — this causes IMMEDIATE build failure
- Do not duplicate dependencies — grep build.gradle(.kts) before adding
- Do not overwrite Activities with real business logic — PATCH only
- Infer credentials from existing CometChat.init or strings.xml before asking
- Never introduce Auth Key if the project already has an auth token endpoint
- init() → login() → render: never break this order
- Auth Key must never appear in committed source — use local.properties or BuildConfig
- minSdk must be >= 24 — CometChat requires Android 7.0+
- compileSdk must be >= 34 — CometChat uses modern Android APIs
- targetSdk should match compileSdk for Play Store compliance
- Kotlin version must be >= 1.8.0 — CometChat uses modern Kotlin features
- AGP (Android Gradle Plugin) must be >= 8.0 for compatibility with CometChat v5
- Do not modify ProGuard/R8 rules unless CometChat explicitly requires it
- Always check for AndroidX — CometChat requires AndroidX, not Support Library
```

---

## AGENT CONTRACT

**Goal:** Integrate CometChat Android UI Kit v5 into a native Android app — experience chosen by user.

**Required inputs:**
- `APP_ID` (string)
- `REGION` ("us" | "eu" | "in")
- `AUTH_KEY` (string, dev only) OR existing auth token endpoint
- Login UID (string) — default: `cometchat-uid-1`
- Experience: "conversation-list" | "one-to-one" | "tab"

**Output files:**
- `local.properties` — credentials (PATCH, add CometChat keys)
- `app/build.gradle(.kts)` — dependencies + minSdk (PATCH)
- `build.gradle(.kts)` (project-level) — maven repo if needed (PATCH)
- `app/src/main/AndroidManifest.xml` — permissions (PATCH)
- `app/src/main/java/.../CometChatConfig.kt` — credential constants (CREATE)
- `app/src/main/java/.../CometChatInitializer.kt` — init + login (CREATE)
- `app/src/main/java/.../ChatActivity.kt` — experience UI (CREATE)

**Invariants:**
1. `CometChatUIKit.init()` callback MUST fire before `login()` is called
2. `login()` callback MUST fire before any CometChat View is inflated
3. Never pass both `user` and `group` to the same component
4. Auth Key only in `local.properties` or `BuildConfig` — never in source strings
5. Gradle DSL must match file extension — Groovy for `.gradle`, Kotlin for `.gradle.kts`

**Failure modes:**
- Build failure: wrong Gradle DSL syntax → Check file extension, use matching syntax
- Build failure: minSdk < 24 → Set minSdk to 24
- Build failure: missing maven repo → Add CometChat maven URL to project build.gradle
- Runtime crash: init/login order broken → Gate UI behind login callback
- Runtime crash: AndroidX not enabled → Check gradle.properties for android.useAndroidX=true
- Silent failure: login error swallowed → Surface error to user in UI

**Completion criteria:**
- `./gradlew assembleDebug` exits 0
- App launches and chat UI renders
- No CometChat errors in Logcat
- Correct Gradle DSL syntax throughout

---

## DECISION LOGIC

```
// Stop early if wrong skill
IF pubspec.yaml exists → STOP, use cometchat-flutter
IF package.json exists → STOP, use appropriate JS/TS skill
IF *.xcodeproj exists AND no build.gradle → STOP, iOS project — no Android skill needed

// GRADLE DSL DETECTION — MUST DO FIRST
IF app/build.gradle exists (no .kts) → GROOVY DSL
  Syntax: minSdkVersion 24, compileSdkVersion 34
  Deps: implementation "group:artifact:version"
  Plugins: apply plugin: 'com.android.application'
IF app/build.gradle.kts exists → KOTLIN DSL
  Syntax: minSdk = 24, compileSdk = 34
  Deps: implementation("group:artifact:version")
  Plugins: id("com.android.application")

// Project-level build file
IF build.gradle exists → Groovy project file
IF build.gradle.kts exists → Kotlin DSL project file
IF settings.gradle.kts has dependencyResolutionManagement → repos are in settings, not project build file

// Android Gradle Plugin version
IF AGP < 8.0 → WARN: CometChat v5 UI Kit requires AGP >= 8.0
IF AGP >= 8.0 AND < 8.3 → version catalog may not be in use
IF AGP >= 8.3 → likely uses version catalog (libs.versions.toml)

// Version catalog detection
IF gradle/libs.versions.toml exists → add CometChat to version catalog
IF not → add dependencies directly in app/build.gradle(.kts)

// Compose vs XML
IF app/build.gradle(.kts) has compose = true or buildFeatures { compose = true } → Jetpack Compose project
IF no compose config → XML Views project
CometChat UI Kit v5 uses XML Views — in Compose projects, use AndroidView wrapper or Fragment container

// Auth strategy
IF project has auth token endpoint → use loginWithAuthToken()
IF project is dev/demo → use login(UID) with Auth Key + TODO

// Credential source
IF local.properties has COMETCHAT_APP_ID → reuse via BuildConfig
IF strings.xml has CometChat credentials → extract and move to BuildConfig
IF existing CometChat.init call in source → extract credentials
```

---

## Step 0 — Check Existing State

Before writing anything:
1. **Read file extensions** — determine Groovy (.gradle) vs Kotlin DSL (.gradle.kts) for BOTH project-level and app-level build files
2. Check `settings.gradle(.kts)` for `dependencyResolutionManagement` (modern repo config)
3. Check `gradle/libs.versions.toml` for version catalog usage
4. Read `app/build.gradle(.kts)` — note current minSdk, compileSdk, targetSdk, existing dependencies
5. Check `gradle.properties` for `android.useAndroidX=true`
6. Read `AndroidManifest.xml` for existing permissions
7. Grep source for existing `CometChat.init` or `CometChatUIKit.init`
8. Read the main Activity — if it has real logic, PATCH only

---

## Step 1 — Confirm Framework

Check for `build.gradle(.kts)` at project root and `app/build.gradle(.kts)`. Confirm this is an Android project.

Check versions:
- **AGP version:** Must be >= 8.0 (check `classpath` or `id("com.android.application") version`)
- **Kotlin version:** Must be >= 1.8.0
- **Gradle wrapper:** Check `gradle/wrapper/gradle-wrapper.properties` for compatibility

---

## Step 2 — Collect Credentials (Infer First)

1. Check `local.properties` for existing CometChat keys
2. Check `BuildConfig` usage in source
3. Grep for existing `CometChat.init` calls
4. Default UID to `cometchat-uid-1`
5. Only ask for missing values

---

## Step 3 — Add Repository

### If using `settings.gradle.kts` with `dependencyResolutionManagement`:

PATCH `settings.gradle.kts`:
```kotlin
dependencyResolutionManagement {
    repositories {
        // ... existing repos ...
        maven { url = uri("https://dl.cloudsmith.io/public/cometchat/cometchat/maven/") }
    }
}
```

### If using `settings.gradle` (Groovy):

PATCH `settings.gradle`:
```gradle
dependencyResolutionManagement {
    repositories {
        // ... existing repos ...
        maven { url "https://dl.cloudsmith.io/public/cometchat/cometchat/maven/" }
    }
}
```

### If repos are in project-level `build.gradle(.kts)`:

PATCH the `allprojects { repositories { } }` block with the same maven URL.

---

## Step 4 — Add Dependencies and Configure SDK Versions

### IF `app/build.gradle` (Groovy):

```gradle
android {
    compileSdkVersion 34

    defaultConfig {
        minSdkVersion 24
        targetSdkVersion 34
    }

    buildFeatures {
        buildConfig true
    }
}

dependencies {
    implementation "com.cometchat:chat-uikit-android:5.0.2"
    // Optional — voice/video calling:
    implementation "com.cometchat:calls-sdk-android:4.0.8"
}
```

### IF `app/build.gradle.kts` (Kotlin DSL):

```kotlin
android {
    compileSdk = 34

    defaultConfig {
        minSdk = 24
        targetSdk = 34
    }

    buildFeatures {
        buildConfig = true
    }
}

dependencies {
    implementation("com.cometchat:chat-uikit-android:5.0.2")
    // Optional — voice/video calling:
    implementation("com.cometchat:calls-sdk-android:4.0.8")
}
```

### IF using version catalog (`gradle/libs.versions.toml`):

PATCH `gradle/libs.versions.toml`:
```toml
[versions]
cometchat-uikit = "5.0.2"
cometchat-calls = "4.0.8"

[libraries]
cometchat-uikit = { group = "com.cometchat", name = "chat-uikit-android", version.ref = "cometchat-uikit" }
cometchat-calls = { group = "com.cometchat", name = "calls-sdk-android", version.ref = "cometchat-calls" }
```

Then in `app/build.gradle.kts`:
```kotlin
dependencies {
    implementation(libs.cometchat.uikit)
    implementation(libs.cometchat.calls)  // Optional
}
```

---

## Step 5 — Permissions

PATCH `app/src/main/AndroidManifest.xml` — add if not present:

```xml
<uses-permission android:name="android.permission.INTERNET" />
<uses-permission android:name="android.permission.CAMERA" />
<uses-permission android:name="android.permission.RECORD_AUDIO" />
<uses-permission android:name="android.permission.MODIFY_AUDIO_SETTINGS" />
<uses-permission android:name="android.permission.ACCESS_NETWORK_STATE" />
```

---

## Step 6 — Credentials via BuildConfig

PATCH `local.properties` (NOT committed to git):
```properties
COMETCHAT_APP_ID=your_app_id
COMETCHAT_REGION=us
COMETCHAT_AUTH_KEY=your_auth_key
```

PATCH `app/build.gradle(.kts)` to read from local.properties:

**Groovy:**
```gradle
def localProps = new Properties()
def localPropsFile = rootProject.file('local.properties')
if (localPropsFile.exists()) { localProps.load(localPropsFile.newDataInputStream()) }

android {
    defaultConfig {
        buildConfigField "String", "COMETCHAT_APP_ID", "\"${localProps.getProperty('COMETCHAT_APP_ID', '')}\""
        buildConfigField "String", "COMETCHAT_REGION", "\"${localProps.getProperty('COMETCHAT_REGION', '')}\""
        buildConfigField "String", "COMETCHAT_AUTH_KEY", "\"${localProps.getProperty('COMETCHAT_AUTH_KEY', '')}\""
    }
}
```

**Kotlin DSL:**
```kotlin
val localProps = java.util.Properties()
val localPropsFile = rootProject.file("local.properties")
if (localPropsFile.exists()) { localProps.load(localPropsFile.inputStream()) }

android {
    defaultConfig {
        buildConfigField("String", "COMETCHAT_APP_ID", "\"${localProps.getProperty("COMETCHAT_APP_ID", "")}\"")
        buildConfigField("String", "COMETCHAT_REGION", "\"${localProps.getProperty("COMETCHAT_REGION", "")}\"")
        buildConfigField("String", "COMETCHAT_AUTH_KEY", "\"${localProps.getProperty("COMETCHAT_AUTH_KEY", "")}\"")
    }
}
```

---

## Step 7 — Init + Login

CREATE `app/src/main/java/<package>/CometChatHelper.kt`:

```kotlin
package com.example.app // Replace with actual package

import android.app.Application
import com.cometchat.chatuikit.shared.cometchatuikit.CometChatUIKit
import com.cometchat.chatuikit.shared.cometchatuikit.UIKitSettings

object CometChatHelper {

    fun init(application: Application, onSuccess: () -> Unit, onError: (String) -> Unit) {
        val settings = UIKitSettings.UIKitSettingsBuilder()
            .setAppId(BuildConfig.COMETCHAT_APP_ID)
            .setRegion(BuildConfig.COMETCHAT_REGION)
            .setAuthKey(BuildConfig.COMETCHAT_AUTH_KEY)
            .subscribePresenceForAllUsers()
            .build()

        CometChatUIKit.init(application, settings,
            object : CometChat.CallbackListener<String>() {
                override fun onSuccess(s: String?) {
                    loginUser(onSuccess, onError)
                }
                override fun onError(e: CometChatException?) {
                    onError("Init failed: ${e?.message}")
                }
            }
        )
    }

    private fun loginUser(onSuccess: () -> Unit, onError: (String) -> Unit) {
        val uid = "cometchat-uid-1" // Replace with real UID

        CometChatUIKit.getLoggedInUser()?.let {
            onSuccess()
            return
        }

        CometChatUIKit.login(uid,
            object : CometChat.CallbackListener<User>() {
                override fun onSuccess(user: User?) {
                    onSuccess()
                }
                override fun onError(e: CometChatException?) {
                    onError("Login failed: ${e?.message}")
                }
            }
        )
    }
}
```

PATCH the main Activity or Application class to call `CometChatHelper.init()` on startup, then navigate to the chat Activity on success.

---

## Step 8A — Experience 1: Conversation List + Messages

CREATE `app/src/main/java/<package>/ChatActivity.kt`:

```kotlin
package com.example.app

import android.os.Bundle
import androidx.appcompat.app.AppCompatActivity
import com.cometchat.chatuikit.conversations.CometChatConversations
import com.cometchat.chatuikit.messages.CometChatMessages
import com.cometchat.chat.models.User
import com.cometchat.chat.models.Group
import com.cometchat.chat.models.Conversation

class ChatActivity : AppCompatActivity() {

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_chat)

        val conversations = findViewById<CometChatConversations>(R.id.conversations)
        conversations.setOnItemClickListener { conversation ->
            val entity = conversation.conversationWith
            when (entity) {
                is User -> showMessages(user = entity)
                is Group -> showMessages(group = entity)
            }
        }
    }

    private fun showMessages(user: User? = null, group: Group? = null) {
        val fragment = CometChatMessages()
        val bundle = Bundle()
        user?.let { bundle.putParcelable("user", it) }
        group?.let { bundle.putParcelable("group", it) }
        fragment.arguments = bundle

        supportFragmentManager.beginTransaction()
            .replace(R.id.messages_container, fragment)
            .commit()
    }
}
```

CREATE `app/src/main/res/layout/activity_chat.xml`:

```xml
<?xml version="1.0" encoding="utf-8"?>
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:orientation="horizontal">

    <com.cometchat.chatuikit.conversations.CometChatConversations
        android:id="@+id/conversations"
        android:layout_width="360dp"
        android:layout_height="match_parent" />

    <FrameLayout
        android:id="@+id/messages_container"
        android:layout_width="0dp"
        android:layout_height="match_parent"
        android:layout_weight="1" />

</LinearLayout>
```

---

## Step 9 — Run

```bash
./gradlew assembleDebug
# Or from Android Studio: Run > Run 'app'
```

---

## Agent Verification Checklist

- [ ] `./gradlew assembleDebug` exits 0
- [ ] Correct Gradle DSL syntax used throughout (Groovy vs Kotlin DSL)
- [ ] minSdk >= 24, compileSdk >= 34
- [ ] CometChat maven repo added to correct location (settings or project build file)
- [ ] `android.useAndroidX=true` in gradle.properties
- [ ] AndroidManifest has INTERNET, CAMERA, RECORD_AUDIO permissions
- [ ] No CometChat View inflated before login() callback fires
- [ ] `user` and `group` never both passed to same component
- [ ] Visible error UI on init/login failure
- [ ] Auth Key only in local.properties / BuildConfig (not in source strings)
- [ ] AGP >= 8.0, Kotlin >= 1.8.0

## ERROR DEBUGGING TABLE

| Symptom | Cause | Fix |
|---|---|---|
| Build failure: "Expecting =" | Groovy syntax in .kts file | Use `=` assignments, `()` for functions |
| Build failure: "Unresolved reference" | Kotlin DSL syntax in .gradle file | Use no `=`, use `"quotes"` not `("parens")` |
| Build failure: minSdk | minSdk < 24 | Set to 24 with correct DSL syntax |
| Build failure: missing repo | CometChat maven repo not added | Add to settings.gradle(.kts) or project build file |
| Build failure: AGP version | AGP < 8.0 | Upgrade AGP — see Android Studio > Project Structure |
| Build failure: Kotlin version | Kotlin < 1.8.0 | Update kotlin version in project build file |
| Runtime crash: init order | login() before init() callback | Gate login() inside init onSuccess callback |
| Runtime crash: AndroidX | Support Library conflict | Ensure android.useAndroidX=true and android.enableJetifier=true |
| Blank screen | UI inflated before login callback | Show loading spinner, swap to chat UI in callback |
| Version catalog not found | Wrong libs.versions.toml path | Must be at gradle/libs.versions.toml |
| .toInteger() error | Groovy method on Kotlin type | You're mixing DSLs — check file extension |
