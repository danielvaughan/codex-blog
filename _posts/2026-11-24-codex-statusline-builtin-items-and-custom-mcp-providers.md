---
title: "The Codex /statusline Command: 33 Built-In Items and the Experimental MCP Provider Contract"
date: 2026-11-24T08:00:00+00:00
last_modified_at: 2026-11-25T19:05:40Z
tags:
  - codex-cli
  - v0.171.0
  - statusline
  - tui
  - mcp
  - configuration
  - skills
  - experimental-features
copy_edited: true
rating: 4.3
depth: 4
originality: 4
accuracy: 5
substack: 4
---

Codex CLI v0.171.0 is described in its release coverage as adding "custom MCP status line authoring". The source is more specific than that, and the earlier summaries in this repository blur two separate things. `/statusline` is first a picker for built-in footer items, which works for everyone. The MCP provider is a second, experimental layer behind a feature flag, and it works differently from how it is usually described: the server returns a UI tree on request, and it does not push updates.

Everything below comes from the `openai/codex` source at commit [`9ba0cea`](https://github.com/openai/codex/commit/9ba0cea) (24 November 2026) unless another source is named. File paths are given relative to the repository root so you can check each claim.

---

## Layer One: The Built-In Picker

The slash command is registered as `SlashCommand::Statusline` with the description "configure which items appear in the status line" (`codex-rs/tui/src/slash_command.rs`). It is marked available during a running task, so you can reconfigure the footer mid-turn.

The picker (`codex-rs/tui/src/bottom_pane/status_line_setup.rs`) is a multi-select list with three behaviours stated in the module documentation: toggle items, reorder them with the left and right arrow keys, and see a live preview of the resulting line. A "Use theme colors" row, described as "Apply colors from the active /theme", sits in the same list.

### Where the choice is stored

Settings live under `[tui]` in `config.toml` (`codex-rs/config/src/types.rs`):

```toml
[tui]
status_line = ["model-with-reasoning", "current-dir", "git-branch", "context-used", "five-hour-limit"]
status_line_use_colors = true
```

- `status_line` is an ordered list of item identifiers. The doc comment says that when it is unset, the TUI defaults to `model-with-reasoning`, `current-dir` and `thread-name`.
- `status_line_use_colors` defaults to `true`; the doc comment says colours derive from the active syntax theme.

### The item catalogue

The `StatusLineItem` enum defines 33 variants. Identifiers are kebab-case, and a few accept aliases (for example `context-usage` is accepted as a legacy spelling of `context-used`). The descriptions in the picker group naturally into six families:

| Family | Items |
|--------|-------|
| Model | `model`, `model-with-reasoning`, `reasoning`, `provider`, `profile`, `fast-mode`, `daybreak` |
| Location and git | `current-dir`, `project-name`, `hostname`, `git-branch`, `pull-request-number`, `branch-changes` |
| Safety posture | `permissions`, `approval-mode` |
| Budget | `context-remaining`, `context-used`, `context-window-size`, `used-tokens`, `total-input-tokens`, `total-output-tokens`, `five-hour-limit`, `weekly-limit` |
| Thread | `thread-name`, `thread-title`, `thread-id`, `run-state`, `task-progress`, `raw-output`, `codex-version` |
| Enterprise | `thread-credits`, `estimated-thread-cost`, `workspace-headline` |

Several descriptions say an item is omitted when its data is missing: git items outside a repository, limit items when the API has not reported them, `thread-id` until a thread exists, and `used-tokens` when the count is zero. The three Enterprise items are described as "Enterprise workspaces only". `task-progress` shows the latest progress from the agent's `update_plan` tool.

Two practical consequences follow from the catalogue. A line with `permissions` and `approval-mode` gives a persistent view of the safety posture, which is otherwise only visible by asking. And `five-hour-limit` and `weekly-limit` ("remaining usage on the primary/secondary usage limit") put rate-limit headroom in view without running a separate command.

---

## Layer Two: The Experimental MCP Provider

The second layer is gated. The feature registry (`codex-rs/features/src/lib.rs`) defines `tui_extensions_statusline` as an experimental feature named "Custom status line", with the menu text "Restart the TUI, then use /statusline.", and it is off by default. Enable it either in `/experimental` or in config:

```toml
[features]
tui_extensions_statusline = true
```

With the flag on, the `/statusline` picker gains two extra rows: "Choose a status line provider" and "Create a custom status line".

### Selecting a provider

The selection is stored as `tui.status_line_provider`. Its type, `StatusLineSelection`, is either the literal `"default"` (built-in items) or a table naming an MCP server and tool:

```toml
[tui]
status_line_provider = { server = "my-statusline", tool = "statusline" }
```

The type's doc comment notes that "an explicit default overrides an MCP provider from a lower configuration layer". Layer rules in `codex-rs/config/src/config_layer_source.rs` add a safety property: the packaged defaults and **project** config layers are not allowed to set a provider, and the setting is stripped from them. User, system, session-flag, enterprise-managed and MDM layers can. In practice, a repository you clone cannot make your terminal footer call one of its MCP servers by shipping a `.codex/config.toml`.

An invalid provider value in a permitted layer is treated as `"default"`, so a typo falls back to the built-in line instead of blanking the footer.

### What a provider must advertise

A provider is an ordinary stdio MCP server with one tool that carries a UI entrypoint in its metadata. `codex-rs/tui/src/statusline_provider/descriptor.rs` shows the rules:

- The tool's `_meta["openai/ui"].entrypoints` array must contain an entry of type `statusline` with a `resourceUri`.
- The URI must start with `ui://` and parse cleanly as a URL.
- If a tool advertises more than one valid statusline entrypoint, Codex logs a warning and ignores the tool entirely.
- The tool's `title` becomes the label in the picker, falling back to the tool name.

### What a provider must return

The bundled sample skill, `create-statusline` (`codex-rs/skills/src/assets/samples/create-statusline/SKILL.md`), documents the runtime protocol:

1. Codex calls `resources/read` on the advertised URI. The current thread's snapshot arrives in `params._meta["com.openai/codex-statusline-state"]`.
2. The server returns exactly one text resource, MIME type `application/json`, containing a JSON UI tree. A plain string renders as literal text; `null` clears the footer.
3. The snapshot type is published as a generated TypeScript file (`references/statusline.d.ts`). Its top level is `{ account, app, config, thread }`. The thread carries model, run state, workspace, token usage, cost, approval and permissions data. The account carries rate-limit snapshots.

The UI tree vocabulary in the same file is small: `text`, `link`, `row`, `bold`, `italic`, `underline`, plus bare strings. `text` takes a colour and weight; `row` takes a `gap`, optional `justify` (`start`, `center`, `end`, `between`), `wrap` and `background`. Colours can be theme names (`primary`, `surface`, `accent`), ANSI names, named status colours such as `status-model` and `status-usage`, or `#RRGGBB`.

### The limits

The skill and the type file together state hard ceilings. Codex accepts at most:

| Limit | Value |
|-------|-------|
| UI tree JSON | 64 KiB |
| Non-null nodes | 256 |
| Tree depth | 16 (root at 0) |
| UTF-8 text across string leaves | 16 KiB |
| Time for each read | 1 second |

After a successful read, Codex waits one second before reading again. An error or a timeout stops polling altogether, and the skill says to reselect the provider in `/statusline` to restart it. If the footer has fallen back, the skill says that F2 shows the failed attempt.

### A minimal provider

The starter (`assets/statusline.py`) is about a hundred lines of newline-delimited JSON-RPC over stdio. Its `render` function is the only part you edit:

```python
def render(state):
    thread = state.get("thread", {})
    model = thread.get("model", {}).get("display_name") or "Codex"
    children = [{"type": "text", "props": {"color": "status-model"}, "children": [model]}]
    if agent_state := thread.get("agent", {}).get("state"):
        children.append({"type": "italic", "children": [agent_state]})
    return {"type": "row", "props": {"gap": 1, "wrap": "wrap"}, "children": children}
```

The skill's enable steps: register the server on the host that runs the app server (`codex mcp add my-statusline -- python3 -u /absolute/path/to/statusline.py`), turn on **Custom status line** in `/experimental`, restart the TUI, start a new conversation in the workspace, then choose the provider in `/statusline`. After editing the Python file, restart its process.

---

## What This Corrects

The v0.171.0 coverage in this repository ([Codex CLI v0.171.0 Stable](/2026/11/20/codex-cli-v0171-stable-gpt62-sol-default-tui-workspace-remote-exec-mcp-statusline/)) says the status line is "authored by the MCP server on each update cycle". The source describes a pull model instead: the TUI reads a resource roughly once a second, with a one-second timeout, and stops on the first failure. Examples such as a build-status or database-connection indicator remain plausible uses, but they are inferences, not documented samples; the only shipped sample displays the model name and agent state.

---

## Choosing Between the Layers

Reasonable guidance, inferred from the constraints above rather than stated in the sources:

- Start with the built-in items. 33 are available, and they cover model, git, budget and safety posture with no code and no flag.
- Write a provider only for data Codex does not already track, or for layout that the picker cannot express.
- Keep a provider's `resources/read` cheap. With a one-second budget and polling that ends on any error, a slow network call in the render path will silently kill the footer.
- Remember that the footer is per-thread input. The snapshot is for "the displayed thread", so a provider should be a pure function of the snapshot plus any fast local state.

---

## What the Sources Do Not Say

- No documentation on developers.openai.com was consulted for this feature; the behaviour above is read from source and the bundled sample skill, so details may change while the flag is experimental.
- The release notes do not say when the provider will leave experimental status.
- How `task-progress`, `daybreak` and the Enterprise items are populated beyond their picker descriptions was not traced.

---

## Sources

- [openai/codex source at `9ba0cea`](https://github.com/openai/codex/commit/9ba0cea): `codex-rs/tui/src/bottom_pane/status_line_setup.rs`, `codex-rs/tui/src/slash_command.rs`, `codex-rs/config/src/types.rs`, `codex-rs/config/src/config_layer_source.rs`, `codex-rs/features/src/lib.rs`, `codex-rs/tui/src/statusline_provider/descriptor.rs`
- Bundled sample skill `codex-rs/skills/src/assets/samples/create-statusline/` (`SKILL.md`, `assets/statusline.py`, `references/statusline.d.ts`)
- [Codex v0.171.0 release](https://github.com/openai/codex/releases/tag/rust-v0.171.0) (2026-11-20), items #61979, #61988, #62037 as cited in this repository's v0.171.0 coverage
