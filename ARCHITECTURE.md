# Architecture

## Overview

NERVE is a single Node.js process that owns one live simulation. Every
connected browser is a thin client: it renders whatever state the server
pushes and sends small command packets back. There is no client-side
simulation logic and no database — state lives in memory in `server.js`.

```
server.js
├── Core algorithms       MinHeap, seededRandom, dijkstra, pathFromSource
├── World generation      buildWorld() — villages, hospitals, road graph
├── Simulation logic      spawnRequest, attemptDispatch, processQueue,
│                         updateAmbulances, updateFacilities, simTick
├── Shared server state   world, running, speedMult, tick loop
└── HTTP + WebSocket       Express static server + `ws` server at /ws
```

## World Model

- **Nodes**: 90 villages + 14 hospitals, placed on a 2D plane.
- **Roads**: each node connects to its 4 nearest neighbors (`K_NEAREST`),
  with weighted edge costs (Euclidean distance × a randomized 0.75–1.45
  multiplier, so the graph isn't purely geometric). A connectivity pass
  guarantees the whole graph is reachable from any node.
- **Hospitals**: each has a bed capacity, a set of on-duty specialties, and
  six medicine categories with independent stock levels that deplete on use
  and periodically restock.
- **Ambulances**: one per hospital plus a few placed at random villages,
  each with a status (`idle`, `to_patient`, `to_hospital`, `returning`).

## Dispatch Algorithm

For every pending request, on every tick:

1. Run Dijkstra from the request's village to get shortest-path distances to
   every node (cached per village per tick, since multiple requests from the
   same village shouldn't repeat the search).
2. Filter to hospitals with free beds, reachable given current road closures.
3. Prefer hospitals with the needed specialty on duty; fall back to
   trauma/general medicine if none is available, then to any reachable
   hospital as a last resort — logged either way.
4. Score each candidate:
   ```
   cost = travel_distance
        + (occupied_beds × 5 + queue_count × 6)      // wait penalty
        + (medicine_stock <= 2 ? 45 : 0)              // stock-risk penalty
   ```
5. Pick the lowest-cost hospital, pick the nearest idle ambulance to the
   village, build the full route (ambulance → village → hospital) by
   concatenating two Dijkstra path lookups, and log the decision — including
   the runner-up's cost, so every dispatch is auditable after the fact.

If no ambulance is idle or no hospital has a free bed, the request stays
queued and its wait time accrues; critical requests that blow past their SLA
budget are logged and counted.

## WebSocket Protocol

All messages are JSON. The server is authoritative — clients never mutate
simulation state directly, only request changes.

**Server → client**

| `type` | Sent | Payload |
|---|---|---|
| `state` | once per tick, broadcast to all clients | full world snapshot: nodes, roads, ambulances, pending queue, active dispatches, decision log, metrics, `running`, `speedMult` |
| `meta` | once per connection, and again after a reset | `specialties`, `urgencies`, and the current `villages` list (id + name) — used to populate the request form |
| `error` | to the sending client only, on a rejected packet | a short human-readable reason |

**Client → server**

| `type` | Payload | Effect |
|---|---|---|
| `toggle` | — | play/pause the shared simulation |
| `speed` | `{value: 0.5–4}` | set tick rate for everyone |
| `spawn` | — | force one randomly-generated emergency |
| `closure` | — | force one random road closure |
| `reset` | — | rebuild the world from scratch |
| `custom_request` | `{villageId, specialty, urgency}` | validated, then injected into the live dispatch queue as a real pending request |

## Input Validation

`custom_request` is the one packet type that carries free-form input, so it's
validated server-side against the live world before touching simulation
state (`submitUserRequest` in `server.js`):

- `villageId` must be an integer that resolves to an actual village node
  (not a hospital, not out of range).
- `specialty` must be one of the six known specialties.
- `urgency` must be one of `Critical` / `Urgent` / `Routine`.
- A hard cap on total pending requests guards against queue-flooding.

Every inbound WebSocket message additionally goes through basic packet
hygiene before it's even parsed for `type`: a size ceiling (4KB) and a
per-connection rate limit (20 messages / 5s), so no single client can flood
the shared simulation for everyone else.

## Why Server-Authoritative?

The original version of this project ran the entire simulation client-side —
each browser tab generated and simulated its own private world. That's fine
for a single-user demo, but it means two people looking at "the same"
dashboard were actually watching two unrelated simulations. Moving the sim
to the server makes NERVE behave like the dispatch system it's modeling:
one shared, consistent picture of the world that every viewer and operator
sees and can act on together.
