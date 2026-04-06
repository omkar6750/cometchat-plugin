---
name: cometchat-flutter
description: Integrate CometChat Flutter UI Kit v5 into a Flutter app. Handles pubspec.yaml dependencies, Android minSdk/Jetifier config, iOS Podfile deployment target, Dart initialization, and login flow. Use for Conversation List + Messages, One-to-One/Group Chat, or Tab-Based Chat in Flutter. Do not use for React, React Native, or Android native.
---

## HARD RULES

```
- Do not duplicate dependencies — check pubspec.yaml before adding
- Do not overwrite files with real business logic — PATCH only
- Infer credentials from existing CometChatUIKit.init or config files before asking
- Never introduce Auth Key if the project already has a server-side auth token endpoint
- init() → login() → render: never break this order
- init() must complete (await) before login() is called — async/await, not fire-and-forget
- Auth Key must never appear in committed source — use environment config or .env with flutter_dotenv
- Return empty Container() or SizedBox.shrink() from widgets on loading/empty states — never return null from build()
- Never call setState() after dispose() — cancel listeners in dispose()
- Always check mounted before setState in async callbacks
```

---

## AGENT CONTRACT

**Goal:** Integrate CometChat Flutter UI Kit v5 into a Flutter app — experience chosen by user.

**Required inputs:**
- `APP_ID` (string)
- `REGION` ("us" | "eu" | "in")
- `AUTH_KEY` (string, dev only) OR existing auth token endpoint
- Login UID (string) — default: `cometchat-uid-1`
- Experience: "conversation-list" | "one-to-one" | "tab"

**Output files:**
- `pubspec.yaml` — dependencies added (PATCH only)
- `android/gradle.properties` — `android.enableJetifier=true`
- `android/app/build.gradle` or `build.gradle.kts` — `minSdk = 24`
- `ios/Podfile` — `platform :ios, '13.0'`
- `lib/cometchat_config.dart` — credentials (CREATE)
- `lib/main.dart` — init + login + navigation (PATCH or CREATE depending on state)
- `lib/cometchat/chat_screen.dart` — experience UI (CREATE)

**Invariants:**
1. `CometChatUIKit.init()` MUST complete (await) before `login()` is called
2. `login()` MUST complete before any CometChat widget renders
3. Never pass both `user` and `group` to the same widget instance
4. Auth Key must not appear in committed Dart source — use a config file added to `.gitignore`
5. `android.enableJetifier=true` MUST be set — CometChat depends on AndroidX

**Failure modes:**
- Blank screen → login() called before init() completes
- Blank screen → widget rendered before login() completes
- Build failure on Android → missing `android.enableJetifier=true` or minSdk < 24
- Build failure on iOS → Podfile deployment target < 13.0 or missing `pod install`
- Runtime crash → setState called after dispose in async callback
- Silent failure → login error caught but not surfaced to UI

**Completion criteria:**
- `flutter build apk --debug` exits 0
- `flutter build ios --no-codesign` exits 0 (on macOS)
- No CometChat widget rendered before login() resolves
- Visible error UI on init/login failure
- Auth Key only in config file (not in committed source)

---

## DECISION LOGIC

```
// Stop early if wrong skill
IF pubspec.yaml does NOT exist → STOP, not a Flutter project
IF package.json exists AND has "react-native" → STOP, use cometchat-react-native-cli or cometchat-react-native-expo
IF package.json exists AND has "react" → STOP, use appropriate React skill

// Gradle DSL detection (CRITICAL — prevents build failures)
IF android/app/build.gradle exists → Groovy DSL
  - Use: minSdkVersion 24 or minSdk 24
  - Use: implementation "..." (quotes)
IF android/app/build.gradle.kts exists → Kotlin DSL
  - Use: minSdk = 24
  - Use: implementation("...") (parentheses)
NEVER apply Groovy syntax to .kts files or vice versa

// Auth strategy
IF project has existing auth token endpoint → use loginWithAuthToken(token)
IF project is clearly dev/demo → use login(UID) with Auth Key (add TODO for production)

// Credential source
IF lib/ contains existing CometChatUIKit.init call → extract credentials
IF .env or config file has APP_ID → reuse
Only ask user for values still missing

// File operations
IF file exists AND has real logic → PATCH only
IF file exists AND is boilerplate stub → REPLACE allowed
IF file does not exist → CREATE in lib/cometchat/ subfolder

// Calling support
IF user requests voice/video → add cometchat_calls_uikit dependency
IF user does not request calling → omit calls dependency (saves ~15MB on app size)
```

