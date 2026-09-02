---
title: "Codex Computer Use on Windows: Foreground Desktop Automation with Agent Sandbox Controls"
description: "On 29 May 2026, OpenAI shipped Computer Use for Windows in Codex App v26.527.1 — giving the agent the ability to see, click, and type inside Windows desktop applications. This article covers the technical architecture, the foreground-only execution model, sandbox constraints, permission boundaries, remote-control integration, and the geographic restrictions that exclude Europe at launch."
type: Technical Article
timestamp: 2026-06-04T00:00:00+00:00
resource: "https://danielvaughan.github.io/codex-resources/articles/2026-06-04-codex-computer-use-windows-foreground-desktop-automation-agent-sandbox-permissions"
tags: ["codex", "computer-use", "windows", "desktop-automation", "sandbox", "permissions", "foreground", "remote-control", "agent"]
date: 2026-06-04T09:00:00+00:00
last_modified_at: 2026-09-02T10:21:19+01:00
---
# Codex Computer Use on Windows: Foreground Desktop Automation with Agent Sandbox Controls


Codex has been able to operate macOS desktop applications since its launch — reading the screen, clicking interface elements, and typing into input fields while the user monitors from a mobile device or another machine. On 29 May 2026, the same capability arrived on Windows with Codex App v26.527.1.[^1] The feature lets the agent interact with any visible Windows application: IDE windows, browsers, database tools, admin consoles, and testing frameworks. This article covers the architecture, the constraints, and what it means for engineering workflows.

---

## What Computer Use actually does

Computer Use gives Codex perceptual access to the host machine's display. The agent can:

- **See** — capture screenshots of the active desktop, parse visual elements, and reason about what is on screen
- **Click** — simulate mouse input to interact with buttons, menus, links, form fields, and other UI controls
- **Type** — input text through simulated keyboard events, filling forms, entering commands, or editing content in any application

This is not clipboard-based automation or accessibility-tree parsing (though it may use those as supplementary signals). The core loop is visual: the agent screenshots the display, reasons about what it sees, decides what to click or type, executes the action, and screenshots again to verify the result.[^2]

The capability is useful for tasks that cannot be accomplished through shell commands or file edits alone:

- **Testing desktop applications** — verifying that UI elements render correctly, forms validate properly, and workflows complete end-to-end
- **Browser and simulator workflows** — interacting with web applications in real browsers, stepping through authentication flows, or operating mobile simulators
- **Accessing locked data sources** — reading from desktop applications that expose no API, CLI, or file export (proprietary database GUIs, legacy admin panels, internal tools)
- **Modifying application settings** — changing IDE preferences, configuring tools through graphical interfaces, or adjusting system settings
- **Reproducing GUI-specific bugs** — stepping through the exact click sequence that triggers a visual rendering issue or an interaction defect[^2]

---

## The foreground-only execution model

The most significant constraint on Windows is that Computer Use runs in the **foreground**. The agent takes over the active desktop session for the duration of the task. You cannot use the machine for other work while Codex is operating it.

This differs from the macOS implementation, where Computer Use can run in the background while the user continues working in other applications. On Windows, the agent needs exclusive access to the display because it relies on capturing the visible screen state to reason about what is happening.

The practical implication: remote control is the intended workflow on Windows. You start a Computer Use task, then monitor and steer it from ChatGPT on iOS or Android, or from another machine running the Codex App. The host machine becomes a dedicated execution environment for the duration of the task.

```
┌─────────────────────────────────────────────────────┐
│                  WINDOWS HOST                        │
│                                                      │
│  ┌─────────────┐    ┌──────────────────────────┐    │
│  │  Codex App  │───>│  Desktop Application      │    │
│  │  (agent)    │    │  (foreground, visible)     │    │
│  │             │<───│                            │    │
│  │  see/click/ │    │  Screenshots → reasoning   │    │
│  │  type loop  │    │  → actions → verification  │    │
│  └──────┬──────┘    └──────────────────────────┘    │
│         │                                            │
│         │  Secure relay                              │
│         │                                            │
│  ┌──────▼──────────────────────────────────────┐    │
│  │  Remote control session                      │    │
│  │  (approvals, diffs, screenshots, terminal)   │    │
│  └──────┬──────────────────────────────────────┘    │
│         │                                            │
└─────────┼────────────────────────────────────────────┘
          │
          ▼
┌─────────────────────┐
│  ChatGPT mobile     │
│  (iOS / Android)    │
│  or another Mac     │
│  running Codex      │
└─────────────────────┘
```

*Figure 1. The foreground execution model. The agent operates the Windows desktop while the user monitors remotely via the secure relay.*

---

## Sandbox and permission boundaries

Computer Use can affect application and system state outside the project workspace. A shell command runs in the sandbox; a click on a desktop application does not. OpenAI addresses this with a permission-prompt model:

- **Every Computer Use action requires explicit approval.** The agent presents what it intends to do, and the user reviews and approves before execution proceeds.
- **Tasks should be kept narrow.** The documentation is explicit: "keep tasks narrow and review permission prompts before continuing."[^2] Broad, open-ended instructions ("clean up my desktop") are not the intended use case.
- **Sandbox rules apply to shell commands within the session.** If Codex runs a terminal command as part of a Computer Use workflow (for example, running `npm test` after clicking through a browser form), the standard sandbox constraints still apply — read/write restrictions, network policies, and execution permissions.

The Windows sandbox itself has matured significantly. The `codex sandbox setup --elevated` command, introduced in v0.136.0, provisions a tightened sandbox environment with:

- Native Windows Sandbox support (Hyper-V based isolation)
- Linux sandboxing via WSL2 as a fallback
- Deny-read rules enforced consistently
- Sandboxed commands that clean up reliably after interruptions[^3]

