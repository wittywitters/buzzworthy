# Buzzworthy design

Buzzworthy is a party quiz game. One computer (the **host**) shows a board of clues on a TV or laptop. Players join from their phones by scanning a QR code or typing a 4-letter room code, and each phone becomes a buzzer. Clues are written as answers, and players reply with the matching question.

The rule that shapes everything here: **it costs nothing to run.** No paid servers, no paid hosting, no accounts.

## How it fits together

```
 ┌──────────────── GitHub Pages (static files, free) ────────────────┐
 │   https://wittywitters.github.io/buzzworthy/                      │
 │     #/host     host screen: lobby, board, clue, scores            │
 │     #/play     phone screen: join, buzzer, wager entry            │
 │     #/edit     board editor                                       │
 └───────────────────────────────────────────────────────────────────┘
        ▲ loads page                         ▲ loads page
        │                                    │
  ┌─────┴──────┐   WebRTC data channels   ┌──┴───────────┐
  │ Host tab   │◀────────────────────────▶│ Phone tab × N │
  │ (authority)│   (direct, peer-to-peer)  └──┬───────────┘
  └─────┬──────┘                              │
        │  kept open all game: every join     │
        │  and every rejoin goes through it   │
        ▼                                     ▼
  PeerJS public broker (0.peerjs.com, free) ◀─┘
```

- **Static site.** Vite + TypeScript build to plain files. A GitHub Actions workflow (`actions/configure-pages`, `actions/upload-pages-artifact`, `actions/deploy-pages`; permissions `pages: write`, `id-token: write`) deploys `dist/` to GitHub Pages on every push to `main`. The repo is public and Pages is already set to deploy from Actions. The site lives under `/buzzworthy/`, so Vite's `base` is `/buzzworthy/` and every asset, link and QR URL is built from `import.meta.env.BASE_URL`.
- **Routing.** Hash routes (`#/host`, `#/play`, `#/play?room=ABCD`, `#/edit`) so GitHub Pages never needs server-side rewrites. The query sits *inside* the hash, so the router parses params from `location.hash`, never `location.search`. In-game route changes use `history.replaceState` so a back swipe can't leave the game.
- **The host is the server.** The host tab holds all game state and makes every decision: who buzzed first, too-early lockouts, scores, which clue is open. Phones only send intents ("buzz", "wager 400") and render what the host tells them.

## Networking

