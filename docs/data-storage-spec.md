# Data Storage Spec

DynamoDB design for the Kahoot-style quiz platform (target: 20,000 to 30,000 players in one game). Derived from the architecture document, `api.yaml` and the project plan. Four tables, two indexes, on-demand capacity.

Design rule: store only what placement, scaling, failure handling and results need. Gameplay state (players, answers, scores) lives in game server memory and never touches the database.

## Conventions

- Attribute names follow `api.yaml` camelCase (`serverId`, `wsUrl`). The architecture doc writes `serverID`, `wsURL`.
- Times are **epoch seconds (N)**. TTL requires it and conditions compare numerically.
- `status` and `ttl` are DynamoDB reserved words. Expressions below use `#st` and `#ttl` via `ExpressionAttributeNames`.
- **Condition expressions cannot do arithmetic.** The architecture's `reserved + :n <= capacity` is invalid. The reserve condition is `reserved <= :maxBefore`, where the caller computes `:maxBefore = capacity - n` from the item it just read. `capacity` never changes after registration, so this is safe.

## Tables

### Servers

PK `serverId` (S). One item per game server, a handful in total.

| Attribute | Type | Notes |
|---|---|---|
| `serverId` | S | EC2 instance ID |
| `address` | S | `ws://{publicIp}:8080`. Directory appends `/ws` to build `wsUrl` |
| `az` | S | Availability zone |
| `status` | S | `ACTIVE`, `DRAINING`, `DEAD` |
| `capacity` | N | Slots, e.g. 32000. Immutable |
| `reserved` | N | Slots held by active games. `0 <= reserved <= capacity` |
| `version` | S | Docker image tag |
| `heartbeatAt` | N | Written every 10 s. Fresh means within 30 s |
| `connected`, `gameCount` | N | Monitoring only |
| `emptySince` | N | Present only while `reserved = 0`. Set by Fleet Manager, removed by the reserve write |

| `status` | Meaning |
|---|---|
| `ACTIVE` | Heartbeating and accepting games |
| `DRAINING` | Chosen for scale-in. Takes no new games |
| `DEAD` | Missed heartbeats. Its games are `FAILED` |

The Directory and Fleet Manager read this table with a `Scan`.

### Games

PK `pin` (S, 6 digits).

| Attribute | Type | Notes |
|---|---|---|
| `pin` | S | `^[0-9]{6}$`, random |
| `gameId` | S | UUID. Key into `Results` |
| `quizId` | S | |
| `serverId` | S | Index key |
| `wsUrl` | S | Copied from the server so `GET /games/{pin}` is one read |
| `status` | S | Enum below. Index key |
| `maxPlayers` | N | 1 to 30000 |
| `reservedSlots` | N | Slots this game holds on its server: `maxPlayers` at creation, the player count after start, `0` once released |
| `hostTokenHash` | S | Hex SHA-256. The token itself is never stored |
| `createdAt` | N | Index sort key |
| `endedAt` | N | Set at the terminal transition |
| `playerCount` | N | Set at the terminal transition |
| `ttl` | N | Set at the terminal transition: `endedAt + 604800` (7 days) |

| `status` | Terminal | Set by |
|---|---|---|
| `CREATED` | no | Directory. Placed, host not connected |
| `LOBBY` | no | Game server. Host connected, players joining |
| `PLAYING` | no | Game server. Joins locked |
| `ENDED` | yes | Game server. Finished, results saved |
| `FAILED` | yes | Fleet Manager or server startup. Server died or restarted |
| `EXPIRED` | yes | Fleet Manager or game server. Host never claimed, or lobby idle |

```
CREATED --host_claim--> LOBBY --start--> PLAYING --final--> ENDED
CREATED --not claimed in t min--> EXPIRED      LOBBY --host end--> ENDED
LOBBY   --idle--> EXPIRED
CREATED|LOBBY|PLAYING --server dead or restarted--> FAILED
```

A game that ends because the host was gone for 10 minutes or ended it early is `ENDED`.

### Quizzes

PK `quizId` (S). One item per quiz: `title`, `questionCount`, `defaultTimeLimitSec`, `questions` (list of `{text, options[4], correctIndex, timeLimitSec}`). Seeded once with the 3-question quiz. `GET /quizzes` is a paged `Scan` that projects away `questions`.

