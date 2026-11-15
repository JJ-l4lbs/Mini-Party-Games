# Product Requirements Document (PRD) — Mini-Party-Games

Authoritative product rules and feature specifications for **Mini-Party-Games**.  
Updated by `ticket-writer` when new features or business-rule changes are specified.

---

## 1. Product Vision & Tech Stack

**Mini-Party-Games** is a zero-friction, session-based multiplayer party game web application designed for friends playing together in person or over voice/video chat.

- **Framework**: Next.js 16 (App Router) + React 19 + TypeScript (strict mode)
- **Styling**: Tailwind CSS v4 with Stitch Design System Tokens (semantic tokens, zero raw hex/rgba in `.tsx`)
- **Backend & Realtime**: Supabase (PostgreSQL, Row Level Security, Realtime Presence & Broadcast)
- **Hosting**: Vercel
- **Accounts / Profiles**: None. All player state is ephemeral.

---

## 2. Core Platform & Lobby Rules

### 2.1 Landing Page
- Displays a **Main Menu** of available minigames:
  1. **Who's the Murderer?**
  2. **Connect dem Words**
- Clicking a minigame navigates to that minigame's **Create / Lobby Setup** screen.
- Includes a global **Join a Lobby** action directly on the landing page (not tied to a specific minigame).

### 2.2 Ephemeral Identity & Rejoin Flow
- **No Profiles**: Players do not create accounts. Reloading the browser wipes local client memory.
- **Lobby Discovery**: Room Code entry only (no public lobby browser).
- **Rejoin Mechanism (`Room Code + Nickname`)**:
  - When joining a lobby, a player enters the **Room Code** and a **Nickname** (unique within that lobby).
  - If a game is **already in play**, new players cannot join as new participants (**Reconnection Only**).
  - Entering an existing `Room Code + Nickname` of a disconnected/reloaded player restores that player's exact slot, role, and game state.
  - While a player is actively connected (tracked via Supabase Realtime Presence), another client cannot hijack their active `Nickname`.

### 2.3 Host Control & Lobby Lifecycle
- **Host Control Transfer**: If the lobby host disconnects or leaves in an active multi-device session, host authority automatically transfers to the next longest-tenured connected player.
- **Automatic Cleanup (TTL)**:
  - A lobby is **immediately wiped** if all players leave and no connected players remain.
  - A lobby is **wiped clean after 30 minutes of inactivity** (no player actions or active game progression).

---

## 3. Game 1: Who's the Murderer?

A social deduction murder mystery game supporting **Automatic Mode** (multi-device session) and **Manual Mode** (single-device pass-and-play with an Admin moderator).

### 3.1 Roles
Each player is randomly assigned one of five roles:
1. **Villager** (Good): No night action. Participates in daytime discussion and voting.
2. **Murderer** (Bad): Sees fellow Murderers. Eliminates target(s) during the Night phase.
3. **Doctor** (Good): Protects one player per Night from being killed.
   - **Self-heal**: Allowed.
   - **Consecutive target**: Cannot heal the same player two nights in a row.
4. **Sheriff** (Good): **Shoot / Eliminate only** during the Night phase (chooses one player to eliminate; no investigation ability).
5. **Detective** (Good): **Investigates** one player per Night to learn whether their identity is **Good** (Villager, Doctor, Sheriff, Detective) or **Bad** (Murderer).

### 3.2 Lobby Configuration
- **Mode**: `Automatic` (multi-device) or `Manual` (single-device).
- **Role Counts** (selectable before start):
  - Number of **Murderers** (`>= 1`)
  - Number of **Doctors** (`>= 0`)
  - Number of **Sheriffs** (`>= 0`)
  - Number of **Detectives** (`>= 0`)
  - Remaining players are assigned **Villager**.
- **Multi-Murderer Kill Rule** (selectable toggle when Murderers > 1):
  - `1_kill_per_murderer`: Each Murderer independently selects 1 target per night.
  - `consensus`: All Murderers must agree on 1 shared target per night.
- **Timers** (configurable in Automatic Mode; defaults shown):
  - Day Discussion: `60s`
  - Day Voting: configurable (e.g., `30s`)
  - Prepare for Night ("Close your eyes"): `20s`
  - Night Action per role: `60s`
  - Inter-phase Transition: `10s` between every phase.

### 3.3 Gameplay Loop (Automatic Mode)
1. **Role Reveal**: Each player privately views their own role. Murderers also see who the other Murderers are.
2. **Day 1**:
   - **Discussion (`60s` default)**: Players talk freely.
   - **No Voting on Day 1**: Voting is skipped on Day 1; proceeds directly to Night prep.