---

## Step 0 — Check Existing State

Before writing anything:
1. Read `pubspec.yaml` — if `cometchat_chat_uikit` already in dependencies, skip install
2. Check `android/app/build.gradle` or `build.gradle.kts` — note which DSL is used
3. Check `android/gradle.properties` — if `android.enableJetifier=true` exists, skip
4. Check `ios/Podfile` — note current platform version
5. Grep `lib/` for `CometChatUIKit.init` — if found, extract credentials
6. Read `lib/main.dart` — if it has real app logic (auth, navigation, state management), PATCH only

---

## Step 1 — Confirm Framework

Read `pubspec.yaml`. Confirm `flutter` SDK dependency is present. Check Flutter version in environment constraints.

**Minimum versions:**
- Flutter 3.0+
- Dart 2.19+
- Android minSdk 24
- iOS deployment target 13.0

---

## Step 2 — Collect Credentials (Infer First)

1. Grep `lib/` for existing CometChat config (AppID, Region, AuthKey)
2. Check for `.env` file with credentials
3. Default LOGIN_UID to `cometchat-uid-1`
4. Only ask user for missing values

---

## Step 3 — Add Dependencies

PATCH `pubspec.yaml` — add only the missing dependencies:

```yaml
dependencies:
  cometchat_chat_uikit: ^5.2.12
  # Optional — only if voice/video calling is needed:
  cometchat_calls_uikit: ^5.0.13
```

Then run:
```bash
flutter pub get
```

---

## Step 4 — Platform Configuration

### Android

**Gradle DSL pre-check — read the file first:**

IF `android/app/build.gradle` (Groovy):
```gradle
android {
    defaultConfig {
        minSdk 24
    }
}
```

IF `android/app/build.gradle.kts` (Kotlin DSL):
```kotlin
android {
    defaultConfig {
        minSdk = 24
    }
}
```

PATCH `android/gradle.properties` — add if not present:
```properties
android.enableJetifier=true
```

### iOS

PATCH `ios/Podfile` — set platform if < 13.0:
```ruby
platform :ios, '13.0'
```

Then run:
```bash
cd ios && pod install && cd ..
```

---

## Step 5 — Create Config File

CREATE `lib/cometchat_config.dart`:

```dart
class CometChatConfig {
  static const String appId = "APP_ID";
  static const String region = "REGION";
  static const String authKey = "AUTH_KEY";
}
```

Add to `.gitignore`:
```
lib/cometchat_config.dart
```

---

## Step 6 — Init + Login

Read `lib/main.dart` first.

**If main.dart is boilerplate (counter app, "Hello World") → REPLACE with:**

