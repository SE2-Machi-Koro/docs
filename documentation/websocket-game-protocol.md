# WebSocket Game Synchronization Protocol

This document defines the client-facing STOMP contract for real-time Machi Koro lobby updates, game action broadcasts, and reconnect synchronization.

## Connection

Clients connect with STOMP over one of the registered WebSocket endpoints:

| Transport | Endpoint | Typical client |
| --- | --- | --- |
| Native WebSocket | `/ws` | Android and WebSocket-capable clients |
| SockJS | `/ws-sockjs` | Browser clients that need SockJS fallback transports |

The application destination prefix is `/app`. Broker subscriptions use `/topic` for shared broadcasts and `/queue` for session-scoped replies.

Every STOMP `CONNECT` frame must include an authenticated session token:

```text
Authorization: Bearer <sessionToken>
```

The server resolves the user identity from this token and ignores client-supplied sender or user fields for authorization-sensitive actions.

## Subscriptions

Clients should subscribe before sending lobby or game commands so they do not miss immediate broadcasts.

| Subscription | Scope | Purpose |
| --- | --- | --- |
| `/topic/game/{gameId}` | All players in one lobby or game | Roster updates (`LOBBY_ROSTER`), lobby leaves (`LOBBY_LEFT`, `HOST_LEFT`), game start, dice rolls and rerolls, phase, purchase, effects, accusation results (`ACCUSATION_RESULT`), in-game chat (`CHAT`), game end, and broadcast errors |
| `/queue/lobby-user{sessionId}` | Single WebSocket session | Private lobby replies: creation result, the joiner's `LOBBY_JOINED` acknowledgement, the `LOBBY_ROSTER` snapshot after join, and lobby-specific request errors |
| `/user/queue/game-sync` | Single WebSocket session | Full `SYNC` snapshot for reconnect and explicit state refresh |
| `/user/queue/errors` | Single WebSocket session | Domain rejections routed privately to the requesting client, including forbidden direct phase advancement |
| `/topic/public` | Legacy / global chat only | Chat and connection announcements from `/app/chat.*`. **Not** a source for lobby or game events — retained only for backwards compatibility. |

`{sessionId}` is the STOMP session id assigned by Spring. Browser clients usually receive it from the STOMP session object after connection. The server sends private lobby replies to `/queue/lobby-user{sessionId}` and sends reconnect sync replies to the resolved broker destination `/queue/game-sync-user{sessionId}`, which clients consume by subscribing to `/user/queue/game-sync`.

Lobby events are delivered on two tiers:

- **Private lobby queue** — the requester's own acknowledgements (`LOBBY_CREATED`, the joiner's `LOBBY_JOINED`), the post-join `LOBBY_ROSTER` snapshot, and lobby request errors are sent only to the requesting session on `/queue/lobby-user{sessionId}`. This requires the STOMP session id; if it is unavailable, the server currently skips these private replies.
- **Lobby broadcast topic** — shared lobby events are received through `/topic/game/{gameId}` once the client knows the game id (from `LOBBY_CREATED` or its private `LOBBY_JOINED` reply). On a normal join the topic carries a `LOBBY_ROSTER` broadcast (not `LOBBY_JOINED`); leaves broadcast `LOBBY_LEFT` or `HOST_LEFT`.

Game clients must not use `/topic/public` for game or lobby state. All game-state and lobby broadcasts are scoped to `/topic/game/{gameId}` or a private queue; `/topic/public` is legacy chat only.

> Note: On the client, the `LOBBY_ROSTER` handler populates `mutablePlayers` during the lobby waiting phase — before `GAME_STARTED` — so the lobby screen reflects the current membership prior to the game's initial state snapshot.

## Send Destinations

