---
name: cometchat-react-native-expo
description: Integrate CometChat React Native UI Kit into an Expo project (managed or bare workflow). Handles Expo config plugins, EAS Build requirements, permissions, and CometChat initialization. Use for Conversation List + Messages, One-to-One/Group Chat, or Tab-Based Chat in Expo. Do not use for bare React Native CLI, React web, or Flutter.
---

## HARD RULES

```
- Do not duplicate dependencies — check package.json before adding
- Do not overwrite files with real business logic — PATCH only
- Infer credentials from existing .env or CometChatUIKit.init before asking
- Never introduce Auth Key if the project already has an auth token endpoint
- init() → login() → render: never break this order
- Auth Key must never appear in committed source — use expo-constants or .env
- CometChat uses native modules — Expo Go WILL NOT WORK. Must use EAS Build or development build.
- NEVER tell user to test with Expo Go — it will crash with native module errors
- Android: minSdk >= 24 via expo config plugin or app.json
- iOS: deployment target >= 13.0 via expo config plugin or app.json
- Always run npx expo prebuild after adding config plugins
- Never call setState on unmounted component
```

---

## AGENT CONTRACT

**Goal:** Integrate CometChat React Native UI Kit into an Expo project — experience chosen by user.

**Required inputs:**
- `APP_ID` (string)
- `REGION` ("us" | "eu" | "in")
- `AUTH_KEY` (string, dev only) OR existing auth token endpoint
- Login UID (string) — default: `cometchat-uid-1`
- Experience: "conversation-list" | "one-to-one" | "tab"

**Output files:**
- `.env` — credentials
- `app.json` or `app.config.js` — Expo config (android.minSdkVersion, ios.deploymentTarget, plugins)
- `src/CometChatInit.tsx` — init + login wrapper (CREATE)
- `src/cometchat/ChatScreen.tsx` — experience UI (CREATE)
- `App.tsx` — patched to include CometChatInit wrapper

**Invariants:**
1. `CometChatUIKit.init()` MUST resolve before `login()`
2. `login()` MUST resolve before any CometChat component renders
3. Never pass both `user` and `group` to the same component
4. Auth Key only in `.env`
5. Must use EAS Build or development build — Expo Go is incompatible

**Failure modes:**
- Crash in Expo Go → CometChat native modules not available in Expo Go
- Blank screen → init/login order broken
- Build failure → missing `npx expo prebuild` after config plugin changes
- Android build failure → minSdkVersion < 24
- iOS build failure → deployment target < 13.0

**Completion criteria:**
- `eas build --platform android --profile development` succeeds
- `eas build --platform ios --profile development` succeeds
- Chat UI renders with messages visible
- No CometChat errors in development build console

---

## DECISION LOGIC

```
// Stop early if wrong skill
IF package.json does NOT have "expo" → STOP, use cometchat-react-native-cli
IF package.json has "next" or "astro" → STOP, use web skill
IF pubspec.yaml exists → STOP, use cometchat-flutter

// Expo workflow detection
IF ios/ and android/ directories exist → bare workflow (npx expo prebuild already run)
IF ios/ and android/ do NOT exist → managed workflow (npx expo prebuild needed)

// Expo SDK version
IF expo SDK >= 50 → use Expo config plugins for native config
IF expo SDK < 50 → WARN: may need manual native config after prebuild

// Expo Router detection
IF "expo-router" in deps → use file-based routing (app/ directory)
IF @react-navigation/native in deps → use React Navigation
IF neither → install @react-navigation/native

// Auth strategy
IF project has auth token endpoint → use loginWithAuthToken()
IF project is dev/demo → use login(UID) with Auth Key + TODO

// Credential source
IF .env exists with credentials → reuse
IF CometChatUIKit.init exists in source → extract
```

---

## Step 0 — Check Existing State

Before writing anything:
1. Read `package.json` — check for existing CometChat packages, Expo SDK version, navigation libraries
2. Check if `ios/` and `android/` directories exist (bare vs managed workflow)
3. Read `app.json` or `app.config.js` — current android/ios config
4. Check `.env` for existing credentials
5. Read `App.tsx` — if it has real logic, PATCH only