```dart
import 'package:flutter/material.dart';
import 'package:cometchat_chat_uikit/cometchat_chat_uikit.dart';
import 'cometchat_config.dart';
import 'cometchat/chat_screen.dart';

void main() => runApp(const MyApp());

class MyApp extends StatelessWidget {
  const MyApp({super.key});

  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      title: 'CometChat',
      theme: ThemeData(useMaterial3: true),
      home: const CometChatInitializer(),
    );
  }
}

class CometChatInitializer extends StatefulWidget {
  const CometChatInitializer({super.key});

  @override
  State<CometChatInitializer> createState() => _CometChatInitializerState();
}

class _CometChatInitializerState extends State<CometChatInitializer> {
  bool _initialized = false;
  String? _error;

  @override
  void initState() {
    super.initState();
    _initAndLogin();
  }

  Future<void> _initAndLogin() async {
    try {
      final settings = (UIKitSettingsBuilder()
            ..appId = CometChatConfig.appId
            ..region = CometChatConfig.region
            ..authKey = CometChatConfig.authKey
            ..subscriptionType = CometChatSubscriptionType.allUsers
            ..autoEstablishSocketConnection = true)
          .build();

      await CometChatUIKit.init(uiKitSettings: settings);

      final existingUser = await CometChatUIKit.getLoggedInUser();
      if (existingUser == null) {
        await CometChatUIKit.login("cometchat-uid-1");
      }

      if (mounted) setState(() => _initialized = true);
    } catch (e) {
      if (mounted) setState(() => _error = e.toString());
    }
  }

  @override
  Widget build(BuildContext context) {
    if (_error != null) {
      return Scaffold(
        body: Center(
          child: Text(_error!, style: const TextStyle(color: Colors.red, fontFamily: 'monospace')),
        ),
      );
    }
    if (!_initialized) {
      return const Scaffold(body: Center(child: CircularProgressIndicator()));
    }
    return const ChatScreen();
  }
}
```

**If main.dart has real app logic → PATCH only:**
1. Add CometChat imports at the top
2. Add the initializer as a wrapper widget or call init in the existing startup flow
3. Do NOT remove existing routing, auth, or state management

---

## Step 7A — Experience 1: Conversation List + Message View

CREATE `lib/cometchat/chat_screen.dart`:

```dart
import 'package:flutter/material.dart';
import 'package:cometchat_chat_uikit/cometchat_chat_uikit.dart';

class ChatScreen extends StatefulWidget {
  const ChatScreen({super.key});

  @override
  State<ChatScreen> createState() => _ChatScreenState();
}

class _ChatScreenState extends State<ChatScreen> {
  User? _selectedUser;
  Group? _selectedGroup;

  void _onConversationTap(Conversation conversation) {
    final entity = conversation.conversationWith;
    setState(() {
      if (entity is User) {
        _selectedUser = entity;
        _selectedGroup = null;
      } else if (entity is Group) {
        _selectedUser = null;
        _selectedGroup = entity;
      }
    });
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      body: Row(
        children: [
          SizedBox(
            width: 360,
            child: CometChatConversations(
              onItemTap: _onConversationTap,
            ),
          ),
          Expanded(
            child: _selectedUser != null
                ? CometChatMessages(user: _selectedUser!)
                : _selectedGroup != null
                    ? CometChatMessages(group: _selectedGroup!)
                    : const Center(child: Text('Select a conversation')),
          ),
        ],
      ),
    );
  }
}
```

---

## Step 7B — Experience 2: One-to-One / Group Chat

CREATE `lib/cometchat/chat_screen.dart`:

```dart
import 'package:flutter/material.dart';
import 'package:cometchat_chat_uikit/cometchat_chat_uikit.dart';

class ChatScreen extends StatelessWidget {
  const ChatScreen({super.key});

  @override
  Widget build(BuildContext context) {
    // Replace with target user UID
    return FutureBuilder<User>(
      future: CometChat.getUser("cometchat-uid-2"),
      builder: (context, snapshot) {
        if (snapshot.hasError) {
          return Scaffold(
            body: Center(child: Text('Error: ${snapshot.error}', style: const TextStyle(color: Colors.red))),
          );
        }
        if (!snapshot.hasData) {
          return const Scaffold(body: Center(child: CircularProgressIndicator()));
        }
        return Scaffold(
          body: CometChatMessages(user: snapshot.data!),
        );
      },
    );
  }
}
```

---

## Step 7C — Experience 3: Tab-Based Chat

CREATE `lib/cometchat/chat_screen.dart`:

