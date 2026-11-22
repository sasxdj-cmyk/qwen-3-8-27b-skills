# Qwen 3.8 27B — Skill e plugin per OpenCode

> Raccolta curata di skill e plugin per OpenCode che migliorano il comportamento di Qwen 3.8 27B (incluse le build GGUF quantizzate IQ3) durante l'uso in OpenCode. **Non** è una model card né una guida alla quantizzazione: raccoglie gli strumenti che fanno lavorare meglio il modello in un loop agente.

**Language:** [English](README.md) | Italiano

---

## Perché questa raccolta

Qwen 3.8 27B è un modello vision-language denso ad attenzione ibrida con ragionamento e MTP. In OpenCode viene solitamente servito in locale (llama.cpp / llama-server / Ollama / LM Studio) con quantizzazione classe IQ3 per adattarsi a GPU 12–16 GB. Qualunque sia lo stack di serving, l'*esperienza del modello dentro OpenCode* è dominata da tre problemi:

1. **Pressione sul contesto** — sessioni agente lunghe annegano la finestra nell'output degli strumenti.
2. **Deriva delle istruzioni** — il system prompt viene riassunto durante la compattazione.
3. **Stile di output** — ragionamento verboso e prosa template-like sprecano token e leggono male.

Le skill e i plugin sotto mirano esattamente a questi tre problemi.

---

## Qwen 3.8 27B — Stack consigliato

### 1. Skill per contesto / performance (impatto massimo)

