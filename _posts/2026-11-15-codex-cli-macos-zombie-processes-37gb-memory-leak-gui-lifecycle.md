---
title: "1,300 Zombies: The macOS Memory Crisis Inside Codex CLI"
date: 2026-11-15T10:00:00+00:00
last_modified_at: 2026-11-25T19:05:40Z
parent: "Articles"
nav_order: 1236
copy_edited: true
tags:
  - codex-cli
  - macos
  - bugs
  - performance
  - memory
  - gui
  - process-lifecycle
  - troubleshooting
rating: 4.2
depth: 4
originality: 4
accuracy: 4
substack: 5
---

![Sketchnote diagram for: 1,300 Zombies: The macOS Memory Crisis Inside Codex CLI](/sketchnotes/articles/2026-11-15-codex-cli-macos-zombie-processes-37gb-memory-leak-gui-lifecycle.png)

You open Codex CLI, work for a few hours, and your Mac slows to a crawl. Activity Monitor reveals something unexpected: 1,300+ child processes in zombie state and host memory consumption at 37 GB. Your machine is not running out of RAM from the task you gave Codex — it is running out of RAM because Codex is not cleaning up after itself.

This is a live, unresolved bug affecting macOS users running extended Codex sessions. Understanding the mechanism is the first step to working around it.

## What a Zombie Process Actually Is

In Unix process semantics, a zombie is a process that has finished executing but has not yet been reaped by its parent. The child exits; the kernel keeps a small record of the exit status in the process table until the parent calls `wait()` or `waitpid()` to collect it. Once the parent collects the exit status, the zombie is removed.

Under normal circumstances, zombies are transient — they appear briefly and are reaped within milliseconds. The problem occurs when the parent process never calls `wait()`. Every unreaped child occupies an entry in the kernel process table indefinitely. At 1,300+ entries, this is not a brief transient state — it is a systematic failure of process lifecycle management in the Codex CLI GUI layer.

## The Root Cause: GUI Process Lifecycle

Codex CLI's GUI spawns child processes for model calls, tool executions, and background operations. The design is correct in principle: offloading work to child processes keeps the GUI responsive. The failure is in teardown.

The GUI process is not calling `wait()` on terminated children. Instead of reaping them, it is allowing them to accumulate in zombie state. On macOS, each zombie holds a process table entry and a small amount of kernel memory. At low counts — a few zombies — the effect is negligible. At 1,300+ zombies, accumulated kernel-side overhead is substantial, and other system resources begin to degrade.

The 37 GB memory figure reported by affected users is not purely zombie overhead — zombies themselves consume very little memory per entry. The figure suggests that some proportion of those child processes are not in true zombie state but are instead leaked: still running, or paused, holding allocated memory that should have been freed. The process count and memory consumption together indicate two related failures: zombie accumulation from missing `wait()` calls, and a subset of processes that are not exiting cleanly at all.

## Connection to Previous macOS Reports

Earlier reports of whole-Mac freezes on 8 GB machines — instances where Codex CLI rendered the entire system temporarily unresponsive — were attributed to memory pressure without a clear mechanism. The zombie process issue provides that mechanism.

On an 8 GB machine, 37 GB of effective memory consumption from a single application is not survivable without aggressive swap activity. macOS will begin compressing and swapping memory aggressively; at extreme pressure, the system scheduler slows to handle the memory management load, producing the apparent freeze. The freeze is not a Codex CLI crash — the application continues running — but the host is unable to keep up with its own memory management overhead.

On machines with 16 GB or more, the same session may produce sluggishness and thermal throttling without a full freeze, which is why the problem has not been universally reported with the same severity.

## Reproducing the Condition

Based on the reports, the zombie accumulation is associated with:

**Extended sessions** — multi-hour use. Short sessions (under 30 minutes) have not been reported as triggers. The accumulation appears proportional to session length and the number of discrete operations performed.

**Agentic task density** — sessions involving many sequential tool calls accumulate zombies faster than conversational sessions. Each tool execution spawns at least one child process; failures to reap stack up over hundreds of calls.

**macOS specifically** — the issue has not been reproduced on Linux. Linux process management and the Codex CLI build may differ in how child process teardown is handled; the macOS GUI layer appears to be the specific site of the failure.

## Impact on Active Sessions

Beyond memory consumption, zombie accumulation has secondary effects on active sessions:

**Process table exhaustion.** macOS imposes a per-user process limit (`kern.maxprocperuid`). The default is 2,666. At 1,300+ zombies, a single Codex CLI session is consuming roughly half the available process table entries. If you are running other applications concurrently, you may hit the limit — at which point macOS will refuse to spawn new processes system-wide until zombies are reaped or the Codex session is terminated.

**Thermal throttling.** The continuous overhead of a large zombie population and the memory pressure it creates — particularly swap activity on machines close to their RAM ceiling — generates sustained CPU load from kernel memory management, raising chassis temperature and triggering thermal throttling on MacBook hardware.

**Degraded Codex performance.** A session already under memory pressure will see slower model call initiation, slower tool execution, and potentially interrupted compaction operations — compounding any existing context management issues.

## Mitigation: Restarting the Session

There is no user-side fix for the underlying bug. The only effective mitigation is session management:

**Restart Codex CLI periodically during long tasks.** Terminating the GUI process forces the kernel to reparent all children to `launchd`, which does call `wait()` and cleans up the zombie backlog. At each session restart, the zombie count resets to zero.

**Monitor with Activity Monitor.** Filter by process name and watch the child process count column. When you observe more than a few hundred child processes, plan a restart at the next natural task boundary.

**Check total memory pressure.** Use `vm_stat` in Terminal or the Memory Pressure graph in Activity Monitor. If memory pressure is consistently in the red during a Codex session, the zombie accumulation is likely a contributing factor even if the total zombie count appears modest.

**Force-quit vs. clean quit.** A clean quit (`Cmd+Q`) should be sufficient to trigger kernel reparenting. If the GUI is unresponsive due to memory pressure, a force-quit via Activity Monitor achieves the same result. Either way, the kernel cleans up after the parent is gone.

## What OpenAI Needs to Fix

The correct fix is straightforward in principle: the GUI process needs to call `wait()` or `waitpid()` for every child it spawns. In practice, this requires auditing every child spawn site in the macOS GUI codebase and ensuring the parent either:

- Calls `wait()` synchronously after the child exits
- Registers a `SIGCHLD` handler that calls `waitpid(-1, WNOHANG)` to reap all available children
- Uses `posix_spawn` with the `POSIX_SPAWN_SETPGROUP` flag in combination with a reaping loop on the process group

For process isolation patterns common in AI coding tools — where child processes handle sandboxed execution — the `SIGCHLD` handler approach is typically the cleanest: a single handler that drains the zombie queue on each signal, without requiring the parent to block on individual child exits.

There is no published fix or ETA as of 15 November 2026.

## Tracking

The primary report is a GitHub issue in the Codex CLI repository. If you are experiencing whole-Mac freezes or memory exhaustion during extended Codex sessions on macOS, leave a comment with your hardware spec (RAM, macOS version, Codex CLI version) and the zombie count at the time of the freeze. The more data points on the issue, the stronger the signal for prioritisation.

---

**Affected:** macOS, extended Codex CLI sessions
**Severity:** High — can render the system unresponsive on 8 GB machines; causes thermal throttling and performance degradation on larger machines
**Status:** Open, no fix confirmed
**Workaround:** Restart Codex CLI every 1–2 hours during long agentic tasks