### Results

PK `gameId` (S), SK `position` (N). The game server writes the **top 100 players** at game end (the API's `limit` maximum), not all 30,000.

| Attribute | Type | Notes |
|---|---|---|
| `gameId`, `position` | S, N | `position` is the display order, 1 to 100, unique |
| `username`, `score` | S, N | |
| `rank` | N | Competition ranking, ties share a rank (1, 2, 2, 4). That is why `rank` is not the key |
| `ttl` | N | Same value as the game's `ttl` |

`GET /games/{pin}/results` reads `Games[pin]` (for `gameId`, `quizId`, `endedAt`, `playerCount`), then runs one `Query`: `gameId = :g AND position BETWEEN 1 AND :limit`.

## Secondary indexes (Games)

Both use projection `ALL`.

| Index | Partition key | Sort key | Used for |
|---|---|---|---|
| `ServerIdIndex` | `serverId` | `createdAt` | Server startup: fail the old games of this server. Fleet Manager: fail a dead server's games |
| `StatusCreatedAtIndex` | `status` | `createdAt` | Fleet Manager: `status = CREATED AND createdAt <= now - t` to expire unclaimed games |

GSIs are eventually consistent, so a sweep may read a stale row. This is safe because every sweep write is a conditional write on the base table that re-checks `status`.

## Conditional writes

`:now` is the caller's epoch seconds.

### W1. Atomic reserve and PIN uniqueness (Directory, `POST /games`)

One `TransactWriteItems`, so a Lambda crash cannot leak a reservation without a game.

```
Update Servers[serverId]
  SET reserved = reserved + :n REMOVE emptySince
  Condition: #st = :active AND heartbeatAt >= :cutoff AND reserved <= :maxBefore

Put Games[pin]   (status CREATED, reservedSlots = :n, no ttl)
  Condition: attribute_not_exists(pin)
```
`:n = maxPlayers`, `:cutoff = now - 30`, `:maxBefore = capacity - :n`. Best-fit selection happens client-side on the `Scan`. This write only has to be safe, not optimal.

On `TransactionCanceledException`, check `CancellationReasons`:
- Action 1 failed (full, stale heartbeat, or no longer ACTIVE): re-Scan, try the next server. If none fit, return 503 `NO_CAPACITY` and invoke Fleet Manager.
- Action 2 failed (PIN taken): new PIN, retry, up to 5 times.

### W2. Server registration (game server, startup)

1. Query `ServerIdIndex` for this `serverId`. For each game in `CREATED`, `LOBBY` or `PLAYING`, run the **games-only fail write** (W7b).
2. `PutItem Servers` with `status = ACTIVE`, `reserved = 0`, `capacity`, `address`, `az`, `version`, `heartbeatAt = now`. **No condition**, on purpose: a restarted process must overwrite its old row.

Failing the old games first means nothing placed after the put is touched.

### W3. Host claim (game server)

```
Games UpdateItem  SET #st = :lobby
Condition: serverId = :me AND #st = :created
```
The server first does a `GetItem` and checks `hostTokenHash`. A host reconnecting in `LOBBY` or `PLAYING` only does the `GetItem`. If the expiry sweep (W5) and the claim race, the `#st = :created` condition lets exactly one win.

### W4. Start: shrink the reservation (game server)

```
TransactWriteItems
  Games   UpdateItem  SET #st = :playing, reservedSlots = :n
          Condition: serverId = :me AND #st = :lobby AND reservedSlots = :max
  Servers UpdateItem  SET reserved = reserved - :delta       (:delta = :max - :n)
          Condition: reserved >= :delta
```
`reservedSlots = :max` makes the shrink happen once, even if the call is retried.

### W5. Reservation release (game server at end, Fleet Manager on expiry)

First `BatchWriteItem` the top 100 into `Results`. Batch writes cannot carry conditions, but keys are deterministic so a retry writes identical data. Then:

```
TransactWriteItems
  Games   UpdateItem  SET #st = :terminal, endedAt = :now, playerCount = :pc,
                          reservedSlots = :zero, #ttl = :ttl
          Condition: serverId = :sid AND #st IN (:from...) AND reservedSlots = :held
  Servers UpdateItem  SET reserved = reserved - :held
          Condition: reserved >= :held
```

| Caller | `:terminal` | `:from` |
|---|---|---|
| Game server, game over or host ended | `ENDED` | `LOBBY`, `PLAYING` |
| Game server, lobby idle | `EXPIRED` | `LOBBY` |
| Fleet Manager, host never claimed | `EXPIRED` | `CREATED` |

`:held` is the `reservedSlots` value the caller read. Whoever commits first sets it to 0, so the other caller fails its condition and the slots are released exactly once. Results are written first, so `status = ENDED` always means results exist.

### W6. DRAINING transition (Fleet Manager, scale-in)

```
Servers UpdateItem  SET #st = :draining
Condition: #st = :active AND reserved = :zero AND emptySince <= :cutoff
```
W1 requires `#st = :active` and this requires `reserved = 0`. Both hit the same item, which DynamoDB serializes, so a game is never placed on a server being removed. After the instance is terminated, remove the row: `DeleteItem Servers[serverId]` with condition `#st = :draining AND reserved = :zero`.

`emptySince` is set separately by the Fleet Manager:
```
Servers UpdateItem  SET emptySince = :now
Condition: #st = :active AND reserved = :zero AND attribute_not_exists(emptySince)
```
W1 removes it again on the next reserve.

### W7. Dead server (Fleet Manager)

**(a)** Mark the server:
```
Servers UpdateItem  SET #st = :dead
Condition: #st IN (:active, :draining) AND heartbeatAt < :cutoff
```
A heartbeat that lands just before this write wins and the server stays alive. The Fleet Manager then terminates the instance.

**(b)** Games-only fail write, for each game from `ServerIdIndex` (also used by W2):
```
Games UpdateItem  SET #st = :failed, endedAt = :now, reservedSlots = :zero, #ttl = :ttl
Condition: serverId = :sid AND #st IN (:created, :lobby, :playing)
```
No `Servers` decrement here, because the server row is dead or about to be overwritten.

### W8. Heartbeat (game server, every 10 s)

```
Servers UpdateItem  SET heartbeatAt = :now, connected = :c, gameCount = :g
Condition: attribute_exists(serverId)
```
Stops a late heartbeat from re-creating a row the Fleet Manager just deleted.

## Capacity and TTL

- **On-demand** for all tables and indexes. Load is bursty (game creation, join storm, idle) and the Learner Lab runs for short sessions, so there is nothing to size. API Gateway throttles at about 2,000 req/s, well inside default on-demand limits.
- **Hot item watch.** In a 30,000-player game every player calls `GET /games/{pin}`, which reads one item. Use a strongly consistent read so players see a game the host just created. About 2,000 req/s is under the roughly 3,000 reads per second one item supports. One `Servers` row takes roughly 500 W1 transactions per second. Both are above our test load.
- **TTL** is enabled on `Games` and `Results`, set only at the terminal transition (7 days). Active games have no `ttl`; the sweeps in W5, W6 and W7 guarantee they reach a terminal state. TTL deletion can lag up to 48 hours, which only means an old ended game lingers.

## Invariants to test

1. For each server, `reserved` equals the sum of `reservedSlots` over its non-terminal games, and `0 <= reserved <= capacity`.
2. A terminal game has `reservedSlots = 0`.
3. An `ENDED` game has Results rows.
4. Under concurrent `POST /games`, reservations never exceed `capacity`, and no two games share a PIN.

## Differences from the architecture document

- **Reserve condition:** no arithmetic in DynamoDB conditions, so `reserved <= :maxBefore` (see Conventions).
- **Results key:** `gameId + position` instead of `gameId + rank`, because tied players share a rank and would overwrite each other.
- **Extra Games attributes:** `reservedSlots` (makes release happen exactly once), `endedAt`, `playerCount`. No public response changes.
- **Transactions:** reserve with create (W1), start (W4) and end (W5) are `TransactWriteItems`. `api.yaml` `x-internal` lists them as separate `UpdateItem` calls. **Hao needs to agree before 11/27.**
- **Servers `az`** and `address = ws://ip:8080` follow the document. The doc's "DRAINING servers that still have games" case cannot occur here, since W6 only drains an empty server and W1 refuses to place on it.
- **Top-100 results:** the doc says "the top results". The count of 100 matches the API `limit` maximum. Players outside the top 100 cannot look up their final rank afterwards, which is accepted for this project.
