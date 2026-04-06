---
name: cometchat-react-native-cli
description: Integrate CometChat React Native UI Kit into a bare React Native CLI project (not Expo). Handles native module linking, Android/iOS permissions, React Navigation setup, and CometChat initialization. Use for Conversation List + Messages, One-to-One/Group Chat, or Tab-Based Chat in React Native CLI. Do not use for Expo, React web, or Flutter.
---

## HARD RULES

```
- Do not duplicate dependencies — check package.json before adding
- Do not overwrite files with real business logic — PATCH only
- Infer credentials from existing .env or CometChatUIKit.init before asking
- Never introduce Auth Key if the project already has an auth token endpoint
- init() → login() → render: never break this order
- Auth Key must never appear in committed source — use react-native-config or .env
- CometChat SDK uses native modules — pod install on iOS is MANDATORY after install
- Android: minSdk must be >= 24, compileSdk >= 34
- iOS: deployment target must be >= 13.0
- Never call setState on unmounted component — use cleanup in useEffect return
- Gradle DSL pre-check: read build.gradle vs build.gradle.kts before patching
```

---

## AGENT CONTRACT

**Goal:** Integrate CometChat React Native UI Kit into a React Native CLI app — experience chosen by user.

**Required inputs:**
- `APP_ID` (string)
- `REGION` ("us" | "eu" | "in")
- `AUTH_KEY` (string, dev only) OR existing auth token endpoint
- Login UID (string) — default: `cometchat-uid-1`
- Experience: "conversation-list" | "one-to-one" | "tab"

**Output files:**
- `.env` — credentials
- `android/app/build.gradle(.kts)` — minSdk patch
- `android/app/src/main/AndroidManifest.xml` — permissions (INTERNET, CAMERA, RECORD_AUDIO, MODIFY_AUDIO_SETTINGS)
- `ios/Podfile` — deployment target patch
- `ios/<project>/Info.plist` — camera/microphone usage descriptions
- `src/CometChatInit.tsx` — init + login wrapper (CREATE)
- `src/cometchat/ChatScreen.tsx` — experience UI (CREATE)

**Invariants:**
1. `CometChatUIKit.init()` MUST resolve before `login()` is called
2. `login()` MUST resolve before any CometChat component renders
3. Never pass both `user` and `group` to the same component
4. `npx pod-install` MUST run after adding native dependencies
5. Auth Key only in `.env` — never in source

**Failure modes:**
- Blank screen → init/login order broken
- Android build failure → minSdk < 24 or compileSdk < 34
- iOS build failure → deployment target < 13.0 or missing pod install
- Red screen → native module not linked (missing pod install or autolinking failure)
- Gradle error → Groovy syntax in .kts file or vice versa

**Completion criteria:**
- Android: `npx react-native run-android` builds and launches
- iOS: `npx react-native run-ios` builds and launches
- Chat UI renders with messages visible
- No red screens or CometChat errors in Metro console

---

## DECISION LOGIC

```
// Stop early if wrong skill
IF package.json has "expo" in dependencies → STOP, use cometchat-react-native-expo
IF package.json has "next" or "astro" → STOP, use appropriate web skill
IF pubspec.yaml exists → STOP, use cometchat-flutter

// Version detection
IF react-native >= 0.73 → New Architecture may be enabled; check newArchEnabled in gradle.properties
IF react-native < 0.70 → WARN: CometChat RN UI Kit v5 requires RN >= 0.70

// Gradle DSL detection (CRITICAL)
IF android/app/build.gradle exists → Groovy DSL
IF android/app/build.gradle.kts exists → Kotlin DSL
NEVER apply Groovy syntax to .kts files or vice versa

// Auth strategy
IF project has auth token endpoint → use loginWithAuthToken()
IF project is dev/demo → use login(UID) with Auth Key + TODO comment

// Navigation
IF @react-navigation/native in deps → use existing navigation
IF no navigation library → install @react-navigation/native + @react-navigation/native-stack
```

---

## Step 0 — Check Existing State