3. **Night Prep (`20s` default)**: All players receive a prompt to close their eyes and prepare for night.
4. **Night Phase** (strict sequential order, with `10s` transition between each):
   1. **Murderer(s)** (`60s`): Prompted to open eyes/act and select target(s) per the configured kill rule.
   2. **Doctor** (`60s`): Selects 1 player to heal (excluding previous night's target).
   3. **Sheriff** (`60s`): Optionally selects 1 player to shoot/eliminate.
   4. **Detective** (`60s`): Selects 1 player to investigate and immediately receives `Good` or `Bad`.
5. **Dawn / Night Resolution**:
   - Night kills are resolved against Doctor heals (if healed, the target survives).
   - Eliminated players are revealed as dead and **can see all players' roles** as spectators.
   - Win/Tie conditions are checked.
6. **Day 2+**:
   - **Discussion (`60s` default)** → **`10s` transition** → **Daytime Voting**.
   - **Voting**: Alive players vote to eliminate 1 suspect. On a **tie vote**, no one is eliminated.
   - Win/Tie conditions are checked; if game continues, cycle returns to Night Prep.

### 3.4 Manual Mode (Single-Device Admin Flow)
- **Setup**: Admin selects total player count and role counts.
- **Role Distribution**: **Tap to Reveal → Hide & Pass to Next Player** privacy screens so each player views their role on the shared device without seeing others.
- **Admin Moderation**: Once all players have seen their roles, Admin starts the game.
  - The Admin is **not** a player; they verbally moderate the room.
  - Timers are **countdown guides only** (non-blocking): Admin can advance to the next phase early or stay past `0s`.
  - Admin has an **Admin Control Panel** to mark players killed/voted out so the app automatically tracks alive counts and evaluates Win/Tie conditions.

### 3.5 Win & Tie Conditions
Evaluated after every Night resolution and every Day vote:
1. **Civilians Win**: `0` Murderers remain alive.
2. **Tie**: Exactly `2` players remain alive and they are `1 Sheriff + 1 Murderer`.
3. **Continue Exception (Sheriff Alive)**:
   - If `1 Murderer + 1 Sheriff + 1 Villager/Doctor/Detective` remain (`3` players), game **continues**.
   - If `2 Murderers + 1 Sheriff + 1 Villager` remain (`4` players, equal `2 vs 2`), game **continues** because a Sheriff is still alive.
4. **Murderers Win**: `Alive Murderers >= Alive Civilians` (and none of the Sheriff Tie/Continue exceptions above apply — e.g., `1 Villager/Doctor + 1 Murderer` is a Murderer Win).

---

## 4. Game 2: Connect dem Words

A cooperative 2-player word-guessing party game with optional **Viewer** judges/rule-orchestrators.

### 4.1 Participants
- **Players**: Strictly **2 Players** working together as a team.
- **Viewers**: Unlimited (`0` to `N`) optional spectators who act as judges and rule-submitters.
- **No-Viewer Mode**: If `0` viewers are present, the 2 players self-police rule violations on a trust system and manually mark round pass/fail.

### 4.2 Setup Phase
- Before Round 1 starts, each of the 2 players enters **5 strings of characters** (`10` strings total).
- **Duplicate Rule**: Players *may* enter the same word in their initial lists, **however**, when the system generates the 5 round pairs (2 strings per round, no string repeated from the pool), **no round may pair two identical strings** (e.g., `banana + banana` is forbidden).

### 4.3 Gameplay Loop (5 Rounds)
- **Round Timer**: `60s` (`1 minute`) per round.
- **Objective**: The 2 players speak/talk to each other to clue and guess both target words for that round without violating any active rules.
  - If they guess both words within `60s` without breaking rules → **Round Won**.
  - Otherwise → **Round Lost**.
- **Rule Stacking**: Rules **accumulate** across rounds (Round 3 enforces Round 1 + Round 2 + Round 3 rules, etc.).
- **System Rules**: 1 random system rule is added at the start of **every round (Rounds 1–5)** from the built-in pool:
  1. *Must rhyme* (clues must be spoken in rhyme)
  2. *No words starting with "W"*
  3. *No yes-or-no questions*
  4. *No filler words* (e.g., "umm", "uh", "hmm", "like")
  5. *No three-or-less letter words* (e.g., "and", "a", "at", "in", "on")
  6. *No proper nouns or brand names* (built-in gap fill)
  7. *All clues must be 5 words or fewer per sentence* (built-in gap fill)
  8. *Must speak in the third person or past tense only* (built-in gap fill)
- **Viewer Rules (Rounds 2–5)**:
  - In Rounds 2–5, **1 additional rule** is randomly picked from viewer submissions (so Rounds 2–5 each add **2 rules**: 1 System + 1 Viewer).
  - If no viewer rules were submitted for that round, only the System rule is added.
  - **Viewer Submission Window**: Viewers submit a rule **1 round before** the target round starts, and submissions lock **10 seconds before the current round ends** (`<= 10s` remaining).
  - **Submission Cap**: Each viewer may submit **at most 1 rule** per game.
