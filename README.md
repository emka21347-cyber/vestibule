# Vestibule

A local multi-agent pipeline that makes a series of short videos from public-domain Internet
Archive footage. Seven agents find a clip, plan changes to it, generate them, and assemble a
~30-second episode. A human approves each episode before anything is published.

The main goal is consistency across many episodes: each run reads a shared memory of what
earlier episodes established, and writes back to it once an episode is approved.

Runs on one machine with open-weight models.

## Pipeline

```
SOURCER → SHOWRUNNER → INTERPRETER → WRITER → DIRECTOR → DOP → EDITOR → human approval
```

Every step passes through SHOWRUNNER, which checks the output against the shared rules before
handing it on.

| Agent | Job |
|---|---|
| SOURCER | Searches the Archive, stores provenance (URL, licence, hash), filters for public domain |
| SHOWRUNNER | Routes work between agents, checks it against the rules, talks to the human, writes memory |
| INTERPRETER | Splits the clip into shots and describes keyframes. Doesn't plan anything |
| WRITER | Reads memory, then writes the on-screen text for this clip |
| DIRECTOR | Decides which frames change and how |
| DOP | Generates the changed frames in ComfyUI, checks them with the vision model, redoes failures |
| EDITOR | Cuts, grades and adds audio |

Each agent keeps a `STATUS.md` (current state) and an append-only `DOCUMENTATION.md` (history), so
it can be restarted without extra context.

## Memory

Markdown files in `memory/`, tracked in git:

- `canon/` — fixed rules, read-only at runtime
- `wiki/` — what the series has established so far; changes are kept, not overwritten
- `log.md` — one line per approved episode
- `index.md` — read first, so agents only load the pages they need

No vector database — at this size an index file is enough. Only SHOWRUNNER writes to memory, and
only after an episode is approved.

The check is looser early in the pipeline and stricter near the end, because rejecting a clip is
cheap and rejecting a finished render isn't.

Recurring elements are defined by outline and behaviour rather than a fixed look, so they're
allowed to vary between episodes. That also means the generator doesn't need to keep a character
visually stable across runs.

## Rules

- A human approves every episode; nothing is published automatically.
- Public-domain footage only, with provenance stored per clip.
- No real people shown saying or doing things.
- Output is labelled as AI-generated fiction.
- Publishing only through official platform APIs.
- Per-episode work runs locally; footage, prompts and output don't go to outside services.

## Stack

RTX 5090 (32 GB VRAM), 64 GB RAM.

- One multimodal MoE model shared by all agents, via Ollama or LM Studio; unloaded while
  ComfyUI generates
- ComfyUI for generation, ffmpeg for video, Internet Archive API for sourcing
- Dashboard: FastAPI + plain JS on `localhost:8765` — agent status, GPU/temperature, review
  queue, library, and a chat to steer a run

To keep it light on the GPU: the most-used experts stay in VRAM and the rest in RAM, fixed prompts
are cached, vision checks are batched, and plain routing doesn't call a model at all.


## Status

Pipeline built; 15 test suites, including a full end-to-end run. The dashboard runs on live data.
Next step is the first full unattended production run.

## Code

The source code is private; this repository is the project page. Footage is public domain;
provenance is stored per clip.
