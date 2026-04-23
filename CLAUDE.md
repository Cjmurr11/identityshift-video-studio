# IdentityShift.io — Video Editing Studio

This workspace is a full video post-production pipeline: raw footage → filler removal → motion graphics → final MP4.

---

## Pipeline Overview

```
RAW VIDEO
    │
    ▼
[Stage 1 — Edit & Filler Removal]  (video-use)
    │  transcribe → pack → EDL → render
    │
    ▼
[Stage 2 — Motion Graphics]  (Hyperframes)
    │  HTML compositions + GSAP → headless Chrome → FFmpeg encode
    │
    ▼
FINAL MP4
```

---

## Tool Stack

| Tool | Location | Purpose |
|------|----------|---------|
| video-use | `C:\Users\cjmur\Developer\video-use` | Filler removal, cuts, color grade, subtitles |
| Hyperframes CLI | npm (global via npx) | Motion graphics rendering |
| hyperframes-student-kit | `./hyperframes-student-kit` | 12 reference motion graphics projects |
| FFmpeg | system PATH | Video encoding backbone |
| ElevenLabs Scribe | API (key in video-use `.env`) | Word-level transcription |

---

## Stage 1: Edit & Filler Removal (video-use)

**Skill file:** `C:\Users\cjmur\Developer\video-use\SKILL.md` — always read this first.

### Quick Start

```bash
cd C:\Users\cjmur\Developer\video-use

# 1. Transcribe the raw video (local Whisper — no API key needed)
uv run python helpers/transcribe_whisper.py "D:/Video Editing- IdentityShift.io/raw/<video.mp4>"

# 2. Pack transcript into readable markdown
uv run python helpers/pack_transcripts.py --edit-dir "D:/Video Editing- IdentityShift.io/raw/"

# 3. View a timeline section (optional, for decision-making)
uv run python helpers/timeline_view.py "<video.mp4>" <start_sec> <end_sec>

# 4. After EDL is generated, render
uv run python helpers/render.py edl.json -o "D:/Video Editing- IdentityShift.io/edits/<output.mp4>" --preview

# 5. Color grade (optional)
uv run python helpers/grade.py <input.mp4> -o <output.mp4>
```

### Pipeline Phases

1. **Inventory** — ffprobe all sources, batch transcribe, pack transcripts
2. **Strategy** — describe content, ask questions, propose cut strategy, **await confirmation**
3. **Execution** — generate `edl.json` (beat-by-beat takes), render with `--preview` (720p)
4. **Self-Eval** — check cut boundaries ±1.5s, audio pops, subtitle alignment (max 3 passes)
5. **Iterate** — accept natural language feedback, re-render without re-transcribing

### Filler Word Removal
There is no single "remove fillers" command. Instead:
- Transcription preserves every word with timestamps
- During EDL generation, identify fillers/slips in `takes_packed.md`
- Cuts snap to word boundaries; silence gaps ≥400ms are preferred targets
- Never cut inside a word; pad edges 30–200ms

### Hard Rules
- Subtitles applied **last** in filter chain (after overlays)
- Per-segment extraction + lossless concat (never single-pass filtergraph)
- 30ms audio fades at every segment boundary
- All outputs go in `<videos_dir>/edit/` only
- Cache transcripts per source (no re-transcription)

### Whisper Models
Default is `small` — best balance for talking-head videos. Pass `--model` to override:
- `tiny` / `base` — fastest, slightly less accurate
- `small` — **default, recommended for solo talking-head**
- `medium` / `large-v3` — most accurate, slower (use if small misses words)

---

## Stage 2: Motion Graphics (Hyperframes)

**Motion philosophy:** `./hyperframes-student-kit/MOTION_PHILOSOPHY.md` — read before every new composition.

### Quick Start

```bash
# Initialize a new motion graphics project
cd "D:\Video Editing- IdentityShift.io"
npx hyperframes init my-video
cd my-video

# Live preview (share localhost:3002 for approval before rendering)
npx hyperframes preview

# Draft render (fast, CRF 28)
npx hyperframes render --quality draft

# Final render (lossless 1080p, CRF 18)
npx hyperframes render --quality standard

# Browse 38 motion blocks (outros, transitions, social overlays, data viz)
npx hyperframes catalog --type block

# Install a catalog block
npx hyperframes add <block-name>

# Word-level transcription (Whisper, on-device)
npx hyperframes transcribe <file>

# On-device TTS narration
npx hyperframes tts "your text here"
```

