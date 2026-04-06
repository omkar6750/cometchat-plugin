# cometchat-init

A Claude Code plugin for integrating CometChat UI Kit into any project. Auto-detects your framework, guides you through setup, and generates a fully functional chat experience.

## Installation

```bash
claude plugin add /path/to/cometchat-init
```

Or for local development:
```bash
claude --plugin-dir /path/to/cometchat-init
```

## Usage

```
/cometchat-init:init          # Interactive — asks for framework and experience
/cometchat-init:init 1        # Conversation List + Message View
/cometchat-init:init 2        # One-to-One Chat
/cometchat-init:init 3        # Tab-Based Chat
```

Or just say "integrate CometChat" or "add chat to my app" — the skill auto-triggers.

## Supported Frameworks

| Framework | Status |
|-----------|--------|
| React.js (Vite / CRA) | Available |
| Next.js (App Router) | Available |
| Astro | Available |
| React Router v6/v7 | Planned |
| React Native | Coming soon |
| Flutter | Coming soon |
| Android | Coming soon |
| iOS | Coming soon |

## What It Does

1. **Detects** your framework from `package.json` / project files
2. **Asks** which chat experience you want (or accepts it as an argument)
3. **Infers** credentials from `.env` files and existing code before asking
4. **Integrates** CometChat with framework-specific patterns (SSR handling, routing, CSS)
5. **Preserves** your existing code — patches minimally, never replaces real logic

## Plugin Structure

```
cometchat-init/
├── .claude-plugin/plugin.json
├── commands/
│   └── init.md                          # /cometchat-init:init entry point
├── skills/
│   ├── cometchat/SKILL.md               # Platform detection + routing
│   ├── cometchat-react-core/
│   │   ├── SKILL.md                     # Shared rules for all React integrations
│   │   └── references/
│   │       ├── react-v6-doc-map.md      # React UI Kit v6 doc URLs
│   │       ├── react-native-doc-map.md  # React Native doc URLs
│   │       ├── flutter-v5-doc-map.md    # Flutter v5 doc URLs
│   │       ├── android-v5-doc-map.md    # Android v5 doc URLs
│   │       └── platform-matrix.md       # Platform support matrix
│   ├── cometchat-react-nextjs/SKILL.md  # Next.js integration
│   ├── cometchat-react-reactjs/SKILL.md # React.js (Vite/CRA) integration
│   └── cometchat-react-astro/SKILL.md   # Astro integration
└── README.md
```

## Prerequisites

- A CometChat account with App ID, Region, and Auth Key
- An existing project with one of the supported frameworks
- Node.js 18+ and npm

## Hard Rules

The plugin enforces these constraints to prevent common integration failures:

- `init()` must resolve before `login()` — always
- `login()` must resolve before any UI component renders — always
- `css-variables.css` imported exactly once per app
- Never pass both `user` and `group` to the same component
- Auth Key stays in `.env` only — never in source files
- Existing auth systems are never replaced — only extended
