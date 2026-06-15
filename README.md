<p align="center">
  <img src="assets/banner.png" alt="chatjimmy-proxy — Claude Code at warp speed" width="100%">
</p>

<h1 align="center">chatjimmy-proxy ⚡</h1>

<p align="center">
  <b>Run Claude Code at warp speed — on <a href="https://chatjimmy.ai">ChatJimmy</a>'s Llama 3.1 8B at ~14,500 tokens/sec.</b><br>
  A tiny local proxy that speaks the Anthropic Messages API, so the full Claude Code harness
  drives a free, blisteringly fast model. Keep the harness; swap in the fastest brain.
</p>

<p align="center">
  <a href="https://dennisonbertram.github.io/chatjimmy-proxy/">Website</a> ·
  <a href="#quickstart">Quickstart</a> ·
  <a href="#how-it-works">How it works</a> ·
  <a href="#speed">Speed</a> ·
  <a href="#quality">Quality</a> ·
  <a href="#safety">Safety</a>
</p>

---

## What is this?

[ChatJimmy](https://chatjimmy.ai) serves Llama 3.1 8B at an absurd **~14,500 tokens/sec** —
roughly **10× faster than Groq** and **150–280× faster than frontier models**. It has no
official API and no native tool-calling.

`chatjimmy-proxy` bridges that gap. It's a local server that:

- **Speaks the Anthropic Messages API** — so [Claude Code](https://claude.com/claude-code) (via [deep-claude](https://github.com/dennisonbertram/deep-claude)) talks to it unmodified.
- **Translates tool calls** — injects a tool-use format into the prompt and parses the model's text back into Anthropic `tool_use` blocks (ChatJimmy has no tools API).
- **Makes a weak 8B reliable** — best-of-N resampling, grounded answer selection, a tuned compact prompt, and a destructive-command guard.

The result: a real agentic coding loop — read, edit, create, search, run — on a free model,
fast enough that 5× oversampling for reliability is essentially free.

> **Why it works:** the speed *buys* the reliability. Each inference is ~3 ms, so the proxy
> can draw the model 5 times per turn to get a valid tool call — and the whole turn is still
> faster than a single frontier-model call.

## Quickstart

### Requirements

- **macOS or Linux**
- **[Claude Code](https://claude.com/claude-code)** — `claude` on your `PATH`
- **[deep-claude](https://github.com/dennisonbertram/deep-claude)** — routes Claude Code to a custom endpoint without touching your real Anthropic login
- **[Node.js](https://nodejs.org/)** 18+

### Install

```bash
git clone https://github.com/dennisonbertram/chatjimmy-proxy
cd chatjimmy-proxy
npm install
npm run build

# register chatjimmy as a deep-claude endpoint (one time)
~/path/to/deep-claude/bin/deep-claude endpoints add chatjimmy http://localhost:3000
```

### Run

```bash
cd your-project          # run it from the project you want to work in
/path/to/chatjimmy-proxy/chatjimmy-code
```

That's it. `chatjimmy-code` auto-starts the proxy and launches a clean Claude Code session on
ChatJimmy. One-shot mode:

```bash
chatjimmy-code -p "create an index.html with a Hello World heading"
```

Tip — add an alias:

```bash
echo 'alias cjcode="/path/to/chatjimmy-proxy/chatjimmy-code"' >> ~/.zshrc
```

> **Run it from your project directory.** Claude Code sandboxes file tools to the working
> directory — paths outside it (like `/tmp`) are blocked.

## How it works

```
Claude Code (Anthropic Messages API)
        │   via deep-claude --endpoint chatjimmy  (isolates your real Anthropic login)
        ▼
  chatjimmy-proxy   :3000
    • strips Claude Code's giant prompt → compact tuned prompt (--system-prompt-file)
    • strips <system-reminder> noise, filters to coding tools
    • injects <tool_call> format into the system prompt
    • best-of-N: re-sample until a valid tool call parses
    • grounded best-of-N: pick the answer most supported by tool output
    • guards against destructive commands
        ▼
  https://chatjimmy.ai/api/chat   →   Llama 3.1 8B  @ ~14,500 tok/s
```

The proxy is **backend-switchable**: set `BACKEND=openrouter` to route to
`meta-llama/llama-3.1-8b-instruct` instead — handy for tuning against the same model on a
billed API without hammering ChatJimmy.

## Speed

Measured with `./eval/speed-benchmark.sh` (ChatJimmy `/api/chat` telemetry + end-to-end runs):

| Metric | ChatJimmy Llama 3.1 8B |
|--------|------------------------|
| Decode rate | **~14,500 tokens/sec** |
| Time-to-first-token | **~1.1 ms** |
| Generate a full HTML page | **~8 ms** of inference (~1.5 s end-to-end incl. harness) |
| Model time per agent turn | **~330 ms** (incl. up to 5× best-of-N + network) |

For comparison (decode rate): Claude / GPT-4 class ≈ 50–100 tok/s · Groq Llama-8B ≈ 1,250 tok/s.

## Quality

After prompt + harness tuning (real Claude Code agent loop):

| Task set | ChatJimmy | OpenRouter llama-3.1-8b |
|----------|-----------|--------------------------|
| Core 4 (read / edit / create / grep) | **5/5 each** (≈20/20) | 20/20 |
| Hard 10 (multi-step, mutations, multi-file) | ≈0.70 | 1.00 |

ChatJimmy's instance is more quantized than OpenRouter's, so absolute quality is lower — but
the core coding loop is reliable, and the speed is unmatched. It's an 8B model: great for
focused file ops, weaker on complex multi-file logic.

Reproduce: `./eval/real-path-eval.sh 5` (core), `EVAL_TASKS=./tasks-hard ... eval/run-eval.js` (hard).

## Safety

The weak model can hallucinate commands, so the proxy refuses to pass through destructive or
privileged shell calls (`sudo`, `rm -rf /`, `mkfs`, `dd`, `curl … | sh`, writes to
`/etc/{passwd,shadow,hosts,sudoers}`, …). It also never *forces* a tool call — a greeting like
"hi" gets a plain-text reply, not an invented command. Claude Code's own permission system
still applies on top.

This is defense-in-depth for a small model, not a security boundary — run it on code and
directories you trust, like any coding agent.

## Configuration

The `chatjimmy-code` launcher bundles the settings that make the weak model work: compact
system prompt, no MCP servers, no skills, coding-only toolset. Override via env:

| Env | Default | What |
|-----|---------|------|
| `BACKEND` | `chatjimmy` | `openrouter` to use `meta-llama/llama-3.1-8b-instruct` |
| `TOOL_SAMPLE_ATTEMPTS` | `5` | best-of-N draws to get a valid tool call |
| `ANSWER_SAMPLE_ATTEMPTS` | `3` | grounded answer candidates |
| `TOOL_ALLOWLIST` | coding set | comma list, or `*` to pass all tools through |
| `MAX_SYSTEM_BYTES` | `18000` | trim the system prompt to fit ChatJimmy's ~24 KB ceiling |
| `CHATJIMMY_TOOLS` | `Read,Write,Edit,Bash,Grep,Glob,LS` | tools advertised to the model |

## Project layout

| Path | Purpose |
|------|---------|
| `src/server.ts` | the proxy: format conversion, tool translation, best-of-N, safety guard |
| `src/tools.ts` | tool-prompt injection + parsing |
| `chatjimmy-code` | one-command launcher (bare-basic Claude Code on ChatJimmy) |
| `llama-system-prompt.txt` | the tuned compact system prompt |
| `eval/` | eval harness, hard task suite, real-path + speed benchmarks |
| `BENCHMARK.md` | the speed story |

## Acknowledgements

Built to ride on [deep-claude](https://github.com/dennisonbertram/deep-claude), which does the
hard work of pointing Claude Code at a custom endpoint while keeping your real Anthropic login
untouched. ChatJimmy provides the ludicrously fast inference.

## License

MIT — see [LICENSE](LICENSE).
