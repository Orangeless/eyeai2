# eyeAI 2 🎬

The intermediate version of the dAIsy page assistant — the step between the [2D sprite-based eyeAI](https://github.com/Orangeless/eyeai) and the [3D VRM-based dAIsy3d](https://github.com/Orangeless/dAIsy3d). eyeAI 2 replaces the hand-drawn canvas sprite with a **pre-rendered video avatar**: MP4 clips for idle, talking, pointing, and more, managed by a scheduler that transitions between states automatically.

> **Evolution:** eyeAI (2D canvas sprites) → **eyeAI 2 (video clips)** → dAIsy3d (Three.js VRM)

---

## What it does

- **Page-aware Q&A** — Scrapes the current page's visible text and sends it alongside the user's question to a Hugging Face LLM. Answers are grounded solely in the page content.
- **Quote highlighting** — After answering, the relevant sentence from the page is highlighted with a pulsing yellow outline and smoothly scrolled into view. The highlight clears automatically after a duration estimated from the answer's word count.
- **Video avatar** — The character is rendered using layered `<video>` elements. A paused video frame serves as the idle still; a second video element plays animations on top when active.
- **Idle animation scheduler** — A 250ms tick loop fires random idle clips from a pool (`idle_main1.mp4`, `idle1.mp4`, `idle2.mp4`) every 30–60 seconds, then holds the last frame as the new still.
- **Talking animation** — While the answer is displayed, `talking_mid.mp4` loops for the estimated read duration, then the avatar returns to idle.
- **Draggable UI** — The avatar shell, answer bubble, and input bar are all independently draggable with mouse and touch support.
- **Collapsible answer bubble** — Clicking the bubble minimizes it to a small pill; clicking "Show answer" restores it.
- **Close button** — Fully hides and disables the widget without a page reload.
- **Graceful fallback** — If the LLM doesn't return structured quotes, the frontend derives the best matching sentence from the page itself using keyword scoring.

---

## Project structure

```
eyeai2/
├── backend/
│   ├── server.js       # Express API — proxies questions to HF Router, returns answer + quotes
│   ├── package.json
│   └── .env            # HF_TOKEN and optional HF_MODEL go here
└── frontend/
    ├── main.js         # Self-contained IIFE — all UI, video logic, highlight engine
    ├── eyeai.css       # Styles for shell, bubble, input, highlight animation
    ├── index.html      # Demo page
    └── assets/
        └── video/
            ├── idle_main1.mp4          # Primary idle (also used as still base)
            ├── idle_main.mp4           # Alternate idle
            ├── idle1.mp4               # Rare idle variant
            ├── idle2.mp4               # Rare idle variant
            ├── talking_start.mp4       # Talk intro
            ├── talking_mid.mp4         # Talk loop (plays during answer display)
            ├── talking_end.mp4         # Talk outro
            ├── pointing.mp4            # Pointing gesture
            ├── explaining_positive.mp4 # Positive explanation gesture
            ├── explaining_negative.mp4 # Negative explanation gesture
            └── goodbye.mp4             # Goodbye animation
```

---

## How the video avatar works

Two `<video>` elements are stacked on top of each other:

1. **`idleStill`** — always present, loaded from `idle_main1.mp4` and seeked to its last frame, then paused. This is what you see when nothing is happening.
2. **`animVideo`** — hidden by default (`display: none`). When an animation needs to play, this element gets a new `src`, plays, then hides again. The idle still underneath shows through the transparent video background.

A scheduler polls every 250ms. When the idle timer fires, it picks a random clip from the pool, plays it through to completion using a `timeupdate`-based segment tracker, then seeks the idle still to that clip's last frame and waits for the next scheduled idle.

When the user asks a question, `animationHold` is set to `true` to suppress the scheduler, `talking_mid.mp4` loops for the estimated answer duration, then the hold is released.

---

## Getting started

### Prerequisites

- [Node.js](https://nodejs.org/) v18+
- A [Hugging Face](https://huggingface.co/) account with an API token

### 1. Clone the repo

```bash
git clone https://github.com/Orangeless/eyeai2.git
cd eyeai2
```

### 2. Configure the backend

```bash
cd backend
npm install
```

Create a `.env` file inside `backend/`:

```env
HF_TOKEN=hf_your_token_here
HF_MODEL=google/gemma-2-2b-it   # optional — this is the default
```

### 3. Start the backend

```bash
npm start
# Backend running at http://localhost:3001
```

Check it's alive: `http://localhost:3001/health`

### 4. Serve the frontend

Must be served over HTTP — open-as-file won't work due to video loading. From the `frontend/` folder:

```bash
npx serve .
# or
python -m http.server 8080
```

Open `http://localhost:8080`.

---

## Embedding on any page

`main.js` is a self-contained IIFE — add one script tag to any page:

```html
<script src="path/to/main.js"></script>
```

Make sure `assets/video/` is accessible at the path set in `CFG.assetsBase` (default: `""`, i.e. relative to the page), and the backend is running at `http://localhost:3001`.

---

## Configuration

All tuneable values are in the `CFG` object at the top of `main.js`:

```js
const CFG = {
  backendUrl:    "http://localhost:3001/api/ask",
  assetsBase:    "",                 // Prefix for all asset paths
  avatarWidth:   640,                // Native resolution — do not change
  avatarHeight:  848,
  avatarUIScale: 0.4,                // Visual size on screen
  avatarPos:     { right: 32, bottom: 92 },
  inputPos:      { right: 32, bottom: 18 },
  bubbleMode:    "fixed",            // "fixed" or relative to avatar
  idleMainEverySecondsMin: 30,       // Min time between idle animations
  idleMainEverySecondsMax: 60,       // Max time between idle animations
  idleMainVideo: "assets/video/idle_main1.mp4",
  idleRareVideos: ["assets/video/idle1.mp4", "assets/video/idle2.mp4"],
  highlightMinMs: 3000,              // Min time quote stays highlighted
  highlightMaxMs: 10000,             // Max time quote stays highlighted
};
```

---

## How it differs from the other versions

| Feature | eyeAI (v1) | eyeAI 2 (this repo) | dAIsy3d (v3) |
|---|---|---|---|
| Avatar | 2D canvas sprites | Pre-rendered MP4 video clips | Three.js + VRM 3D model |
| Idle animation | Procedural float + blink | Scheduled video clip pool | VRMA clip system |
| Talking animation | Mouth-flap frames | `talking_mid.mp4` loop | Pointing VRMA clip |
| TTS | ✅ Web Speech API | ❌ | ❌ |
| Quote highlighting | ❌ | ✅ | ✅ |
| Draggable UI | ❌ | ✅ | ✅ |
| Avatar spin/tilt | ❌ | ❌ | ✅ (mouse drag) |
| Close button | ❌ | ✅ (hides widget) | ✅ |
| Dependencies | None | None | Three.js, @pixiv/three-vrm |

---

## Tech stack

| Layer | Technology |
|---|---|
| Avatar rendering | HTML5 `<video>` elements, MP4 clips |
| Backend | Express (Node.js) |
| LLM | Hugging Face Inference Router (OpenAI-compatible) |
| Default model | `google/gemma-2-2b-it` |

---

## License

Copyright (No use)
