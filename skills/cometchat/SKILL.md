---
name: cometchat
description: Integrate CometChat into any app. Auto-detects your platform and framework, then walks you through picking and applying a chat experience. Start here — do not invoke platform-specific skills directly. Trigger with "/cometchat", "integrate cometchat", or "add chat to my app".
---

# CometChat Integration

You are the entry point for all CometChat integrations. Follow every step in order.

> **Trigger phrases:** `/cometchat`, `/cometchat 1`, `/cometchat 2`, `/cometchat 3`, "integrate CometChat", "add CometChat", "add chat to my app"

---

## AUTONOMOUS MODE

If ALL of the following are true, proceed without asking for confirmation:

1. Framework is unambiguously detected (only one framework matches)
2. Credentials already exist in .env / .env.local (APP_ID, REGION, AUTH_KEY present)
3. Experience was passed as an argument (e.g. /cometchat 1 or /cometchat conversation-list)

If ANY of the following require clarification, ask once and proceed:

- Multiple frameworks detected (e.g. Next.js + Astro in a monorepo)
- No .env file and no existing CometChat init in source
- No experience specified and cannot be inferred from existing code

DO NOT ask for confirmation on:

- Framework version detection (read package.json and decide)
- Whether to patch vs create (follow FILE OPERATION RULES from core)
- Whether to skip install (read package.json and decide)
- Whether CSS is already imported (read the file and decide)

---

## Step 1 — Detect Platform

Read the following files (whichever exist) to identify the platform:

| File to read                             | Signal             |
| ---------------------------------------- | ------------------ |
| `package.json`                           | Web / React Native |
| `pubspec.yaml`                           | Flutter            |
| `app/build.gradle` or `build.gradle.kts` | Android            |
| `*.xcodeproj/` (glob) or `Package.swift` | iOS                |

**Detection rules (check in this order):**

1. `package.json` exists → read `dependencies` and `devDependencies`:
    - `"next"` present → **Next.js**
    - `"@react-router/dev"` present → **React Router v7**
    - `"react-router-dom"` present (no `@react-router/dev`) → **React Router v6**
    - `"astro"` present → **Astro**
    - `"react-native"` present → **React Native** _(coming soon)_
    - `"expo"` present → **Expo** _(coming soon)_
    - `"react"` present (none of the above) → **React.js / Vite**
    - None of the above → **Unknown web project**

2. `pubspec.yaml` exists → **Flutter** _(coming soon)_

3. `app/build.gradle` or `build.gradle.kts` exists → **Android** _(coming soon)_

4. `*.xcodeproj` or `Package.swift` exists → **iOS** _(coming soon)_

5. Nothing matched → ask the user: "What platform are you building on?"

---

## Step 2 — Confirm Detection

If detection is unambiguous, state what was detected and continue. Only pause for confirmation if ambiguous.

Example (unambiguous): "Detected **Next.js** (App Router) — continuing."
Example (ambiguous): "Detected both Next.js and Astro in this monorepo. Which should I target?"

If the user corrects you, use their answer.

---

## Step 3 — Check UI Kit Support

**Supported now:**

- React.js / Vite / CRA → `cometchat-react-reactjs`
- Next.js → `cometchat-react-nextjs`
- React Router v6 / v7 → `cometchat-react-react-router`
- Astro → `cometchat-react-astro`

**Coming soon (not yet available):**

- React Native, Expo, Flutter, Android, iOS

If the detected platform is _coming soon_, say:

> "CometChat skills for **[platform]** are not available yet. You can integrate manually using the [CometChat docs](https://www.cometchat.com/docs). Want me to open the docs for your platform?"

Then stop — do not attempt an integration without a skill file.

---

## Step 4 — Present Experience Options

If experience was passed as argument (e.g. `/cometchat 1`, `/cometchat conversation-list`), skip this step and use the specified experience.

Otherwise, ask:

> "Which chat experience would you like to integrate?"

Show all three options with descriptions:

**[1] Conversation List + Messages**
A full chat UI. The left panel shows all conversations (users and groups). Clicking one opens the message thread on the right — header, message list, and composer. Best for apps where users need to browse and switch between multiple conversations.

**[2] One-to-One Chat**
Embeds a direct message window with a single hardcoded user or group. No conversation list — just the message header, list, and composer filling the screen. Best for apps where the chat partner is known in advance (e.g., customer support, matched pairs).

**[3] Tab-Based Chat**
The left panel has a bottom tab bar with four tabs: Chats, Calls, Users, Groups. Each tab shows the corresponding CometChat list component. Clicking a conversation or user opens messages on the right. Best for full-featured messenger-style apps.

Wait for the user to choose [1], [2], or [3].

**Autonomous fallback:** If running in non-interactive / automated mode and no experience was specified:

1. Default to Experience 1 (Conversation List + Messages)
2. DO NOT ask the user — proceed immediately
3. Log the assumption in output: "No experience specified — defaulting to Experience 1 (Conversation List + Messages)"

---

## Step 5 — Collect Credentials

**Inference order — execute all before asking:**

1. Check `.env` / `.env.local` / `.env.development` for `APP_ID`, `REGION`, `AUTH_KEY`
2. Grep source files for `CometChatUIKit.init` — extract credentials from existing call if found
3. Check for server-side auth token endpoint (`/api/auth`, `/api/cometchat-token`) → if found, use `loginWithAuthToken()` and skip Auth Key entirely
4. Default `LOGIN_UID` to `cometchat-uid-1` if not found (pre-created in every CometChat app)

Only ask the user for values still missing after the steps above.

> ⚠ Auth Key is for development only. In production use server-generated Auth Tokens.

---

## Step 6 — Delegate to Platform Skill

Based on the detected platform, follow the corresponding skill **in full**, passing through the chosen experience and credentials:

| Platform              | Skill to follow                |
| --------------------- | ------------------------------ |
| React.js / Vite / CRA | `cometchat-react-reactjs`      |
| Next.js               | `cometchat-react-nextjs`       |
| React Router v6 / v7  | `cometchat-react-react-router` |
| Astro                 | `cometchat-react-astro`        |

Start from **Step 0** of the target skill (idempotency check). Skip steps the target skill says to skip. Apply the experience the user chose in Step 4 (maps to Step 7A / 7B / 7C in the target skill).

Do not ask the user for credentials again — carry them through from Step 5.