| Skill / Plugin | Cosa fa | Licenza | Link |
|---|---|---|---|
| **opencode-context-compress** | Pruning in background + folds + squash. Output vecchi diventano stub, edit/write/risultati subagenti restano intatti. Sostituisce l'auto-compaction nativa. | MIT | [AidenGeunGeun/opencode-context-compress](https://github.com/AidenGeunGeun/opencode-context-compress) |
| **opencode-dynamic-context-pruning (DCP)** | Deduplica tool call ripetuti, supersede write, purga errori vecchi. Espone tool `distill` / `compress` / `prune` che l'agente può chiamare da solo. | — | [yiranzai/opencode-dynamic-context-pruning](https://github.com/yiranzai/opencode-dynamic-context-pruning) |
| **context-optimizer** | LLMLingua-2 + rerank + dedupe sulla compattazione. Gira in locale, fail-open. Mantiene sessioni piccole senza perdere contesto critico. | — | [evermeer/context-optimizer](https://github.com/evermeer/context-optimizer) |
| **opencode-magic-context** | Historian in background + memoria cross-session + consolidamento notturno. Tutto cache-aware. | — | [bf-frieder/opencode-magic-context](https://github.com/bf-frieder/opencode-magic-context) |
| **opencode-slim** | Panel TUI contesto, compressione semantica, pruning cost-aware, soglie adattive. | — | [serkanalgur/opencode-slim](https://github.com/serkanalgur/opencode-slim) |
| **gguf-context-skill** | Compattazione contesto locale tramite modello GGUF piccolo (es. Qwen 2.5 1.5B IQ3) via `llama-server`. Zero chiamate API esterne, tutto offline. | MIT | [./gguf-context-skill](./gguf-context-skill) |

> **Nota:** su un setup Qwen 3.8 27B IQ3 locale, la finestra di contesto è già stretta. Scegli **un solo** gestore di contesto, non diversi — si sovrappongono e possono confliggere. La skill GGUF locale richiede `llama-server` con un modello piccolo su una porta separata.

### 2. Skill per comportamento / output

| Skill | Cosa fa | Link |
|---|---|---|
| **prompt-engineering** | Pattern few-shot, chain-of-thought, prompt optimization. Si carica quando scrivi prompt, hook, skill o istruzioni per subagenti. | [montenoki/dotfiles](https://github.com/montenoki/dotfiles/blob/main/.config/opencode/skill/prompt-engineering/SKILL.md) |
| **stop-slop** | Rimuove i "tells" della scrittura AI (aperture vuote, em-dash, "Not X. But Y.", gergo business). Rubrica 1–10, riscrive sotto 35/50. | [hardikpandya/stop-slop](https://github.com/hardikpandya/stop-slop) |
| **optimize-instructions** | Comprime e riscrive in-place file markdown di istruzioni / prompt / skill preservando intent, placeholder e semantica dei comandi. | [KatphLab/opencode-config](https://github.com/KatphLab/opencode-config) |
| **opencode-plain-style** | Inietta regole plain-language nel system prompt a ogni chiamata (+26.6 punti compliance misurati). Sopravvive alla compattazione. | [vasilvestre/opencode-plain-style](https://github.com/vasilvestre/opencode-plain-style) |
| **Caveman** | Taglia token output ~65% rimuovendo narrazione e filler, mantiene codice e fatti tecnici byte-per-byte. | Julius Brussee (vedi roundup skill OpenCode) |
| **peso** | Pipeline di prompt-enhancement con quality gate; adatta intensità al tier del modello e mantiene prompt utente puliti. | [bulga138/peso](https://github.com/bulga138/peso) |

### 3. Baseline che OpenCode include già

- **Compaction** — La compaction nativa di OpenCode è il fallback. Se installi un plugin contesto che sostituisce l'auto-compaction, imposta `compaction.auto` a `false` come richiesto dal plugin.
- **Skills discovery** — OpenCode legge `~/.config/opencode/skills`, `.opencode/skills`, e i path di compatibilità `.claude/skills` / `.agents/skills`. Metti un `SKILL.md` in uno di quelli e viene pubblicizzato al modello automaticamente.

---

## Esempi installazione

### A. Plugin contesto (opencode-context-compress)

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

Copia la cartella `prompt-engineering` in `~/.config/opencode/skills/prompt-engineering/`, mantenendo il suo `SKILL.md`.

### D. Local GGUF context manager (no external API)

```bash
# 1. Scarica un modello GGUF piccolo
hf download Qwen/Qwen2.5-1.5B-Instruct-GGUF --include "*IQ3_XXS*" --local-dir ./models/qwen-ctx

# 2. Avvia llama-server su porta locale
llama-server -m ./models/qwen-ctx/qwen2.5-1.5b-instruct-iq3_xxs.gguf -c 4096 -ngl 99 -t 4 --port 11434 --host 127.0.0.1

# 3. Copia la skill
cp -r gguf-context-skill ~/.config/opencode/skills/gguf-context-skill
```

---

## Note di serving per Qwen 3.8 27B IQ3

Queste sono impostazioni di *serving*, non skill. Contano perché definiscono il budget di contesto che le skill sopra devono gestire.

- Preferisci un quant che sta interamente sulla GPU (es. `UD-IQ3_XXS` ≈ 11.9 GB, `UD-Q3_K_XL` ≈ 13.4 GB).
- Quantizza la KV cache (`--cache-type-k q4_0 --cache-type-v q4_0`) se serve contesto lungo su 16 GB.
- Mantieni `--ctx-size` realistico: 16K–32K basta per la maggior parte dei turni agente; 64K–128K solo con KV quantization.
- Disabilita o budgetta MTP se VRAM è stretta; costa memoria per un guadagno di velocità modesto.
- In OpenCode, `reasoning_effort` è prompt-driven, non enforced: `low` chiede brevemente, `xhigh` chiede a fondo, e il modello decide comunque. Mantieni i loop agente a `low`/`medium` a meno che il task non richieda davvero ragionamento profondo.

---

## Criteri di selezione

Una skill o plugin è incluso qui solo se:

1. Funziona con OpenCode V2 (o documenta chiaramente la linea V1), e
2. Cambia cosa raggiunge il modello (contesto, prompt, o output), e
3. Ha un repository pubblico con sorgente leggibile.

Niente ranking per stelle, niente link affiliati, niente ridistribuzione pesi modello.

---

## Licenza

Questa lista curata è rilasciata sotto licenza MIT. Ogni progetto linkato mantiene la propria licenza.