# CometChat Platform & Skill Matrix

## UI Kit Skills — Current Status

| Platform | Skill Name | Status | Doc Map |
|----------|-----------|--------|---------|
| React.js (Vite / CRA) | `cometchat-react-reactjs` | Available | react-v6-doc-map.md |
| Next.js (App Router) | `cometchat-react-nextjs` | Available | react-v6-doc-map.md |
| React Router v6/v7 | `cometchat-react-react-router` | **Not yet created** | react-v6-doc-map.md |
| Astro | `cometchat-react-astro` | Available | react-v6-doc-map.md |
| React Native (CLI) | — | Coming soon | react-native-doc-map.md |
| React Native (Expo) | — | Coming soon | react-native-doc-map.md |
| Flutter | — | Coming soon | flutter-v5-doc-map.md |
| Android (Kotlin/Java) | — | Coming soon | android-v5-doc-map.md |
| iOS (Swift) | — | Coming soon | — |
| Angular | — | Coming soon | — |
| Vue | — | Coming soon | — |

## Shared Foundation

All React-based UI Kit skills (React.js, Next.js, React Router, Astro) depend on:
- `cometchat-react-core` — shared hard rules, init/login/render order, auth strategy, file operation rules, TypeScript shapes, error debugging table

## UI Kit Versions

| Platform | Current Version | Previous Version |
|----------|----------------|------------------|
| React (Web) | v6 | v5 |
| React Native | v5 | v4 |
| Flutter | v5 | v4 |
| Android | v5 | v4 |
| iOS | v5 | v4 |

## CometChat Product Matrix

| Product | Description | Doc Base URL |
|---------|-------------|--------------|
| UI Kits | Prebuilt UI components per platform | /docs/ui-kit/{platform} |
| SDKs | Backend-connected client SDKs (full control, no UI) | /docs/sdk/{platform} |
| Chat Widgets | Drop-in embeddable chat widgets | /docs/widget |
| UIKit Builder | No-code visual builder for chat UI | /docs/ui-kit-builder |
| AI Agents | CometChat AI agent configuration | /docs/ai |
| REST API | Server-side API for admin/backend actions | /docs/rest-api |

## Feature Availability Across Platforms

| Feature | React Web | React Native | Flutter | Android | iOS |
|---------|-----------|-------------|---------|---------|-----|
| Text messaging | Yes | Yes | Yes | Yes | Yes |
| Media messages | Yes | Yes | Yes | Yes | Yes |
| Typing indicators | Dashboard | Dashboard | Dashboard | Dashboard | Dashboard |
| Read receipts | Dashboard | Dashboard | Dashboard | Dashboard | Dashboard |
| Reactions | SDK code | SDK code | SDK code | SDK code | SDK code |
| Voice/video calls | Calls SDK | Calls SDK | Calls SDK | Calls SDK | Calls SDK |
| Threaded messages | Yes | Yes | Yes | Yes | Yes |
| AI smart replies | Dashboard | Dashboard | Dashboard | Dashboard | Dashboard |
