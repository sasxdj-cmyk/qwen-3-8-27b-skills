# Qwen 3.8 27B — OpenCode Skills & Plugins

> Curated OpenCode skills and plugins that improve how Qwen 3.8 27B (including the IQ3 quantized GGUF builds) behaves while you use it inside OpenCode. This is **not** a model card and **not** a quantization guide: it collects the tools that make the model work better in an agent loop.

**Language:** English | [Italiano](README.it.md)

---

## Why this collection

Qwen 3.8 27B is a dense, hybrid-attention vision-language model with reasoning and MTP. In OpenCode it is usually served locally (llama.cpp / llama-server / Ollama / LM Studio) at IQ3 class quantization to fit 12–16 GB GPUs. Whatever the serving stack, the *model experience inside OpenCode* is dominated by three things:

1. **Context pressure** — long agent sessions drown the window in tool output.
2. **Instruction drift** — the system prompt gets summarized away during compaction.
3. **Output style** — verbose reasoning and template-like prose waste tokens and read poorly.

The skills and plugins below target exactly those three problems.

---

## Qwen 3.8 27B — Recommended stack

### 1. Context / performance skills (highest impact)

| Skill / Plugin | What it does | License | Link |
|---|---|---|---|
| **opencode-context-compress** | Background pruning + folds + squash. Old tool outputs become a stub, edits/writes/subagent results stay intact. Replaces native auto-compaction. | MIT | [AidenGeunGeun/opencode-context-compress](https://github.com/AidenGeunGeun/opencode-context-compress) |
| **opencode-dynamic-context-pruning (DCP)** | Dedupes repeated tool calls, supersedes writes, purges stale errors. Exposes `distill` / `compress` / `prune` tools the agent can call itself. | — | [yiranzai/opencode-dynamic-context-pruning](https://github.com/yiranzai/opencode-dynamic-context-pruning) |
| **context-optimizer** | LLMLingua-2 + rerank + dedupe on compaction. Runs locally, fails open. Keeps sessions small without losing critical context. | — | [evermeer/context-optimizer](https://github.com/evermeer/context-optimizer) |
| **opencode-magic-context** | Background historian + cross-session project memory + overnight consolidation. All cache-aware. | — | [bf-frieder/opencode-magic-context](https://github.com/bf-frieder/opencode-magic-context) |
| **opencode-slim** | TUI context panel, semantic compression, cost-aware pruning, adaptive thresholds. | — | [serkanalgur/opencode-slim](https://github.com/serkanalgur/opencode-slim) |
| **gguf-context-skill** | Local context compression/summarization using a small GGUF model (e.g. Qwen 2.5 1.5B IQ3) via `llama-server`. No external API, runs fully offline. | MIT | [./gguf-context-skill](./gguf-context-skill) |

> **Note:** on a Qwen 3.8 27B IQ3 local setup, context window is already tight. Choose **one** context manager, not several — they overlap and can fight each other. The local GGUF skill requires `llama-server` running a small model on a separate port.

### 2. Behavior / output skills

| Skill | What it does | Link |
|---|---|---|
| **prompt-engineering** | Few-shot, chain-of-thought, and prompt-optimization patterns. Loads when you write prompts, hooks, skills, or subagent instructions. | [montenoki/dotfiles](https://github.com/montenoki/dotfiles/blob/main/.config/opencode/skill/prompt-engineering/SKILL.md) |
| **stop-slop** | Removes AI-writing tells (throat-clearing, em-dashes, "Not X. But Y.", business jargon). 1–10 rubric, revises anything under 35/50. | [hardikpandya/stop-slop](https://github.com/hardikpandya/stop-slop) |
| **optimize-instructions** | Compresses and rewrites markdown instruction / prompt / skill files in place while preserving intent, placeholders, and command semantics. | [KatphLab/opencode-config](https://github.com/KatphLab/opencode-config) |
| **opencode-plain-style** | Injects plain-language rules into the system prompt on every call (+26.6 points measured compliance). Survives compaction. | [vasilvestre/opencode-plain-style](https://github.com/vasilvestre/opencode-plain-style) |
| **Caveman** | Cuts output tokens ~65% by stripping narration and filler while keeping code and technical facts byte-for-byte. | Julius Brussee (see OpenCode skills roundups) |
| **peso** | Prompt-enhancement pipeline with quality gate; adapts intensity to the model tier and keeps user prompts clean. | [bulga138/peso](https://github.com/bulga138/peso) |

### 3. Baseline that already ships with OpenCode

- **Compaction** — OpenCode's native compaction is the fallback. If you install a context plugin that replaces auto-compaction, set `compaction.auto` to `false` as the plugin requires.
- **Skills discovery** — OpenCode reads `~/.config/opencode/skills`, `.opencode/skills`, and the `.claude/skills` / `.agents/skills` compatibility paths. Drop a `SKILL.md` in one of those and it is advertised to the model automatically.

---

## Install examples

### A. Context plugin (opencode-context-compress)

```jsonc
// opencode.jsonc
{
  "compaction": { "auto": false },
  "plugins": [{ "package": "@skybluejacket/opencode-context-compress" }]
}
```

### B. Skill (stop-slop)

```bash
git clone https://github.com/hardikpandya/stop-slop.git ~/.config/opencode/skills/stop-slop
```

### C. Skill (prompt-engineering)

Copy the `prompt-engineering` folder into `~/.config/opencode/skills/prompt-engineering/`, keeping its `SKILL.md`.

### D. Local GGUF context manager (no external API)

```bash
# 1. Download a small GGUF model
hf download Qwen/Qwen2.5-1.5B-Instruct-GGUF --include "*IQ3_XXS*" --local-dir ./models/qwen-ctx

# 2. Start llama-server on a local port
llama-server -m ./models/qwen-ctx/qwen2.5-1.5b-instruct-iq3_xxs.gguf -c 4096 -ngl 99 -t 4 --port 11434 --host 127.0.0.1

# 3. Copy the skill
cp -r gguf-context-skill ~/.config/opencode/skills/gguf-context-skill
```

---

## Serving notes for Qwen 3.8 27B IQ3

These are *serving* settings, not skills. They matter because they define the context budget the skills above have to work with.

- Prefer a quant that fits fully on the GPU (e.g. `UD-IQ3_XXS` ≈ 11.9 GB, `UD-Q3_K_XL` ≈ 13.4 GB).
- Quantize the KV cache (`--cache-type-k q4_0 --cache-type-v q4_0`) if you need long context on 16 GB.
- Keep `--ctx-size` realistic: 16K–32K is enough for most agent turns; 64K–128K only with KV quantization.
- Disable or budget MTP if VRAM is tight; it costs memory for a modest speed gain.
- In OpenCode, `reasoning_effort` is prompt-driven, not enforced: `low` asks briefly, `xhigh` asks thoroughly, and the model still decides. Keep agent loops at `low`/`medium` unless the task truly needs deep reasoning.

---

## Selection criteria

A skill or plugin is included here only if it:

1. Works with OpenCode V2 (or clearly documents the V1 line), and
2. Changes what reaches the model (context, prompt, or output), and
3. Has a public repository with readable source.

No stars-based ranking, no affiliate links, no model-weight redistribution.

---

## License

This curated list is released under the MIT License. Each linked project keeps its own license.
