# Cheating and Accusations (Insider Trading)

This document describes the full cheat/accusation mechanic ("Schummelfunktion")
as implemented in `SE2-Machi-Koro/Server`, plus the in-app tutorial and feedback
in `SE2-Machi-Koro/Client`. It covers how the Insider Trading cheat works, how a
player self-reports it, how accusations are made and adjudicated, the penalty
rules, the broadcast payload, and where the feature is persisted.

The server is the **sole authority**: it silently records cheat usage and decides
every accusation. Clients never adjudicate — they only send intents and render
the result the server broadcasts.

The pure wire contract (frames and JSON shapes) lives in
[websocket-game-protocol.md](websocket-game-protocol.md#cheating-and-accusations);
this document is the game-rules/feature reference that sits behind it.

## The Insider Trading cheat

Insider Trading is a hidden-information mechanic. On their **own turn**, the
active player can secretly peek at the strongest establishment they can currently
afford — the best card to buy is highlighted in green in the shop. The peek is:

- triggered by **shaking the phone** or tapping the **"Insider tip"** button,
- valid for the **current turn only**, and
- **silently reported to the server** the moment it is used.

The cheat gives an information edge, not an automatic win — but using it leaves a
server-side flag that opponents can cash in on if they correctly accuse the
player. Cheating is therefore a calculated risk, not a free advantage.

## Self-reporting the cheat — `/app/game.reportCheat`

When the active player uses the cheat, their client sends a silent self-report:

| Aspect | Behavior |
| :--- | :--- |
| Destination | `/app/game.reportCheat` (payload `ReportCheatRequest { gameId }`) |
| Effect | Server marks the active player as having an **outstanding (uncaught) cheat** |
| Authority | Only the active player on their own turn may report (`GameStateGuard.ensureSenderIsActivePlayer`) — a client can only ever incriminate **itself**, never frame another player |
| Broadcast | **None.** Only the server learns; opponents must still guess |
| Idempotency | Reporting twice keeps a single flag, so a player who cheats repeatedly without being caught is still "caught at most once" per accusation |

Authorization failures (`NOT_YOUR_TURN`, `UNAUTHENTICATED`, `GAME_FINISHED`) are
delivered privately to the reporter on `/user/queue/errors`.

## Accusing and adjudication — `/app/game.accuse`

Any other player may accuse a suspected cheater by sending
`AccuseRequest { gameId, accusedPlayerId }` to `/app/game.accuse`. The server
adjudicates the whole accusation inside a **single transaction**
(`AccusationService.accuse`):

1. Validate the game is running and that both accuser and accused are members.
2. Enforce the once-per-turn and double-submit guardrails (see below).
3. **Atomically consume** the accused player's cheat flag. The flag's
   affected-row count is the verdict:
   - **1 row removed → caught.** The accused really had an outstanding cheat.
   - **0 rows removed → wrong.** No outstanding cheat; the accuser was wrong.
4. Apply the coin penalty to whichever player lost, clamped at 0.
5. Broadcast the outcome with a fresh state snapshot.

All IDs in the request and the response are `PlayerModel.id` (player IDs), **not**
user IDs.

### Penalty rules

| Outcome | Who is penalized | Coins lost |
| :--- | :--- | :--- |
| **Caught** (accused had an outstanding cheat) | the cheater (accused) | `CHEATER_PENALTY` = **2** |
| **Wrong** (no outstanding cheat) | the accuser | `WRONG_ACCUSER_PENALTY` = **1** |

Coin balances never go negative: each penalty is clamped at 0, and the broadcast
`penaltyCoins` reports the coins **actually** deducted (which can be less than the
nominal penalty when the balance is already low).

### Guardrails and concurrency

- **One accusation per accuser per turn.** A second accusation by the same
  accuser in the same turn (keyed by `roundNumber` + `currentTurnIndex`) is
  rejected with `INVALID_ACCUSATION` ("You can only accuse once per turn"). The
  turn slot is claimed only **after** the transaction commits, so a rollback or
  an internal retry never burns the slot.
- **Double-submit guard.** An in-flight set of accuser IDs closes the race where
  the same accuser fires two accusations concurrently.
- **Race-safe catches.** Because the verdict is the flag-delete's affected-row
  count, only the one accusation whose delete actually removed the row counts as
  a catch — two simultaneous accusers can never double-penalize the same cheater.

Invalid accusations (self-accusation, a non-member accuser or accused, or a
second accusation in the same turn) throw `INVALID_ACCUSATION` and are delivered
**privately** to the accuser on `/user/queue/errors`; they are never broadcast.

## Broadcast payload — `ACCUSATION_RESULT`

On a valid accusation the server broadcasts to `/topic/game/{gameId}` so every
client updates coin totals:

```json
{
  "type": "ACCUSATION_RESULT",
  "sender": "server",
  "payload": {
    "accuserPlayerId": 1,
    "accusedPlayerId": 2,
    "caught": true,
    "penalizedPlayerId": 2,
    "penaltyCoins": 2,
    "state": { "...": "GameStateDto snapshot" }
  },
  "timestamp": 1714000000000,
  "gameId": 1
}
```

| Field | Meaning |
| :--- | :--- |
| `accuserPlayerId` | player ID of the accuser |
| `accusedPlayerId` | player ID of the accused |
| `caught` | `true` if the accused really cheated; `false` if the accuser was wrong |
| `penalizedPlayerId` | the player who lost coins (cheater if caught, accuser if wrong) |
| `penaltyCoins` | coins actually deducted (clamped at 0) |
| `state` | fresh `GameStateDto` snapshot so coin totals update for everyone |

These keys are the wire contract agreed in server issue #361 / client issue #280
and must stay in lockstep with the client parser.

## Persistence

Outstanding cheats are durable so a cheat reported before a backend restart stays
catchable:

- **Table:** `cheat_flags` — one row per player, created by Flyway migration
  `V3__add_cheat_flags.sql`. A row present means "this player cheated and has not
  yet been caught."
- **DAO:** `CheatFlagDao` — `setOutstanding` (idempotent insert), `consume`
  (atomic delete returning the affected-row count), `hasOutstanding` (diagnostic).
- **Cleanup:** `player_id` is a foreign key with `ON DELETE CASCADE`, so flags are
  removed automatically when a player or game is deleted.

The once-per-turn accusation slot is kept in memory on purpose: losing it on
restart merely re-opens at most one extra accusation for the in-flight turn.

## In-app tutorial and feedback (Client)

The requirement is *"Schummelfunktion integriert **und tutorialisiert**."* The
client tutorializes the mechanic and gives the player feedback:

- **Tutorial.** From the Home screen (after login/registration) the **"Rules"**
  button opens the in-app rules viewer (`PdfViewerScreen`). The viewer shows the
  rulebook first, and its **final page** is a dedicated **"Cheating &
  Accusations"** tutorial (`CheatRulesContent`, client issue #336) that explains
  the Insider Trading cheat and how to accuse — using the same labels players see
  in-game ("Insider tip", "Accuse of cheating"). A player paging through the
  rules discovers it naturally.
- **Cheat feedback.** Activating the cheat shows an "Insider Trading active" toast
  (client issue #203).
- **Accusation feedback.** The `ACCUSATION_RESULT` outcome is shown as a toast,
  and when the **local player's own accusation correctly catches a cheater**, the
  client plays a celebratory **"SIUUU"** sound (client issue #353,
  `SoundManager.play(GameSound.SIUUU)`). It plays only for the player who made
  the correct accusation — not on a wrong accusation, and not when a rival makes
  the catch.

## Source references

**Server (`SE2-Machi-Koro/Server`)**

- `controller/GameController.kt` — `/game.reportCheat`, `/game.accuse`, `broadcastAccusationResult`
- `service/AccusationService.kt` — adjudication, penalties, guardrails
- `dao/CheatFlagDao.kt`, `database/CheatFlags` table
- `dto/ReportCheatRequest.kt`, `dto/AccuseRequest.kt`, `dto/AccusationOutcome.kt`
- `resources/db/migration/V3__add_cheat_flags.sql`

**Client (`SE2-Machi-Koro/Client`)**

- `ui/home/CheatRulesContent.kt` — the in-app tutorial page
- `ui/home/PdfViewerScreen.kt`, `ui/home/HomeScreen.kt` — rules viewer entry point
- `domain/cheat/InsiderTrading.kt`, `ui/cheat/ShakeDetector.kt` — the cheat trigger
- `domain/model/state/AccusationResult.kt`, `MainActivity.kt` — outcome handling + SIUUU
- `ui/game/SoundManager.kt` (`GameSound.SIUUU`, `res/raw/siuuu.mp3`)

## Related documentation

- [websocket-game-protocol.md](websocket-game-protocol.md#cheating-and-accusations) — the STOMP wire contract for `reportCheat`/`accuse` and `ACCUSATION_RESULT`.
- [Business-Logic.md](Business-Logic.md) — overall game rules and turn phases.
