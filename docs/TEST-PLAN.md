# Test Plan & QA Matrix — Mini-Party-Games

Covers **Vitest** unit/integration suites and **Playwright** multi-context E2E suites.

---

## 1. Unit & State-Machine Tests (Vitest)

### 1.1 Platform & Lobby
- Room code generation & normalization (case-insensitive lookup).
- Rejoin rules: `waiting` allows new players; `in_progress` blocks new nicknames (`403`) and allows offline `Room Code + Nickname` reclaim.
- Host transfer selects oldest connected player (`joined_at ASC`).
- 30-minute inactivity check & empty-lobby wipe.

### 1.2 Game 1: *Who's the Murderer?* (`murder-engine.ts`)
- Role randomization matches exact configured counts (`Murderers`, `Doctors`, `Sheriffs`, `Detectives`, remainder `Villagers`).
- **Day 1**: Voting is skipped; transitions directly to `NightPrep`.
- **Day 2+**: Tie vote results in `0` eliminations; majority eliminates target.
- **Doctor**: Self-heal succeeds; healing same target 2 nights in a row is rejected (`400`).
- **Sheriff**: Shoot/eliminate removes target (unless healed by Doctor).
- **Detective**: Returns `'bad'` for `Murderer`, `'good'` for `Villager | Doctor | Sheriff | Detective`.
- **Multi-Murderer**: `1_kill_per_murderer` vs `consensus` resolution.
- **Win/Tie Conditions**:
  - `0 Murderers` → `civilians` win.
  - `1 Sheriff + 1 Murderer` (`2` alive) → `tie`.
  - `1 Villager/Doctor + 1 Murderer` (`2` alive) → `murderers` win.
  - `1 Murderer + 1 Sheriff + 1 Villager` (`3` alive) → `continue`.
  - `2 Murderers + 1 Sheriff + 1 Villager` (`4` alive) → `continue`.
- **Secret Filtering**: Alive non-murderers receive `null` for others' roles; dead players receive all roles.

### 1.3 Game 2: *Connect dem Words* (`connect-engine.ts`)
- Pair-deduplication: Even when Player 1 and Player 2 submit overlapping words (e.g., `"banana"` in both lists), generated 5 rounds never pair `"banana"` with `"banana"` and never reuse a string instance.
- Rule stacking: Round `N` active rules list includes all rules from Rounds `1..N`.
- Viewer rule lockout: Rejects submissions when `remainingSeconds <= 10`; enforces max `1` rule per viewer.

---

## 2. End-to-End Tests (Playwright — Desktop Chrome)

All E2E tests run with `{ ...devices['Desktop Chrome'] }` using multiple isolated `BrowserContext` instances to simulate concurrent players.

1. **Lobby & Rejoin Journey**:
   - Host creates lobby → Player 2 joins via Landing Page `Room Code + Nickname` → Player 2 reloads page (state wiped) → Player 2 rejoins with same `Room Code + Nickname` and recovers role → Host closes tab → Host badge transfers to Player 2.
2. **Who's the Murderer? (Automatic Mode)**:
   - 5 browser contexts join, start game, verify role privacy (Villager cannot see Murderer; Murderers see each other), complete Day 1 (no vote) → Night (Murderer → Doctor → Sheriff → Detective) → Dawn resolution → Dead player sees all roles.
3. **Who's the Murderer? (Manual Mode)**:
   - Single context creates Manual lobby → cycles through Tap-to-Reveal → Hide & Pass screens → Admin advances phases early/late and marks eliminations until Win/Tie triggers.
4. **Connect dem Words**:
   - 2 Player contexts + 1 Viewer context → Players submit 5 strings → Round 1 starts with 1 System Rule → Viewer submits rule before `10s` lockout (verified locked at `<= 10s`) → Round 2 stacks System + Viewer rules → completes 5 rounds.
   - 2 Player contexts with `0` Viewers → verifies No-Viewer trust mode controls.

