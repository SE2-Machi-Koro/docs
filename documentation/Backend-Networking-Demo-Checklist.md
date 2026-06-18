# Backend Networking Demo Verification Checklist

This checklist verifies the backend networking requirements for Sprint 3 demo
and review. It is written for a backend build that contains the authenticated
STOMP flow, game-topic broadcasts, reconnect sync, actuator health, and
Docker/GHCR deployment setup.

## Prerequisites

- Backend repository: `SE2-Machi-Koro/Server`.
- Backend branch/build includes these WebSocket destinations:
  - `/app/lobby.create`
  - `/app/chat.addUser`
  - `/app/game.start`
  - `/app/game.rollDice`
  - `/app/game.resolveEffects`
  - `/app/game.purchase`
  - `/app/game.endTurn`
  - `/app/game.sync`
- Two test users exist or can be registered through `/auth/register`.
- The STOMP client sends an `Authorization` native header on `CONNECT`:

```text
Authorization: Bearer <sessionToken>
```

- The clients subscribe to:
  - `/topic/public`
  - `/topic/game/{gameId}`
  - `/user/queue/game-sync`
  - `/user/queue/errors`

## Local Backend Startup

From the backend repository:

```bash
cp .env.example .env
docker compose -f compose-dev.yaml --env-file .env up -d postgres
./gradlew bootRun
```

Expected result:

```text
Tomcat started on port 8080
```

Verify health:

```bash
curl -s http://localhost:8080/actuator/health
```

Expected response:

```json
{"status":"UP"}
```

## Pre-Demo Automated Checks

Run the full backend verification suite:

```bash
./gradlew check
```

For a faster networking-focused check, run:

```bash
./gradlew test --tests "org.machikoro.server.config.HealthEndpointTest"
./gradlew test --tests "org.machikoro.server.config.WebSocketConfigTests"
./gradlew test --tests "org.machikoro.server.controller.WebSocketControllerTests"
./gradlew test --tests "org.machikoro.server.controller.GameControllerTest"
./gradlew test --tests "org.machikoro.server.controller.LobbyWebSocketControllerTest"
./gradlew test --tests "org.machikoro.server.service.GameSyncServiceTest"
./gradlew test --tests "org.machikoro.server.service.WebSocketConnectionTrackerTest"
```

Expected result:

```text
BUILD SUCCESSFUL
```

## Authentication Setup

Register or log in two players and capture their session tokens.

Register player A:

```bash
curl -s -X POST http://localhost:8080/auth/register \
  -H "Content-Type: application/json" \
  -d '{"username":"demo-a","password":"Password123!"}'
```

Register player B:

```bash
curl -s -X POST http://localhost:8080/auth/register \
  -H "Content-Type: application/json" \
  -d '{"username":"demo-b","password":"Password123!"}'
```

Login player A:

```bash
curl -s -X POST http://localhost:8080/auth/login \
  -H "Content-Type: application/json" \
  -d '{"username":"demo-a","password":"Password123!"}'
```

Login player B:

```bash
curl -s -X POST http://localhost:8080/auth/login \
  -H "Content-Type: application/json" \
  -d '{"username":"demo-b","password":"Password123!"}'
```

Expected response shape:

```json
{
  "sessionToken": "<opaque-token>",
  "username": "demo-a"
}
```

Store the returned values as:

```text
TOKEN_A=<player-a-session-token>
TOKEN_B=<player-b-session-token>
```

## Manual WebSocket Demo Flow

Use the Android client, Springwolf/STOMP tooling, or another STOMP client that
can set native `CONNECT` headers. The flow below is the required observable
sequence.

### 1. Connect Two Authenticated Players

Player A connects to:

```text
ws://localhost:8080/ws
```

with:

```text
Authorization: Bearer TOKEN_A
```

Player B connects to:

```text
ws://localhost:8080/ws
```

with:

```text
Authorization: Bearer TOKEN_B
```

Expected result:

- Both STOMP connections are accepted.
- A missing or invalid token is rejected during `CONNECT`.

### 2. Subscribe Both Players

Both clients subscribe to:

```text
/topic/public
/user/queue/errors
```

Player A creates a lobby:

```text
SEND /app/lobby.create
```

Payload:

```json
{
  "type": "JOIN",
  "sender": "demo-a"
}
```

Expected broadcast on `/topic/public`:

```json
{
  "type": "LOBBY_CREATED",
  "sender": "SERVER",
  "content": "Lobby created",
  "gameId": 1,
  "payload": {
    "lobbyCode": "ABC123",
    "hostUserId": 1,
    "status": "WAITING"
  }
}
```

Record the returned `gameId`.

Both clients subscribe to:

```text
/topic/game/{gameId}
/user/queue/game-sync
```

### 3. Join The Game Topic

Player A sends:

```text
SEND /app/chat.addUser
```

Payload:

```json
{
  "type": "JOIN",
  "sender": "demo-a",
  "gameId": 1
}
```

Player B sends:

```text
SEND /app/chat.addUser
```

Payload:

```json
{
  "type": "JOIN",
  "sender": "demo-b",
  "gameId": 1
}
```

Expected result:

