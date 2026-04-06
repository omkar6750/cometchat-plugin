---
name: cometchat-react-react-router
description: Integrate CometChat React UI Kit v6 into a React Router v6+ app. Enforces client-only rendering via lazy imports and a mounted state check. Use for Conversation List + Message View, One-to-One/Group Chat, or Tab-Based Chat in React Router. Do not use for plain React, Next.js, or Astro.
---

## HARD RULES

```
- Do not duplicate CSS imports — grep for css-variables.css before adding
- Do not recreate existing routes — read app/routes.ts or App.tsx first, skip if chat route present
- Do not overwrite files with real business logic — PATCH only
- Infer credentials from .env or existing CometChatUIKit.init before asking
- Never introduce Auth Key if /api/auth or /api/cometchat-token exists
- init() → login() → render: never break this order
- v7 detection: @react-router/dev in devDeps OR react-router >= 7 → file-based routing (app/routes.ts)
- v6 detection: react-router-dom in deps AND version < 7 → JSX routing (<BrowserRouter> + <Routes>)
- CSS goes in app/app.css (v7) or src/index.css (v6) — detect which exists, do not hardcode
- Return null (not undefined) from components on loading/empty states
```

---

## AGENT CONTRACT

**Goal:** Integrate CometChat React UI Kit v6 into a React Router app — experience chosen by user.

**Required inputs:**
- `VITE_COMETCHAT_APP_ID` (string)
- `VITE_COMETCHAT_REGION` ("us" | "eu" | "in")
- `VITE_COMETCHAT_AUTH_KEY` (string, dev only) OR existing auth token endpoint
- Login UID (string) — default: `cometchat-uid-1`
- Experience: "conversation-list" | "one-to-one" | "tab"

**Output files (v7 — `@react-router/dev` present or `react-router` >= 7):**
- `.env` — credentials
- `app/app.css` — CSS variables import (top of file)
- `app/routes/CometChat.tsx` — route component with `lazy()` + `mounted` guard
- `app/routes.ts` — route registration (patch only if route not already present)
- `app/cometchat/CometChatNoSSR.tsx` — init + login + experience UI
- `app/cometchat/CometChatNoSSR.css` — layout styles
- `app/cometchat/CometChatSelector.tsx` — (Experience 1 and 3 only)
- `app/cometchat/CometChatTabs.tsx` + `CometChatTabs.css` — (Experience 3 only)
- `public/assets/*.svg` — (Experience 3 only)

**Output files (v6 — `react-router-dom` < 7, no `@react-router/dev`):**
- `.env` — credentials
- `app/app.css` or `src/index.css` — CSS variables import (top of file)
- `src/cometchat/CometChatNoSSR.tsx` — init + login + experience UI (adjust path to match project structure)
- `src/cometchat/CometChatNoSSR.css` — layout styles
- `src/cometchat/CometChatSelector.tsx` — (Experience 1 and 3 only)
- `src/cometchat/CometChatTabs.tsx` + `CometChatTabs.css` — (Experience 3 only)
- `public/assets/*.svg` — (Experience 3 only)
- No `routes.ts` — route is added inline to existing `<Routes>` in `App.tsx` or equivalent

**Invariants:**
1. `init()` MUST resolve before `login()`
2. `login()` MUST resolve before any component renders (`if (!user) return null` guard)
3. `css-variables.css` imported exactly once per app — in `app/app.css` (v7) or `src/index.css` (v6)
4. Never pass `user` AND `group` to the same component instance
5. SSR rule: `lazy()` + `Suspense` + `mounted` state check in route component; `if (typeof window === "undefined") return` inside `CometChatNoSSR` useEffect

**Failure modes:**
- Blank screen at `/chat` → `mounted` guard missing OR `lazy()` not used
- `401 Unauthorized` → wrong App ID or Auth Key
- Broken styles → `css-variables.css` missing or imported twice
- TypeScript error `Object is possibly undefined` → use `init()?.then()` not `init().then()`

**Completion criteria:**
- `npm run build` exits 0 with no TypeScript errors
- `css-variables.css` imported exactly once — in `app/app.css` (v7) or `src/index.css` (v6)
- Chat UI loads at `/chat` and messages are visible
- No CometChat errors in browser console

---

## DECISION LOGIC

