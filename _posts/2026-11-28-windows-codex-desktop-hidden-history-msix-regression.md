---
title: "Windows Codex Desktop Is Hiding Your Conversation History — MSIX Rollback Fixes It"
date: 2026-11-28T13:00:00+00:00
last_modified_at: 2026-11-28T13:07:40Z
tags:
  - codex-cli
  - windows
  - bug
  - msix
  - desktop
  - session-management
  - workaround
  - regression
nav_order: 2134
---

# Windows Codex Desktop Is Hiding Your Conversation History — MSIX Rollback Fixes It

If you updated Windows Codex Desktop after 20 November and your conversation history appears empty, your data is not gone. A regression in the current MSIX package is hiding messages from that date onward. Rolling back to the last working release restores everything.

---

## What Is Happening

GitHub issue [#64435](https://github.com/openai/codex/issues/64435) (opened 28 November 2026) documents the problem precisely. After installing the current Windows x64 Codex Desktop release, all conversation messages since 20 November disappear from the visible history. The projects list remains intact; existing sessions are still accessible by name. Opening them shows an empty conversation pane.

The last working MSIX package is **26.1111.9664.0**, which was available until 20 November. The first affected version is **26.1117.3838.0**. The same user who reported the issue confirmed that:

1. Installing 26.1111.9664.0 immediately restores all hidden messages.
2. Allowing the app to auto-update to the current release causes the messages to disappear again.
3. Repeating the cycle is fully reproducible.

That reproducibility is important. It means the data is present — the app is simply failing to load or display it under the new version. This is a history-rendering regression, not data loss.

---

## Why It Probably Happened

Windows Codex Desktop ships as an MSIX package. MSIX packages use virtualized file system paths, and conversation history is indexed separately from the raw message store. The regression almost certainly lives in one of two places:

**Option A — History index migration.** The update shipped a schema change to the local message index without a migration path for data written by older versions. Newer app code looks for index entries that do not exist in pre-upgrade records, finds nothing, and renders an empty pane.

**Option B — Virtualized path resolution.** MSIX virtualizes `%AppData%` and similar locations. If the update changed where the app resolves session storage — even by one path component — messages written by the old version would become invisible to the new one, even though they sit untouched on disk.

The rollback working perfectly points to Option A or B being reversible without data manipulation. The messages were never at risk; the index pointer broke.

This is not the first time Windows Codex Desktop has had MSIX-related history regressions. Issue [#59389](https://github.com/openai/codex/issues/59389) documented a similar disappearing-history pattern in October 2026 (thousands of persisted reasoning items causing flickering replies and missing visible history after restart). Issue [#54573](https://github.com/openai/codex/issues/54573) covered a related problem in August 2026 where persisted progress messages disappeared and internal markup leaked. The pattern suggests the Windows Desktop session-persistence layer accumulates technical debt across release cycles.

---

## How to Roll Back

You will need to obtain MSIX package version **26.1111.9664.0** and block automatic updates temporarily.

**Step 1 — Obtain the working MSIX.** If you have access to the Microsoft Store package download history, pull the 26.1111.9664.0 release directly. Alternatively, check the OpenAI Codex Desktop GitHub release page for the archived Windows asset, or ask in the community forum — several users have shared direct download links in the issue thread.

**Step 2 — Uninstall the current release.** Open Windows Settings → Apps → Installed apps, find "OpenAI Codex", and uninstall it. Your session data is stored separately from the application binaries and survives uninstall.

**Step 3 — Install 26.1111.9664.0.** Double-click the `.msixbundle` file and follow the installer prompt.

**Step 4 — Verify history is restored.** Open an existing session that should contain messages from 20–28 November. If the messages appear, the rollback succeeded.

**Step 5 — Pause automatic updates.** In Microsoft Store settings, turn off automatic app updates temporarily, or exclude OpenAI Codex from the update list. This prevents the app from re-applying the broken release.

If you do not have access to the older MSIX, the workaround is to stay on the current version, export any critical conversation content you can still read (via copy-paste), and wait for OpenAI to ship a fix.

---

## Context: The Windows Platform Health Cluster

Issue #64435 is the latest addition to a cluster of Windows-specific Codex Desktop problems that have accumulated since late October 2026. The cluster now includes:

- **History regression** (#64435, 28 November) — messages hidden since Nov 20 update
- **MSIX plugin directory isolation** (#63557, early November) — `codex_app` becomes unreachable when `cmd.exe` cannot see the virtualized plugin path
- **Computer Use mode availability** (#64332, mid-November) — Computer Use not reachable on certain Windows Desktop configurations
- **Work mode message length cap** (#64326, mid-November) — max-length constraint applied inconsistently in Windows Work mode threads
- **MSIX/Cloudpaging conflict** (#64295, mid-November) — Citrix/AppSense Cloudpaging virtualisation layer fights MSIX virtualisation, corrupting session state

The pattern across these issues is consistent: the MSIX packaging layer interacts badly with Windows path virtualisation at multiple points in the session lifecycle — startup, plugin discovery, message routing, and history loading. Each issue manifests differently but the root cause domain is the same.

This cluster has not attracted a formal OpenAI response or a dedicated fix milestone. The issues are tagged individually as bugs but no aggregated Windows Desktop stability report has been posted.

---

## What to Watch

**An OpenAI response on #64435.** The issue is well-documented with a clear reproduction case and version pinpointing. A fix should be straightforward once the regression is located. If the issue remains without an OpenAI comment for 48 hours, it is worth escalating on the developer forum.

**A new MSIX release.** The fix will come as an updated package — watch for version numbers beyond 26.1120.6357.0 (the latest published package as of 28 November). Any new Windows Desktop release should be treated with caution until the community confirms the history is loading correctly.

**More history-loss reports.** If the issue is wider than a single account or configuration, additional reports will appear in the issue thread. A second independent confirmation would signal this affects a substantial portion of Windows users who have recently updated.

---

## Summary

| Item | Detail |
|------|--------|
| Affected platform | Windows x64 Codex Desktop |
| First broken version | 26.1117.3838.0 (released ~20 November 2026) |
| Last working version | 26.1111.9664.0 |
| Symptom | All conversation messages since 20 November hidden from history pane |
| Data status | Not lost — rendering/index regression, not deletion |
| Workaround | MSIX rollback to 26.1111.9664.0 |
| Fix status | No official fix as of 28 November 2026 |
| GitHub issue | [#64435](https://github.com/openai/codex/issues/64435) |

---

*Sources: GitHub issue [#64435](https://github.com/openai/codex/issues/64435) (ncusanoai-cloud, 28 November 2026); GitHub issue [#59389](https://github.com/openai/codex/issues/59389) (October 2026); GitHub issue [#54573](https://github.com/openai/codex/issues/54573) (August 2026); GitHub issue [#63557](https://github.com/openai/codex/issues/63557) (November 2026); KB article triage pass 13, 28 November 2026.*
