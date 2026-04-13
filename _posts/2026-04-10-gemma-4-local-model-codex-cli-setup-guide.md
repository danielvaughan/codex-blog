---
title: "Running Gemma 4 Locally with the Codex CLI: What Actually Works"
date: 2026-04-10T06:00:00+00:00
featured: true
tags:
  - local-models
  - gemma-4
  - llama-cpp
  - ollama
  - config-toml
  - tool-calling
  - self-hosted
---

![Sketchnote diagram for: Running Gemma 4 Locally with the Codex CLI: What Actually Works](/sketchnotes/articles/2026-04-10-gemma-4-local-model-codex-cli-setup-guide.png)

# Running Gemma 4 Locally with the Codex CLI: What Actually Works

---

Every token you send to a cloud API is a cost you accept, a latency you tolerate, and a privacy decision you make on behalf of your codebase. Google's Gemma 4, released in April 2026, is the first open-weights model family where local tool calling works well enough to drive the Codex CLI harness end to end. This guide documents the two setups that actually work, based on real testing on a 24 GB M4 MacBook Pro and a Dell Pro Max GB10. Everything here was verified hands-on. What was not tested is not included.

**Hardware tested:**

- MacBook Pro M4 Pro, 24 GB unified memory
- Dell Pro Max GB10, NVIDIA Grace Blackwell, ~120 GB usable unified memory

**Software versions tested:**

- Codex CLI v0.118.0 / v0.120.0
- Ollama v0.20.3 (Mac), v0.20.5 (GB10)
- llama.cpp ggml v0.9.11 (Homebrew build)

---

## Why Gemma 4 Matters for Tool Calling

Previous Gemma generations were unusable with Codex CLI. Gemma 3 27B scored 6.6% on the tau2-bench function-calling benchmark. Gemma 4 31B scores 86.4% on the same benchmark[^13]. The 26B-A4B MoE scores only marginally lower. That jump is what makes this entire guide possible.

Gemma 4 introduces six dedicated special tokens for structured function calling[^2] -- `<|tool>`, `<|tool|>`, `<|tool_call>`, `<|tool_call|>`, `<|tool_response>`, `<|tool_response|>`. These are first-class vocabulary tokens embedded during pre-training, not prompt-engineered conventions. The llama-server logs confirmed these tokens are present in the GGUF vocabulary.

Codex CLI provides a fixed set of tools (`apply_patch`, `Bash`, `Read`, `Write`, `Glob`, `Grep`, `WebFetch`)[^3]. The model only needs to emit the correct tool name and arguments. The harness handles execution.

---

## Setup 1: Mac -- llama.cpp + 26B MoE Q4_K_M on 24 GB M4 Pro

This is the only working path for Gemma 4 tool calling on Apple Silicon. Ollama does not work (see "What Broke" below).

### Install llama.cpp

```bash
brew install llama.cpp
```