```
// Stop early if wrong skill
IF "react-router" is NOT in package.json AND "@react-router/dev" is NOT in package.json → STOP, use cometchat-react-reactjs instead
IF "next" in package.json → STOP, use cometchat-react-nextjs instead
IF "astro" in package.json → STOP, use cometchat-react-astro instead

// React Router version detection
IF "@react-router/dev" in devDependencies OR "react-router" >= "7.0.0"
  → v7 mode: use file-based routing (app/routes.ts), follow Step 7 (v7 path)
IF "react-router-dom" in dependencies AND version < "7.0.0"
  → v6 mode: use JSX routing (<BrowserRouter> + <Routes>), follow Step 7 (v6 path)

// SSR detection
IF "@react-router/dev" is present → SSR is enabled; typeof window === "undefined" guard is REQUIRED inside CometChatNoSSR useEffect
IF pure client-side v6 app → guard is optional but harmless

// Auth strategy
IF server-side auth token endpoint exists (e.g. /api/auth, /api/cometchat-token) → use loginWithAuthToken(), do NOT use Auth Key
IF project is clearly dev/demo with no existing auth → use login(UID) with Auth Key (add TODO comment for production)

// Credential source
IF .env already has VITE_COMETCHAT_APP_ID → reuse, do NOT overwrite
IF CometChatUIKit.init call already exists in source → extract credentials from there

// File operations
IF app/routes.ts exists AND chat route already registered → skip route registration
IF file exists AND contains app logic beyond boilerplate → PATCH only, preserve all existing logic
IF file exists AND is boilerplate stub (placeholder content only) → REPLACE allowed
IF file does not exist → CREATE

// CSS import
IF css-variables.css already imported anywhere in the project → DO NOT add again
IF not imported → add to top of app/app.css (v7) or src/index.css (v6) — detect which exists
```

---

## QUICK INTEGRATION

Fast path for Experience 1 (Conversation List + Message View) on a fresh React Router v7 project.

**1. Install**
```bash
npm install @cometchat/chat-uikit-react
```

**2. `.env`**
```
VITE_COMETCHAT_APP_ID=your_app_id
VITE_COMETCHAT_REGION=us
VITE_COMETCHAT_AUTH_KEY=your_auth_key
```

**3. `app/app.css` — top of file**
```css
@import url("@cometchat/chat-uikit-react/css-variables.css");
html, body { height: 100%; }
```

**4. `app/routes/CometChat.tsx` — route wrapper**
```tsx
import React, { lazy, Suspense, useEffect, useState } from "react";

const CometChatNoSSR = lazy(() => import("../cometchat/CometChatNoSSR"));

export default function CometChatRoute() {
  const [mounted, setMounted] = useState(false);
  useEffect(() => { setMounted(true); }, []);

  return mounted ? (
    <Suspense fallback={<div>Loading...</div>}>
      <CometChatNoSSR />
    </Suspense>
  ) : <div>Loading...</div>;
}
```

**5. `app/routes.ts` — add the chat route**
```ts
import { type RouteConfig, index, route } from "@react-router/dev/routes";
export default [
  index("routes/home.tsx"),
  route("chat", "routes/CometChat.tsx"),
] satisfies RouteConfig;
```

**6. Navigate to `/chat` after `npm run dev`**

For other experiences (One-to-One, Tab-Based), see Steps 8B and 8C below.

---

# CometChat + React Router Integration

Integrate CometChat React UI Kit v6 into a React Router v6+ app. CometChat requires browser APIs — use `lazy()` + a `mounted` state check to prevent SSR execution.

Follow all rules in `cometchat-react-core`. The invariants and decision logic in core override anything in this file.

This skill owns: SSR pattern (`lazy()` + `Suspense` + `mounted` guard + `typeof window` check), env prefix (`VITE_`), canonical CSS location (`app/app.css`), route registration in `app/routes.ts`, file layout, and experience code examples.

**Minimal diff rule:** Prefer integrating into the user's current app structure. Only scaffold a demo structure when the repository is clearly a starter app.

---

## Step 0 — Check Existing State and Hard Rules

**No-duplicate rules — check these before writing anything:**
- Do not add duplicate CSS imports (`css-variables.css` must appear exactly once across all files).
- Do not create a duplicate chat route if one already exists.
- Do not recreate components that already exist — patch them instead.

Before writing any files:
1. Read `package.json` — if `@cometchat/chat-uikit-react` is already in dependencies, skip the install step.
2. Check if `app/cometchat/` already exists — if so, read the existing files and patch only what is missing rather than overwriting. If files already have app logic, patch minimally — do not replace.
3. Check if `.env` or `.env.development` exists — if it already has `VITE_COMETCHAT_APP_ID`, reuse those values. Only ask the user for credentials that are missing.
4. Read `app/routes.ts` — if the chat route is already registered, do not add it again. If routes.ts has existing routes, APPEND the chat route — DO NOT replace the file. See Pattern D in `cometchat-react-core` EXISTING PROJECT PATCH GUIDE.
5. Check if `@cometchat/chat-uikit-react/css-variables.css` is already imported in `app/app.css` — if so, do not add it again.
6. DO NOT modify `app/root.tsx` except to verify it imports `app.css`.