- **Library.** [PeerJS](https://peerjs.com) 1.5.5 wraps WebRTC. All PeerJS use sits behind one `Transport` interface (`src/net/transport.ts`) so game code never imports PeerJS.
- **Broker.** The free public PeerJS broker introduces peers. It is needed for the **whole game**, not just setup: every new join and every phone rejoin (which on iOS happens on every screen lock) goes through it. The broker URL is a config value (`VITE_PEER_HOST`, `VITE_PEER_PORT`, `VITE_PEER_PATH`, `VITE_PEER_SECURE`) so we can switch to our own free PeerServer, and tests use a local one.
- **ICE servers, set explicitly.** We pass `config.iceServers` ourselves instead of relying on PeerJS defaults, so a library update can't change behaviour silently. We use Google STUN plus PeerJS's free, best-effort TURN relay (`turn:eu-0.turn.peerjs.com:3478`, `turn:us-0.turn.peerjs.com:3478`). The relay helps on strict networks at no cost, but it's unsupported and sometimes down. When neither direct nor relay works, the phone says so plainly. We never run or pay for our own TURN.
- **Connections.** Phones always call `peer.connect(hostId, { reliable: true, serialization: 'json' })`. PeerJS defaults to *unordered* channels and binarypack, and our "full state on every change" model needs ordered JSON.
- **Timeouts.** PeerJS has none of its own, so we add them:
  - Peer open > 10 s → "Can't reach the matchmaking server. Check your internet and try again."
  - Phone data connection open > 12 s → "Couldn't reach the host. Try joining the same Wi-Fi as the host."
  - `peer-unavailable` → "Room ABCD not found. Check the code on the TV."
- **Staying registered.** The host listens for `peer.on('disconnected')` and calls `peer.reconnect()` with backoff. While disconnected it shows "New players can't join right now" (players already connected keep playing). A closed DataConnection is gone: rejoining always opens a new one, and creates a new Peer if the old one was destroyed.

## Rooms and joining

- **Room code:** 4 letters from `ABCDEFGHJKMNPQRSTUVWXYZ` (23 letters, no I, L or O). Typed codes are trimmed and uppercased before use.
- **Host peer ID:** `buzzworthy-v1-<CODE>`. The prefix avoids clashes in the broker's shared namespace, and `v1` lets the protocol change later.
- **Host token.** When a room is created, the host makes a random token and saves `{code, token}` in localStorage. It always creates its Peer with `new Peer(id, { token })`. The broker lets a client with the same token re-attach to an ID whose old socket hasn't timed out yet (up to ~90 s), which is exactly the host-reload case.
- **ID taken:**
  - *New room:* pick a new code and try again.
  - *Resuming a room:* never switch codes silently. Show "Reclaiming room ABCD…" and retry the same ID with backoff for up to 90 s (each retry is a new Peer, since `unavailable-id` destroys it). Only then offer "Start a new room".
- **Joining:** the TV shows a QR code (`qrcode` package) for `<origin><BASE_URL>#/play?room=ABCD`, plus the short site address and the code in big letters. `#/play` with no room shows a 4-letter code box, for people who'd rather type.
- **Rooms aren't private.** Codes are guessable and the namespace is public. The host validates every message and ignores anything that doesn't parse. The host can kick a player from the lobby.
- **Players:** at most 12. Names are 1–16 characters, trimmed. A duplicate name gets a number added ("Sam 2").
- **Player identity:** on first join a phone makes a random `playerId` and keeps it with the name in storage. Rejoining with the same `playerId` gets the same seat and score back. If two tabs use the same `playerId`, the host keeps the newest connection and tells the old one `replaced`, which shows "Opened in another tab".
- **Storage wrapper.** All localStorage use goes through a wrapper with try/catch and an in-memory fallback (Safari private tabs lose storage when closed; storage can be blocked).

## Staying connected (phones are unreliable)

iOS Safari suspends WebRTC and the broker socket when the screen locks or the user switches apps. So reconnecting is the normal path, not an edge case.

- **Wake lock.** If `navigator.wakeLock` exists, the phone first requests it inside the Join tap handler. It requests again on `visibilitychange` only when `visibilityState === 'visible'`, and catches every rejection. When no lock is held, the phone shows one line: "Keep your screen on. If it keeps disconnecting, turn off Auto-Lock."
- **Coming back to the page.** On `visibilitychange` to visible, the phone disables the buzzer ("Reconnecting…") and sends `ping`. If no `pong` arrives within 1.5 s, it tears down and rejoins. The buzzer is re-enabled only after a fresh `welcome` or `state`. This stops buzzes going into a channel that says "open" but is dead.
- **Auto-rejoin.** When the channel closes, the phone reconnects its Peer to the broker (or makes a new one), opens a new connection with backoff (0.5 s, 1 s, 2 s, then every 4 s), and sends `hello {playerId, name}`. The host restores the seat. Disconnected players show greyed out on the host, never removed.
- **Heartbeat.** Host and phones ping every 5 s. Missing 3 marks a peer disconnected. The interval and miss count are configurable so tests can use short values.
- **Host refresh.** The host saves the room (code, token, roster with scores) and, once game flow exists, the full game state to storage after every change. On reload it offers "Resume game ABCD". Phones rejoin on their own without rescanning.

## Message protocol

Every message is JSON with a `type` and a protocol version `v: 1`. Types and a validating parser live in `src/net/protocol.ts`, shared by host and phone. Anything that fails to parse is dropped.

Phone → host: `hello {playerId, name}`, `buzz {clueId}`, `wager {clueId, amount}`, `response {clueId, text}`, `ping`.

Host → phone: `welcome {playerId, seat, score}`, `state {seq, phase, clueId?, buzzersOpen, lockedOutUntil?, yourScore, ...}`, `buzzResult {winnerId}`, `replaced`, `kicked`, `pong`.

`state.seq` goes up by one with every change. A phone ignores any `state` with a `seq` lower than the last one it applied. The host sends each phone the full `state` it needs on every change, so a phone that missed messages is correct after one update.

### Phases

`lobby` → `board` (picking a clue) → `clue` (clue shown, buzzers closed) → `open` (buzzers open) → `answering` (one player has buzzed) → back to `clue`/`open` after an incorrect reply, or `board` after a correct one or when the host skips. Wager clues add `wager` before `clue`. The final round adds `finalWager` → `finalClue` → `finalReveal`. The game ends in `ended`.

## Buzzing and fairness

- The host reads the clue aloud, then presses **Open buzzers**.
- **The host decides "too early".** A buzz that arrives while buzzers are closed puts that player in a 0.5 s lockout, which is kept in the arbitration module and sent as `state.lockedOutUntil`. The phone may show "Too early" at once, but the host's record is what counts.
- The **first valid `buzz` to arrive** after opening wins. Every other phone is told immediately and locks.
- The host marks the reply **Correct** (adds the clue value) or **Incorrect** (subtracts it and reopens buzzers for players who haven't tried this clue). The host can also **Skip**, which closes the clue with no score change.
- **Who picks next:** the last player to answer correctly is "in control" and is highlighted on the TV. The host clicks the clue they call out. At the start, the host picks.
- **Buzz input.** The buzz fires on `pointerdown`, not `click`, to avoid iOS's tap delay and so a firm press isn't slower than a quick tap. The buzzer has `touch-action: none`, `user-select: none` and `-webkit-touch-callout: none`.
- Arrival order is fair enough on one Wi-Fi network (a few ms of jitter). Clock-sync fairness is a possible later improvement, not v1.
- **Arbitration, lockouts, the tried-set and score changes are one pure module** (`src/game/clue.ts`) with no network or DOM code, so they're unit-tested directly.

## Wagers

- **Hidden-wager clue** (`wager: true`): only the player in control plays it. Their phone asks for a wager before the clue is shown. Allowed: 0 up to the larger of their score and the highest clue value in the round. Correct adds the wager, incorrect subtracts it. Nobody else may buzz.
- **Final wager round:** every player with a score above 0 plays. Each phone shows the category and asks for a wager from 0 to their score. Then the clue appears on the TV, and each phone types a reply (`response`) with 30 seconds to answer. The host reveals and judges each reply in order from lowest to highest score.
- Wager and response rules live in a pure module (`src/game/wager.ts`) with unit tests.

## Boards

- A board is a JSON file: `{ version: 1, title, rounds: [{ name, categories: [{ name, clues: [{ value, clue, response, wager? }] }] }], final?: { category, clue, response } }`. Up to 6 categories × 5 clues per round, any number of rounds (usually 2).
- **Clue IDs** are derived by position: `r{round}-c{category}-{row}` (zero-based), for example `r0-c2-4`. They're not stored in the file.
- Types and a validator (`src/boards/schema.ts`) give clear errors for bad files ("Round 1, category 3, clue 2 has no response").
- The editor (`#/edit`) creates and edits boards, saves them in browser storage, calls `navigator.storage.persist()`, and imports or exports `.json` files. It says plainly that browser storage isn't a backup and keeps Export easy to find.
- The repo ships a few original sample boards in `public/boards/`. The host picks one, or a board from the editor, before starting.

## Look and feel

- The Buzzworthy identity: electric yellow (#ffd60a), buzzer red (#e63946), black, chunky display type (Rubik Mono One, self-hosted under the SIL Open Font License) with a mono accent. Colours and fonts are CSS variables from the first card on, so every screen uses them as it's built.
- The phone screen is almost entirely one big red buzzer.
- **Phone layout:** `100dvh` (not `100vh`), `viewport-fit=cover` with `env(safe-area-inset-*)` padding, `overscroll-behavior: none` on `html` and `body` (no pull-to-refresh mid-game), and inputs at least 16px so iOS doesn't zoom.
- **Sound:** made in code with the Web Audio API (no audio files to license). The host unlocks audio on the "Create room" click. Phones give visual feedback for every result, because sound can be muted and iOS has no vibration API.
- Use original styling and our own round names. Don't copy any TV show's board colours, fonts, sounds or terminology.

## Testing

- **Check command:** `npm ci && npm run check` runs the typecheck, unit tests (Vitest) and build. It never touches the public broker, so it can't fail because of someone else's server.
- **Unit tests** cover the router and hash params, room codes, the storage wrapper, protocol parsing, board validation, clue arbitration and scoring, wagers, heartbeat timing (Vitest fake timers against an in-memory transport), and wake-lock requests (stubbed `navigator.wakeLock`).
- **End-to-end tests** (Playwright, `npm run e2e`) start a **local PeerServer** (the `peer` npm package) on localhost and the Vite dev server pointed at it. One host page and several phone pages run in separate browser contexts, so each has its own storage and `playerId`. This exercises the real PeerJS transport with no internet. CI runs `check` and `e2e` on every push.
- **Smoke test** (`npm run smoke`, manual only, not in CI): one host and one phone against the real public broker, including a host reload that reclaims the room with its token.
- The in-memory `Transport` fake is for unit tests only, where host and phone run in one process.

## Out of scope for v1

Our own TURN relay, accounts, cloud-saved boards, spectators, more than 12 players, native apps, clock-synced buzz fairness.

## Prerequisite on the dev computer

Node.js 22 LTS or newer must be installed on any computer that builds or runs checks (for example `winget install OpenJS.NodeJS.LTS`).
