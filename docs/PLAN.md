# CometChat Plugin — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build a Claude Code plugin that lets any developer type `/cometchat-init:init` and get a fully integrated CometChat chat experience in their project — no docs reading required.

**Architecture:** A plugin with a single slash command entry point (`/cometchat-init:init`) that delegates to a hierarchy of skills: a router skill (`cometchat`) detects the framework, then delegates to a framework-specific skill (e.g., `cometchat-react-nextjs`) which inherits shared rules from `cometchat-react-core`. Documentation is never embedded — lookup tables point to live URLs fetched on demand.

**Tech Stack:** Claude Code plugin system (markdown skills, slash commands), CometChat React UI Kit v6, CometChat SDKs for React Native / Flutter / Android (future phases)

---

## Current State (What's Done)

The following files are already created and should NOT be modified unless explicitly noted:

### Plugin scaffold
- `.claude-plugin/plugin.json` — manifest
- `commands/init.md` — `/cometchat-init:init` entry point
- `README.md` — usage docs

### Skills (your existing files, copied unchanged)
- `skills/cometchat/SKILL.md` — platform detection + routing dispatcher
- `skills/cometchat-react-core/SKILL.md` — shared hard rules for all React integrations
- `skills/cometchat-react-nextjs/SKILL.md` — Next.js integration
- `skills/cometchat-react-reactjs/SKILL.md` — React.js (Vite/CRA) integration
- `skills/cometchat-react-astro/SKILL.md` — Astro integration

### Reference files (generated from your JSON doc structures)
- `skills/cometchat-react-core/references/react-v6-doc-map.md` — 60+ URL lookup table
- `skills/cometchat-react-core/references/react-native-doc-map.md` — 50+ URL lookup table
- `skills/cometchat-react-core/references/flutter-v5-doc-map.md` — 50+ URL lookup table
- `skills/cometchat-react-core/references/android-v5-doc-map.md` — 40+ URL lookup table
- `skills/cometchat-react-core/references/platform-matrix.md` — cross-platform feature/status matrix

---

## Phase 1: Validate & Fix Current Plugin (React Web)

**Goal:** Ensure the plugin works end-to-end for React web frameworks before expanding.

### Task 1: React Router Skill (Missing)

The `cometchat` dispatcher references `cometchat-react-react-router` but no skill file exists yet.

**Files:**
- Create: `skills/cometchat-react-react-router/SKILL.md`

- [ ] **Step 1: Check if you have a React Router skill file**

Check `C:/Users/Omkar/Downloads/` for a file named `cometchat-react-react-router.md`. If it exists, copy it unchanged. If not, proceed to Step 2.

- [ ] **Step 2: If no file exists, create a placeholder skill**

```markdown
---
name: cometchat-react-react-router
description: Integrate CometChat React UI Kit v6 into a React Router v6/v7 app. Use for Conversation List + Message View, One-to-One/Group Chat, or Tab-Based Chat with React Router. Do not use for plain React, Next.js, or Astro.
---

# CometChat + React Router Integration

**Status:** This skill is under development. For now, refer to the React.js skill (`cometchat-react-reactjs`) as a starting base — React Router integration follows the same patterns with routing additions.

Follow all rules in `cometchat-react-core`. The invariants and decision logic in core override anything in this file.

See the React Router documentation: https://www.cometchat.com/docs/ui-kit/react/react-router
```

- [ ] **Step 3: Commit**

```bash
git add skills/cometchat-react-react-router/SKILL.md
git commit -m "feat: add placeholder React Router skill"
```

---

### Task 2: End-to-End Test — Fresh Vite Project

**Goal:** Verify the plugin works on a real project.

- [ ] **Step 1: Create a fresh Vite + React + TS project**

```bash
cd /tmp
npm create vite@latest cometchat-test -- --template react-ts
cd cometchat-test
npm install
```

- [ ] **Step 2: Install the plugin locally**

```bash
claude --plugin-dir C:/dev/CometChat/cometchat-init
```

- [ ] **Step 3: Run /cometchat-init:init 1**

Verify:
- Framework detected as React.js / Vite
- Credentials requested (no .env exists)
- Files created: `.env`, `src/index.css` patched, `src/main.tsx` patched, `src/App.tsx` replaced, `src/cometchat/CometChatSelector.tsx` created
- `npm run build` exits 0

- [ ] **Step 4: Test Experience 2 and 3 on fresh projects**

Repeat Steps 1-3 with `/cometchat-init:init 2` and `/cometchat-init:init 3`.

- [ ] **Step 5: Document any failures and fix**

If any integration fails, fix the skill file and re-test.

---

### Task 3: End-to-End Test — Existing Next.js Project

- [ ] **Step 1: Use one of your existing projects**

```bash
cd C:/dev/CometChat/uikitbuilder_nextjs_1
```

- [ ] **Step 2: Run /cometchat-init:init 1**

Verify:
- Framework detected as Next.js
- Existing `.env.local` credentials reused (not overwritten)
- Existing `page.tsx` NOT replaced (has real content)
- New `/chat` route created instead
- `npm run build` exits 0

---

## Phase 2: Expand to Remaining Platforms

Each phase below adds one platform. The pattern is identical each time:
1. Write the framework-specific SKILL.md
2. Test on a real project
3. Update the `cometchat` dispatcher if needed (it already references these skill names)

### Task 4: React Native (CLI + Expo)

**Files:**
- Create: `skills/cometchat-react-native/SKILL.md`
- Create: `skills/cometchat-react-native/references/` (if needed)

