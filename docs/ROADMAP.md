# Project Roadmap

High-level feature roadmap for **Mini-Party-Games**.  
Use `/orchestrator` or `/ticket-writer` to turn unchecked items into implementation-ready tickets in `docs/tickets/`.

## Planned Features

- [ ] **Milestone 1: Platform Core, Database & Ephemeral Lobby System**
  - [ ] Initialize Next.js 16 + Tailwind CSS v4 (Stitch tokens) + Supabase schema & RLS
  - [ ] Landing page (Minigame selector + Room Code Join)
  - [ ] Ephemeral `Room Code + Nickname` join/rejoin, host transfer, and 30-minute TTL cleanup
- [ ] **Milestone 2: Game 1 — Who's the Murderer? (Automatic Mode)**
  - [ ] Lobby configuration (role counts, multi-murderer kill mode, timers)
  - [ ] Private role reveal & server-filtered Realtime state sync
  - [ ] Day 1 (talk only) & Day 2+ (talk + voting with tie = no elimination)
  - [ ] Night phase sequence (Murderer → Doctor → Sheriff → Detective) & Dawn resolution
  - [ ] Win/Tie condition engine & dead-player full-role spectator view
- [ ] **Milestone 3: Game 1 — Who's the Murderer? (Manual Mode)**
  - [ ] Single-device Tap-to-Reveal → Hide & Pass role distribution
  - [ ] Non-blocking Admin countdown timers & Admin Control Panel (mark killed/voted out + auto win-check)
- [ ] **Milestone 4: Game 2 — Connect dem Words**
  - [ ] 2-Player + Viewer lobby & 5-string setup with pair-deduplication algorithm
  - [ ] 5-Round 60s gameplay loop with stacked System Rules (8+ built-in rules)
  - [ ] Viewer rule submission (1 per viewer, locked 10s before round end) & Viewer judging / No-Viewer trust mode
- [ ] **Milestone 5: QA, Accessibility & Production Hardening**
  - [ ] WCAG 2.2 AA audit & responsive mobile/desktop polish
  - [ ] Vitest unit/state-machine suite & Playwright multi-context E2E suite
