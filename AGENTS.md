# AGENTS.md — Called Bluff

Called Bluff is a 2–4 player browser game: chess with hidden piece identities plus poker-style bluff calling.
Every piece is face-down to opponents. Every move is a claim ("this is a Bishop"). Any claim can be challenged with chips.

This file is the source of truth for rules and architecture. If code and this file disagree, this file wins.
If a rule is ambiguous, choose the simplest reading, implement it, and note it in `engine/RULES_NOTES.md`.

---

## 1. Repository layout

npm workspaces monorepo, TypeScript strict mode everywhere.

```
/engine   pure rules engine + bots (no DOM, no Node APIs, no dependencies). Vitest tests.
/client   Vite + React + TypeScript. Runs "Play vs bots" fully in-browser using /engine.
/server   Node + Express + Socket.IO. Online rooms. Holds the only full game state.
```

Commands that must work from the repo root: `npm install`, `npm test`, `npm run dev` (client + server), `npm run build`.
All tests must pass before a task is considered done.

---

## 2. Board

- 10×10 grid, coordinates (x, y), 0–9. (0,0) is bottom-left from South's view.
- The four 2×2 corners are removed: a square is OFF the board if (x < 2 or x > 7) and (y < 2 or y > 7). 84 playable squares.
- Home zones (6 wide × 2 deep, 12 squares each):
  - South: y ∈ {0,1}, x ∈ 2..7
  - North: y ∈ {8,9}, x ∈ 2..7
  - West:  x ∈ {0,1}, y ∈ 2..7
  - East:  x ∈ {8,9}, y ∈ 2..7
- Seats: S, W, N, E. Turn order is clockwise: S → W → N → E. South moves first.
- 2 players: S and N. 3 players: S, W, N (East arm empty). 4 players: all.

### Players, seats and teams (2–8 players)

The game supports 2–8 humans, each on their own device, joining by room code. The board always has at most 4 seats;
extra players become partners on a seat.

| Humans | Seats |
|---|---|
| 2–4 | one human per seat; empty seats may be filled by bots |
| 5–8 | 4 seats; seats are filled first with one human each, then a second human is added to seats in order S, W, N, E |

Team seat rules (a seat with two humans):
- Both partners see the seat's true piece types and share the chip stack.
- Partners alternate who controls the seat's MOVE each time it is that seat's turn (the "driver" badge shows whose turn it is).
- Either partner may press **Bluff!** during an opponent's challenge window, and either partner may respond (Show / Fold / Raise / Call / Back down) — the first valid action wins.
- Partners get quick pings visible only to their team: "Bluff it", "Play honest", "Call it", "Let it go".
- The engine is unchanged: it still sees one seat per arm. Teams are a server/client concept mapping several sockets to one seat.

Host lobby: shows all joined players, lets the host drag players between seats, and auto-balances (solo seats first, then pairs).
The game must be fully playable with 8 humans on 8 devices.

## 3. Pieces

Each player has 8 pieces: K, Q, R, B, N, P, P, P. This composition is public.

Movement (standard chess geometry):
- K: 1 square any direction.
- Q: any distance orthogonally or diagonally.
- R: any distance orthogonally.
- B: any distance diagonally.
- N: L-shape, jumps over pieces.
- P: 1 square "forward"; captures 1 square diagonally forward. Forward = S: +y, N: −y, W: +x, E: −x.
- Sliding pieces stop at the first occupied square (capturing it if it is an enemy) and cannot pass through off-board squares.
- No double pawn step, castling, en passant, check/checkmate, or promotion.
- A pawn that cannot move forward (edge or blocked) simply has no pawn-claim moves.

## 4. Setup

- Every player starts with 20 chips.
- Each player secretly places their 8 pieces on any 8 of their 12 home squares. "Quick setup" places them randomly.
- Opponents see only blank tokens in that player's colour.

## 5. Turn: move and claim

1. Pick one of your pieces and declare a type (K, Q, R, B, N, P).
2. The move must be legal for the DECLARED type. The true type can be anything.
3. You cannot declare a type that is publicly gone: if all of your pieces of that type have been revealed (by challenge or capture) and removed, or are revealed and still on board elsewhere, the claim is blocked. Concretely: a claim of type T for piece p is blocked if (number of your pieces publicly known to be T, excluding p) ≥ (your starting count of T).
4. A revealed piece may only claim its true type.
5. The claim is appended to the piece's public `claims` history.
6. If the destination holds an enemy piece, it is captured provisionally (see §7).

If a player has no legal move under any claim, they pass and pay 1 chip.

## 6. Challenges

After each declared move, a challenge window opens for 8 seconds.
- Only alive opponents of the mover may challenge. Only one challenge per move: the first `Bluff!` wins.
- The challenger picks an opening stake from 2 to 6, capped at min(challenger chips, mover chips).
- If nobody challenges, the move stands.