| Client sends to | Payload | Server publishes |
| --- | --- | --- |
| `/app/lobby.create` | `WebSocketMessage` | `LOBBY_CREATED` to the private lobby queue `/queue/lobby-user{sessionId}` |
| `/app/lobby.join` | `WebSocketMessage` with `payload.lobbyCode` | Private `LOBBY_JOINED` acknowledgement and `LOBBY_ROSTER` snapshot to the joiner's queue `/queue/lobby-user{sessionId}`, plus a `LOBBY_ROSTER` broadcast to `/topic/game/{gameId}` for the existing members |
| `/app/lobby.leave` | `WebSocketMessage` with `payload.gameId` | `LOBBY_LEFT` or `HOST_LEFT` to `/topic/game/{gameId}` |
| `/app/game.start` | `StartGameRequest` | `GAME_STARTED` and a `GAME_ACTION` snapshot to `/topic/game/{gameId}` |
| `/app/game.enterScreen` | `EnterGameScreenRequest` | When the host enters a still-`WAITING` game: `GAME_STARTED` and a `GAME_ACTION` snapshot to `/topic/game/{gameId}`. Otherwise a silent no-op (game already in progress, or caller is not the host) |
| `/app/game.rollDice` | `RollDiceRequest` | `ROLL_DICE` and a `GAME_ACTION` snapshot to `/topic/game/{gameId}` |
| `/app/game.rerollDice` | `RollDiceRequest` | `ROLL_DICE` (`event: DICE_REROLLED`) and a `GAME_ACTION` snapshot to `/topic/game/{gameId}`. Requires a built Radio Tower; once per turn |
| `/app/game.resolveEffects` | `ResolveEffectsRequest` | `GAME_ACTION` income/effects result to `/topic/game/{gameId}` |
| `/app/game.advancePhase` | `AdvancePhaseRequest` | Private error on `/user/queue/errors` (`DIRECT_PHASE_ADVANCE_FORBIDDEN`); never mutates game state |
| `/app/game.purchase` | `PurchaseRequest` | `GAME_ACTION` purchase snapshot or `ERROR` purchase failure to `/topic/game/{gameId}` |
| `/app/game.endTurn` | `EndTurnRequest` | `GAME_ACTION` next-turn snapshot or `GAME_END` to `/topic/game/{gameId}` |
| `/app/game.reportCheat` | `ReportCheatRequest` | Nothing is broadcast — the server silently flags the active player as having used the Insider Trading cheat. Authorization failures (e.g. `NOT_YOUR_TURN`) go privately to `/user/queue/errors` |
| `/app/game.accuse` | `AccuseRequest` | `ACCUSATION_RESULT` plus a state snapshot to `/topic/game/{gameId}`; a rejected accusation (`INVALID_ACCUSATION`) goes privately to `/user/queue/errors` |
| `/app/game.sync` | `SyncGameRequest` | `SYNC` to `/queue/game-sync-user{sessionId}` |
| `/app/chat.send` | `WebSocketMessage` | Chat message to `/topic/public` (legacy global chat) |
| `/app/game.chat.send` | `GameChatRequest` | `CHAT` message to `/topic/game/{gameId}` (in-game chat, scoped to one game) |
| `/app/chat.addUser` | `WebSocketMessage` | Join message to `/topic/public`; may also trigger `SYNC` for an active in-progress game |

## Message Envelope

All WebSocket responses use `WebSocketMessage`:

```json
{
  "type": "GAME_ACTION",
  "sender": "server",
  "content": "Optional human-readable text",
  "payload": {},
  "timestamp": 1714000000000,
  "gameId": 1
}
```

Core game message types:

| Type | Meaning |
| --- | --- |
| `GAME_STARTED` | The host started the lobby. The payload is the full `GameStateDto`. |
| `GAME_ACTION` | A state-changing game action completed. The payload includes `event`, `turnPhase`, `activePlayerId`, and `state`. |
| `ROLL_DICE` | Dice were rolled. The payload includes the dice values, total, completion flag, and state snapshot. |
| `SYNC` | Private reconnect snapshot. The payload includes `targetUserId`, `targetSessionId`, and `state`. |
| `GAME_END` | The game has ended. The payload includes `winnerId`, `roundsPlayed`, and final `state`. |
| `ACCUSATION_RESULT` | A cheating accusation was adjudicated. The payload includes `accuserPlayerId`, `accusedPlayerId`, `caught`, `penalizedPlayerId`, `penaltyCoins`, and `state`. |
| `CHAT` | An in-game chat message broadcast to `/topic/game/{gameId}` via `/app/game.chat.send`. `sender` is the username and `content` is the message text. |
| `ERROR` | The command failed. The payload is a `WebSocketErrorDto` with stable `code`, `message`, and `timestamp` fields. |

