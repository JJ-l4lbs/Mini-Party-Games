# Backend Architecture — Mini-Party-Games

Defines the Next.js 16 Route Handlers, Zod validation boundaries, authoritative game engines, and error contracts.

---

## 1. Service & Validation Layer (`lib/` & `services/`)

```text
lib/
├── supabase/
│   ├── server.ts           # Service-role Supabase client (never imported in client code)
│   └── client.ts           # Anon client for Realtime Presence/Broadcast only
├── validations/
│   ├── lobby.schema.ts     # Room code, nickname, config Zod schemas
│   ├── murder.schema.ts    # Role counts, night actions, votes, manual-mode updates
│   └── connect.schema.ts   # 5-string setup, viewer rule submission, round outcome
services/
├── lobby.service.ts        # Create, join/rejoin, host transfer, 30m TTL cleanup
├── murder-engine.ts        # Pure state machine: role assignment, night resolution, win/tie check
└── connect-engine.ts       # Pure state machine: pair deduplication, rule stacking, 10s lockout
```

---

## 2. API Route Handlers (`app/api/`)

Every mutating endpoint validates payloads with `zod.safeParse()` (returning `400` with structured field errors on failure), verifies the caller's ephemeral `session_token` cookie, and updates `lobbies.last_activity_at = NOW()`.

### 2.1 Lobby Endpoints
- `POST /api/lobbies` — Create a new lobby for `whos_the_murderer` or `connect_dem_words`, generate unique 6-char `room_code`, set creator as host.
- `POST /api/lobbies/join` — Join or rejoin via `{ roomCode, nickname }`. Enforces `Reconnection Only` when `status === 'in_progress'`, prevents hijacking active nicknames, and issues `session_token` cookie.
- `POST /api/lobbies/[code]/leave` — Marks player disconnected; transfers host if host left; deletes lobby if 0 connected players remain.
- `POST /api/lobbies/[code]/transfer-host` — Triggered on Presence disconnect to promote oldest connected player.
- `GET /api/lobbies/[code]/state` — Returns caller-filtered lobby & game state (hiding secret roles per `DATABASE-AND-REALTIME.md`).

### 2.2 Game 1: *Who's the Murderer?* Endpoints
- `POST /api/games/murder/[code]/start` — Host only. Validates role counts `<= totalPlayers`, randomizes roles, initializes `murder_games` (`role_reveal` or Manual Mode reveal).
- `POST /api/games/murder/[code]/advance` — Advances phase (host/timer-expiry in Auto Mode; Admin anytime in Manual Mode). Handles `10s` transitions and skips absent Night roles.
- `POST /api/games/murder/[code]/night-action` — Validates caller is alive & matches current Night role (`murderer`, `doctor`, `sheriff`, `detective`). Enforces Doctor consecutive-heal rule and Murderer `consensus` vs `1_kill_per_murderer`. Returns `good`/`bad` immediately to `detective`.
- `POST /api/games/murder/[code]/vote` — Day `>= 2` voting. Records vote; tallies on completion (tie = no elimination) and runs Win/Tie check.
- `POST /api/games/murder/[code]/manual-status` — Manual Mode Admin only. Marks a player alive/dead and re-runs Win/Tie check.

### 2.3 Game 2: *Connect dem Words* Endpoints
- `POST /api/games/connect/[code]/start` — Host only. Requires exactly 2 `player` slots; transitions to `word_setup`.
- `POST /api/games/connect/[code]/words` — Player 1 or 2 submits 5 strings. When both submit, generates 5 non-duplicate pairs (`wordA !== wordB`), assigns Round 1 System Rule, and starts Round 1 (`60s`).
- `POST /api/games/connect/[code]/viewer-rule` — Viewer submits 1 rule for `current_round + 1`. Rejects with `409` if viewer already submitted or if `round_ends_at - now() <= 10s`.
- `POST /api/games/connect/[code]/round-outcome` — Records `won` or `lost` (Viewer judge or No-Viewer player trust mode), advances to next round (stacking 1 System Rule + 1 random eligible Viewer Rule) or `game_over` after Round 5.

---

## 3. Standard Error Contract

All API errors return sanitized JSON (never stack traces):
- `400 Bad Request`: Zod validation failure `{ error, fieldErrors }`
- `401 Unauthorized`: Missing/invalid ephemeral `session_token`
- `403 Forbidden`: Wrong role, dead player acting, or late joiner blocked (`Reconnection Only`)
- `404 Not Found`: Lobby missing or expired (`> 30 min` inactive)
- `409 Conflict`: Nickname already connected, duplicate viewer rule, or 10-second lockout active
- `500 Internal Server Error`: Sanitized generic message