- Both clients receive `JOIN` messages on `/topic/public`.
- The backend registers each STOMP session against the game.

### 4. Start The Game

Player A, as host, sends:

```text
SEND /app/game.start
```

Payload:

```json
{
  "gameId": 1
}
```

Expected broadcast on `/topic/game/{gameId}`:

```json
{
  "type": "GAME_STARTED",
  "sender": "server",
  "content": "Game 1 has started",
  "gameId": 1,
  "payload": {
    "game": {},
    "players": [],
    "playerCards": {},
    "turnOrder": []
  }
}
```

Checklist result:

- Player A receives the broadcast.
- Player B receives the same broadcast.
- Both clients transition to the game board using the same `GameStateDto`
  snapshot.

### 5. Verify Shared Game Broadcasts

The active player sends one game action. For dice rolling:

```text
SEND /app/game.rollDice
```

Payload:

```json
{
  "gameId": 1,
  "playerId": 1,
  "rollTwoDice": false
}
```

Expected broadcast on `/topic/game/{gameId}`:

```json
{
  "type": "ROLL_DICE",
  "sender": "SERVER",
  "content": "Player 1 rolled: 4",
  "payload": {
    "dice": [4],
    "total": 4
  }
}
```

Checklist result:

- Player A receives the `ROLL_DICE` event.
- Player B receives the same `ROLL_DICE` event.
- No game-state broadcast uses stale `/topic/public` routing for this action.

### 6. Verify Reconnect Sync

Disconnect player B's WebSocket session without ending the game.

Reconnect player B to:

```text
ws://localhost:8080/ws
```

with:

```text
Authorization: Bearer TOKEN_B
```

Player B subscribes again to:

```text
/topic/game/{gameId}
/user/queue/game-sync
/user/queue/errors
```

Player B explicitly requests a sync:

```text
SEND /app/game.sync
```

Payload:

```json
{
  "gameId": 1
}
```

Expected user-specific response on `/user/queue/game-sync`:

```json
{
  "type": "SYNC",
  "sender": "server",
  "content": "State sync for reconnecting player",
  "gameId": 1,
  "payload": {
    "targetUserId": 2,
    "targetSessionId": "<new-session-id>",
    "state": {
      "game": {},
      "players": [],
      "playerCards": {},
      "turnOrder": []
    }
  }
}
```

Checklist result:

- Only player B receives the sync message.
- The message is delivered through `/user/queue/game-sync`.
- The state payload reflects the persisted in-progress game.

### 7. Verify Backend Restart Recovery

With the game still in progress, restart the backend while keeping the database
volume alive.

For Docker Compose deployment:

```bash
docker compose restart backend
docker compose ps
```

For local `bootRun`, stop and start the application again:

```bash
./gradlew bootRun
```

After restart:

1. Player A reconnects with `TOKEN_A`.
2. Player B reconnects with `TOKEN_B`.
3. Both subscribe to `/topic/game/{gameId}` and `/user/queue/game-sync`.
4. Each client sends `/app/game.sync` with the active `gameId`.

Expected result:

- `/actuator/health` returns `UP`.
- Each player receives a `SYNC` snapshot from persisted database state.
- The game id, players, turn order, phase, and owned cards match the state from
  before restart.

## Production Deployment Verification (Railway)

The backend is deployed to **Railway** on the HTTPS domain
`machi-koro.up.railway.app`.

Production backend endpoint:

```text
https://machi-koro.up.railway.app
```

Production WebSocket endpoint:

```text
wss://machi-koro.up.railway.app/ws
```

Verify health:

```bash
curl -s https://machi-koro.up.railway.app/actuator/health
```

Expected response:

```json
{"status":"UP"}
```

Verify the deployment on Railway:

- In the Railway dashboard, confirm both the backend service and the managed
  PostgreSQL service show as deployed/healthy.
- Confirm the backend service is deploying `ghcr.io/se2-machi-koro/server:latest`
  (or the pinned `IMAGE_TAG`).

Expected result:

- The backend image resolves from `ghcr.io/se2-machi-koro/server:latest`.
- The backend service becomes healthy.
- `https://machi-koro.up.railway.app/actuator/health` returns `UP`.
- `wss://machi-koro.up.railway.app/ws` accepts authenticated STOMP connections.

> **Legacy (AAU):** The former AAU group 6 endpoint was
> `http://se2-demo.aau.at:53210` (WebSocket `ws://se2-demo.aau.at:53210/ws`),
> verified over SSH (`ssh grp-6@se2-demo.aau.at -p 53200`) and
> `docker compose ps`. This is no longer the active deployment.

## Pass Criteria

The demo passes when all of the following are true:

- Two authenticated players connect over `/ws`.
- Both players subscribe to `/topic/game/{gameId}`.
- A game action from one player is broadcast to both clients on
  `/topic/game/{gameId}`.
- A disconnected player reconnects and receives a `SYNC` snapshot through
  `/user/queue/game-sync` after sending `/app/game.sync`.
- Restarting the backend does not lose the persisted in-progress game state.
- `/actuator/health` returns `UP` locally and on the Railway deployment endpoint.
- The Railway HTTPS and WebSocket endpoints are reachable.
- The referenced smoke/unit/integration tests pass.