The elevated provisioning path is separate from Computer Use but complements it: you can run Computer Use for GUI interactions while keeping all shell-level execution sandboxed.

---

## Remote control integration

Computer Use on Windows was announced alongside the broader remote-control Windows support.[^1] The two features are designed to work together:

1. **Pair the host with your mobile device** — open "Set up Codex mobile" in the Codex App sidebar, scan the QR code with ChatGPT on iOS or Android
2. **Start a Computer Use task remotely** — describe what you need Codex to do on the Windows machine
3. **Monitor the session** — review screenshots, diffs, terminal output, and approval prompts from your phone
4. **Approve actions** — each permission prompt surfaces on the mobile device for review

The v0.137.0 release enhanced remote control further with app-server v2 RPCs for pairing management and short-lived server tokens (replacing the earlier ChatGPT access token approach) for websocket authentication.[^4] This means:

- **Controller grants are managed programmatically** — `remoteControl/client/list` and `remoteControl/client/revoke` JSON-RPC methods let you audit and revoke device access
- **Tokens expire quickly** — the short-lived server tokens reduce the window of exposure if a token is intercepted
- **No ChatGPT access token on the wire** — the separation between session authentication and relay authentication is cleaner

Windows cannot currently control another machine. It can only serve as a controlled host. If you need to control a Windows machine from another Windows machine, you will need an intermediary (a Mac running Codex, or a mobile device).

---

## What you can automate today

Computer Use on Windows opens several practical workflows for engineering teams:

### Cross-browser testing with visual verification

Ask Codex to open a web application in Edge, Chrome, and Firefox, navigate through a test scenario in each, and report visual differences. The agent sees the rendered output, not the DOM, so it catches rendering issues that automated DOM-based testing misses.

### Legacy application data extraction

Many enterprise environments have internal tools — database admin consoles, ERP interfaces, monitoring dashboards — that expose no API. Computer Use can navigate these interfaces, read data from the screen, and write it to structured files in the workspace.

### IDE automation for complex refactoring

Some refactoring tasks are easier through the IDE's graphical tools than through CLI commands. Rename refactoring across a large Java project, for example, or using Visual Studio's built-in database schema comparison tool. Computer Use lets Codex operate these tools directly.

### Accessibility testing

Point Codex at your application and ask it to navigate using only keyboard input, or to verify that screen-reader landmarks are present and correctly ordered. The visual perception model can identify UI elements that lack proper labelling.

### Screenshot-based bug reproduction

Paste a bug report with a screenshot, and ask Codex to reproduce the exact visual state on the Windows host. The agent navigates to the same page, performs the same actions, and compares what it sees against the reported screenshot.

---

## Geographic restrictions

Computer Use is **not available** in the European Economic Area, the United Kingdom, or Switzerland at launch.[^2] This applies to all platforms — macOS and Windows.

The restriction is regulatory rather than technical. The visual perception model processes screenshots of the user's desktop, which raises data-handling considerations under GDPR and related frameworks. OpenAI has not announced a timeline for European availability.

For teams with developers in restricted regions, Computer Use tasks must be initiated and monitored from an approved geography, with the host machine also located outside the restricted area.

---

## Setup checklist

For teams ready to adopt Computer Use on Windows:

1. **Install the Codex App from the Microsoft Store** — the feature requires the desktop application, not the CLI alone
2. **Update to at least v26.527.1** — the version that introduced Windows Computer Use support
3. **Run `codex sandbox setup --elevated`** — provisions the Windows sandbox for tighter shell-level isolation
4. **Pair a mobile device** — set up remote control via QR code in the Codex App sidebar
5. **Start narrow** — begin with well-defined, single-application tasks (test this form, read this dashboard, click through this wizard) before attempting multi-application workflows
6. **Review every permission prompt** — Computer Use operates outside the project sandbox; treat every action as requiring explicit approval

---

## Limitations and known constraints

- **Foreground-only on Windows** — the host machine is unavailable for other work during Computer Use
- **No Windows-to-Windows control** — a Windows machine cannot control another Windows machine
- **No European availability** — EEA, UK, and Switzerland are excluded
- **Visual-only perception** — the agent reasons from screenshots, not from the accessibility tree or DOM, so it may misread low-contrast text, unusual fonts, or heavily animated interfaces
- **Permission fatigue risk** — frequent approval prompts for long workflows can lead to rubber-stamping; keep tasks short and specific
- **No CLI-only path** — Computer Use requires the Codex App; it is not available through `codex` CLI alone

---

## Summary

Computer Use on Windows gives Codex the ability to see, click, and type inside any visible desktop application — a capability that extends the agent beyond file edits and shell commands into the graphical layer of the operating system. The foreground-only execution model, combined with the permission-prompt security boundary, makes remote control the natural workflow: start the task, then monitor from your phone. The v0.137.0 release strengthened the remote-control security model with short-lived tokens and programmatic device management. Geographic restrictions currently exclude Europe. For teams outside those regions, the feature opens up cross-browser testing, legacy application automation, IDE-driven refactoring, and screenshot-based bug reproduction — tasks that were previously inaccessible to agentic tools.

---

[^1]: OpenAI shipped Computer Use and remote control for Windows in Codex App v26.527.1, announced 29 May 2026. [OpenAI Developer Changelog](https://developers.openai.com/codex/changelog)
[^2]: Computer Use feature documentation. [Codex App Features](https://developers.openai.com/codex/app/features)
[^3]: Windows sandbox setup and `--elevated` provisioning introduced in v0.136.0. [Codex CLI Releases](https://github.com/openai/codex/releases)
[^4]: Remote-control security improvements in v0.137.0: short-lived server tokens and app-server v2 RPCs. [Codex CLI v0.137.0 Release Notes](https://github.com/openai/codex/releases)
