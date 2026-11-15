# Frontend Architecture — Mini-Party-Games

Defines the Next.js 16 App Router structure, Server-First RSC boundaries, Tailwind CSS v4 + Stitch Design System integration, and responsive/accessibility standards.

---

## 1. Directory & Route Structure

```text
app/
├── layout.tsx                      # Root RSC layout, fonts, global theme tokens
├── page.tsx                        # Landing Page (Main Menu + Join Lobby trigger)
├── globals.css                     # Tailwind v4 @theme & @utility Stitch tokens
├── lobby/
│   ├── create/[game]/page.tsx      # Minigame-specific lobby creation
│   └── [code]/page.tsx             # Active Lobby & Game Shell (RSC + Suspense)
└── api/                            # Route Handlers (see BACKEND-ARCHITECTURE.md)

components/
├── ui/                             # Reusable Stitch primitives (Button, Card, Input, Badge, Modal, Timer)
├── common/                         # Header, ErrorBoundary, ReconnectBanner, LoadingSkeleton
├── lobby/                          # JoinLobbyForm (Room Code + Nickname), LobbyRoster, HostTransferBadge
└── games/
    ├── murder/
    │   ├── auto-game-view.tsx      # Multi-device phase views + role card
    │   ├── night-action-panel.tsx  # Role-specific Night UI (Murderer/Doctor/Sheriff/Detective)
    │   ├── day-voting-panel.tsx    # Day 2+ voting UI
    │   ├── spectator-roles-view.tsx# Dead-player full-role reveal
    │   └── manual-admin-view.tsx   # Tap-to-Reveal pass-and-play + Admin Control Panel
    └── connect-words/
        ├── word-setup-form.tsx     # 5-string input form for Players 1 & 2
        ├── round-play-view.tsx     # 60s timer, 2 target words, stacked rules list
        └── viewer-judge-panel.tsx  # Viewer rule submission (10s lockout) + Win/Loss judging
```

---

## 2. Server-First Composition & State Management

- **RSC Shell (`page.tsx`)**: Validates route params, renders `<Suspense>` boundaries, and delegates interactive gameplay to `"use client"` leaf components.
- **Code Splitting**: Heavy game views (`manual-admin-view.tsx`, `auto-game-view.tsx`, `round-play-view.tsx`) are loaded via `next/dynamic` so players only download the bundle for the game they are playing.
- **Ephemeral Client State**:
  - **Zero** `localStorage` or `sessionStorage` persistence for player identity—reloading the page returns the user to a clean state where they can rejoin via `Room Code + Nickname` (validated against the HTTP-only session cookie or reclaim flow).
  - Live game state is held in React state, synchronized via Supabase Realtime Broadcast + Presence and hydrated via `GET /api/lobbies/[code]/state`.

---

## 3. Styling & Stitch Design System Tokens

- **Tailwind CSS v4**: Configured exclusively in `app/globals.css` using `@theme` and `@utility` (no `tailwind.config.js`).
- **Semantic Tokens Only**: All `.tsx` files use semantic Tailwind classes (`bg-surface`, `text-foreground`, `border-border`, `bg-primary`, `text-danger`) merged via `cn()`. Zero raw `#hex` or `rgba()` values in `.tsx`.
- **Animations**: Pure CSS transitions and `@keyframes` (e.g., countdown progress ring, role-card flip in Manual Mode).

---

## 4. Responsive & Accessibility Standards (WCAG 2.2 AA)

- **Multi-Viewport**:
  - Mobile-first (`< 640px`) for party players on phones (large touch targets `>= 44x44px`, zero horizontal overflow).
  - Tablet/Desktop (`>= 640px`) for Admin/Viewer dashboards and split-pane rosters.
- **Pass-and-Play Privacy (Manual Mode)**:
  - Full-screen opaque privacy interstitial (*"Pass device to [Name] — Tap to Reveal"*) with zero role info in the DOM until tapped.
- **A11y**:
  - Semantic `<button type="button">`, `<main>`, `<section>`, visible `:focus-visible` rings, `cursor-pointer` on interactive controls.
  - Form validation errors linked via `aria-invalid`, `aria-describedby`, and `role="alert"`.
  - Countdown timers announce critical thresholds (e.g., `10s` lockout warning) via `aria-live="polite"`.