Before writing anything:
1. Read `package.json` — check for existing CometChat packages, React Native version, navigation libraries
2. Check `android/app/build.gradle(.kts)` — note DSL type, current minSdk, compileSdk
3. Check `ios/Podfile` — note deployment target
4. Check `.env` for existing credentials
5. Grep `src/` or `App.tsx` for `CometChatUIKit.init`
6. Read `App.tsx` — if it has real logic, PATCH only

---

## Step 1 — Confirm Framework

Read `package.json`. Confirm `react-native` is present and `expo` is NOT. If Expo is detected, redirect to `cometchat-react-native-expo`.

Check React Native version — must be >= 0.70.

---

## Step 2 — Collect Credentials (Infer First)

1. Read `.env` — if credentials exist, reuse
2. Grep source for existing `CometChatUIKit.init`
3. Check for auth token endpoint
4. Default UID to `cometchat-uid-1`
5. Only ask for missing values

---

## Step 3 — Install Dependencies

```bash
npm install @cometchat/chat-uikit-react-native @cometchat/chat-sdk-react-native
# Optional — voice/video calling:
npm install @cometchat/calls-sdk-react-native

# Navigation (if not already installed):
npm install @react-navigation/native @react-navigation/native-stack react-native-screens react-native-safe-area-context

# iOS pods — MANDATORY:
npx pod-install
```

---

## Step 4 — Android Configuration

### Gradle DSL Pre-Check

Read the file first. Determine Groovy vs Kotlin DSL.

**IF `android/app/build.gradle` (Groovy):**

PATCH — set minSdk and compileSdk:
```gradle
android {
    compileSdkVersion 34

    defaultConfig {
        minSdkVersion 24
    }
}
```

**IF `android/app/build.gradle.kts` (Kotlin DSL):**

PATCH — set minSdk and compileSdk:
```kotlin
android {
    compileSdk = 34

    defaultConfig {
        minSdk = 24
    }
}
```

### AndroidManifest.xml

PATCH `android/app/src/main/AndroidManifest.xml` — add permissions if not present:
```xml
<uses-permission android:name="android.permission.INTERNET" />
<uses-permission android:name="android.permission.CAMERA" />
<uses-permission android:name="android.permission.RECORD_AUDIO" />
<uses-permission android:name="android.permission.MODIFY_AUDIO_SETTINGS" />
```

---

## Step 5 — iOS Configuration

PATCH `ios/Podfile` — set deployment target if < 13.0:
```ruby
platform :ios, '13.0'
```

PATCH `ios/<ProjectName>/Info.plist` — add usage descriptions if not present:
```xml
<key>NSCameraUsageDescription</key>
<string>This app needs camera access for video calls</string>
<key>NSMicrophoneUsageDescription</key>
<string>This app needs microphone access for voice and video calls</string>
```

Then run:
```bash
npx pod-install
```

---

## Step 6 — Credentials

CREATE `.env` (if not present):
```
COMETCHAT_APP_ID=your_app_id
COMETCHAT_REGION=us
COMETCHAT_AUTH_KEY=your_auth_key
```

Add `.env` to `.gitignore` if not already there.

---

## Step 7 — Init + Login Wrapper

CREATE `src/CometChatInit.tsx`:

```tsx
import React, { useEffect, useState } from 'react';
import { View, Text, ActivityIndicator, StyleSheet } from 'react-native';
import { CometChatUIKit, UIKitSettings } from '@cometchat/chat-uikit-react-native';

const APP_ID = 'APP_ID'; // Replace or read from env
const REGION = 'REGION';
const AUTH_KEY = 'AUTH_KEY';
const UID = 'cometchat-uid-1';

interface Props {
  children: React.ReactNode;
}

export const CometChatInit: React.FC<Props> = ({ children }) => {
  const [ready, setReady] = useState(false);
  const [error, setError] = useState<string | null>(null);

  useEffect(() => {
    let cancelled = false;

    const init = async () => {
      try {
        const settings = new UIKitSettings({
          appId: APP_ID,
          region: REGION,
          authKey: AUTH_KEY,
          subscriptionType: 'allUsers',
        });

        await CometChatUIKit.init(settings);

        const existingUser = await CometChatUIKit.getLoggedInUser();
        if (!existingUser) {
          await CometChatUIKit.login(UID);
        }

        if (!cancelled) setReady(true);
      } catch (e: unknown) {
        if (!cancelled) setError(String(e));
      }
    };

    init();
    return () => { cancelled = true; };
  }, []);

  if (error) {
    return (
      <View style={styles.center}>
        <Text style={styles.error}>{error}</Text>
      </View>
    );
  }

  if (!ready) {
    return (
      <View style={styles.center}>
        <ActivityIndicator size="large" />
      </View>
    );
  }

  return <>{children}</>;
};

const styles = StyleSheet.create({
  center: { flex: 1, justifyContent: 'center', alignItems: 'center' },
  error: { color: 'red', fontFamily: 'monospace', padding: 16 },
});
```

