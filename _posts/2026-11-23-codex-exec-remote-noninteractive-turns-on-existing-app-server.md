---
title: "codex exec --remote: Running Non-Interactive Turns on an Existing App Server"
date: 2026-11-23T08:00:00+00:00
last_modified_at: 2026-11-23T11:04:52Z
tags:
  - codex-cli
  - v0.171.0
  - exec
  - remote
  - app-server
  - non-interactive
  - ci-cd
  - thread-resume
  - authentication
rating: 4.0
copy_edited: true
---

`codex exec` has always started a fresh agent in the process that ran it. From v0.171.0 it can instead be a thin client: point it at an app server that is already running, and the turn runs there, using that server's configuration, credentials and filesystem. This article reads the implementing change, [openai/codex#61707](https://github.com/openai/codex/pull/61707) ("Support remote app servers in `codex exec`", merged 16 November 2026), to pin down what the flag does, what it refuses to do, and where earlier coverage in this knowledge base was imprecise.

All behaviour below comes from that pull request's description and diff unless another source is named. The v0.171.0 release tag is [`rust-v0.171.0`](https://github.com/openai/codex/releases/tag/rust-v0.171.0), published 20 November 2026.

---

## What the Flag Does

The PR description states the contract: `codex exec --remote <ADDR>` runs "a noninteractive turn using an existing app server's configuration and credentials", can start a new thread or resume one by UUID, prints human-readable or JSON output, and supports optional bearer-token authentication through `--remote-auth-token-env`.

Two new options are added to the exec command line (`codex-rs/exec/src/cli.rs` in the diff):

| Option | Help text in source | Notes |
|---|---|---|
| `--remote <ADDR>` | "Connect to an existing app server." | Declared `global`, so it is accepted on `exec` and `exec resume` |
| `--remote-auth-token-env <ENV_VAR>` | "Environment variable containing the remote WebSocket bearer token." | The token is read from the named environment variable, never passed on the command line |

The PR also lets you put the remote options before `exec`. The dispatcher copies the root-level `--remote` and `--remote-auth-token-env` onto the exec command, and fails with "`codex exec` received conflicting remote server endpoints" if the two positions name different endpoints. The test for this uses two `wss://` addresses — the only address format the source documents.

---

## Server Configuration, Not Local Configuration

The key design choice is in the first lines of `codex-rs/exec/src/remote.rs`: the remote path **rejects** nearly every flag that would configure the agent locally. The guard covers `--trust`, `--ignore-user-config`, `--ignore-rules`, `--strict-config`, `--ephemeral`, `--output-schema`, `--image`, `--model`, `--oss`, `--sandbox`, `--cd`, `--add-dir`, `--worktree`, `-c` config overrides, profiles, auto-review and `--dangerously-bypass-approvals-and-sandbox`. The error says: "`--remote` currently supports only prompt, resume by UUID, and output options; configure the agent on the server".

Practical consequences:

- Model, sandbox mode, working directory and credentials are whatever the **server** was started with. The client cannot widen or narrow them.
- A command that works locally (`codex exec --model … --sandbox …`) will fail, not silently ignore the flags, when `--remote` is added.
- Approval policy is fixed by the client: both the thread start and the turn start request `AskForApproval::Never`. A remote exec turn never pauses for a human.

What is still accepted: the prompt, `exec resume`, `--json`, `--last-message-file`, and colour options.

---

## Starting and Resuming Threads

Without a subcommand, the client connects, sends a thread-start request, then a turn-start request with the prompt, and streams events until the turn completes. It does **not** attach to a session that is already running.

This corrects an earlier article here ([v0.171.0 Stable](2026-11-20-codex-cli-v0171-stable-gpt62-sol-default-tui-workspace-remote-exec-mcp-statusline.md)), which described `--remote` as attaching to a running session. In the source, a plain `exec --remote` creates a new thread on the server. To continue earlier work you use resume:

```bash
codex exec --remote wss://agent-host.example:443 --remote-auth-token-env CODEX_REMOTE_TOKEN "run the test suite and summarise failures"
codex exec resume --remote wss://agent-host.example:443 --remote-auth-token-env CODEX_REMOTE_TOKEN <THREAD_UUID> "now fix the first failure"
```

(The address and variable names are illustrative; only the flag shapes and the `wss://` scheme come from the source.)

Resume has stricter rules than local resume:

- It requires a thread **UUID**. `--last`, `--all` and `--image` are rejected ("remote resume requires a thread UUID"), as is any non-UUID id.
- The thread must be idle. If the server reports it as active, the command stops with "remote thread already has an active turn". You cannot run two turns on one thread at once.

The PR's integration test confirms the shared-state behaviour: after one new-thread exec and one resume against the same server, a `ThreadRead` shows the thread holding two turns.

---

## Output and Thread IDs

Because a remote run creates or reuses a server-side thread, the new `print_thread_started` hook reports its id:

- **Human output:** a `thread id: <uuid>` line on **stderr**, so stdout stays clean for the agent's answer.
- **`--json`:** a `thread.started` event with the `thread_id` as the first event, followed by the usual events, ending in `turn.completed`.

Capture that id if you intend to resume later. A short sketch of the pattern, with the token variable set by your CI secret store:

```bash
tid=$(codex exec --remote "$ADDR" --remote-auth-token-env AGENT_TOKEN --json "inspect the failing build" \
  | head -1 | jq -r .thread_id)
```

Note that `head -1` closes the pipe after the first event; for a real job, write the full stream to a file and read the first line from it instead.

---

## Failure Modes

- **Server disconnects mid-turn.** The change adds an explicit error, "app-server disconnected before turn completion", for the remote case only. Locally, a closed event stream simply ends the loop. In CI this means a dropped connection fails the job rather than looking like success.
- **Ctrl-C.** The client installs a handler and forwards an interrupt for the running turn.
- **No local isolation.** Since the turn executes on the server, whatever it changes (files, git state) lands in the server's working directory. The client's `--cd` and `--worktree` are rejected, so isolation has to be arranged when the server is started.

---

## When to Use It

Reasonable fits, inferred from the constraints above rather than stated in the PR:

1. **A shared build or dev host** that already holds the repository, toolchain and credentials; CI jobs send prompts to it instead of installing credentials on every runner.
2. **Resumable multi-step jobs**: step one starts a thread and records the UUID, step two resumes it with a follow-up prompt.
3. **Scripted use of a long-lived server** where `--json` output feeds other tooling.

Poor fits: anything needing per-run model, sandbox or directory choices, structured output via `--output-schema`, or an approval step.

---

## What the Sources Do Not Say

- The PR does not document the accepted address formats beyond the `wss://host:port` form in its tests, nor how to start the server side; see the existing app-server articles in this repository for that.
- It does not say how the bearer token is validated; only that it is read from the named environment variable and sent as a WebSocket bearer token.
- The `/rc` and `/remote` TUI commands mentioned in the v0.171.0 release coverage were a separate change (#62121) and are not part of this one.

---

## Sources

- [openai/codex#61707: Support remote app servers in `codex exec`](https://github.com/openai/codex/pull/61707) (description, diff and tests; merged 2026-11-16)
- [Codex v0.171.0 release](https://github.com/openai/codex/releases/tag/rust-v0.171.0) (2026-11-20)