Mover responds with one of:
- **Show**: the piece is revealed (`revealed = true`, stays revealed forever).
  - Claim TRUE: challenger pays stake to mover. Move stands.
  - Claim FALSE: mover pays stake to challenger. The piece is BUSTED (removed). The move is undone; any captured piece is restored.
- **Fold**: mover pays ceil(stake / 2) to challenger. Move is undone (captured piece restored). Piece stays hidden. Turn ends.
- **Raise** (once per challenge): stake becomes min(stake × 2, mover chips, challenger chips). Challenger then chooses:
  - **Call**: resolves exactly as Show at the raised stake.
  - **Back down**: challenger pays the ORIGINAL stake to mover. Move stands. Piece stays hidden.

Payments: if a player cannot cover a payment, they pay everything they have and are eliminated (§8).

## 7. Captures

- Moving onto an enemy piece captures it whatever its true type. The captured piece is revealed to all and removed once the move stands (no challenge, or challenge resolved with the move standing).
- If the move is undone (bust or fold), the captured piece returns to its square unrevealed.
- Capturing a King eliminates its owner; the capturer receives that player's entire chip stack.
- A busted King (the mover's own King caught lying) eliminates the mover; the challenger receives the mover's stack.

## 8. Elimination and winning

- Eliminated when: King captured, King busted, or chips reach 0.
- Eliminated players' pieces are removed from the board. They cannot challenge and are skipped in turn order.
- Last player standing wins.
- Turn cap: after 60 total turns (a turn = one player's move attempt, including passes and folds), the player with the most chips wins; tie-break = most pieces on board; then earliest seat in turn order.

## 9. Config (engine/config.ts)

All balance numbers live here, never inline in rule code:

```ts
export const CONFIG = {
  startingChips: 20,
  minStake: 2,
  maxOpeningStake: 6,
  foldCostFraction: 0.5,   // rounded up
  raiseMultiplier: 2,
  challengeWindowMs: 8000,
  relaxedChallengeWindowMs: 20000,
  turnTimerMs: 45000,
  disconnectGraceMs: 30000,
  turnCap: 60,
  passPenalty: 1,
};
```

---

## 10. Engine API

Pure, deterministic functions. Use a seeded RNG stored in state (e.g. mulberry32). No `Math.random` in /engine.

```ts
type Seat = 'S' | 'W' | 'N' | 'E';
type PieceType = 'K' | 'Q' | 'R' | 'B' | 'N' | 'P';

interface Piece { id: string; owner: Seat; type: PieceType; x: number; y: number; revealed: boolean; alive: boolean; claims: PieceType[]; }
interface Player { seat: Seat; name: string; chips: number; alive: boolean; isBot: boolean; botStyle?: 'sentinel' | 'joker' | 'shark'; }

interface PendingMove { pieceId: string; claim: PieceType; from: [number, number]; to: [number, number]; capturedId?: string; }

type Phase =
  | { kind: 'setup'; submitted: Seat[] }
  | { kind: 'move'; seat: Seat }
  | { kind: 'challengeWindow'; move: PendingMove; endsAt: number }
  | { kind: 'moverResponse'; move: PendingMove; challenger: Seat; stake: number }
  | { kind: 'challengerResponse'; move: PendingMove; challenger: Seat; stake: number; originalStake: number }
  | { kind: 'gameOver'; winner: Seat; reason: 'lastStanding' | 'turnCap' };

interface GameState { players: Player[]; pieces: Piece[]; phase: Phase; turn: number; log: LogEntry[]; rngState: number; }
```

Functions:
- `createGame(config: { seats: Seat[]; names: Record<Seat,string>; bots: Partial<Record<Seat, BotStyle>>; seed: number }): GameState`
- `legalClaims(state, pieceId): { claim: PieceType; destinations: [number,number][] }[]`
- `applyAction(state, action): GameState` — actions: `setup`, `quickSetup`, `declareMove`, `pass`, `challenge`, `noChallenge` (window expired), `show`, `fold`, `raise`, `call`, `backDown`, `timeout`.
- `viewFor(state, seat | 'spectator'): PublicState` — redacts `type` to `null` for every piece not owned by `seat` and not `revealed`. Spectators see no hidden types.
- Invalid actions throw a typed `RuleError` with a human-readable message.

`LogEntry` carries a plain-English `text` (e.g. "Amber moved a 'Bishop' c3→f6", "BUSTED — it was a Pawn. Kai +4") and an optional animation cue (`reveal`, `bust`, `chips`, `capture`, `eliminate`).

Tests (Vitest): every rule in §§2–8 and every edge case, including: blocked claims, pawn direction per seat, sliding through removed corners, capture restoration on bust and fold, raise caps, back-down payment, King bust elimination, stack transfer on King capture, bankruptcy mid-payment, pass penalty, turn cap and tie-breaks, and that `viewFor` never exposes hidden types.

---

## 11. Bots (engine/bots)

One engine, three personalities. `chooseMove(view, rng)`, `chooseChallenge(view, rng)`, `chooseResponse(view, rng)` — bots receive ONLY their own `viewFor` plus their own true piece types (no cheating).

| Bot | Bluff rate | Challenge rule | Raise when bluffing |
|---|---|---|---|
| sentinel | 0.10 | P(lie) > 0.65 | 0.15 |
| joker | 0.45 | P(lie) > 0.35 | 0.60 |
| shark | 0.25 | EV rule (below) | 0.40, +0.15 if chips ≥ 30 |

Move choice: enumerate all (piece, claim, destination). Score = captured value (K 100, Q 9, R 5, B 3, N 3, P 1; unknown hidden enemy = 3) − own true-piece value if the destination is attackable by any enemy claim + 0.2 × progress toward centre + noise(±0.5). An option is a bluff if claim ≠ true type; bluffs are only eligible with probability = bluff rate, and get +2 if they unlock a capture. The King never moves more than 1 square.

P(lie) estimate for an opponent's claim of type T:
- base = 0.25 + 0.4 × (1 − unaccountedT / hiddenCount), where unaccountedT = that player's pieces of type T not yet publicly known, hiddenCount = that player's unrevealed alive pieces.
- +0.4 if the piece's claim history already contains a different type.
- +0.1 for a Q/R/B move of 3+ squares; +0.1 if the move captures.
- clamp to [0.05, 0.95].

Shark challenges when P(lie) − (1 − P(lie)) > 0.1, with stake = round(2 + 4 × P(lie)).

Responding: honest piece → Show, except an honest King → Fold. Lying piece → Raise with raise probability, else Fold. As challenger facing a raise → Call if P(lie) > 0.5, else Back down.

Bots act after a random 0.8–2.5 s delay (client/server side, not in /engine).

Simulation script: `npm run simulate -- --games 500` plays bot-only games with fixed seeds and prints: mean turns, % moves challenged, % claims that were bluffs, % games ending by bankruptcy, win rate by seat, win rate by bot style.

---

## 12. Server (online rooms)

- The server holds the only full `GameState`. Every socket receives `viewFor(state, theirSeat)` after every change. Never emit full state.
- Clients send intents only; the server validates every action through the engine.
- Events, client → server: `room:create`, `room:join {code, name, rejoinToken?}`, `seat:setBot {seat, style|null}` (host only), `game:start` (host only), `setup:submit {placements}`, `setup:quick`, `move:declare {pieceId, claim, to}`, `move:pass`, `challenge:call {stake}`, `challenge:respond {action}`.
- Events, server → client: `state {view}`, `event {logEntry}`, `error {message}`.
- Room codes: 4 uppercase letters (no I or O). Share link: `/room/CODE`.
- Rejoin: a token stored in localStorage reclaims the same seat after refresh.
- Disconnect: after 30 s a sentinel bot plays the seat; the human can reclaim it on return.
- Timers (challenge window, 45 s turn timer, disconnect grace) run on the server; clients get `endsAt` timestamps to draw countdowns.
- Spectators join by link and get `viewFor(state, 'spectator')`.
- Teams: a seat may map to up to 2 sockets. Both receive `viewFor(state, seat)`. The server tracks the current driver per seat and rejects `move:declare` / `move:pass` from the non-driver. Team pings use `team:ping {kind}` and are delivered only to the partner.
- Room capacity: 8 players plus unlimited spectators. A 9th joiner becomes a spectator.

---

## 13. Client UX

- Landing: big **Play vs bots** and **Create room** buttons, small **How to play** link. A new user should be in a game within 10 seconds.
- My pieces: show true type, a corner badge with current claim, and a red outline when the current claim is a lie.
- Opponent pieces: blank tokens in seat colour with seat letter (S/W/N/E), labelled with last claim; tap/hover shows full claim history.
- Claim picker: select piece → 6 type buttons (blocked types disabled with a tooltip explaining why) → only legal destinations highlight.
- Challenge banner: countdown ring, stake slider (2–6), big **Bluff!** button. Mover sees Show / Fold / Raise; challenger sees Call / Back down.
- Narrated move log, animated chip transfers, TRUE (green) / BUSTED (red) reveal stamp, sound effects with visible mute toggle.
- End screen: winner, bluffs made, bluffs caught, biggest pot, Play again.
- Tutorial (5 steps vs a scripted bot): honest move → successful bluff → get challenged and Show → call a bot's bluff → Raise to scare off a challenger.
- Mobile: board on top, controls in a bottom sheet, tap targets ≥ 44 px. Colour-blind safe seat colours.
- "Take your time" toggle uses the 20 s challenge window. Bot games can be paused.

---

## 14. Non-negotiables

1. No code path may send an unrevealed opponent piece's type to a client (online mode).
2. Rules live only in /engine. Client and server never re-implement rules.
3. Balance numbers live only in `engine/config.ts`.
4. `npm test` passes at the end of every task.