### Composition Rules (never skip)
- Root `<div>`: `id`, `data-composition-id`, `data-start="0"`, `data-width`, `data-height`
- Timed elements: `class="clip"`, `data-start`, `data-duration`, `data-track-index`
- `<video>` must be `muted`; audio goes in sibling `<audio>` element
- Register exactly one GSAP timeline (paused) on `window.__timelines["<id>"]`
- Never call `.play()`, `.pause()`, or `.currentTime` on media
- Wrap `<video>` in `<div>` to animate position/size (direct animation freezes frames)
- No `Date.now()`, unseeded randomness, or network calls at render time

### Two-Gate Approval (mandatory)
1. **Gate 1 — Live Studio**: `npx hyperframes preview` → share `http://localhost:3002` → wait for verbal OK
2. **Gate 2 — Draft MP4**: `npx serve . -p 8080 -n` → share URL → wait for scrub approval → then run final render

### Available Blocks (38 total)
- **Transitions**: glitch, whip-pan, cinematic-zoom, light-leak, ripple-waves, domain-warp-dissolve, sdf-iris, thermal-distortion, swirl-vortex
- **Social overlays**: Instagram, TikTok, YouTube, X, Reddit, Spotify
- **UI**: app-showcase, ui-3d-reveal, flowchart, data-chart
- **Outros**: logo-outro
- **Components**: grain-overlay, shimmer-sweep, grid-pixelate-wipe

### Motion Shorthand
- **Easing**: smooth / snappy / bouncy / springy / dramatic / dreamy
- **Caption energy**: hype / corporate / tutorial / storytelling / social
- **Transition energy**: calm (blur) / medium (push) / high (zoom, glitch)

### Reference Projects (student kit)
Located in `./hyperframes-student-kit/video-projects/`:
- `aisoc-app-release`, `aisoc-hype`, `claude-edit-intro`, `clickup-demo`
- `first-agent-promo`, `golden-ratio-demo`, `hyperframes-sizzle`
- `linear-promo-30s`, `may-shorts-6`, `may-shorts-18`, `may-shorts-19`

Study these to understand real composition patterns before building new ones.

### Docs
- Full reference: https://hyperframes.heygen.com/reference/html-schema
- Catalog: `npx hyperframes docs examples`

---

## Combining Both Stages

Typical full pipeline for a talking-head video:

```
1. Drop raw .mp4 into D:\Video Editing- IdentityShift.io\raw\
2. Run video-use transcription + EDL generation (Stage 1)
3. Review edits, approve
4. Render edited clean cut → edit\clean.mp4
5. Re-encode for Hyperframes compatibility:
   ffmpeg -i edit\clean.mp4 -c:v libx264 -preset medium -crf 20 -c:a aac -b:a 192k assets\clip.mp4
6. Build motion graphics composition in Hyperframes (Stage 2)
7. Reference clean.mp4 as <video> asset inside HTML composition
8. Preview → approve → draft render → approve → final render
```

---

## Directory Structure

```
D:\Video Editing- IdentityShift.io\
├── CLAUDE.md                    ← this file
├── raw\                         ← drop raw footage here
├── edits\                       ← video-use output
├── hyperframes-student-kit\     ← 12 reference motion graphics projects
└── <project-slug>\              ← each Hyperframes project lives here
    ├── index.html
    ├── compositions\
    ├── assets\
    └── renders\
```

---

## Setup Status

| Dependency | Status |
|------------|--------|
| Node.js | ✅ v24.14.0 |
| FFmpeg | ✅ installed (restart shell to activate PATH) |
| Python 3.12 | ✅ installed |
| uv | ✅ installed |
| video-use Python deps | ✅ installed via `uv sync` |
| faster-whisper (local Whisper) | ✅ installed — no API key needed |
| Hyperframes CLI | ✅ available via `npx hyperframes` |

---

## Key Source Repos

- video-use: https://github.com/browser-use/video-use
- Hyperframes core: https://github.com/heygen-com/hyperframes
- Student kit: https://github.com/nateherkai/hyperframes-student-kit