**EXISTING PROJECT STRATEGY:**
- DO NOT modify existing route files (home.tsx, about.tsx, dashboard.tsx, etc.)
- APPEND one line to `app/routes.ts` — never replace or reorder existing routes
- CREATE `app/cometchat/` directory for CometChat components
- CREATE `app/routes/CometChat.tsx` as the chat route component
- DO NOT modify `app/root.tsx`

**Concrete existing-project example (React Router v7 with 3 existing routes):**

Given an existing `app/routes.ts`:
```ts
import { type RouteConfig, index, route } from "@react-router/dev/routes";

export default [
  index("routes/home.tsx"),
  route("about",     "routes/about.tsx"),
  route("dashboard", "routes/dashboard.tsx"),
] satisfies RouteConfig;
```

Patch it by appending ONE line (do not remove or reorder existing routes):
```ts
import { type RouteConfig, index, route } from "@react-router/dev/routes";

export default [
  index("routes/home.tsx"),
  route("about",     "routes/about.tsx"),
  route("dashboard", "routes/dashboard.tsx"),
  route("chat",      "routes/CometChat.tsx"),
] satisfies RouteConfig;
```

Patch `app/app.css` by adding ONE line at the top (preserve all existing styles):
```css
@import url("@cometchat/chat-uikit-react/css-variables.css");
/* ... existing styles below stay untouched ... */
```