Lobby-specific message types include `LOBBY_CREATED`, `LOBBY_JOINED`, `LOBBY_ROSTER`, `LOBBY_LEFT`, and `HOST_LEFT`.

## Error Payloads

All WebSocket `ERROR` messages use the same payload shape:

```json
{
  "code": "NOT_YOUR_TURN",
  "message": "It is not your turn",
  "timestamp": 1714000000000,
  "context": {
    "event": "ROLL_FAILED"
  }
}
```

Clients must parse `payload.code`, `payload.message`, and `payload.timestamp` as the stable contract. `payload.context` is endpoint-specific metadata used for UI cleanup or diagnostics.

Expected codes include:

| Code | Meaning |
| --- | --- |
| `UNAUTHENTICATED` | The WebSocket session has no authenticated principal. |
| `INVALID_PAYLOAD` | The incoming message body does not match the required structure. |
| `INVALID_LOBBY_CODE` | The requested lobby code is missing, malformed, or not found. |
| `UNKNOWN_SESSION` | The game command came from an unregistered WebSocket session. |
| `NOT_HOST` | The authenticated user attempted a host-only action. |
| `NOT_YOUR_TURN` | The authenticated user is not the active player. |
| `GAME_NOT_STARTED` | The game is still waiting and cannot accept the action. |
| `GAME_FINISHED` | The game already ended. |
| `DIRECT_PHASE_ADVANCE_FORBIDDEN` | Clients attempted to advance a turn phase directly. |
| `ROLL_ALREADY_COMPLETED` | Dice were already rolled for the current turn. |
| `EFFECTS_ALREADY_RESOLVED` | Income/card effects were already resolved for the current turn. |
| `PURCHASE_ALREADY_MADE` | The active player already purchased this turn. |
| `DUPLICATE_PURPLE_ESTABLISHMENT` | The player tried to buy a purple establishment they already own. |
| `INVALID_ACCUSATION` | A cheating accusation was rejected: self-accusation, a non-member accuser or accused, or a second accusation by the same accuser in the same turn. |
| `INTERNAL_ERROR` | An unexpected server-side failure occurred; details are logged server-side only. |

## Authoritative Turn Loop

Clients request gameplay actions; they do not choose phases. For an in-progress turn, only the active player's legal request can produce the following transition:

| Current phase | Accepted action | Resulting phase and state |
| --- | --- | --- |
| `ROLL_DICE` | `/app/game.rollDice` | Persists exactly one roll and enters `RESOLVE_EFFECTS` |
| `RESOLVE_EFFECTS` | `/app/game.resolveEffects` | Applies the stored roll exactly once and enters `BUY_OR_BUILD` |
| `BUY_OR_BUILD` | `/app/game.purchase` | Applies at most one affordable, available purchase; phase remains `BUY_OR_BUILD` |
| `BUY_OR_BUILD` | `/app/game.endTurn` | Checks for a winner; otherwise changes active player and enters `ROLL_DICE` |

Ending a turn without purchasing is the supported skip action. A continuing next turn clears `lastDiceRoll` and `hasPurchasedThisTurn`. Invalid, duplicated, out-of-order, unaffordable, unavailable, or non-active-player requests do not mutate the stored game state.

Every successful action publishes a server snapshot. Clients must replace local phase, roll, coins, supply, ownership, and active-player state from that snapshot.

## Purchase Rejections

When `/app/game.purchase` is rejected after authorization succeeds, the server broadcasts an `ERROR` message on `/topic/game/{gameId}`. Clients with a pending shop action must consume `payload.context.event = "PURCHASE_FAILED"` as a failed purchase and clear their pending state.

For example, buying the same purple establishment twice produces:

```json
{
  "type": "ERROR",
  "sender": "server",
  "payload": {
    "code": "DUPLICATE_PURPLE_ESTABLISHMENT",
    "message": "Player already owns purple establishment STADIUM",
    "timestamp": 1714000000000,
    "context": {
      "event": "PURCHASE_FAILED",
      "purchaseType": "ESTABLISHMENT",
      "cardType": "STADIUM"
    }
  },
  "gameId": 42
}
```

For rejected landmark purchases the payload uses `landmarkType` instead of `cardType`.

## Screen Entry and Auto-Initialization

`/app/game.enterScreen` is sent by every player when their client navigates to
the game board. It is an idempotent alternative trigger for game start: when the
**host** sends it while the game is still `WAITING`, the server initializes the
game and broadcasts the same `GAME_STARTED` + `GAME_ACTION` pair as
`/app/game.start`. When the game is already `IN_PROGRESS`, or when a non-host
sends it, the call is a silent no-op so late or non-host arrivals neither restart
nor duplicate initialization.

Send:

```json
SEND /app/game.enterScreen

{
  "gameId": 1
}
```

Broadcast on host-triggered initialization (to `/topic/game/{gameId}`):

```json
{
  "type": "GAME_STARTED",
  "sender": "server",
  "content": "Game 1 has started",
  "payload": { "...": "full GameStateDto" },
  "timestamp": 1714000000000,
  "gameId": 1
}
```

A standard `GAME_ACTION` snapshot (`payload.event = "GAME_STARTED"`) follows
immediately. Unauthenticated sessions are rejected with `UNAUTHENTICATED` on
`/user/queue/errors`.

## Dice Reroll (Radio Tower)

`/app/game.rerollDice` lets the active player reroll once per turn when they have
built the Radio Tower landmark. It shares the `RollDiceRequest` payload and the
`ROLL_DICE` envelope with `/app/game.rollDice`; the only wire difference is
`payload.event`, which is `DICE_REROLLED` instead of `DICE_ROLLED`.

Send:

```json
SEND /app/game.rerollDice

{
  "gameId": 1,
  "rollTwoDice": false
}
```

Broadcast (to `/topic/game/{gameId}`):

```json
{
  "type": "ROLL_DICE",
  "sender": "SERVER",
  "content": "Player 10 rolled: 8",
  "payload": {
    "event": "DICE_REROLLED",
    "turnPhase": "RESOLVE_EFFECTS",
    "activePlayerId": 10,
    "playerId": 10,
    "result": [3, 5],
    "total": 8,
    "completed": true,
    "extraTurnGranted": false,
    "roundNumber": 4,
    "timestamp": 1714000000000,
    "state": { "...": "GameStateDto snapshot" }
  },
  "gameId": 1
}
```

A matching `GAME_ACTION` snapshot (`payload.event = "DICE_REROLLED"`) follows.
Rejections broadcast an `ERROR` on `/topic/game/{gameId}` with
`payload.context.event = "REROLL_FAILED"`.

## In-Game Chat

`/app/game.chat.send` broadcasts a chat message to a single game's subscribers.
It is distinct from `/app/chat.send`, which targets the legacy global
`/topic/public` channel. The sender is resolved from the authenticated session
principal, and the server verifies the sender is a participant of the game. Blank
messages, messages longer than 300 characters, and messages from non-participants
are rejected silently (logged, with no broadcast and no error reply). Chat does
not mutate or persist game state.

Send:

```json
SEND /app/game.chat.send

{
  "gameId": 1,
  "message": "Good roll!"
}
```

Broadcast (to `/topic/game/{gameId}`):

```json
{
  "type": "CHAT",
  "sender": "alice",
  "content": "Good roll!",
  "payload": null,
  "timestamp": 1714000000000,
  "gameId": 1
}
```

## Cheating and Accusations

The Insider Trading cheat is a hidden-information mechanic: the active player's
client may self-report that it used the cheat, and any other player may accuse a
suspected cheater. The server is the sole authority and adjudicates every
accusation. This section documents only the wire contract; the full game-rules
detail, penalty rules, persistence, and in-app tutorial for this mechanic are
documented in [Backend-Cheating.md](Backend-Cheating.md).

### `/app/game.reportCheat`

