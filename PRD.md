# Poker HUD – Product Requirements Document (PRD)

> **Version:** 0.1.0 · **Status:** Draft · **Last Updated:** 2026-02-19

---

## Table of Contents

1. [Overview](#1-overview)
2. [Goals & Success Criteria](#2-goals--success-criteria)
3. [Tech Stack](#3-tech-stack)
4. [Architecture Overview](#4-architecture-overview)
5. [Feature Specifications](#5-feature-specifications)
   - 5.1 [Real-Time HUD](#51-real-time-hud)
   - 5.2 [Session Health Meter](#52-session-health-meter)
   - 5.3 [Stop-Loss / Stop-Win Alerts](#53-stop-loss--stop-win-alerts)
   - 5.4 [One-Click Gut Check](#54-one-click-gut-check)
   - 5.5 [Visual Player Heatmaps](#55-visual-player-heatmaps)
   - 5.6 [AI Pro Feedback Engine](#56-ai-pro-feedback-engine)
   - 5.7 [Post-Game Report Card](#57-post-game-report-card)
6. [AI System Prompt Template](#6-ai-system-prompt-template)
7. [Equity Calculator Integration](#7-equity-calculator-integration)
8. [User Experience Principles](#8-user-experience-principles)
9. [Non-Functional Requirements](#9-non-functional-requirements)
10. [Out of Scope (v1)](#10-out-of-scope-v1)
11. [Open Questions](#11-open-questions)

---

## 1. Overview

**Poker HUD** is a desktop overlay application purpose-built for [PokerNow](https://www.pokernow.club/) home games. It reads the live `.csv` log files produced by PokerNow, displays real-time opponent statistics on an unobtrusive heads-up display (HUD), and provides AI-powered post-hand coaching feedback aimed at micro-stakes players.

The primary audience is **casual-to-semi-serious home-game players** who want actionable insight without needing to understand complex solver output or statistical notation.

---

## 2. Goals & Success Criteria

### Primary Goal
> *Leave every session with more money than you started with.*

### Measurable Success Criteria

| Metric | Target |
|---|---|
| Session win-rate improvement after 30 days of use | +10% vs. baseline |
| Time-to-install for a non-technical user | ≤ 5 minutes |
| AI feedback latency per hand (post-hand analysis) | ≤ 8 seconds |
| HUD overlay CPU overhead during play | < 3% CPU |
| User retention (weekly active users after month 1) | ≥ 60% |

---

## 3. Tech Stack

### Frontend

| Layer | Choice | Rationale |
|---|---|---|
| UI Framework | **React** (with Vite) | Large ecosystem, component reuse, easy state management |
| Styling | **Tailwind CSS** | Utility-first, fast iteration, consistent design tokens |
| Animations | **Framer Motion** | Smooth HUD update transitions without custom CSS |
| Desktop Shell | **Electron** | Cross-platform, file-system access, overlay support |

### Backend / Logic

| Layer | Choice | Rationale |
|---|---|---|
| Log Watcher & Parser | **Python + FastAPI** | Simple async server; Python has excellent CSV/text tooling |
| Real-time Push | **WebSockets** (via FastAPI) | Low-latency push to the Electron frontend |
| AI Integration | **OpenAI API** (GPT-4o) | Best-in-class reasoning for poker hand analysis |
| Equity Calculator | **Peal / Calamari API** (Phase 2) | Mathematically verified equity figures |

### Data

| Concern | Approach |
|---|---|
| Session storage | Local SQLite database (no cloud required) |
| Log ingestion | Tail-watch the PokerNow `.csv` export folder |
| User preferences | JSON config file in app data directory |

---

## 4. Architecture Overview

```
┌─────────────────────────────────────────────────────────────────┐
│  Electron Shell (Desktop App)                                   │
│  ┌─────────────────────────────────┐  ┌────────────────────┐   │
│  │  React UI (HUD Overlay)         │  │  Electron Main     │   │
│  │  - Player stat cards            │◄─┤  - Window mgmt     │   │
│  │  - Session Health Meter         │  │  - System tray     │   │
│  │  - Gut Check widget             │  └────────┬───────────┘   │
│  │  - Stop-Loss / Stop-Win alerts  │           │               │
│  └───────────────┬─────────────────┘           │               │
│                  │ WebSocket                    │               │
└──────────────────┼──────────────────────────────┼───────────────┘
                   │                              │
      ┌────────────▼──────────────────────────────▼────────────┐
      │  Python FastAPI Server (localhost)                      │
      │  ┌──────────────┐  ┌──────────────┐  ┌─────────────┐  │
      │  │  Log Watcher │  │ Hand Parser  │  │  AI Engine  │  │
      │  │  (watchdog)  │─►│  & Stats DB  │─►│  (GPT-4o)   │  │
      │  └──────────────┘  └──────────────┘  └─────────────┘  │
      │                            │                            │
      │                     ┌──────▼──────┐                    │
      │                     │  SQLite DB  │                    │
      │                     └─────────────┘                    │
      └─────────────────────────────────────────────────────────┘
                   │
      ┌────────────▼───────────────┐
      │  PokerNow .csv Log File    │
      │  (watched in real-time)    │
      └────────────────────────────┘
```

---

## 5. Feature Specifications

### 5.1 Real-Time HUD

**Description:** An overlay of player stat cards shown during an active PokerNow session.

**Stats displayed per player:**

| Stat | Full Name | Description |
|---|---|---|
| VPIP | Voluntarily Put $ In Pot | % of hands where player voluntarily entered pot |
| PFR | Pre-Flop Raise | % of hands where player raised pre-flop |
| AF | Aggression Factor | Ratio of aggressive to passive actions post-flop |
| 3-Bet% | 3-Bet Percentage | % of hands where player 3-bet pre-flop |
| Fold to 3-Bet | — | % of times player folded to a 3-bet |
| WTSD | Went to Showdown | % of times player saw a showdown when reaching the flop |
| Hands | — | Sample size (shown as confidence indicator) |

**UX Requirements:**
- Cards are compact (≤ 150 × 80 px) and semi-transparent.
- Low sample size (< 20 hands) shows stats in gray/muted to signal unreliability.
- Cards update within 1 second of a new log line being written.
- A toggle hotkey (default: `Alt+H`) shows/hides the overlay.

---

### 5.2 Session Health Meter

**Description:** A persistent visual bar that shows the player's current profit or loss relative to their buy-in for the session.

**States:**

| Zone | Condition | Color | Display |
|---|---|---|---|
| Up Big | Profit ≥ 50% of buy-in | 🟢 Green (bright) | "+2.5 buy-ins" |
| Ahead | Profit 20–49% of buy-in | 🟢 Green (soft) | "+1.3 buy-ins" |
| Even | -20% to +20% of buy-in | 🟡 Yellow | "Breakeven" |
| Behind | Loss 20–49% of buy-in | 🟠 Orange | "-0.8 buy-ins" |
| Red Zone | Loss ≥ 2 buy-ins | 🔴 Red (pulsing) | "Stop-Loss Zone" |

**Behavior:**
- Updates in real-time as pots are won/lost.
- Shows session duration alongside profit (e.g., "2h 15m · +$45").
- Optional: plays a subtle chime when crossing from Ahead → Up Big (positive reinforcement).

---

### 5.3 Stop-Loss / Stop-Win Alerts

**Description:** Contextual alerts that help players avoid tilt-driven losses and lock in winning sessions.

#### Stop-Win Alert (Green Zone)

- **Trigger:** Player is up ≥ 20% of starting buy-in AND Gut Check indicates "Tired" or "Bored."
- **Message:** *"You're up [X]% tonight. Your gut says you're fading — locking in this win is the disciplined play."*
- **Actions:** `Keep Playing` / `End Session`

#### Stop-Loss Alert (Red Zone)

- **Trigger:** Player has lost ≥ 2 buy-ins in the session.
- **Message:** *"You've hit your stop-loss limit. Taking a break now protects your bankroll. Chasing rarely works."*
- **Actions:** `Take a 15-min Break` / `End Session` / `Dismiss (I'm sure)`
- **Behavior:** Cannot be permanently dismissed — re-triggers after every additional 0.5 buy-in lost.

---

### 5.4 One-Click Gut Check

**Description:** A quick emotional self-assessment widget. The player taps one button at any point during a session to log their mental state.

**Options:**

| State | Icon | Meaning |
|---|---|---|
| Confident | 💪 | Playing well, reading the table clearly |
| Neutral | 😐 | Average focus |
| Tilted | 😤 | Frustrated, prone to emotional plays |
| Tired | 😴 | Fatigued, reaction time and focus slipping |
| Bored | 😑 | Distracted, playing too many marginal hands |

**Behavior:**
- Gut Check timestamps are stored and correlated with hand outcomes in the session database.
- The Post-Game Report Card uses this data to surface insights like: *"You played 12 hands while Tilted and lost 1.8 buy-ins during that stretch."*
- Stop-Loss / Stop-Win alerts incorporate the current Gut Check state in their recommendations (see §5.3).

---

### 5.5 Visual Player Heatmaps

**Description:** Instead of raw numbers, player tendencies are rendered as intuitive visual icons.

**Icon System:**

| Tendency | Icon | Threshold |
|---|---|---|
| Ultra-aggressive | 🔥🔥🔥 Three flames | VPIP > 60 or AF > 5 |
| Aggressive | 🔥🔥 Two flames | VPIP 40–59 or AF 3–5 |
| Moderate | 🔥 One flame | VPIP 25–39 |
| Tight-aggressive (TAG) | 🎯 Target | VPIP 15–24, PFR/VPIP > 0.6 |
| Tight-passive (Rock) | 🪨 Rock | VPIP < 15 |
| Calling station | 📞 Phone | VPIP > 35, AF < 1.5 |

**UX Note:** Raw numbers remain accessible as a tooltip on hover for players who want the data. The icon is the default display.

---

### 5.6 AI Pro Feedback Engine

**Description:** An automated pipeline that analyzes completed hands and provides GTO-informed coaching feedback.

#### Pipeline

```
1. Log Parser       →  Detects hand completion in .csv
2. Hand Builder     →  Reconstructs hand into readable narrative
3. Context Injector →  Appends session context (stack sizes, position, villain stats)
4. LLM Call         →  Sends prompt to GPT-4o with GTO coach system prompt
5. Response Render  →  Displays coaching note in the UI, linked to hand history
```

#### Hand Narrative Format (input to AI)

```
Hand #42 — Blinds: $0.25/$0.50 — 6-Max

Positions:
  UTG:  PlayerA  ($48.00)  [VPIP: 45%, PFR: 18%]
  HJ:   PlayerB  ($62.50)  [VPIP: 22%, PFR: 19%]
  CO:   Hero     ($50.00)  [Ah Kd]
  BTN:  PlayerC  ($55.00)  [VPIP: 68%, PFR: 32%] 🔥🔥🔥
  SB:   PlayerD  ($38.00)  [VPIP: 15%, PFR: 14%]
  BB:   PlayerE  ($50.00)  [VPIP: 30%, PFR: 10%]

Pre-Flop:
  UTG folds. HJ folds.
  Hero (CO) raises to $1.50.
  BTN calls $1.50. SB folds. BB folds.
  Pot: $3.50

Flop: [As 7h 2c]
  Hero bets $2.00 (57% pot).
  BTN raises to $6.00.
  Hero calls $4.00.
  Pot: $15.50

Turn: [Ks]
  Hero checks.
  BTN bets $10.00 (65% pot).
  Hero raises all-in to $42.50.
  BTN folds.
  Pot: $35.50

Result: Hero wins $35.50.
```

#### Trigger Conditions

- Analysis is triggered automatically after every hand where Hero was dealt in.
- Analysis can also be manually requested via a "Analyze This Hand" button in the hand history log.

#### Latency Target
- AI response displayed within **8 seconds** of hand completion.
- If analysis is delayed, a loading spinner with "Thinking…" is shown.

---

### 5.7 Post-Game Report Card

**Description:** An automated end-of-session summary generated by combining session statistics with AI narrative analysis.

**Sections:**

#### Session Summary
```
Session: Friday Night Game  ·  Duration: 3h 22m
Hands Played: 87  ·  VPIP: 24%  ·  PFR: 18%

Result: +$68.50  (+1.37 buy-ins)
```

#### Key Stats vs. GTO Benchmarks

| Stat | Your Session | GTO Baseline | Assessment |
|---|---|---|---|
| VPIP | 24% | 22–28% | ✅ Good range |
| PFR | 18% | 16–22% | ✅ Balanced |
| 3-Bet% | 4% | 6–9% | ⚠️ Under 3-betting |
| WTSD | 32% | 28–35% | ✅ Normal |
| Fold to 3-Bet | 72% | 55–65% | ⚠️ Over-folding to 3-bets |

#### Biggest Leaks (AI-generated)
> *"Your biggest leak tonight was folding too often to 3-bets out of position. You folded 72% of the time — GTO suggests 55–65%. Consider defending with suited broadways and pocket pairs more often."*

#### Emotional Correlation
> *"You logged 'Tilted' at 10:45 PM after a bad beat. In the 18 hands that followed, you posted a net loss of $22. Consider implementing a 15-minute break rule after significant losses."*

#### Recommended Focus for Next Session
1. Defend 3-bets more aggressively from the BTN and CO.
2. Implement your stop-loss at -2 buy-ins.
3. Practice identifying calling stations early and value-betting thinner against them.

---

## 6. AI System Prompt Template

The following system prompt is used for the AI Pro Feedback Engine (§5.6).

```
SYSTEM PROMPT
─────────────────────────────────────────────────────────────────────
You are an expert micro-stakes poker coach specializing in
Game Theory Optimal (GTO) strategy for 6-max cash games.

Your role is to analyze completed poker hands and provide clear,
actionable feedback. Your student is a recreational player whose
primary goal is consistent session profitability at micro-stakes
($0.25/$0.50 to $1/$2).

When analyzing a hand:
1. Briefly validate or critique the pre-flop action (2 sentences max).
2. Analyze the most important street decision in 3–4 sentences.
3. Identify the single biggest mistake (if any) or confirm the
   best line was taken.
4. Provide one specific, actionable improvement suggestion.
5. End with a confidence score: "Line Quality: X/10"

Tone: Honest but encouraging. Avoid jargon beyond VPIP, PFR, and
pot odds. Never use solver percentages unless asked.

Important constraints:
- Do NOT invent equity percentages — use only general ranges
  (e.g., "roughly a coin flip," "a significant favorite").
- If you are unsure about the correct play, say so explicitly.
- Keep responses under 200 words.
─────────────────────────────────────────────────────────────────────
```

---

## 7. Equity Calculator Integration

### Phase 1 (MVP): AI-Only Analysis

The initial version relies solely on the LLM for hand analysis. To mitigate the risk of hallucinated poker math:

- The system prompt explicitly prohibits the AI from quoting specific equity percentages.
- AI output is labeled with a disclaimer: *"AI feedback is strategy-oriented. Exact equities are not verified."*

### Phase 2: Equity Calculator API

Integrate a dedicated equity calculator API (e.g., [Peal](https://peal.io) or [Calamari](https://calamari.io)) to provide mathematically verified equity figures.

**Integration Points:**
- Equity is calculated at each decision point in the hand reconstruction.
- Results are injected into the hand narrative before the AI call: `"Hero equity on the flop: ~64% vs. BTN's likely range."`
- Equity figures shown in the UI are sourced from the calculator, not the AI.

**Benefits:**
- Eliminates risk of AI "hallucinating" poker math.
- Enables features like "Hero had 78% equity when they bet the river" in the Report Card.
- Required before any commercial product launch.

---

## 8. User Experience Principles

1. **Zero configuration for casual users.** The app auto-detects the PokerNow log folder on first launch.
2. **Numbers are a last resort.** Visual icons, colors, and plain language come first; raw stats are one tap away.
3. **Respect the game flow.** No disruptive popups during a hand. All AI analysis appears between hands.
4. **Privacy first.** No hand data leaves the user's machine except for AI analysis calls (opt-in).
5. **One-click install.** Distributed as a signed `.dmg` (macOS) and `.exe` installer (Windows).

---

## 9. Non-Functional Requirements

| Category | Requirement |
|---|---|
| Performance | HUD overlay renders at ≥ 60 fps with < 3% CPU overhead |
| Latency | Log file changes reflected in HUD within 1 second |
| AI latency | AI feedback displayed within 8 seconds of hand end |
| Reliability | App recovers gracefully from log file gaps or corrupted rows |
| Offline support | All HUD and stat features work fully offline; AI features require internet |
| Security | OpenAI API key stored in OS keychain, never in plain text config |
| Platform | macOS 12+ and Windows 10+ at launch |
| Accessibility | All colors supplemented with text or icon labels (colorblind-safe) |

---

## 10. Out of Scope (v1)

- Live multi-table tournament (MTT) support.
- Mobile or tablet companion app.
- Hand history import from sites other than PokerNow.
- Multiplayer / shared session views.
- Real-money payment processing or subscription billing.
- Solver integrations (GTO+, PioSOLVER).

---

## 11. Open Questions

| # | Question | Owner | Status |
|---|---|---|---|
| 1 | Which equity calculator API (Peal vs. Calamari) offers the best free tier for an MVP? | Engineering | Open |
| 2 | Should the Gut Check widget auto-prompt on a timer, or only on user request? | Design | Open |
| 3 | What is the minimum hand sample size before HUD stats are shown at all? | Product | Open |
| 4 | Do we need a PokerNow-specific ToS review before distributing publicly? | Legal | Open |
| 5 | Should AI analysis be opt-in per session, or always-on with an API key set? | Product | Open |
