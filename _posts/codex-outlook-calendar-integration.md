---
title: "Codex and Outlook Calendar: Five Ways to Connect Your Agent to Microsoft 365 Scheduling"
description: "ChatGPT has native Outlook connectors baked into the product. As of April 16, 2026, the Codex Desktop App does too."
date: 2026-04-28T00:00:00+00:00
last_modified_at: 2026-06-18T02:12:52+01:00
layout: article
tags:
  - codex-cli
  - outlook
  - calendar
  - mcp
  - microsoft-graph
  - microsoft-365
  - work-iq
  - composio
  - enterprise
  - scheduling
  - integration
word_count_target: 1500
---

![Sketchnote diagram for: Codex and Outlook Calendar: Five Ways to Connect Your Agent to Microsoft 365 Scheduling](/sketchnotes/articles/codex-outlook-calendar-integration.png)

# Codex and Outlook Calendar: Five Ways to Connect Your Agent to Microsoft 365 Scheduling

ChatGPT has native Outlook connectors baked into the product. As of April 16, 2026, the Codex Desktop App does too — the **Microsoft 365 Suite plugin** in the 90+ plugin marketplace includes Outlook email and calendar alongside Excel, Word, PowerPoint, Teams, and SharePoint. Authentication uses standard OAuth flows, and no custom API setup is required. Install via the `/plugins` command in the Codex App and search for "Microsoft Suite."

The Codex Desktop App's Microsoft Suite plugin supports email compose and send, calendar event listing and creation, and email draft management. For many users, this is the simplest path — it works out of the box with the same OAuth consent flow as ChatGPT's native connectors.

For Codex **CLI** users, however, the Desktop App's plugin marketplace is not available. The CLI is a terminal-first tool with its own integration model: the Model Context Protocol. MCP gives the CLI the same integration surface as the Desktop App's plugins, and in some cases more control. Here are four CLI approaches, ordered from lightest setup to most enterprise-grade.

## Option 1: Composio — Managed MCP with OAuth Out of the Box

