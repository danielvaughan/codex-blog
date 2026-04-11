---
title: "Axios Supply Chain Attack Reaches OpenAI macOS Signing Pipeline — What Codex CLI Users Need to Know"
date: 2026-04-12T07:00:00+00:00
tags:
  - security
  - supply-chain
  - axios-attack
  - macos
  - certificate-rotation
  - codex-cli
  - breaking-change
---

# Axios Supply Chain Attack Reaches OpenAI macOS Signing Pipeline


The March 31 Axios npm supply chain attack — already covered in our [source map incident article](2026-04-09-source-map-incident-supply-chain-security-codex-cli.md) — has a direct impact on Codex CLI users that became public on April 10-11, 2026. OpenAI's macOS app-signing pipeline downloaded the malicious `axios@1.14.1` during a GitHub Actions workflow, exposing signing certificates and notarization materials.

## What Happened

On March 31, 2026, a compromised version of the Axios npm library (`v1.14.1`) was published to npm by a North Korean threat actor. A GitHub Actions workflow used in OpenAI's macOS application signing process pulled this malicious dependency. The affected apps include:

- **ChatGPT Desktop** (macOS)
- **Codex App** (macOS)
- **Codex CLI** (macOS installer)
- **Atlas** (macOS)

The malicious package had access to the CI environment where Apple code-signing certificates and notarization credentials were stored. OpenAI stated they found **no evidence of user data access or code tampering**, but as a precaution rotated all affected certificates.

## What You Need to Do

**Update all OpenAI macOS apps immediately.** OpenAI has issued new builds signed with rotated certificates. Older versions signed with the compromised certificates:

- Will stop receiving updates
- **May cease functioning after May 8, 2026** as Apple revokes the old certificates

If you installed Codex CLI via the macOS installer (`.pkg` or Homebrew cask), update now:

```bash
# If installed via Homebrew
brew upgrade codex-cli

# If installed via npm (not affected — npm distribution doesn't use macOS signing)
npm update -g @openai/codex

# If installed via cargo (not affected)
cargo install codex-cli
```

**npm and cargo installations are NOT affected** — the signing pipeline compromise only impacts macOS native app distribution.

## Why This Matters for Agentic Workflows

1. **CI/CD pipelines using `codex exec`** on macOS runners should verify their Codex CLI version is post-rotation. Pin to v0.120.0+ to be safe.
2. **Enterprise deployments** with managed macOS fleets need to push the update before May 8 to avoid disruption.
3. **The attack vector** — a compromised transitive dependency in a CI pipeline — is exactly the kind of supply chain risk that Codex plugins and skills are exposed to. See our [hardening checklist](2026-04-09-source-map-incident-supply-chain-security-codex-cli.md) for mitigation strategies.

## Timeline

| Date | Event |
|------|-------|
| March 31, 2026 00:21 UTC | Malicious `axios@1.14.1` published to npm |
| March 31, 2026 ~01:00 UTC | OpenAI CI pipeline downloads compromised package |
| March 31, 2026 03:29 UTC | Malicious version detected and pulled (~3hr window) |
| April 10, 2026 | OpenAI discloses impact to macOS signing pipeline |
| April 11, 2026 | Security advisory published; certificate rotation complete |
| May 8, 2026 | Old certificates revoked — pre-rotation builds stop working |

## Sources

- [Socket.dev — Axios supply chain attack reaches OpenAI macOS signing pipeline](https://socket.dev/blog/axios-supply-chain-attack-reaches-openai-macos-signing-pipeline-forces-certificate-rotation)
- [Cybernews — OpenAI warns Mac users to update apps](https://cybernews.com/news/openai-warns-mac-users-to-update-apps-after-third-party-security-issue/)
- [startupnews.fyi — OpenAI update advisory](https://startupnews.fyi/2026/04/11/openai-says-to-update-mac-apps-including-chatgpt-and-codex-as-security-precaution/)

*Added 2026-04-12 via hourly research.*