- [ ] **Step 1: Write the React Native skill**

Follow the same structure as `cometchat-react-reactjs`:
- HARD RULES section
- AGENT CONTRACT
- DECISION LOGIC (detect CLI vs Expo from package.json)
- Step-by-step integration (install, init, login, render)
- Experience code for all 3 experiences
- Verification checklist

Key differences from React web:
- Navigation uses React Navigation, not React Router
- Permissions needed for calls (camera, microphone)
- Expo needs config plugins
- No CSS — uses StyleSheet or CometChatTheme object
- No SSR concerns

Use `react-native-doc-map.md` for documentation references.

- [ ] **Step 2: Update `cometchat` dispatcher**

The dispatcher already has detection logic for React Native (`"react-native"` in package.json). Add the skill delegation entry:

```
| React Native (CLI)  | cometchat-react-native     |
| React Native (Expo) | cometchat-react-native     |
```

- [ ] **Step 3: Test on a real RN project**

```bash
cd C:/dev/CometChat/rn_uikit
```

- [ ] **Step 4: Commit**

---

### Task 5: Flutter

**Files:**
- Create: `skills/cometchat-flutter/SKILL.md`

- [ ] **Step 1: Write the Flutter skill**

Key differences:
- Detection: `pubspec.yaml` exists
- Package manager: `flutter pub add`
- No CSS — uses CometChatTheme and CometChatColorPalette
- Widgets instead of components
- Platform permissions in AndroidManifest.xml and Info.plist

Use `flutter-v5-doc-map.md` for documentation references.

- [ ] **Step 2: Update dispatcher**
- [ ] **Step 3: Test**
- [ ] **Step 4: Commit**

---

### Task 6: Android

**Files:**
- Create: `skills/cometchat-android/SKILL.md`

- [ ] **Step 1: Write the Android skill**

Key differences:
- Detection: `build.gradle.kts` or `app/build.gradle`
- Package manager: Gradle dependencies
- XML layouts or Jetpack Compose
- AndroidManifest.xml permissions
- View + ViewModel + Adapter architecture

Use `android-v5-doc-map.md` for documentation references.

- [ ] **Step 2: Update dispatcher**
- [ ] **Step 3: Test**
- [ ] **Step 4: Commit**

---

## Phase 3: Advanced Features

### Task 7: Auth Token Integration Skill

**Files:**
- Create: `skills/cometchat-auth/SKILL.md`
- Create: `skills/cometchat-auth/references/backend-patterns.md`

- [ ] **Step 1: Write auth skill**

This skill covers:
- Modifying existing login/signup backend to call CometChat REST API
- Generating auth tokens server-side
- Patterns for Express, Next.js API routes, Flask, Django
- Token refresh and session management

- [ ] **Step 2: Test with a Next.js project that has NextAuth**

---

### Task 8: Theming & Customization Skill

**Files:**
- Create: `skills/cometchat-customize/SKILL.md`

- [ ] **Step 1: Write customization skill**

Covers:
- CSS variable overrides (React web)
- CometChatTheme object (React Native, Flutter)
- XML theming (Android)
- Dark mode setup
- Message bubble customization
- Component-level style overrides

Auto-triggers when user says "customize chat", "change chat colors", "dark mode for chat", etc.

---

### Task 9: Calling Integration Skill

**Files:**
- Create: `skills/cometchat-calling/SKILL.md`

- [ ] **Step 1: Write calling skill**

Covers:
- Installing `@cometchat/calls-sdk-javascript` (web) or platform equivalents
- Dashboard configuration for calling
- IncomingCall + OutgoingCall component setup
- CallButtons integration
- Permissions (camera, microphone) per platform

---

## Phase 4: Distribution

### Task 10: Package and Publish

- [ ] **Step 1: Add .gitignore**

```
node_modules/
.env
*.local.md
```

- [ ] **Step 2: Initialize git repo**

```bash
cd C:/dev/CometChat/cometchat-init
git init
git add .
git commit -m "feat: initial cometchat-init plugin"
```

- [ ] **Step 3: Test installation from path**

```bash
claude plugin add C:/dev/CometChat/cometchat-init
```

Verify `/cometchat-init:init` appears in `/help` output.

- [ ] **Step 4: Publish to GitHub**

Push to a public repo. Users install via:
```bash
claude plugin add github:your-org/cometchat-init
```

---

## Architecture Diagram

```
User: /cometchat-init:init [experience]
        │
        ▼
commands/init.md ──► skills/cometchat/SKILL.md
                           │
                           ├─ reads package.json / pubspec.yaml / build.gradle
                           ├─ detects: Next.js
                           ▼
                    skills/cometchat-react-nextjs/SKILL.md
                           │
                           ├─ inherits: skills/cometchat-react-core/SKILL.md
                           ├─ references: references/react-v6-doc-map.md (on demand)
                           ├─ references: references/platform-matrix.md (on demand)
                           │
                           ▼
                    Executes integration steps:
                    .env.local → globals.css → page.tsx → CometChatNoSSR.tsx
```

## Priority Order

1. **Now:** Validate current React web plugin works (Tasks 1-3)
2. **Next:** React Native (Task 4) — you already have test projects
3. **Then:** Flutter (Task 5) — you have flutter_uikit
4. **Later:** Android (Task 6), Auth (Task 7), Customization (Task 8), Calling (Task 9)
5. **Finally:** Distribution (Task 10)
