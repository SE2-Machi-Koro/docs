# Backend Recovery And Sync Snapshot Contract

This document describes the backend recovery guarantees for reconnecting
players and backend restarts. It is based on the current recovery implementation
in `SE2-Machi-Koro/Server`.

## Recovery Scope

The backend supports two recovery cases:

| Case | Supported behavior |
| :--- | :--- |
| Player reconnect | A reconnecting authenticated player can receive a game-state snapshot through `/app/game.sync` and `/user/queue/game-sync`. |
| Backend restart | Persisted game state survives as long as the PostgreSQL volume/database is preserved. WebSocket sessions do not survive and clients must reconnect. |

Recovery is snapshot-based. The backend does not replay missed WebSocket events
after reconnect. Clients should treat the sync snapshot as the authoritative
state after any reconnect or backend restart.

## Player Reconnect Flow

Clients connect to the backend through STOMP over:

```text
ws://<host>:<port>/ws
```

The STOMP `CONNECT` frame must include the authenticated session token:

```text
Authorization: Bearer <sessionToken>
```

After reconnect, the client must subscribe to:

```text
/topic/game/{gameId}
/user/queue/game-sync
/user/queue/errors
```

Then the client sends:

```text
SEND /app/game.sync
```

Payload with explicit game context:

```json
{
  "gameId": 1
}
```

Payload without explicit game context:

```json
{
  "gameId": null
}
```

When `gameId` is omitted, the backend attempts to resolve the caller's active
`IN_PROGRESS` game from persisted player membership.

## Automatic Sync During Join/Reconnect

The backend can also emit a sync snapshot automatically during the
`/app/chat.addUser` flow.

When an authenticated player sends:

```text
SEND /app/chat.addUser
```

with:

```json
{
  "type": "JOIN",
  "sender": "demo-player",
  "gameId": 1
}
```

the backend:

1. reads the authenticated user from the STOMP session principal;
2. ignores client-supplied identity for authorization decisions;
3. maps the STOMP session to the user's game;
4. checks whether the user already belongs to an active in-progress game;
5. sends a `SYNC` message to `/user/queue/game-sync` when an active game exists.

This automatic path covers the common reconnect case where a player reconnects
after the game has already started or during the transition from lobby to game.

## Explicit Sync Response

Successful sync is delivered only to the reconnecting user:

```text
/user/queue/game-sync
```

Response envelope:

```json
{
  "type": "SYNC",
  "sender": "server",
  "content": "State sync for reconnecting player",
  "gameId": 1,
  "payload": {
    "targetUserId": 2,
    "targetSessionId": "session-rejoin",
    "state": {
      "game": {},
      "players": [],
      "playerCards": {},
      "turnOrder": []
    }
  }
}
```

The backend validates that an explicit `gameId` belongs to the authenticated
user before sending a snapshot. If the user is not a member of that game, the
backend does not publish a sync snapshot.

## Current `GameStateDto` Snapshot

The currently implemented `GameStateDto` contains:

| Field | Type | Description |
| :--- | :--- | :--- |
| `game` | `GameModel` | Persisted game state, including status, phase, current turn index, round number, and last dice roll. |
| `players` | `List<PlayerModel>` | Persisted players for the game. Each player contains both `id` and `userId`. |
| `playerCards` | `Map<Int, List<PlayerCardModel>>` | Owned cards keyed by `playerId`, not by `userId`. Players with no cards may be present with an empty list or absent from the DAO result before normalization. |
| `turnOrder` | `List<Int>` | Ordered list of player IDs, sorted by `PlayerModel.turnOrder`. |

Current example:

```json
{
  "game": {
    "id": 1,
    "status": "IN_PROGRESS",
    "hostUserId": 10,
    "lobbyCode": "ABC1234",
    "maxPlayers": 4,
    "currentTurnIndex": 0,
    "turnPhase": "ROLL_DICE",
    "lastDiceRoll": null,
    "roundNumber": 1,
    "hasPurchasedThisTurn": false
  },
  "players": [
    {
      "id": 5,
      "gameId": 1,
      "userId": 10,
      "turnOrder": 0,
      "coins": 3,
      "lastSeenAt": null
    },
    {
      "id": 6,
      "gameId": 1,
      "userId": 11,
      "turnOrder": 1,
      "coins": 3,
      "lastSeenAt": null
    }
  ],
  "playerCards": {
    "5": [
      {
        "id": 1,
        "cardType": "BAKERY",
        "quantity": 1
      }
    ],
    "6": []
  },
  "turnOrder": [5, 6]
}
```

