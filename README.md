# Cook Timer — Frontend

**Live:** [prashantapurkayastha.github.io/cook-timer-front](https://prashantapurkayastha.github.io/cook-timer-front)  
**Backend repo:** [cook-timer-backend-](https://github.com/prashantapurkayastha/cook-timer-backend-)

Type in any dish — "2 chicken breasts, pan-fried" or "basmati rice, 2 cups" — and the app calls a Gemini-backed API to generate a multi-stage cooking timer specific to that dish, then counts it down stage by stage with audio and visual alerts.

---

## What It Does

Most cooking timers are single countdown clocks. This one understands the dish. Tell it you're making spaghetti bolognese and it breaks that into stages — browning the mince, simmering the sauce, boiling the pasta — with realistic times for each. It advances automatically from one stage to the next, beeps at each transition, and fires a persistent audio + screen flash alert when the last stage completes.

The intelligence is entirely in the backend (Gemini 2.5 Flash). The frontend is a state machine that renders and drives whatever stage data the API returns.

---

## Architecture

```
User types dish name
        ↓
POST /cook → Railway backend → Gemini 2.5 Flash
        ↓
{ dish, emoji, totalMinutes, stages: [{name, minutes, tip}] }
        ↓
Frontend state machine: INPUT → LOADING → TIMER → DONE
```

No framework. No build step. Single HTML file deployed to GitHub Pages.

---

## Frontend State Machine

The UI has four mutually exclusive states, each a `<div>` that shows or hides:

**INPUT** — Text field + example chips. Submits on button click or Enter key.

**LOADING** — CSS spinner shown while the API call is in flight. The submit button is disabled to prevent duplicate requests.

**TIMER** — The main cooking view. Three components run simultaneously:
- A large `MM:SS` countdown clock (turns `--accent` orange when under 60 seconds)
- A stage list showing upcoming, active (highlighted green), and completed (faded) stages
- A `setInterval` tick loop decrementing `secondsLeft` each second

When a stage hits zero, a short transition beep fires (`beepShort()`), `currentStage` increments, and `startStage()` restarts the clock for the next stage. When the last stage completes, the state moves to DONE.

**DONE** — Shows the dish name and total cook time. Fires `startPersistentAlert()`.

---

## Audio Implementation

Audio uses the Web Audio API directly — no libraries.

**Stage transition beep (`beepShort`):** Creates a fresh `AudioContext` per beep, a 660Hz sine oscillator, and an exponential gain ramp to zero over 0.4s. Disposable — the context is not reused.

**Completion alert (`startPersistentBeep`):** A single long-lived `AudioContext` with two oscillators running simultaneously: 880Hz and 1108Hz (a major third interval, matching iOS notification tones). The gain schedule is pre-computed at call time — a repeating on/off pattern (0.6s on, 0.3s off) for up to 300 seconds, written into the AudioContext timeline using `setValueAtTime` and `linearRampToValueAtTime`. This means the beep pattern continues without any JavaScript interval — it's baked into the audio graph.

**Stopping the alert:** `alertAudioCtx.close()` terminates the entire context and all its nodes instantly. This was the fix for an earlier bug where individual `oscillator.stop()` calls left the AudioContext alive and subsequent alerts failed to start — closing the context is the only reliable way to reset Web Audio state.

**Visual alert:** A CSS `@keyframes flash-bg` animation (`background` pulsing to `#ffcc00`) applied via a `.flashing` class on `<body>`. A forced reflow (`void document.body.offsetWidth`) before adding the class ensures the animation restarts cleanly if triggered multiple times.

---

## Design

| Token | Value | Purpose |
|---|---|---|
| `--bg` | `#faf9f6` | Warm off-white page background |
| `--accent` | `#d4622a` | Terracotta — primary action + urgency |
| `--active` | `#2a7d4f` | Forest green — active stage highlight |
| `--text` | `#1a1a18` | Near-black |
| Font (display) | DM Serif Display | Headers, dish name, countdown clock |
| Font (body) | DM Sans | Labels, tips, controls |

The colour palette is intentionally warm and kitchen-adjacent — terracotta and cream rather than the cold blue-and-white of most utility apps. The active stage uses green because green = "currently happening" is a universal reading.

The clock uses `DM Serif Display` at `4.5rem` — the largest element on the screen because during cooking, the phone is typically at arm's length or sitting on a counter.

---

## Stack

- **Vanilla JS** — no framework, no dependencies, no build step
- **Web Audio API** — multi-oscillator alerts with pre-scheduled gain envelopes
- **CSS custom properties** — all colours and transitions via `:root` variables
- **GitHub Pages** — zero-config deployment, single file

---

## Local Development

```bash
# No install needed. Open directly in a browser:
open index.html

# Or serve locally to avoid CORS issues with the API:
python3 -m http.server 8080
# then visit http://localhost:8080
```

To point at a local backend instead of production:

```js
// index.html line 1:
const API_BASE = "http://localhost:3000";
```

---

## Project Structure

```
cook-timer-front/
└── index.html    # Entire frontend — HTML, CSS, JS in one file
```

---

## Related

- **Backend:** [cook-timer-backend-](https://github.com/prashantapurkayastha/cook-timer-backend-) — Express proxy that calls Gemini 2.5 Flash and returns structured stage data