**CRITICAL: Warn the user immediately:**
> CometChat uses native modules and is NOT compatible with Expo Go. You must use a development build (`npx expo run:android` / `npx expo run:ios`) or EAS Build.

---

## Step 1 — Confirm Framework

Read `package.json`. Confirm `expo` is present. Check Expo SDK version.

---

## Step 2 — Collect Credentials (Infer First)

Same as React Native CLI — check .env, grep source, default UID.

---

## Step 3 — Install Dependencies

```bash
npx expo install @cometchat/chat-uikit-react-native @cometchat/chat-sdk-react-native
# Optional — voice/video:
npx expo install @cometchat/calls-sdk-react-native

# Navigation (if not already installed):
npx expo install @react-navigation/native @react-navigation/native-stack react-native-screens react-native-safe-area-context
```

---

## Step 4 — Configure app.json

PATCH `app.json` (or `app.config.js`) — add/update these fields:

```json
{
  "expo": {
    "android": {
      "minSdkVersion": 24,
      "permissions": [
        "INTERNET",
        "CAMERA",
        "RECORD_AUDIO",
        "MODIFY_AUDIO_SETTINGS"
      ]
    },
    "ios": {
      "deploymentTarget": "13.0",
      "infoPlist": {
        "NSCameraUsageDescription": "This app needs camera access for video calls",
        "NSMicrophoneUsageDescription": "This app needs microphone access for voice and video calls"
      }
    },
    "plugins": []
  }
}
```

---

## Step 5 — Prebuild (if managed workflow)

If `ios/` and `android/` directories do NOT exist:
```bash
npx expo prebuild
```

This generates the native projects with the config from `app.json`.

If they already exist (bare workflow), skip this step — the config plugins will apply on next build.

---

## Step 6 — Credentials

CREATE `.env`:
```
COMETCHAT_APP_ID=your_app_id
COMETCHAT_REGION=us
COMETCHAT_AUTH_KEY=your_auth_key
```

Add `.env` to `.gitignore`.

---

## Step 7 — Init + Login Wrapper

CREATE `src/CometChatInit.tsx` — identical to React Native CLI version (see cometchat-react-native-cli skill Step 7).

---

## Step 8A/8B/8C — Experiences

The experience components are identical to the React Native CLI skill (Steps 8A, 8B, 8C). The only difference is file organization may use Expo Router conventions if `expo-router` is installed:

**If using Expo Router (file-based routing):**
- CREATE `app/chat.tsx` as the chat route
- Import `CometChatInit` as a wrapper in `app/_layout.tsx`

**If using React Navigation:**
- Same pattern as React Native CLI

---

## Step 9 — Run

```bash
# Development build (NOT Expo Go):
npx expo run:android
npx expo run:ios

# OR with EAS:
eas build --profile development --platform android
eas build --profile development --platform ios
```

**NEVER suggest `npx expo start` with Expo Go for CometChat — it will crash.**

---

## Agent Verification Checklist

- [ ] Development build launches on Android
- [ ] Development build launches on iOS
- [ ] `minSdkVersion >= 24` in app.json android config
- [ ] iOS deployment target >= 13.0
- [ ] `npx expo prebuild` was run (managed workflow)
- [ ] No CometChat component rendered before login() resolves
- [ ] `user` and `group` never both passed to same component
- [ ] Visible error UI on init/login failure
- [ ] Auth Key only in .env
- [ ] User was warned that Expo Go is incompatible

## ERROR DEBUGGING TABLE

| Symptom | Cause | Fix |
|---|---|---|
| Crash: "Native module cannot be null" | Using Expo Go instead of dev build | Use `npx expo run:android/ios` or EAS Build |
| Blank screen | init/login order broken | Await init() before login() |
| Build failure: minSdk | minSdkVersion < 24 in app.json | Set android.minSdkVersion to 24 |
| Build failure: iOS target | deploymentTarget < 13.0 | Set ios.deploymentTarget to "13.0" |
| Config not applied | Missing prebuild | Run `npx expo prebuild --clean` |
| Permission denied: camera | Missing Info.plist entries | Add to ios.infoPlist in app.json |