## ID Semantics

The current backend contract uses player IDs for turn-order fields:

- `turnOrder` contains `PlayerModel.id` values.
- `playerCards` is keyed by `PlayerModel.id`.
- `game.currentTurnIndex` is an index into `turnOrder`.
- The active player can be derived as:

```text
activePlayerId = turnOrder[game.currentTurnIndex]
```

This is a player ID, not a user ID.

Clients that need the active user can map the derived `activePlayerId` through
the `players` list:

```text
activeUserId = players.first { it.id == activePlayerId }.userId
```

The issue text asks whether `turnOrder` and `activePlayerId` use user IDs. They
do not in the current implementation. `turnOrder` uses player IDs, and
`activePlayerId` is not currently emitted as a separate snapshot field.

## Issue-Listed Snapshot Fields Not Yet Emitted

The issue lists a richer snapshot contract. The following fields are not present
in the current implemented `GameStateDto`:

| Field | Current status |
| :--- | :--- |
| `playerLandmarks` | Persisted landmark rows are initialized on game start, but they are not included in the sync snapshot. |
| `marketplace` | Marketplace rows are initialized on game start, but they are not included in the sync snapshot. |
| `cardDefinitions` | Card definitions exist in backend data access, but they are not included in the sync snapshot. |
| `landmarkDefinitions` | Landmark definitions exist in backend data access, but they are not included in the sync snapshot. |
| `activePlayerId` | Not emitted. Clients derive it from `turnOrder[game.currentTurnIndex]`. |

Until these fields are added to `GameStateDto`, clients must not assume they are
available in `/user/queue/game-sync` payloads.

## Backend Restart Recovery

Backend restart recovery depends on the database, not on WebSocket session
state.

What survives a backend restart:

- games stored in PostgreSQL;
- players stored in PostgreSQL;
- player cards stored in PostgreSQL;
- turn phase, current turn index, round number, last dice roll, and purchase
  state stored on the game row;
- marketplace and landmark rows stored in PostgreSQL, although they are not yet
  included in the current sync snapshot.

What does not survive a backend restart:

- active STOMP sessions;
- topic subscriptions;
- in-memory session-to-user/game mappings;
- in-flight messages that were not persisted.

Client behavior after backend restart:

1. reconnect to `/ws` with the session token;
2. re-subscribe to `/topic/game/{gameId}`, `/user/queue/game-sync`, and
   `/user/queue/errors`;
3. send `/app/game.sync`;
4. replace local game state with the received snapshot.

## Failure Cases

The backend intentionally does not publish a sync snapshot when:

- the STOMP session is missing;
- the authenticated principal is missing;
- the requested `gameId` does not belong to the authenticated user;
- no active `IN_PROGRESS` game can be resolved;
- the resolved game is not currently `IN_PROGRESS`.

These failures are recorded through reconnect sync metrics in the backend:

```text
machikoro.reconnect.sync.success.total
machikoro.reconnect.sync.failure.total
machikoro.reconnect.sync.latency.ms
```

## Recovery Test References

Relevant backend tests:

| Test class | Coverage |
| :--- | :--- |
| `WebSocketControllerTests` | Automatic sync on `/app/chat.addUser`, explicit `/app/game.sync`, user-specific `/queue/game-sync` delivery, unauthorized game rejection, missing principal handling. |
| `GameSyncServiceTest` | Snapshot construction, active in-progress game lookup, membership validation, missing game handling. |
| `LobbyServiceTest` | Reconnect behavior when a user is already in a lobby/game. |
| `WebSocketConnectionTrackerTest` | Thread-safe session-to-user/game tracking. |
| `GameControllerTest` | Game-start broadcast and game-topic message behavior. |

Run the focused recovery checks from the backend repository:

```bash
./gradlew test --tests "org.machikoro.server.controller.WebSocketControllerTests"
./gradlew test --tests "org.machikoro.server.service.GameSyncServiceTest"
./gradlew test --tests "org.machikoro.server.service.LobbyServiceTest"
./gradlew test --tests "org.machikoro.server.service.WebSocketConnectionTrackerTest"
./gradlew test --tests "org.machikoro.server.controller.GameControllerTest"
```

Run the full backend suite:

```bash
./gradlew check
```