Version matters. These benchmarks were taken with ggml 0.9.11 (Homebrew build 8680). A [reported 3.3x regression](https://github.com/ggml-org/llama.cpp/issues/21655) between build b8680 and later master builds dropped M4 generation from ~27 tok/s to ~8 tok/s on the base M4. If generation speed is unexpectedly low after an upgrade, check for regressions and consider pinning your version.

### Download the Model

```bash
huggingface-cli download ggml-org/gemma-4-26B-A4B-it-GGUF \
  gemma-4-26B-A4B-it-Q4_K_M.gguf \
  --local-dir ~/models
```

The download is approximately 16 GB. Note the path reported by `huggingface-cli` -- you will need the full snapshot path for the `-m` flag.

### Start the Server

Every flag here is load-bearing. Omitting any of them will cause failures on 24 GB.

```bash
llama-server \
  -m ~/.cache/huggingface/hub/models--ggml-org--gemma-4-26B-A4B-it-GGUF/snapshots/<hash>/gemma-4-26B-A4B-it-Q4_K_M.gguf \
  --port 1234 \
  -ngl 99 \
  -c 32768 \
  -np 1 \
  --jinja \
  -ctk q8_0 \
  -ctv q8_0
```

| Flag | Why It Matters on 24 GB |
|---|---|
| `-m /path/to/file.gguf` | The `-hf` flag auto-downloads a 1.1 GB vision projector (mmproj) that causes OOM. Use `-m` to skip it. |
| `-c 32768` | Codex CLI system prompt + tools = ~27K tokens. 32K is the minimum viable context. |
| `-np 1` | Multiple slots multiply KV cache memory. Single slot is mandatory on 24 GB. |
| `-ctk q8_0 -ctv q8_0` | Reduces KV cache from ~940 MB (f16) to ~499 MB. Makes 32K context fit. |
| `--jinja` | Required for Gemma 4 tool calling. |
| `-ngl 99` | Full Metal offload for maximum speed. |

Metal reports `recommendedMaxWorkingSetSize` of approximately 19 GB on the 24 GB M4 Pro. Model weights (~17 GB) plus KV cache (499 MB with q8_0) plus overhead fits within this ceiling. Attempting `-c 32768` with the default 4 slots or with f16 KV cache causes `kIOGPUCommandBufferCallbackErrorOutOfMemory`.

### Configure Codex CLI

Create or edit `~/.codex/config.toml`:

```toml
[profiles.local]
model = "ggml-org/gemma-4-26B-A4B-it-GGUF:Q4_K_M"
model_provider = "mac_local"
model_context_window = 32768
web_search = "disabled"

[model_providers.mac_local]
name = "MacBook Pro llama.cpp"
base_url = "http://localhost:1234/v1"
wire_api = "responses"
stream_idle_timeout_ms = 1800000
```

Configuration notes:

- **`web_search = "disabled"`**: Mandatory. Codex CLI sends a `web_search_preview` tool type that llama.cpp rejects (it expects all tools to have `"type": "function"`). This is tracked at codex#10635 and llama.cpp#19343.
- **`model_context_window = 32768`**: Must match the `-c` flag. Setting this higher causes silent prompt truncation and broken tool calls.
- **`stream_idle_timeout_ms = 1800000`**: 30 minutes. A single Read tool call plus response takes approximately 1 minute 39 seconds on this hardware.
- **`wire_api = "responses"`**: The only supported wire format. Codex CLI dropped `wire_api = "chat"` as of v0.118.0.
- **Port 1234**: Avoids conflict with Ollama's default 11434.

### Run and Verify

```bash
codex --profile local "Read the file package.json and tell me the project name"
```

If tool calling works, the model will invoke the `Read` tool, return the correct file contents, and answer the question.

### Validated Results

All six tests passed with Gemma 4 26B MoE Q4_K_M via llama.cpp on M4 Pro 24 GB:

| Test | Result |
|---|---|
| Text generation (`Say hello`) | PASS |
| Read tool (`Read README.md`) | PASS |
| Write tool (`Create test.txt`) | PASS |
| Bash tool (`ls -la`) | PASS |
| Multi-step (`fizzbuzz create/run/fix`) | PASS |
| 5-step stress test (create/read/count/cat/delete) | PASS |

---

## Setup 2: GB10 -- Ollama v0.20.5 + 31B Dense on NVIDIA Blackwell

The GB10 has 120 GB usable unified memory, so the larger 31B Dense model runs without constraint. Ollama v0.20.5 on NVIDIA provides working tool calling -- both text generation and agentic tool use are confirmed functional.

The system ships with Ubuntu and the NVIDIA kernel (aarch64), not DGX OS. CUDA 13.0 is pre-installed (at `/usr/local/cuda/bin/nvcc`, not on the default PATH). GPU compute capability is sm_121 (Blackwell). `nvidia-smi` reports "N/A" for FB Memory and BAR1 Memory because the unified memory architecture uses NVLink C2C. To verify GPU memory:

```bash
python3 -c "import torch; print(torch.cuda.get_device_properties(0).total_memory / 1024**3)"
```

### Install Ollama and Pull the Model

```bash
# Install Ollama (detects NVIDIA GPU automatically)
curl -fsSL https://ollama.com/install.sh | sudo sh

# Pull the 31B Dense model (Q4_K_M, ~19 GB)
ollama pull gemma4:31b
```

### Access from Your Mac via SSH Tunnel

Codex CLI's `--oss` mode only checks localhost. Set up a tunnel:

```bash
ssh -f -N -L 11434:localhost:11434 <user>@<gb10-ip>
```

Replace `<user>` and `<gb10-ip>` with your GB10 credentials and IP address. mDNS hostnames like `gb10.local` may not resolve -- find the IP via your router admin or `ip addr` on the device.

### Run Codex CLI

```bash
codex --oss -m gemma4:31b "Read README.md and tell me what this project is about"
```

Both text generation and tool calling are confirmed working. This was the first successful end-to-end agentic tool call with Gemma 4 on the GB10.

### Why 31B Dense on GB10

With 120 GB available, there is no reason to compromise. The 31B Dense outperforms the 26B MoE on benchmarks -- LMArena text score 1452 vs 1441, LiveCodeBench 80.0% vs 77.1%[^13] -- with better reasoning and more reliable tool calling in testing. On a 24 GB Mac, the MoE's memory efficiency justifies the quality trade-off. On the GB10, there is no trade-off.

---

## What Broke

These are the paths that failed during testing. Documenting them here saves you the hours we spent finding out.

### Ollama on Mac (v0.20.3) -- Tool Calling Broken

Two compounding bugs make Ollama unusable with Gemma 4 on Apple Silicon:

1. **Flash Attention freeze.** Gemma 4's hybrid attention (256-dim sliding window + 512-dim global heads) breaks Ollama's FA kernel on Apple Silicon. The system freezes above approximately 500 tokens. The Codex CLI system prompt alone exceeds this threshold. Workaround: `OLLAMA_FLASH_ATTENTION=0 OLLAMA_KV_CACHE_TYPE=q8_0 ollama serve` -- but this only fixes the freeze, not the tool-calling bug.

2. **Streaming bug.** Tool calls are routed to the reasoning field instead of the tool_calls field. The model tries `mcp:filesystem:read_file` instead of using Codex CLI's tools. Both `--oss` mode and manual provider config fail identically. This persists through at least v0.20.3 on Apple Silicon[^8].

**Verdict:** Text generation works with the FA workaround. Tool calling does not work at all. Use llama.cpp.

### vLLM on GB10 -- ABI Incompatibility

vLLM 0.19.0's compiled `_C.abi3.so` is built against the torch 2.10.0 ABI. But torch 2.10.0 for aarch64 is CPU-only. The CUDA-enabled torch 2.11.0+cu128 (required for Blackwell sm_121) has an incompatible ABI. Result: `ImportError: undefined symbol: _ZN3c1013MessageLoggerC1EPKciib` at startup.

Additional complications: Python 3.12 on Ubuntu enforces PEP 668 (must use a venv), the default pip aarch64 torch wheel is CPU-only, and you need to force-install `torch 2.11.0+cu128` from the PyTorch index after installing vLLM. Even after all that, the binary crashes.

**Verdict:** `pip install vllm` does not work on GB10 Blackwell. Docker or building from source might work but were not tested.

### llama.cpp on GB10 -- Builds Fine, Tool Calling Fails with Codex CLI

llama.cpp builds cleanly on GB10 with CUDA. CMake auto-detects Blackwell `121a-real`:

```bash
git clone --depth 1 https://github.com/ggml-org/llama.cpp.git
cd llama.cpp
cmake -B build -DGGML_CUDA=ON -DLLAMA_CURL=ON
cmake --build build --config Release -j$(nproc)
```

Direct inference and benchmarking work perfectly (the llama-bench results in this article come from this build). However, Codex CLI's `wire_api = "responses"` sends non-function tool types (`web_search_preview`, custom `apply_patch` type) that llama.cpp's `/v1/responses` endpoint rejects. Setting `web_search = "disabled"` resolves the web search tool but the remaining incompatibilities mean Ollama with `--oss` is the reliable path for Codex CLI on GB10.

### Flash Attention (Ollama + Apple Silicon)

Gemma 4's hybrid attention architecture combines 256-dim sliding-window heads with 512-dim global heads. Ollama's Flash Attention kernel on Apple Silicon cannot handle this. The symptom is a complete system freeze above approximately 500 tokens of context -- no error message, no crash log, just a frozen machine requiring a hard restart.

The environment variable workaround (`OLLAMA_FLASH_ATTENTION=0`) disables FA and allows text generation to work, but does not fix the separate tool-calling streaming bug. llama.cpp's own FA implementation handles the hybrid architecture correctly.

### `wire_api = "chat"` Removed

Codex CLI v0.118.0 removed support for `wire_api = "chat"`. Attempting to use it produces: `wire_api = "chat" is no longer supported`. Only `wire_api = "responses"` is valid[^5].

---

## Benchmarks (Measured)

All benchmarks were gathered during testing using llama-bench and timed Codex CLI sessions.

### Raw Speed (llama-bench)

| Metric | Mac (26B MoE Q4_K_M) | GB10 (31B Dense Q4_K_M) | GB10 (31B Dense Q8_0) |
|---|---|---|---|
| **Generation (tg128)** | **51.73 tok/s** | 10.18 tok/s | 6.74 tok/s |
| **Prompt processing (pp128)** | 406.94 tok/s | 692.95 tok/s | 534.80 tok/s |
| **Prompt processing (pp512)** | 589.98 tok/s | 673.86 tok/s | 499.19 tok/s |
| **Prompt processing (pp8192)** | 530.54 tok/s | 547.52 tok/s | 425.99 tok/s |

The Mac generates tokens 5.1x faster than the GB10 despite identical 273 GB/s memory bandwidth. The MoE architecture is the reason: the 26B MoE only activates approximately 3.8 billion parameters per token, so the effective per-token memory read is approximately 1.9 GB (at Q4_K_M). The 31B Dense reads all 17.4 GB of Q4_K_M weights per token. Under Codex CLI load, the Mac's prompt eval measured approximately 21 tok/s (versus 407-590 raw) because the ~27K system prompt dominates.

### Code Generation Benchmark (parse_csv_summary Task)

All three were given the same task: create a `parse_csv_summary` function with error handling and tests.

| Metric | Cloud (GPT-5.4) | GB10 (31B Dense) | Mac (26B MoE) |
|---|---|---|---|
| **Wall-clock time** | 1m 05s | 6m 59s | 4m 42s |
| **Tokens used** | 21,268 | 185,091 | 29,501 |
| **Tests passed** | 5/5 first try | 5/5 first try | 4/4 after 5 failed rewrites |
| **Code quality** | 5/5 | 4/5 (clean, single-pass) | 3/5 (dead code left in) |
| **Test quality** | 5/5 | 4/5 (good coverage) | 2/5 (final tests OK, 5 failed attempts) |
| **Tool calls** | ~5 (clean) | 3 (clean) | ~10 (messy retries) |
| **Type hints** | Yes | No | No |

The GB10's 31B Dense produced clean code on the first attempt with 3 tool calls. The Mac's 26B MoE finished faster but needed approximately 10 tool calls with 5 failed test-file rewrites -- the model repeatedly mangled variable names and string literals when writing via shell heredocs. It also left dead code in the implementation.

**Key finding for agentic coding: model quality matters more than raw token speed.** A model that gets it right first time (3 tool calls) beats one that generates tokens faster but needs 10 tool calls with retries.

### GB10 Model Comparison

| Configuration | Generation (tg) | Prompt Processing (pp) | Notes |
|---|---|---|---|
| 31B Dense, Q4_K_M, Ollama | ~10 tok/s | ~548-693 tok/s | Recommended. Ollama default quantisation. |
| 31B Dense, Q8_0, llama-bench | ~6.7 tok/s | ~426-535 tok/s | Near-lossless quality, slower generation. |

Generation is memory-bandwidth limited at 273 GB/s. Q4_K_M (17.4 GB weights) achieves 10.18 tok/s; Q8_0 (30.4 GB weights) achieves 6.74 tok/s -- a 51% improvement for Q4_K_M confirming bandwidth as the bottleneck.

---

## Troubleshooting

Every entry here comes from an issue actually hit during testing.

| Symptom | Cause | Fix |
|---|---|---|
| Ollama freezes / hangs with Gemma 4 on Apple Silicon | Flash Attention bug with Gemma 4's hybrid attention | `OLLAMA_FLASH_ATTENTION=0 OLLAMA_KV_CACHE_TYPE=q8_0 ollama serve` -- but tool calling still broken. Use llama.cpp. |
| Tool calling broken on Ollama + Apple Silicon (model hallucinates `mcp:filesystem:read_file`) | Streaming bug routes tool calls to reasoning field (v0.20.3) | Use llama.cpp instead of Ollama on Apple Silicon |
| `'type' of tool must be 'function'` error from llama.cpp | Codex CLI sends `web_search_preview` tool type | Add `web_search = "disabled"` to profile config |
| OOM with `-hf` flag on 24 GB Mac | `-hf` auto-downloads 1.1 GB vision projector (mmproj) | Use `-m /path/to/file.gguf` instead of `-hf` |
| OOM with `-c 32768` and multiple slots | KV cache for 4 x 32K exceeds Metal working set (~19 GB) | Use `-np 1` (single slot) |
| OOM with `-c 32768 -np 1` and f16 KV cache | Model (17 GB) + f16 KV cache (~940 MB) + overhead exceeds ceiling | Add `-ctk q8_0 -ctv q8_0` to reduce KV cache to ~499 MB |
| `wire_api = "chat" is no longer supported` | Codex CLI v0.118.0+ removed Chat Completions wire format | Use `wire_api = "responses"` |
| vLLM `ImportError: undefined symbol` on GB10 | ABI incompatibility: vLLM pip wheel built against torch 2.10.0, Blackwell needs 2.11.0+cu128 | Use Ollama or llama.cpp instead |
| `--oss` mode cannot reach remote Ollama | `--oss` only checks localhost | SSH tunnel: `ssh -f -N -L 11434:localhost:11434 <user>@<gb10-ip>` |
| `--oss` mode tries to `ollama pull` against llama.cpp | llama.cpp lacks the `/api/pull` endpoint | Use `--local-provider lmstudio` to skip the pull, or use a custom profile with `wire_api = "responses"` |
| `nvidia-smi` shows "N/A" for memory on GB10 | Unified memory architecture | Use `python3 -c "import torch; print(torch.cuda.get_device_properties(0).total_memory / 1024**3)"` |
| Session hangs / times out | `stream_idle_timeout_ms` too low for local inference | Set to `1800000` (30 min) or higher in provider config |
| `apply_patch` invoked as bash command instead of tool call | Known model behaviour (Issue #2235)[^10] | Re-prompt with "use the apply_patch tool" |
| Context window mismatch errors | `model_context_window` in config.toml does not match server `-c` | Ensure both values match exactly |
| `pip install vllm` fails on GB10 Ubuntu | PEP 668 blocks system pip installs | Must create a venv first -- but vLLM still fails due to ABI issue |

---

## What Works and What Does Not

Based on actual testing, not spec-sheet claims.

### Works

| Capability | Reliability | Notes |
|---|---|---|
| Text generation | High | Both setups |
| File reading (`Read`, `Glob`, `Grep`) | High | Tool calls are simple, single-argument |
| Bash command execution | High | Model reliably calls the `Bash` tool |
| Simple tool chains (read, edit, verify) | Medium-High | Works for 2-3 step chains |
| Single-file edits via `apply_patch` | Medium | Works most of the time |
| 5-step stress test | Tested | GB10 31B Dense passed first attempt. Mac 26B MoE passed but needed retries. |

### Fragile

**`apply_patch` reliability.** The model sometimes invokes `apply_patch` as a bash command instead of as a tool call. This is tracked as Issue #2235[^10]. Workaround: re-prompt with explicit language like "use the apply_patch tool to modify the file."

**Complex multi-tool chains.** Chains beyond 3-4 sequential tool calls become less reliable. The GB10's 31B Dense handled a 5-step stress test on the first attempt. The Mac's 26B MoE needed multiple retries for comparable tasks.

**Long context sessions.** As conversation grows beyond 32K tokens, quality degrades. Keep sessions focused and short. Use `/compact` aggressively.

### Does Not Work

| Capability | Status |
|---|---|
| Ollama tool calling on Apple Silicon | Broken (v0.20.3) |
| vLLM on GB10 Blackwell (pip install) | Broken (ABI incompatibility) |
| `wire_api = "chat"` | Removed from Codex CLI |
| Multi-file refactors (5+ files) | Unreliable -- use a cloud model |
| Reasoning tokens (`model_reasoning_effort`) | Not applicable to Gemma 4 |
| Image input via Codex CLI | Not supported by the harness for local models |

---

## Summary

Two setups work. On Apple Silicon, use llama.cpp with the 26B-A4B MoE at Q4_K_M. On the GB10, use Ollama v0.20.5 with the 31B Dense. Everything else we tried either crashed, froze, or failed to call tools.

The Mac is 5.1x faster at token generation (52 vs 10 tok/s) thanks to the MoE architecture. The GB10 produces higher quality output that succeeds on the first attempt. For agentic coding, first-attempt success matters more than raw speed.

Local inference is not a replacement for cloud models. It is a complement. Use local Gemma 4 for fast, private, zero-cost iteration. Use cloud models for complex refactors and production-critical work.

---

## Citations

[^2]: Gemma 4 Function Calling -- <https://ai.google.dev/gemma/docs/capabilities/text/function-calling-gemma4>

[^3]: Codex CLI Advanced Configuration -- <https://developers.openai.com/codex/config-advanced>

[^5]: Codex CLI Discussion #7782 (Responses API mandate) -- <https://github.com/openai/codex/discussions/7782>

[^8]: Ollama Issue #15315 (tool calling fix) -- <https://github.com/ollama/ollama/issues/15315>

[^10]: Codex Issue #2235 (apply_patch as bash) -- <https://github.com/openai/codex/issues/2235>

[^13]: Gemma 4 Benchmarks -- <https://gemma4all.com/blog/gemma-4-benchmarks-performance>
