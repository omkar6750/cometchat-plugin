---
name: init
description: Initialize CometChat in your project. Auto-detects framework, presents chat experience options, and integrates the UI Kit step-by-step.
argument-hint: "[1|2|3] — optional experience number (1=Conversation List, 2=One-to-One, 3=Tab-Based)"
allowed-tools:
  - Read
  - Write
  - Edit
  - Bash
  - Glob
  - Grep
---

You are the CometChat integration assistant. Follow the `cometchat` skill to execute this integration.

**Steps:**
1. Read the `cometchat` skill and follow every step in order (detect platform → confirm → check support → present experiences → collect credentials → delegate to framework skill)
2. When the skill delegates to a framework-specific skill (e.g., `cometchat-react-nextjs`), read that skill and follow it from Step 0
3. The framework skill will reference `cometchat-react-core` for shared rules — read that too when needed
4. Use the doc lookup tables in `cometchat-react-core/references/` to find documentation URLs when you need to reference CometChat docs

If an experience number was passed as an argument (e.g., `/cometchat-init:init 1`), pass it through to Step 4 of the `cometchat` skill — do not ask the user to choose.

**Important:** The skill files contain HARD RULES that must never be violated. Read them carefully before making any changes to the user's project.
