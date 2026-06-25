# Backend WebSocket Protocol (Superseded)

> **This document has been superseded by
> [websocket-game-protocol.md](websocket-game-protocol.md), which is the current
> single source of truth for the STOMP protocol.**

This file was a point-in-time snapshot verified against `SE2-Machi-Koro/Server`
at commit `f0b5ccf` for the Sprint 3 review. The server has since moved well
beyond that snapshot, so the descriptions that used to live here are no longer
accurate. Notable drift:

- `/app/game.start`, `/app/game.rollDice`, `/app/game.purchase`, and
  `/app/game.sync` are now implemented. This snapshot described them as "not
  present."
- `/app/game.advancePhase` is now a **rejected compatibility endpoint** that
  always replies with `DIRECT_PHASE_ADVANCE_FORBIDDEN` on `/user/queue/errors`.
  Turn phases advance only as a result of `rollDice`, `resolveEffects`, and
  `endTurn`; the endpoint cannot skip required turn actions. This snapshot
  documented it as a working phase-advance command.
- `/app/game.leave` **never existed** on the server. Only `/app/lobby.leave`
  does. (See the related Server-repo issue about the dead `ensureSenderOwnsPlayer`
  guard: SE2-Machi-Koro/Server#421.)
- Several destinations added since this snapshot — `game.enterScreen`,
  `game.rerollDice`, `game.reportCheat`, `game.accuse`, and `game.chat.send` —
  are documented in the current contract.

For the authoritative STOMP contract — connection, subscriptions, send
destinations, message envelopes, error codes, the cheat/accusation protocol, and
the reconnect/sync flow — see
**[websocket-game-protocol.md](websocket-game-protocol.md)**.