---

## Step 8A — Experience 1: Conversation List + Messages

CREATE `src/cometchat/ChatScreen.tsx`:

```tsx
import React, { useState } from 'react';
import { View, StyleSheet } from 'react-native';
import {
  CometChatConversations,
  CometChatMessages,
} from '@cometchat/chat-uikit-react-native';
import { CometChat } from '@cometchat/chat-sdk-react-native';

export const ChatScreen: React.FC = () => {
  const [selectedUser, setSelectedUser] = useState<CometChat.User | undefined>();
  const [selectedGroup, setSelectedGroup] = useState<CometChat.Group | undefined>();

  return (
    <View style={styles.container}>
      <View style={styles.conversations}>
        <CometChatConversations
          onItemPress={(conversation: CometChat.Conversation) => {
            const entity = conversation.getConversationWith();
            if (entity instanceof CometChat.User) {
              setSelectedUser(entity);
              setSelectedGroup(undefined);
            } else if (entity instanceof CometChat.Group) {
              setSelectedUser(undefined);
              setSelectedGroup(entity);
            }
          }}
        />
      </View>
      <View style={styles.messages}>
        {selectedUser && <CometChatMessages user={selectedUser} />}
        {selectedGroup && <CometChatMessages group={selectedGroup} />}
      </View>
    </View>
  );
};

const styles = StyleSheet.create({
  container: { flex: 1, flexDirection: 'row' },
  conversations: { width: 320 },
  messages: { flex: 1 },
});
```

**PATCH `App.tsx`:**

```tsx
import React from 'react';
import { CometChatInit } from './src/CometChatInit';
import { ChatScreen } from './src/cometchat/ChatScreen';

export default function App() {
  return (
    <CometChatInit>
      <ChatScreen />
    </CometChatInit>
  );
}
```

---

## Step 8B — Experience 2: One-to-One Chat

CREATE `src/cometchat/ChatScreen.tsx`:

```tsx
import React, { useEffect, useState } from 'react';
import { View, Text, ActivityIndicator, StyleSheet } from 'react-native';
import { CometChatMessages } from '@cometchat/chat-uikit-react-native';
import { CometChat } from '@cometchat/chat-sdk-react-native';

const CHAT_UID = 'cometchat-uid-2'; // Target user

export const ChatScreen: React.FC = () => {
  const [user, setUser] = useState<CometChat.User | undefined>();
  const [error, setError] = useState<string | null>(null);

  useEffect(() => {
    CometChat.getUser(CHAT_UID)
      .then(setUser)
      .catch((e: unknown) => setError(String(e)));
  }, []);

  if (error) return <View style={styles.center}><Text style={styles.error}>{error}</Text></View>;
  if (!user) return <View style={styles.center}><ActivityIndicator size="large" /></View>;

  return <CometChatMessages user={user} />;
};

const styles = StyleSheet.create({
  center: { flex: 1, justifyContent: 'center', alignItems: 'center' },
  error: { color: 'red', fontFamily: 'monospace', padding: 16 },
});
```

---

## Step 8C — Experience 3: Tab-Based Chat

Use React Navigation bottom tabs. Install if not present:
```bash
npm install @react-navigation/bottom-tabs
```

