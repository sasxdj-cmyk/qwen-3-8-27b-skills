---
name: Local GGUF Context Manager
description: Context compression/summarization using a local GGUF model (llama.cpp), no external API calls
metadata:
  opencode/autoinvoke: false
---

# Local GGUF Context Manager

Skill per la gestione del contesto di OpenCode interamente in locale tramite un modello GGUF piccolo (es. Qwen 2.5 1.5B o Llama-3.2 1B quantizzato IQ3/IQ4) servito da `llama-server`. Nessuna chiamata API esterna, nessun servizio cloud, zero dipendenze da Jev o TypeSafe.

## Quando usarlo

- Usi Qwen 3.8 27B IQ3 su 12–16 GB di VRAM e non puoi permetterti chiamate esterne per la compattazione del contesto.
- Vuoi evitare i rate limit e i costi dei servizi di compattazione remota (Jev, TypeSafe, ecc.).
- Vuoi che il contesto delle tue sessioni OpenCode venga gestito completamente offline.

## Cosa fa

Questa skill definisce un flusso di compattazione del contesto locale che:

1. **Rileva** quando il contesto della sessione si avvicina al limite (es. >80% della finestra).
2. **Chiama** `llama-server` locale con un modello GGUF piccolo per sintetizzare i vecchi turni in un blocco compatto.
3. **Sostituisce** il contesto originale con la sintesi, preservando le decisioni, i path di file, i comandi eseguiti e gli stati di task.
4. **Mantiene** i tool call critici (scrittura, modifica, subagent) intatti e non compressi.

## Installazione

### 1. Scarica un modello GGUF piccolo per la compattazione

Scegli uno dei seguenti, quantizzato per girare in locale con pochi GB di RAM/VRAM:

| Modello GGUF | Dimensione | Quantizzazione consigliata |
|--------------|------------|---------------------------|
| Qwen 2.5 1.5B | ~0.9 GB | IQ3_XXS o Q4_K_M |
| Llama-3.2 1B | ~0.7 GB | IQ4_XS o Q4_K_M |
| Phi-3 mini 3.8B | ~2.2 GB | IQ4_XS |

Esempio con Hugging Face:

```bash
hf download Qwen/Qwen2.5-1.5B-Instruct-GGUF --include "*IQ3_XXS*" --local-dir ./models/qwen-ctx
```

### 2. Avvia llama-server locale

```bash
llama-server -m ./models/qwen-ctx/qwen2.5-1.5b-instruct-q3_k_xxs.gguf -c 4096 -ngl 99 -t 4 --port 11434 --host 127.0.0.1
```

### 3. Attiva la skill in OpenCode

Copia questa cartella in `~/.config/opencode/skills/gguf-context-skill/` (o nel tuo progetto sotto `.opencode/skills/`).

OpenCode la caricherà automaticamente quando rileva la parola chiave `gguf-context` o quando richiedi esplicitamente `skill: gguf-context-skill`.

### 4. Configura il sistema prompt per usare la compattazione locale

Aggiungi a `opencode.jsonc` o al tuo AGENTS.md:

```markdown
## Compattazione contesto locale (GGUF)

Quando il contesto supera l'80% della finestra, chiedi all'agente di:
1. Riepilogare i turni precedenti in un blocco compatto (max 2000 token).
2. Conservare: obiettivo utente, decisioni prese, file modificati, comandi eseguiti, task in corso.
3. Rimuovere: output di tool call ridondanti, errori risolti, narrazione superflua.
4. Inviare il riepilogo a `llama-server` locale su `http://127.0.0.1:11434` per la sintesi finale.
5. Sostituire il contesto con il riepilogo sintetizzato.
```

## Vantaggi vs Jev/TypeSafe

| Aspetto | Jev / TypeSafe | GGUF locale |
|---------|---------------|-------------|
| Chiamate esterne | Sì (API remota) | No, tutto in locale |
| Dipendenza rete | Obbligatoria | Opzionale (solo per download iniziale) |
| Costo | Per token API | Zero |
| Privacy | Dati escono dal PC | Dati restano sul PC |
| Latienza | Dipende dal servizio | Dipende dall'hardware locale |

## Limitazioni

- La qualità della sintesi dipende dal modello GGUF scelto: modelli sotto 2B possono perdere dettagli su sessioni molto lunghe.
- `llama-server` deve essere attivo in background prima di invocare la compattazione.
- Non sostituisce la compattazione nativa di OpenCode: se vuoi disabilitarla, imposta `compaction.auto` a `false` in `opencode.jsonc`.