[Composio](https://composio.dev/toolkits/outlook/framework/codex) is the fastest path. It is a managed MCP server platform that handles OAuth token management, refresh cycles, and scope configuration automatically across 870+ app integrations including Outlook.

**Setup:**

```bash
# Add the Composio MCP server to Codex
codex mcp add composio

# Authenticate (opens browser for OAuth consent)
codex mcp login composio

# Verify
codex mcp list
```

For the Codex App (desktop), navigate to Settings, MCP Servers, Add Server, select Streamable HTTP, and provide your Composio API key in the header: `{ "x-consumer-api-key": "ck_*******" }`.

**Calendar capabilities:** Create events, retrieve schedules, check free/busy status, manage invitations, create recurring meetings. Approximately 40 Outlook tools are exposed, covering email, calendar, contacts, and folder management.

**When to choose Composio:** You want calendar access working in under five minutes. You are comfortable with a third-party service handling your OAuth tokens. You do not need to customise the tool definitions.

**Limitation:** Composio is a hosted service. Your Microsoft Graph API calls route through their infrastructure. Enterprise security teams may object to this.

## Option 2: outlook-mcp — Self-Hosted Node.js MCP Server

[XenoXilus/outlook-mcp](https://github.com/XenoXilus/outlook-mcp) is an open-source Node.js MCP server that connects directly to the Microsoft Graph API. It covers email, calendar, SharePoint, and OneDrive, with automatic document text extraction for PDFs, Word, PowerPoint, and Excel files.

**Setup:**

Register an Azure AD application:
1. Go to Azure Portal, App Registrations, New Registration
2. Set redirect URI to `http://localhost/callback`
3. Enable "Allow public client flows" under Authentication
4. Grant delegated permissions: `Calendars.Read`, `Calendars.ReadWrite`, `Mail.Read`, `Mail.ReadWrite`, `Mail.Send`, `User.Read`, `Files.Read.All`, `Sites.Read.All`, `offline_access`

Configure in `.codex/mcp.json` or `~/.config/mcp/servers.json`:

```json
{
  "outlook-mcp": {
    "command": "node",
    "args": ["/path/to/outlook-mcp/server/index.js"],
    "env": {
      "AZURE_CLIENT_ID": "your-application-id",
      "AZURE_TENANT_ID": "your-directory-id"
    }
  }
}
```

**Calendar capabilities:** View upcoming appointments, create events, retrieve daily schedules. The server uses OAuth 2.0 with PKCE (no client secret required), stores tokens encrypted via the OS keychain, and handles automatic refresh.

**When to choose this:** You want full control over the server, your tokens stay on your machine, and you need SharePoint and OneDrive access alongside calendar. MIT-licensed.

**Limitation:** Calendar tooling is functional but less comprehensive than the Microsoft or Composio options. No availability checking or meeting-time suggestions.

## Option 3: microsoft-mcp — Python MCP Server with 40+ Tools

[elyxlz/microsoft-mcp](https://github.com/elyxlz/microsoft-mcp) is a Python-based MCP server implementing over 40 Microsoft Graph tools across email, calendar, OneDrive, and contacts. It includes multi-account support -- useful if you manage both a personal and work Microsoft account.

**Setup:**

```bash
git clone https://github.com/elyxlz/microsoft-mcp.git
cd microsoft-mcp
uv sync

# Authenticate
export MICROSOFT_MCP_CLIENT_ID="your-azure-app-id"
python authenticate.py
```

Configure for Codex:

```json
{
  "microsoft-mcp": {
    "command": "uv",
    "args": ["run", "--directory", "/path/to/microsoft-mcp", "python", "-m", "microsoft_mcp"],
    "env": {
      "MICROSOFT_MCP_CLIENT_ID": "your-azure-app-id"
    }
  }
}
```

**Calendar capabilities:**
- `list_events` — retrieve events with time range filtering
- `create_event` — full event creation with attendees, recurrence, online meeting links
- `update_event` — modify existing events
- `delete_event` — remove events
- `respond_event` — accept or decline invitations
- `check_availability` — free/busy lookup
- `search_events` — query events by subject, attendee, or date

**When to choose this:** You want the most comprehensive open-source calendar toolset. Multi-account support is a differentiator for consultants or contractors who straddle organisations. Python ecosystem if your team prefers that over Node.js.

**Limitation:** Token cache stored in a JSON file (`~/.microsoft_mcp_token_cache.json`) rather than the OS keychain. Less secure than Option 2's approach for sensitive environments.

## Option 4: Microsoft Work IQ — Official Enterprise MCP (Preview)

Microsoft's own [Work IQ Calendar MCP Server](https://learn.microsoft.com/en-us/microsoft-agent-365/mcp-server-reference/calendar) is the enterprise-grade option. It is part of the Microsoft Agent 365 platform, currently in preview, and requires a Microsoft 365 Copilot licence.

**Server ID:** `mcp_CalendarTools`

**Available tools (12):**

| Tool | Purpose |
|------|---------|
| `graph_createEvent` | Create events with recurrence, online meetings, attendee types |
| `graph_listEvents` | List events with OData filtering and ordering |
| `graph_listCalendarView` | Time-range view with occurrence expansion |
| `graph_getEvent` | Retrieve single event with OData $select/$expand |
| `graph_updateEvent` | Modify any event property including recurrence |
| `graph_deleteEvent` | Remove events |
| `graph_acceptEvent` | Accept invitation with optional comment |
| `graph_declineEvent` | Decline invitation with optional comment |
| `graph_cancelEvent` | Cancel and notify attendees |
| `graph_findMeetingTimes` | Suggest meeting times based on attendee availability |
| `graph_getSchedule` | Free/busy lookup for users, distribution lists, or rooms |

**When to choose this:** Your organisation already has Microsoft 365 Copilot licences. You need the deepest calendar integration — meeting-time suggestions, room resource booking, distribution list scheduling, recurring event patterns, and full attendee lifecycle management. Pre-certified by Microsoft, so enterprise security teams accept it without custom review.

**Limitation:** Requires a Copilot licence (currently the most expensive Microsoft 365 add-on). Preview status means the API surface may change. Not yet available as a standalone MCP server you can self-host.

## Comparison Matrix

| Capability | Composio | outlook-mcp | microsoft-mcp | Work IQ |
|---|---|---|---|---|
| Setup time | 2 min | 15 min | 10 min | Org-dependent |
| Create/read/update/delete events | Yes | Partial | Yes | Yes |
| Free/busy availability | Yes | No | Yes | Yes |
| Meeting-time suggestions | No | No | No | Yes |
| Recurring events | Yes | Limited | Yes | Yes |
| Online meeting links (Teams) | Unknown | No | Unknown | Yes |
| Room/resource booking | No | No | No | Yes |
| Multi-account | No | No | Yes | Via delegation |
| Self-hosted | No | Yes | Yes | No (Microsoft cloud) |
| Licence requirement | Composio plan | Free (MIT) | Free (MIT) | M365 Copilot |
| Token storage | Composio cloud | OS keychain | JSON file | Microsoft Identity |

## Practical Example: Blocking Focus Time from Codex CLI

Once any of these MCP servers is configured, you can interact with your calendar conversationally:

```
> codex "Check my calendar for tomorrow afternoon and block out a 2-hour
  focus time slot in the first available gap. Title it 'Deep Work —
  Agent Harness Refactoring'. No attendees."
```

The agent will:
1. Call `listEvents` or `listCalendarView` for tomorrow's date range
2. Identify gaps between existing events
3. Call `createEvent` with the first available 2-hour window
4. Confirm the booking

For teams using Symphony (OpenAI's new orchestration spec), calendar integration enables agents to check engineer availability before assigning Linear tickets, preventing work allocation during meetings or leave.

## Which Option Should You Choose?

**Solo developer, quick start:** Composio. Two commands and you are running.

**Team that owns its infrastructure:** outlook-mcp or microsoft-mcp. Self-hosted, MIT-licensed, tokens stay on your machines. Choose Node.js or Python based on your stack.

**Enterprise with Copilot licences:** Work IQ. Pre-certified, deepest feature set, no custom Azure AD setup required.

**The Desktop App closes the gap natively** via the Microsoft Suite plugin — install it and Outlook calendar works out of the box, just like ChatGPT's native connector. For CLI users, MCP makes this a configuration problem, not a capability problem. The same pattern applies to Google Calendar (via google-calendar-mcp), Apple Calendar (via apple-calendar-mcp), and any other scheduling system with a Graph or REST API. The agent does not need native connectors. It needs a protocol.

**Desktop App users:** Install the Microsoft Suite plugin via `/plugins` — that is Option 0 and the simplest path.

**CLI users:** Choose from the four MCP options above based on your control and licensing requirements.

## Citations

0. OpenAI, 'Codex Desktop v26.415 — Codex for (almost) everything,' April 16, 2026. 90+ plugin marketplace including Microsoft 365 Suite (Outlook, Excel, Word, PowerPoint, Teams, SharePoint). OAuth authentication, no custom API setup required. macOS only at launch. https://openai.com/codex/
1. XenoXilus, 'outlook-mcp,' GitHub, 2026. MCP server for Microsoft Office 365 Outlook — email, calendar, and SharePoint integration via Microsoft Graph API. MIT licence. https://github.com/XenoXilus/outlook-mcp
2. elyxlz, 'microsoft-mcp,' GitHub, 2026. Python MCP server for Microsoft Graph API with 40+ tools covering Outlook, Calendar, OneDrive, and Contacts. Multi-account support. MIT licence. https://github.com/elyxlz/microsoft-mcp
3. Composio, 'Outlook MCP Integration for Codex,' composio.dev, 2026. Managed MCP platform handling OAuth, token refresh, and 870+ app integrations including Outlook calendar. https://composio.dev/toolkits/outlook/framework/codex
4. Microsoft, 'Work IQ Calendar reference (preview),' Microsoft Learn, March 2026. Official Microsoft MCP server specification for calendar operations via Microsoft Graph. Requires Microsoft 365 Copilot licence. 12 tools covering full event lifecycle, availability, and meeting-time suggestions. https://learn.microsoft.com/en-us/microsoft-agent-365/mcp-server-reference/calendar
5. Microsoft, 'Work IQ MCP overview (preview),' Microsoft Learn, 2026. Overview of pre-certified MCP servers for Microsoft 365 services. https://learn.microsoft.com/en-us/microsoft-agent-365/tooling-servers-overview
