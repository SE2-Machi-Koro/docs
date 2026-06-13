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
| `/topic/game/{gameId}` | All players in one lobby or game | Roster updates (`LOBBY_ROSTER`), lobby leaves (`LOBBY_LEFT`, `HOST_LEFT`), game start, dice, phase, purchase, effects, game end, and broadcast errors |
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
| `/app/game.rollDice` | `RollDiceRequest` | `ROLL_DICE` and a `GAME_ACTION` snapshot to `/topic/game/{gameId}` |
| `/app/game.resolveEffects` | `ResolveEffectsRequest` | `GAME_ACTION` income/effects result to `/topic/game/{gameId}` |
| `/app/game.advancePhase` | `AdvancePhaseRequest` | Private error on `/user/queue/errors` (`DIRECT_PHASE_ADVANCE_FORBIDDEN`); never mutates game state |
| `/app/game.purchase` | `PurchaseRequest` | `GAME_ACTION` purchase snapshot or `ERROR` purchase failure to `/topic/game/{gameId}` |
| `/app/game.endTurn` | `EndTurnRequest` | `GAME_ACTION` next-turn snapshot or `GAME_END` to `/topic/game/{gameId}` |
| `/app/game.sync` | `SyncGameRequest` | `SYNC` to `/queue/game-sync-user{sessionId}` |
| `/app/chat.send` | `WebSocketMessage` | Chat message to `/topic/public` |
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