CREATE `src/cometchat/ChatScreen.tsx` with tab navigation:

```tsx
import React, { useState } from 'react';
import { View, StyleSheet } from 'react-native';
import { createBottomTabNavigator } from '@react-navigation/bottom-tabs';
import {
  CometChatConversations,
  CometChatUsers,
  CometChatGroups,
  CometChatCallLogs,
  CometChatMessages,
} from '@cometchat/chat-uikit-react-native';
import { CometChat } from '@cometchat/chat-sdk-react-native';

const Tab = createBottomTabNavigator();

export const ChatScreen: React.FC = () => {
  const [selectedUser, setSelectedUser] = useState<CometChat.User | undefined>();
  const [selectedGroup, setSelectedGroup] = useState<CometChat.Group | undefined>();

  const ChatsTab = () => (
    <CometChatConversations
      onItemPress={(conversation: CometChat.Conversation) => {
        const entity = conversation.getConversationWith();
        if (entity instanceof CometChat.User) { setSelectedUser(entity); setSelectedGroup(undefined); }
        else if (entity instanceof CometChat.Group) { setSelectedUser(undefined); setSelectedGroup(entity); }
      }}
    />
  );

  const UsersTab = () => (
    <CometChatUsers
      onItemPress={(user: CometChat.User) => { setSelectedUser(user); setSelectedGroup(undefined); }}
    />
  );

  const GroupsTab = () => (
    <CometChatGroups
      onItemPress={(group: CometChat.Group) => { setSelectedUser(undefined); setSelectedGroup(group); }}
    />
  );

  const CallsTab = () => <CometChatCallLogs />;

  return (
    <View style={styles.container}>
      <View style={styles.sidebar}>
        <Tab.Navigator screenOptions={{ headerShown: false }}>
          <Tab.Screen name="Chats" component={ChatsTab} />
          <Tab.Screen name="Calls" component={CallsTab} />
          <Tab.Screen name="Users" component={UsersTab} />
          <Tab.Screen name="Groups" component={GroupsTab} />
        </Tab.Navigator>
      </View>
      <View style={styles.messages}>
        {selectedUser && <CometChatMessages user={selectedUser} />}
        {selectedGroup && <CometChatMessages group={selectedGroup} />}
      </View>
    </View>
  );
};

const styles = StyleSheet.create({
  container: { flex: 1, flexDirection: 'row' },
  sidebar: { width: 320 },
  messages: { flex: 1 },
});
```

---

## Step 9 — Run

```bash
# Android
npx react-native run-android

# iOS
npx react-native run-ios
```

---

## Agent Verification Checklist

- [ ] Android: `npx react-native run-android` builds and launches
- [ ] iOS: `npx react-native run-ios` builds and launches
- [ ] minSdk >= 24, compileSdk >= 34 (correct DSL syntax used)
- [ ] iOS deployment target >= 13.0
- [ ] `npx pod-install` was run after dependency installation
- [ ] No CometChat component rendered before login() resolves
- [ ] `user` and `group` never both passed to same component
- [ ] Visible error UI on init/login failure
- [ ] Auth Key only in .env (not in committed source)
- [ ] AndroidManifest.xml has INTERNET, CAMERA, RECORD_AUDIO permissions
- [ ] iOS Info.plist has camera/microphone usage descriptions

## ERROR DEBUGGING TABLE

| Symptom | Cause | Fix |
|---|---|---|
| Blank screen | login() before init() resolves | Await init() before calling login() |
| Red screen: native module not found | Missing pod install | Run `npx pod-install` |
| Android build failure: minSdk | minSdk < 24 | Set to 24 in build.gradle(.kts) with correct DSL syntax |
| iOS build failure: deployment target | Podfile target < 13.0 | Set platform :ios, '13.0' and pod install |
| Gradle syntax error | Groovy syntax in .kts or vice versa | Check file extension, use matching DSL |
| setState on unmounted | Async callback after unmount | Use cancelled flag in useEffect cleanup |
| Camera/mic permission denied | Missing Info.plist entries | Add NSCameraUsageDescription, NSMicrophoneUsageDescription |