Sent by the active player's client to silently flag that it used the cheat this
game. **Nothing is broadcast** — only the server learns, so opponents must still
guess. Only the active player may report (enforced by the active-player guard),
so a client can only ever incriminate itself. Authorization failures
(`NOT_YOUR_TURN`, `UNAUTHENTICATED`, `GAME_FINISHED`) are delivered privately on
`/user/queue/errors`.

Send:

```json
SEND /app/game.reportCheat

{
  "gameId": 1
}
```

### `/app/game.accuse`

One player accuses another of cheating. On a valid accusation the server
adjudicates, applies the coin penalty, and broadcasts `ACCUSATION_RESULT` with a
fresh state snapshot so every client updates coin totals. A caught cheater loses
2 coins; a wrong accuser loses 1 coin; balances are clamped at 0, and
`penaltyCoins` reports the coins actually deducted. Each accuser may accuse at
most once per turn. All IDs in the request and response are `PlayerModel.id`
(player IDs), not user IDs.

Send:

```json
SEND /app/game.accuse

{
  "gameId": 1,
  "accusedPlayerId": 2
}
```

Broadcast on a valid accusation (to `/topic/game/{gameId}`):

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

Invalid accusations (self-accusation, a non-member accuser or accused, or a
second accusation in the same turn) are rejected with `INVALID_ACCUSATION` on
`/user/queue/errors` and are never broadcast.

## Two-Player Synchronization Flow

1. Player A and Player B connect to `/ws` or `/ws-sockjs` with `Authorization: Bearer <sessionToken>`.
2. Player A subscribes to `/queue/lobby-user{sessionId}` and sends `/app/lobby.create`.
3. Player A receives `LOBBY_CREATED` with `gameId` and `lobbyCode`.
4. Both players subscribe to `/topic/game/{gameId}`.
5. Player B subscribes to `/queue/lobby-user{sessionId}` and sends `/app/lobby.join` with the lobby code.
6. Player B receives `LOBBY_JOINED` and `LOBBY_ROSTER` privately. Both players receive `LOBBY_ROSTER` on `/topic/game/{gameId}`.
7. Player A sends `/app/game.start`.
8. Both players receive `GAME_STARTED` and the initial `GAME_ACTION` snapshot on `/topic/game/{gameId}`.
9. During the game, the active player sends `rollDice`, `resolveEffects`, an optional `purchase`, and `endTurn`. Both players consume the resulting `ROLL_DICE`, `GAME_ACTION`, or `GAME_END` messages from `/topic/game/{gameId}` and replace their local state with the included `state` snapshot.

Clients should treat the server snapshot as authoritative. Local UI state may optimistically display pending actions, but it must reconcile to the latest broadcast payload.

## Reconnect and Explicit Sync

A reconnecting player must establish a new STOMP connection with the same authenticated session token and re-subscribe to its private and topic destinations before requesting a sync:

```text
/user/queue/errors              # private domain rejections
/user/queue/game-sync           # SYNC snapshot reply
/queue/lobby-user{sessionId}    # private lobby queue
/topic/game/{gameId}            # any game/lobby id already known to the client
```

The client re-subscribes to the error, sync, and lobby-queue destinations, plus every `/topic/game/{gameId}` it already knows about. `/topic/public` is **not** re-subscribed for lobby or game state; subscribe to it only if the client intentionally keeps the legacy chat channel.

Then the client sends:

```json
SEND /app/game.sync

{
  "gameId": 1
}
```

The server validates that the authenticated user belongs to the requested game and that the game is in progress. On success, it publishes:

```json
{
  "type": "SYNC",
  "sender": "server",
  "content": "State sync for reconnecting player",
  "payload": {
    "targetUserId": 10,
    "targetSessionId": "abc123",
    "state": {
      "game": {
        "id": 1,
        "status": "IN_PROGRESS",
        "turnPhase": "ROLL_DICE"
      },
      "activePlayerId": 10,
      "turnOrder": [10, 11],
      "players": [],
      "marketplace": []
    }
  },
  "gameId": 1
}
```

If `gameId` is omitted, the server attempts to find the authenticated user's active in-progress game. Unauthorized or inactive sync requests are rejected without broadcasting state to other players.
