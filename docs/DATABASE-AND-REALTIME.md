# Database Schema, Realtime Sync & Security Architecture

Defines the Supabase PostgreSQL schema, Row Level Security (RLS) anti-cheat boundaries, Realtime channels, and 30-minute TTL cleanup for **Mini-Party-Games**.

---

## 1. Core Tables

### 1.1 `lobbies`
| Column | Type | Notes |
| :--- | :--- | :--- |
| `id` | `uuid` PK | Default `gen_random_uuid()` |
| `room_code` | `text` UNIQUE | 6-char uppercase alphanumeric (e.g., `K9X4P2`) |
| `game_type` | `text` | `'whos_the_murderer'` \| `'connect_dem_words'` |
| `mode` | `text` | `'automatic'` \| `'manual'` |
| `status` | `text` | `'waiting'` \| `'in_progress'` \| `'finished'` |
| `host_player_id` | `uuid` NULL | References `lobby_players.id` |
| `config` | `jsonb` | Game-specific settings (role counts, timers, kill mode) |
| `last_activity_at` | `timestamptz` | Updated on every mutating action; used for 30m TTL |
| `created_at` | `timestamptz` | Default `now()` |

### 1.2 `lobby_players`
| Column | Type | Notes |
| :--- | :--- | :--- |
| `id` | `uuid` PK | Default `gen_random_uuid()` |
| `lobby_id` | `uuid` FK | References `lobbies.id` ON DELETE CASCADE |
| `nickname` | `text` | Unique per `(lobby_id, lower(nickname))` |
| `session_token_hash` | `text` | Ephemeral per-tab token hash (never exposed to clients) |
| `participant_role` | `text` | `'player'` \| `'viewer'` \| `'admin'` |
| `is_host` | `boolean` | Default `false` |
| `is_connected` | `boolean` | Synced with Realtime Presence |
| `joined_at` | `timestamptz` | Used for deterministic host transfer (`ORDER BY joined_at ASC`) |

---

## 2. Game 1 Tables (*Who's the Murderer?*)

### 2.1 `murder_games`
| Column | Type | Notes |
| :--- | :--- | :--- |
| `lobby_id` | `uuid` PK/FK | References `lobbies.id` ON DELETE CASCADE |
| `day_number` | `integer` | Starts at `1` |
| `phase` | `text` | `'role_reveal'` \| `'day_discussion'` \| `'transition'` \| `'day_voting'` \| `'night_prep'` \| `'night_murderer'` \| `'night_doctor'` \| `'night_sheriff'` \| `'night_detective'` \| `'night_resolution'` \| `'game_over'` |
| `next_phase` | `text` NULL | Target phase after `10s` `'transition'` |
| `phase_ends_at` | `timestamptz` NULL | Authoritative server deadline |
| `winner` | `text` NULL | `'civilians'` \| `'murderers'` \| `'tie'` |

### 2.2 `murder_player_states`
| Column | Type | Notes |
| :--- | :--- | :--- |
| `player_id` | `uuid` PK/FK | References `lobby_players.id` ON DELETE CASCADE (or manual-mode slot UUID) |
| `lobby_id` | `uuid` FK | References `lobbies.id` ON DELETE CASCADE |
| `display_name` | `text` | Player name (used in both Auto and Manual modes) |
| `secret_role` | `text` | `'villager'` \| `'murderer'` \| `'doctor'` \| `'sheriff'` \| `'detective'` |
| `is_alive` | `boolean` | Default `true` |
| `last_healed_target_id` | `uuid` NULL | Enforces Doctor "no 2 nights in a row" rule |

### 2.3 `murder_night_actions` & `murder_votes`
- `murder_night_actions`: `(id, lobby_id, day_number, actor_player_id, action_type ['kill'|'heal'|'shoot'|'investigate'], target_player_id, result ['good'|'bad'|null], created_at)`
- `murder_votes`: `(id, lobby_id, day_number, voter_player_id, target_player_id NULL, created_at)` — unique per `(lobby_id, day_number, voter_player_id)`.

---

## 3. Game 2 Tables (*Connect dem Words*)

### 3.1 `connect_games`
| Column | Type | Notes |
| :--- | :--- | :--- |
| `lobby_id` | `uuid` PK/FK | References `lobbies.id` ON DELETE CASCADE |
| `current_round` | `integer` | `0` (setup) to `5` |
| `phase` | `text` | `'word_setup'` \| `'round_active'` \| `'round_judging'` \| `'game_over'` |
| `round_ends_at` | `timestamptz` NULL | Authoritative `60s` round deadline |
| `score` | `integer` | Rounds won (`0..5`) |

### 3.2 `connect_word_submissions`
- `(id, lobby_id, player_id, words jsonb)` — stores each player's 5 strings.

### 3.3 `connect_rounds`
- `(lobby_id, round_number [1..5], word_a, word_b, system_rule, viewer_rule NULL, outcome ['pending'|'won'|'lost'])`

### 3.4 `connect_viewer_rules`
- `(id, lobby_id, viewer_player_id, target_round, rule_text, created_at)`
- Unique constraint on `(lobby_id, viewer_player_id)` enforcing **max 1 rule per viewer**.

---

## 4. Security & Anti-Cheat (Secret Role Protection)

- **Strict RLS**: Clients **never** query `murder_player_states` or `murder_night_actions` directly via Supabase client SDK.
- **Server-Filtered State Endpoint (`GET /api/lobbies/[code]/state`)**:
  - Reads the caller's ephemeral HTTP-only session cookie (`session_token`).
  - Returns **only** what that caller is allowed to see:
    - Alive `Villager`/`Doctor`/`Sheriff`/`Detective`: own `secret_role` only; other players' roles are `null`.
    - Alive `Murderer`: own `secret_role` + fellow `Murderer` IDs; non-murderer roles are `null`.
    - **Dead Player** (`is_alive === false`) or `phase === 'game_over'`: all players' `secret_role` values are revealed.
    - **Detective investigation result**: returned only to the Detective who performed it.
- **Realtime Broadcast**: Broadcasts only **public** phase transitions (`phase_changed`, `player_eliminated`, `timer_sync`, `round_started`). Clients then fetch their personalized state from `/api/lobbies/[code]/state`.

---

## 5. Realtime Channels & TTL Cleanup

- **Channel `lobby:{room_code}`**:
  - **Presence**: Tracks connected `player_id`s to detect disconnects, prevent active-nickname hijacking, trigger host transfer, and detect empty lobbies.
  - **Broadcast**: Sends lightweight events (`lobby_updated`, `phase_changed`, `murderer_consensus_pick`, `viewer_rule_submitted`).
- **30-Minute Inactivity & Empty-Lobby Wipe**:
  1. **Supabase `pg_cron`** (every 5 minutes): `DELETE FROM lobbies WHERE last_activity_at < NOW() - INTERVAL '30 minutes';`
  2. **Lazy Expiration**: Every API route checks `last_activity_at` before serving a lobby; if expired, deletes it and returns `404`.
  3. **Empty Lobby Wipe**: When Presence/Leave reduces connected players to `0` (after a `15s` grace period for page reloads), the lobby is deleted.

