# Poker-HUD

A real-time heads-up display (HUD) and AI coaching tool designed specifically for [PokerNow](https://www.pokernow.club/) home games.

## Documentation

- [**PRD.md**](./PRD.md) — Full Product Requirements Document covering the tech stack, feature specifications, AI feedback engine, and roadmap.

## Quick Concept

Poker-HUD reads your live PokerNow `.csv` log file and provides:

- **Real-time opponent stats** (VPIP, PFR, AF, 3-Bet%) displayed as an overlay
- **Visual heatmaps** — intuitive flame/icon indicators instead of raw numbers
- **Session Health Meter** — track your profit vs. buy-in at a glance
- **Stop-Loss / Stop-Win alerts** — protect your bankroll and lock in winning sessions
- **One-Click Gut Check** — log your emotional state and see how it correlates with results
- **AI Pro Feedback** — automated GTO coaching analysis after every hand
- **Post-Game Report Card** — identify your biggest leaks and get an improvement plan

## Tech Stack

| Layer | Technology |
|---|---|
| Desktop Shell | Electron |
| Frontend | React + Tailwind CSS + Framer Motion |
| Backend | Python + FastAPI (local server) |
| AI Analysis | OpenAI GPT-4o |
| Database | SQLite (local) |

## Status

🚧 **Pre-development** — PRD complete, implementation in progress.

See [PRD.md](./PRD.md) for the full specification.