Then CREATE these new files (safe — they don't exist yet):
- `app/routes/CometChat.tsx` — route wrapper with `lazy()` + `mounted` guard
- `app/cometchat/CometChatNoSSR.tsx` — init + login + experience UI
- `app/cometchat/CometChatNoSSR.css` — layout styles
- `app/cometchat/CometChatSelector.tsx` — (Experience 1 and 3 only)

Leave `app/root.tsx`, `app/routes/home.tsx`, `app/routes/about.tsx`, `app/routes/dashboard.tsx` completely untouched.

---

## Step 1 — Confirm Framework and Version

Read `package.json`. Confirm `"react-router"` or `"@react-router/dev"` is in dependencies. If not, use the correct framework skill.

Check the React Router major version:
- **React Router v7** (`"react-router": "^7.x"` or `"@react-router/dev"` present): uses file-based routing with `app/routes.ts`. Use `route()` from `@react-router/dev/routes` to register the chat route. This skill targets v7.
- **React Router v6** (`"react-router-dom": "^6.x"`, no `@react-router/dev`): uses `<BrowserRouter>` + `<Routes>` in JSX. Wrap `CometChatNoSSR` in a `<Route>` component instead of registering in `routes.ts`.

Also check for SSR: if `@react-router/dev` is present, the app has SSR enabled — the `typeof window === "undefined"` guard and `lazy()` + `mounted` pattern are required. If it's a pure client-side v6 app (no SSR), the guard is optional but harmless to include.

---

## Step 2 — Collect Credentials (Infer First)

Execute in order before asking the user anything:

1. Read `.env` and `.env.development` — if `VITE_COMETCHAT_APP_ID` exists, reuse it. Do NOT overwrite.
2. Grep source for `CometChatUIKit.init` — extract APP_ID, REGION, AUTH_KEY from existing call.
3. Check for auth token endpoint (`/api/auth`, `/api/cometchat-token`) — if found, plan to use `loginWithAuthToken()`. Do not add Auth Key.
4. Default `LOGIN_UID` to `cometchat-uid-1` unless a different UID is evident from the codebase.

Only ask for values still missing after steps 1–4. If experience was passed as an argument to the dispatcher, skip asking about it.

---

## Step 3 — Install

```bash
npm install @cometchat/chat-uikit-react
```

---

## Step 4 — Write credentials to `.env`

Only create or update `.env` if `VITE_COMETCHAT_APP_ID` is not already present. Add `.env` to `.gitignore` if not already there.

```
VITE_COMETCHAT_APP_ID=your_app_id
VITE_COMETCHAT_REGION=us
VITE_COMETCHAT_AUTH_KEY=your_auth_key
```

> Warning: Auth Key is for development only. In production, omit `VITE_COMETCHAT_AUTH_KEY` and use server-generated Auth Tokens with `loginWithAuthToken()` instead.

---

## Step 5 — Add CSS

Determine the CSS path before writing — do NOT hardcode:
- v7 (file-based routing): `app/app.css` — check it exists; if not, check for `app/root.css`
- v6 (JSX routing): check whether `src/index.css` or `app/app.css` exists; use whichever is the project's root stylesheet

Add at the top of the detected CSS file. Do not add if already present.

```css
@import url("@cometchat/chat-uikit-react/css-variables.css");
html, body { height: 100%; }
```

---

## Step 6 — Create the Route Component

Create `app/routes/CometChat.tsx`. Uses `lazy()` + `mounted` check for client-only rendering:

```tsx
import React, { lazy, Suspense, useEffect, useState } from "react";

const CometChatNoSSR = lazy(() => import("../cometchat/CometChatNoSSR"));

export default function CometChatRoute() {
  const [mounted, setMounted] = useState(false);
  useEffect(() => { setMounted(true); }, []);

  return mounted ? (
    <Suspense fallback={<div>Loading...</div>}>
      <CometChatNoSSR />
    </Suspense>
  ) : <div>Loading...</div>;
}
```

---

## Step 7 — Register the Route

### v7 path (file-based routing)

Applies when `"@react-router/dev"` is in devDependencies or `"react-router"` >= `"7.0.0"`.

Read `app/routes.ts` first. If the chat route already exists, skip this step.

```ts
import { type RouteConfig, index, route } from "@react-router/dev/routes";
export default [
  index("routes/home.tsx"),
  route("chat", "routes/CometChat.tsx"),
] satisfies RouteConfig;
```

Navigate to `/chat` to see the chat UI.

---

### v6 path (JSX routing)

Applies when `"react-router-dom"` is in dependencies and its version is < `"7.0.0"` (no `@react-router/dev`).

No `routes.ts` file is needed. Wrap the lazy component in the existing router. Find the file where `<Routes>` lives (commonly `App.tsx` or `main.tsx`) and add the route there:

```tsx
// In your existing router setup (App.tsx or wherever <Routes> lives):
import { lazy, Suspense, useEffect, useState } from "react";
const CometChatNoSSR = lazy(() => import("./cometchat/CometChatNoSSR"));

// Add this route inside your existing <Routes>:
<Route path="/chat" element={<CometChatRouteWrapper />} />

// Route wrapper component:
function CometChatRouteWrapper() {
  const [mounted, setMounted] = useState(false);
  useEffect(() => { setMounted(true); }, []);
  return mounted ? (
    <Suspense fallback={<div>Loading...</div>}>
      <CometChatNoSSR />
    </Suspense>
  ) : <div>Loading...</div>;
}
```

Navigate to `/chat` to see the chat UI.

---

## Step 8A — Experience 1: Conversation List + Message View

**`app/cometchat/CometChatSelector.tsx`**
```tsx
import { useEffect, useState } from "react";
import { CometChat } from "@cometchat/chat-sdk-javascript";
import { CometChatConversations, CometChatUIKitLoginListener } from "@cometchat/chat-uikit-react";

interface SelectorProps {
  onSelectorItemClicked?: (
    input: CometChat.Conversation | CometChat.User | CometChat.Group,
    type: string
  ) => void;
}

export const CometChatSelector = ({ onSelectorItemClicked = () => {} }: SelectorProps) => {
  const [loggedInUser, setLoggedInUser] = useState<CometChat.User | null>(null);
  const [activeConversation, setActiveConversation] = useState<CometChat.Conversation | undefined>();

  useEffect(() => {
    setLoggedInUser(CometChatUIKitLoginListener.getLoggedInUser());
  }, []);

  return (
    <>
      {loggedInUser && (
        <CometChatConversations
          activeConversation={activeConversation}
          onItemClick={(e) => {
            setActiveConversation(e);
            onSelectorItemClicked(e, "updateSelectedItem");
          }}
        />
      )}
    </>
  );
};
```

**`app/cometchat/CometChatNoSSR.tsx`:**
```tsx
import React, { useEffect, useState } from "react";
import {
  CometChatMessageComposer,
  CometChatMessageHeader,
  CometChatMessageList,
  CometChatUIKit,
  UIKitSettingsBuilder,
} from "@cometchat/chat-uikit-react";
import { CometChat } from "@cometchat/chat-sdk-javascript";
import { CometChatSelector } from "./CometChatSelector";
import "@cometchat/chat-uikit-react/css-variables.css";
import "./CometChatNoSSR.css";

const COMETCHAT_CONSTANTS = {
  APP_ID: import.meta.env.VITE_COMETCHAT_APP_ID as string,
  REGION: import.meta.env.VITE_COMETCHAT_REGION as string,
  AUTH_KEY: import.meta.env.VITE_COMETCHAT_AUTH_KEY as string,
};
const UID = "UID"; // replace with the login UID

const CometChatNoSSR: React.FC = () => {
  const [user, setUser] = useState<CometChat.User | null>(null);
  const [error, setError] = useState<string | null>(null);
  const [selectedUser, setSelectedUser] = useState<CometChat.User | undefined>();
  const [selectedGroup, setSelectedGroup] = useState<CometChat.Group | undefined>();

  useEffect(() => {
    if (typeof window === "undefined") return; // SSR guard

    const settings = new UIKitSettingsBuilder()
      .setAppId(COMETCHAT_CONSTANTS.APP_ID)
      .setRegion(COMETCHAT_CONSTANTS.REGION)
      .setAuthKey(COMETCHAT_CONSTANTS.AUTH_KEY)
      .subscribePresenceForAllUsers()
      .build();

    CometChatUIKit.init(settings)?.then(() => {
      CometChatUIKit.getLoggedinUser().then((u) => {
        if (!u) {
          CometChatUIKit.login(UID)
            .then(setUser)
            .catch((e: unknown) => setError(String(e)));
        } else {
          setUser(u);
        }
      });
    }).catch((e: unknown) => setError(String(e)));
  }, []);

  if (error) return <div style={{ color: "red", padding: 16, fontFamily: "monospace" }}>{error}</div>;
  if (!user) return null;

  return (
    <div className="conversations-with-messages">
      <div className="conversations-wrapper">
        <CometChatSelector
          onSelectorItemClicked={(activeItem) => {
            const item =
              activeItem instanceof CometChat.Conversation
                ? activeItem.getConversationWith()
                : activeItem;
            if (item instanceof CometChat.User) {
              setSelectedUser(item);
              setSelectedGroup(undefined);
            } else if (item instanceof CometChat.Group) {
              setSelectedUser(undefined);
              setSelectedGroup(item);
            }
          }}
        />
      </div>
      {selectedUser || selectedGroup ? (
        <div className="messages-wrapper">
          {selectedUser && <CometChatMessageHeader user={selectedUser} />}
          {selectedGroup && <CometChatMessageHeader group={selectedGroup} />}
          {selectedUser && <CometChatMessageList user={selectedUser} />}
          {selectedGroup && <CometChatMessageList group={selectedGroup} />}
          {selectedUser && <CometChatMessageComposer user={selectedUser} />}
          {selectedGroup && <CometChatMessageComposer group={selectedGroup} />}
        </div>
      ) : (
        <div className="empty-conversation">Select a conversation to start chatting</div>
      )}
    </div>
  );
};
export default CometChatNoSSR;
```

**`app/cometchat/CometChatNoSSR.css`**
```css
.conversations-with-messages { display: flex; height: 100%; width: 100%; }
.conversations-wrapper { height: 100%; width: 480px; overflow: hidden; display: flex; flex-direction: column; }
.conversations-wrapper > .cometchat { overflow: hidden; }
.messages-wrapper { width: calc(100% - 480px); height: 100%; display: flex; flex-direction: column; }
.empty-conversation { height: 100%; width: 100%; display: flex; justify-content: center; align-items: center; background: white; color: var(--cometchat-text-color-secondary, #727272); }
.cometchat .cometchat-message-composer { border-radius: 0; }
```

---

## Step 8B — Experience 2: One-to-One / Group Chat

**`app/cometchat/CometChatNoSSR.tsx`:**
```tsx
import React, { useEffect, useState } from "react";
import {
  CometChatMessageComposer,
  CometChatMessageHeader,
  CometChatMessageList,
  CometChatUIKit,
  UIKitSettingsBuilder,
} from "@cometchat/chat-uikit-react";
import { CometChat } from "@cometchat/chat-sdk-javascript";
import "@cometchat/chat-uikit-react/css-variables.css";
import "./CometChatNoSSR.css";

const COMETCHAT_CONSTANTS = {
  APP_ID: import.meta.env.VITE_COMETCHAT_APP_ID as string,
  REGION: import.meta.env.VITE_COMETCHAT_REGION as string,
  AUTH_KEY: import.meta.env.VITE_COMETCHAT_AUTH_KEY as string,
};
const LOGIN_UID = "UID"; // replace with login UID
const CHAT_UID = "cometchat-uid-1"; // replace with target UID

const CometChatNoSSR: React.FC = () => {
  const [user, setUser] = useState<CometChat.User | null>(null);
  const [error, setError] = useState<string | null>(null);
  const [selectedUser, setSelectedUser] = useState<CometChat.User | undefined>();

  useEffect(() => {
    if (typeof window === "undefined") return; // SSR guard

    const settings = new UIKitSettingsBuilder()
      .setAppId(COMETCHAT_CONSTANTS.APP_ID)
      .setRegion(COMETCHAT_CONSTANTS.REGION)
      .setAuthKey(COMETCHAT_CONSTANTS.AUTH_KEY)
      .subscribePresenceForAllUsers()
      .build();

    CometChatUIKit.init(settings)?.then(() => {
      CometChatUIKit.getLoggedinUser().then((u) => {
        if (!u) {
          CometChatUIKit.login(LOGIN_UID)
            .then(setUser)
            .catch((e: unknown) => setError(String(e)));
        } else {
          setUser(u);
        }
      });
    }).catch((e: unknown) => setError(String(e)));
  }, []);

  useEffect(() => {
    if (user) {
      CometChat.getUser(CHAT_UID)
        .then(setSelectedUser)
        .catch((e: unknown) => setError(String(e)));
    }
  }, [user]);

  if (error) return <div style={{ color: "red", padding: 16, fontFamily: "monospace" }}>{error}</div>;
  if (!user) return null;

  return selectedUser ? (
    <div className="messages-wrapper">
      <CometChatMessageHeader user={selectedUser} />
      <CometChatMessageList user={selectedUser} />
      <CometChatMessageComposer user={selectedUser} />
    </div>
  ) : (
    <div className="empty-conversation">Set CHAT_UID to start chatting</div>
  );
};
export default CometChatNoSSR;
```

**`app/cometchat/CometChatNoSSR.css`** (for one-to-one)
```css
.messages-wrapper { width: 100%; height: 100%; display: flex; flex-direction: column; }
.empty-conversation { height: 100%; width: 100%; display: flex; justify-content: center; align-items: center; background: white; color: var(--cometchat-text-color-secondary, #727272); }
.cometchat .cometchat-message-composer { border-radius: 0; }
```

---

## Step 8C — Experience 3: Tab-Based Chat

Create icon files in `public/assets/`. Reference them as `/assets/chats.svg` (absolute URL, not ES imports).

**`public/assets/chats.svg`**
```svg
<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 24 24"><path d="M20 2H4c-1.1 0-2 .9-2 2v18l4-4h14c1.1 0 2-.9 2-2V4c0-1.1-.9-2-2-2z"/></svg>
```

**`public/assets/calls.svg`**
```svg
<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 24 24"><path d="M6.62 10.79c1.44 2.83 3.76 5.14 6.59 6.59l2.2-2.2c.27-.27.67-.36 1.02-.24 1.12.37 2.33.57 3.57.57.55 0 1 .45 1 1V20c0 .55-.45 1-1 1-9.39 0-17-7.61-17-17 0-.55.45-1 1-1h3.5c.55 0 1 .45 1 1 0 1.25.2 2.45.57 3.57.11.35.03.74-.25 1.02l-2.2 2.2z"/></svg>
```

**`public/assets/users.svg`**
```svg
<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 24 24"><path d="M12 12c2.21 0 4-1.79 4-4s-1.79-4-4-4-4 1.79-4 4 1.79 4 4 4zm0 2c-2.67 0-8 1.34-8 4v2h16v-2c0-2.66-5.33-4-8-4z"/></svg>
```

**`public/assets/groups.svg`**
```svg
<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 24 24"><path d="M16 11c1.66 0 2.99-1.34 2.99-3S17.66 5 16 5c-1.66 0-3 1.34-3 3s1.34 3 3 3zm-8 0c1.66 0 2.99-1.34 2.99-3S9.66 5 8 5C6.34 5 5 6.34 5 8s1.34 3 3 3zm0 2c-2.33 0-7 1.17-7 3.5V19h14v-2.5c0-2.33-4.67-3.5-7-3.5zm8 0c-.29 0-.62.02-.97.05 1.16.84 1.97 1.97 1.97 3.45V19h6v-2.5c0-2.33-4.67-3.5-7-3.5z"/></svg>
```

**`app/cometchat/CometChatTabs.tsx`**
```tsx
import { useState } from "react";
import "./CometChatTabs.css";

const TABS = [
  { name: "CHATS", icon: "/assets/chats.svg" },
  { name: "CALLS", icon: "/assets/calls.svg" },
  { name: "USERS", icon: "/assets/users.svg" },
  { name: "GROUPS", icon: "/assets/groups.svg" },
];

export const CometChatTabs = ({
  onTabClicked = () => {},
  activeTab,
}: {
  onTabClicked?: (t: { name: string }) => void;
  activeTab?: string;
}) => {
  const [hover, setHover] = useState("");
  return (
    <div className="cometchat-tab-component">
      {TABS.map((tab) => {
        const active =
          activeTab === tab.name.toLowerCase() || hover === tab.name.toLowerCase();
        return (
          <div
            key={tab.name}
            className="cometchat-tab-component__tab"
            onClick={() => onTabClicked(tab)}
          >
            <div
              className={`cometchat-tab-component__tab-icon${active ? " cometchat-tab-component__tab-icon-active" : ""}`}
              style={{ WebkitMaskImage: `url("${tab.icon}")`, maskImage: `url("${tab.icon}")` }}
              onMouseEnter={() => setHover(tab.name.toLowerCase())}
              onMouseLeave={() => setHover("")}
            />
            <div
              className={`cometchat-tab-component__tab-text${active ? " cometchat-tab-component__tab-text-active" : ""}`}
              onMouseEnter={() => setHover(tab.name.toLowerCase())}
              onMouseLeave={() => setHover("")}
            >
              {tab.name}
            </div>
          </div>
        );
      })}
    </div>
  );
};
```

**`app/cometchat/CometChatTabs.css`**
```css
.cometchat-tab-component { display: flex; width: 100%; padding: 0 8px; gap: 8px; border-top: 1px solid var(--cometchat-border-color-light, #F5F5F5); background: var(--cometchat-background-color-01, #FFF); }
.cometchat-tab-component__tab { display: flex; padding: 12px 0 10px; flex-direction: column; align-items: center; gap: 4px; flex: 1; min-height: 48px; cursor: pointer; }
.cometchat-tab-component__tab-icon { width: 32px; height: 32px; background: var(--cometchat-icon-color-secondary, #A1A1A1); -webkit-mask-size: contain; -webkit-mask-position: center; -webkit-mask-repeat: no-repeat; mask-size: contain; mask-position: center; mask-repeat: no-repeat; }
.cometchat-tab-component__tab-text { color: var(--cometchat-text-color-secondary, #727272); font: var(--cometchat-font-caption1-medium, 500 12px Roboto); }
.cometchat-tab-component__tab-icon-active { background: var(--cometchat-icon-color-highlight); }
.cometchat-tab-component__tab-text-active { color: var(--cometchat-text-color-highlight); }
```

**`app/cometchat/CometChatSelector.tsx`** (tab-aware version)
```tsx
import { useEffect, useState } from "react";
import { CometChat } from "@cometchat/chat-sdk-javascript";
import type { Call, Conversation, Group, User } from "@cometchat/chat-sdk-javascript";
import {
  CometChatCallLogs,
  CometChatConversations,
  CometChatGroups,
  CometChatUIKitLoginListener,
  CometChatUsers,
} from "@cometchat/chat-uikit-react";
import { CometChatTabs } from "./CometChatTabs";

interface SelectorProps {
  onSelectorItemClicked?: (
    input: User | Group | Conversation | Call,
    type: string
  ) => void;
}

export const CometChatSelector = ({ onSelectorItemClicked = () => {} }: SelectorProps) => {
  const [loggedInUser, setLoggedInUser] = useState<CometChat.User | null>(null);
  const [activeConversation, setActiveConversation] = useState<CometChat.Conversation | undefined>();
  const [activeUser, setActiveUser] = useState<CometChat.User | undefined>();
  const [activeGroup, setActiveGroup] = useState<CometChat.Group | undefined>();
  const [activeCall, setActiveCall] = useState<Call | undefined>();
  const [activeTab, setActiveTab] = useState("chats");

  useEffect(() => {
    setLoggedInUser(CometChatUIKitLoginListener.getLoggedInUser());
  }, []);

  return (
    <>
      {loggedInUser && (
        <>
          {activeTab === "chats" && (
            <CometChatConversations
              activeConversation={activeConversation}
              onItemClick={(e) => {
                setActiveConversation(e);
                onSelectorItemClicked(e, "updateSelectedItem");
              }}
            />
          )}
          {activeTab === "calls" && (
            <CometChatCallLogs
              activeCall={activeCall}
              onItemClick={(e: Call) => {
                setActiveCall(e);
                onSelectorItemClicked(e, "updateSelectedItemCall");
              }}
            />
          )}
          {activeTab === "users" && (
            <CometChatUsers
              activeUser={activeUser}
              onItemClick={(e) => {
                setActiveUser(e);
                onSelectorItemClicked(e, "updateSelectedItemUser");
              }}
            />
          )}
          {activeTab === "groups" && (
            <CometChatGroups
              activeGroup={activeGroup}
              onItemClick={(e) => {
                setActiveGroup(e);
                onSelectorItemClicked(e, "updateSelectedItemGroup");
              }}
            />
          )}
        </>
      )}
      <CometChatTabs
        activeTab={activeTab}
        onTabClicked={(t) => setActiveTab(t.name.toLowerCase())}
      />
    </>
  );
};
```

**`app/cometchat/CometChatNoSSR.tsx`** (tab experience):
```tsx
import React, { useEffect, useState } from "react";
import {
  CometChatMessageComposer,
  CometChatMessageHeader,
  CometChatMessageList,
  CometChatUIKit,
  UIKitSettingsBuilder,
} from "@cometchat/chat-uikit-react";
import { CometChat } from "@cometchat/chat-sdk-javascript";
import { CometChatSelector } from "./CometChatSelector";
import "@cometchat/chat-uikit-react/css-variables.css";
import "./CometChatNoSSR.css";

const COMETCHAT_CONSTANTS = {
  APP_ID: import.meta.env.VITE_COMETCHAT_APP_ID as string,
  REGION: import.meta.env.VITE_COMETCHAT_REGION as string,
  AUTH_KEY: import.meta.env.VITE_COMETCHAT_AUTH_KEY as string,
};
const UID = "UID"; // replace with the login UID

const CometChatNoSSR: React.FC = () => {
  const [user, setUser] = useState<CometChat.User | null>(null);
  const [error, setError] = useState<string | null>(null);
  const [selectedUser, setSelectedUser] = useState<CometChat.User | undefined>();
  const [selectedGroup, setSelectedGroup] = useState<CometChat.Group | undefined>();

  useEffect(() => {
    if (typeof window === "undefined") return; // SSR guard

    const settings = new UIKitSettingsBuilder()
      .setAppId(COMETCHAT_CONSTANTS.APP_ID)
      .setRegion(COMETCHAT_CONSTANTS.REGION)
      .setAuthKey(COMETCHAT_CONSTANTS.AUTH_KEY)
      .subscribePresenceForAllUsers()
      .build();

    CometChatUIKit.init(settings)?.then(() => {
      CometChatUIKit.getLoggedinUser().then((u) => {
        if (!u) {
          CometChatUIKit.login(UID)
            .then(setUser)
            .catch((e: unknown) => setError(String(e)));
        } else {
          setUser(u);
        }
      });
    }).catch((e: unknown) => setError(String(e)));
  }, []);

  if (error) return <div style={{ color: "red", padding: 16, fontFamily: "monospace" }}>{error}</div>;
  if (!user) return null;

  return (
    <div className="conversations-with-messages">
      <div className="conversations-wrapper">
        <CometChatSelector
          onSelectorItemClicked={(activeItem) => {
            const item =
              activeItem instanceof CometChat.Conversation
                ? activeItem.getConversationWith()
                : activeItem;
            if (item instanceof CometChat.User) {
              setSelectedUser(item);
              setSelectedGroup(undefined);
            } else if (item instanceof CometChat.Group) {
              setSelectedUser(undefined);
              setSelectedGroup(item);
            }
          }}
        />
      </div>
      {selectedUser || selectedGroup ? (
        <div className="messages-wrapper">
          {selectedUser && <CometChatMessageHeader user={selectedUser} />}
          {selectedGroup && <CometChatMessageHeader group={selectedGroup} />}
          {selectedUser && <CometChatMessageList user={selectedUser} />}
          {selectedGroup && <CometChatMessageList group={selectedGroup} />}
          {selectedUser && <CometChatMessageComposer user={selectedUser} />}
          {selectedGroup && <CometChatMessageComposer group={selectedGroup} />}
        </div>
      ) : (
        <div className="empty-conversation">Select a conversation to start chatting</div>
      )}
    </div>
  );
};
export default CometChatNoSSR;
```

Use the same `CometChatNoSSR.css` as Experience 1 for the tab experience. Use the same route wrapper from Step 6.

---

## Step 9 — Substitute Credentials

Replace `"APP_ID"`, `"REGION"`, `"AUTH_KEY"`, `"UID"` in all generated files with values from `.env`.

---

## Step 10 — Run

```bash
npm run dev
# Navigate to http://localhost:5173/chat
```

---

## Agent Verification Checklist

Before finishing, verify each item and report pass or fail:

- [ ] `npm run build` exits with no TypeScript errors
- [ ] `css-variables.css` imported exactly once across all files
- [ ] No component renders before `login()` resolves (`if (!user) return null` guard in place, render gate depends on login success not just init)
- [ ] `user` and `group` never both passed to the same component instance (conditional rendering used)
- [ ] Visible error UI reachable on init/login failure (`if (error) return <div ...>` renders the message)
- [ ] No Auth Key in source files (only in `.env`)

### Runtime verification (browser)

**Experience 1 — Conversation List + Message View:**
- [ ] Conversation list renders in left panel with at least one conversation
- [ ] Clicking a conversation shows MessageHeader, MessageList, and MessageComposer on the right
- [ ] Sending a message appears in the list in real time
- [ ] Selecting a group conversation shows group name in header

**Experience 2 — One-to-One Chat:**
- [ ] MessageHeader shows the target user's name and avatar
- [ ] MessageList shows existing message history (or empty state)
- [ ] MessageComposer accepts input and sends on Enter/button click

**Experience 3 — Tab-Based Chat:**
- [ ] Tab bar visible at bottom of left panel with CHATS / CALLS / USERS / GROUPS
- [ ] Each tab click switches the list component
- [ ] Selecting a user from USERS tab opens message view on the right
- [ ] Selecting a group from GROUPS tab opens message view on the right