```dart
import 'package:flutter/material.dart';
import 'package:cometchat_chat_uikit/cometchat_chat_uikit.dart';

class ChatScreen extends StatefulWidget {
  const ChatScreen({super.key});

  @override
  State<ChatScreen> createState() => _ChatScreenState();
}

class _ChatScreenState extends State<ChatScreen> {
  int _currentTab = 0;
  User? _selectedUser;
  Group? _selectedGroup;

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      body: Row(
        children: [
          SizedBox(
            width: 360,
            child: Column(
              children: [
                Expanded(child: _buildTabContent()),
              ],
            ),
          ),
          Expanded(
            child: _selectedUser != null
                ? CometChatMessages(user: _selectedUser!)
                : _selectedGroup != null
                    ? CometChatMessages(group: _selectedGroup!)
                    : const Center(child: Text('Select a conversation')),
          ),
        ],
      ),
      bottomNavigationBar: NavigationBar(
        selectedIndex: _currentTab,
        onDestinationSelected: (i) => setState(() => _currentTab = i),
        destinations: const [
          NavigationDestination(icon: Icon(Icons.chat), label: 'Chats'),
          NavigationDestination(icon: Icon(Icons.call), label: 'Calls'),
          NavigationDestination(icon: Icon(Icons.person), label: 'Users'),
          NavigationDestination(icon: Icon(Icons.group), label: 'Groups'),
        ],
      ),
    );
  }

  Widget _buildTabContent() {
    switch (_currentTab) {
      case 0:
        return CometChatConversations(
          onItemTap: (conversation) {
            final entity = conversation.conversationWith;
            setState(() {
              if (entity is User) { _selectedUser = entity; _selectedGroup = null; }
              else if (entity is Group) { _selectedUser = null; _selectedGroup = entity; }
            });
          },
        );
      case 1:
        return CometChatCallLogs(
          onItemClick: (callLog) {},
        );
      case 2:
        return CometChatUsers(
          onItemTap: (user) => setState(() { _selectedUser = user; _selectedGroup = null; }),
        );
      case 3:
        return CometChatGroups(
          onItemTap: (group) => setState(() { _selectedUser = null; _selectedGroup = group; }),
        );
      default:
        return const SizedBox.shrink();
    }
  }
}
```

---

## Step 8 — Substitute Credentials

Replace all placeholders in generated files with real values from Step 2.

---

## Step 9 — Run

```bash
flutter run
```

---

## Agent Verification Checklist

- [ ] `flutter build apk --debug` exits 0
- [ ] `flutter build ios --no-codesign` exits 0 (macOS only)
- [ ] `android.enableJetifier=true` in gradle.properties
- [ ] `minSdk >= 24` in android/app/build.gradle(.kts)
- [ ] iOS deployment target >= 13.0 in Podfile
- [ ] No CometChat widget rendered before login() completes
- [ ] `user` and `group` never both passed to same widget
- [ ] Visible error UI on init/login failure
- [ ] Auth Key only in config file (added to .gitignore)
- [ ] Correct Gradle DSL syntax used (Groovy vs Kotlin DSL)

## ERROR DEBUGGING TABLE

| Symptom | Cause | Fix |
|---|---|---|
| Blank screen | login() before init() completes | Use await on init() before calling login() |
| Blank screen | Widget rendered before login() completes | Gate rendering with state flag set after login() |
| Android build failure: minSdk | minSdk < 24 | Set minSdk to 24 in build.gradle(.kts) |
| Android build failure: Jetifier | Missing android.enableJetifier=true | Add to gradle.properties |
| iOS build failure: deployment target | Podfile target < 13.0 | Set platform :ios, '13.0' and pod install |
| Gradle syntax error in .kts | Groovy syntax applied to Kotlin DSL file | Use = for assignments, parentheses for functions |
| setState after dispose | Async callback runs after widget disposed | Check `mounted` before setState |
| Silent failure | Error caught but not shown | Surface error to UI with error state widget |
