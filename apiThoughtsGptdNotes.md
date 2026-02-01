# DeeDay2026
## Backend goals (what the server must guarantee)

1. **Authoritative world state**
   One canonical infection state per region (and per “layer” if you add variants later).

2. **Deterministic ticking**
   Spread runs on a schedule (e.g., every 5 minutes). A tick produces:

   * new `region_state`
   * a compact **diff** for clients

3. **Action evaluation + anti-abuse**
   Deploys (chem mixes) are validated, evaluated, recorded append-only, and applied as controlled deltas.

4. **Realtime fanout**
   Clients receive diffs fast (WebSocket). They should never need to poll except for initial load/reconnect.

---

## Recommended stack (simple and solid)

* **Node + Express (TS)** for HTTP
* **WebSocket server** (ws) or Socket.IO (ws is leaner)
* **MongoDB replica set** (since you’re already thinking that way): enables transactions if you want them, and gives you more safety for writes.
* **BullMQ + Redis** (optional, later) for scheduled ticks and background processing, but you can start with a single in-process scheduler and move later.

If you’re single-VPS day 1, do:

* One API process (PM2)
* One “tick worker” process (PM2) OR run tick inside the API with a leader lock in DB

---

## Data model (backend-first)

### 1) Regions (static-ish)

Store once; serve to clients as GeoJSON.

* `regions`

  * `_id` = regionId (string)
  * `name`
  * `level` (`country|province`)
  * `centroid` (lon/lat)
  * `neighbors` (string[])
  * `geojson` (optional in DB; or store on disk + CDN)

**Tip:** Keep geometry out of hot path. Clients rarely need it again once cached.

### 2) Region state (hot path)

* `region_state`

  * `regionId` (unique)
  * `infectionBp` (0..10000) // basis points (2 decimals)
  * `variant` (optional later)
  * `resistance` (small object, e.g. `{ antiviral: 0.2, antibiotic: 0.1 }`)
  * `lastTick` (int tick number)
  * `updatedAt`

Indexes:

* unique on `regionId`
* index on `updatedAt` for monitoring

### 3) Tick log (append-only, for replay/debug)

* `ticks`

  * `tickId` (monotonic)
  * `startedAt`, `endedAt`
  * `seed` (string/int) for deterministic RNG
  * `diff` (compressed updates: array of `{regionId, infectionBp}`)
  * `stats` (counts, avg infection, etc.)

Index:

* unique on `tickId`

### 4) Actions (deploys) (append-only, critical)

* `actions`

  * `actionId`
  * `playerId` (or wallet)
  * `regionId`
  * `mix` (e.g. `{ compoundA: 35, compoundB: 65 }`)
  * `submittedAt`
  * `tickIdApplied` (which tick it applied in)
  * `result` (`success|fail`)
  * `deltaBp`
  * `serverProof` (hash of payload + secret, if you want tamper-evidence)

Indexes:

* `regionId + submittedAt`
* `playerId + submittedAt`
* **unique** `refId` if you do idempotency (recommended)

### 5) Cooldowns / anti-spam

* `player_cooldowns`

  * `playerId`
  * `nextAllowedAt`
  * `rateWindowCount`, etc.

---

## API surface (MapLibre friendly)

### Public (no auth) – MVP

1. **GET `/v1/regions`**

* returns FeatureCollection with `id = regionId`
* clients add it once to MapLibre source

2. **GET `/v1/state/snapshot`**

* returns compact map of infection values:

```json
{
  "tickId": 1280,
  "updatedAt": "2026-02-02T00:00:00Z",
  "infection": { "AU-NSW": 1320, "US-CA": 845, "...": 9999 }
}
```

3. **GET `/v1/ticks/latest`** (optional)

* current tickId + next tick time (nice for UI countdown)

### Auth-required (token mode) – Deploy flow

4. **POST `/v1/deploy`**
   Request:

```json
{
  "refId": "uuid-or-client-generated",
  "regionId": "AU-NSW",
  "mix": { "compoundA": 40, "compoundB": 60 }
}
```

Response:

```json
{
  "ok": true,
  "tickId": 1280,
  "result": "success",
  "deltaBp": -220,
  "newInfectionBp": 1100,
  "fx": { "kind": "ripple", "strength": 0.7 }
}
```

**Important:** Make this endpoint idempotent using `refId` unique index.

### Realtime

5. **WS `/v1/realtime`**
   Server emits:

* `snapshot` on connect (or tell client to fetch snapshot)
* `state` diffs after each tick
* `fx` events on deploy outcomes

Example messages:

```json
{ "t":"state", "tickId":1281, "updates":[["AU-NSW",1100],["US-CA",900]] }
{ "t":"fx", "kind":"ripple", "regionId":"AU-NSW", "strength":0.7 }
```

Use tuple arrays for compactness.

---

## Tick engine (the heart of the backend)

### Requirements

* Runs every N minutes.
* Guarantees **only one** tick runs at a time (leader/lock).
* Produces a diff and persists it.
* Broadcasts diff to WS clients.

### “Single VPS” leader lock (Mongo pattern)

Use a `locks` collection:

* `_id: "tick"`
* `expiresAt`
* `holderId`

Algorithm:

1. Try `findOneAndUpdate` with condition `expiresAt < now` to acquire
2. If acquired, run tick
3. Update `expiresAt` periodically while running (or set generous TTL)
4. Release at end

This makes it safe even if you run multiple API instances later.

### Tick computation pattern

**Don’t update region docs one-by-one with separate writes.**
Do:

* Load state needed (infection + neighbors)
* Compute next infection in memory
* Bulk write with `bulkWrite`
* Store diff in `ticks`
* Broadcast updates

Pseudo flow:

1. `tickId = lastTickId + 1`
2. Load `region_state` for all regions (or chunked)
3. For each region:

   * compute growth + neighbor pressure
   * apply decay from recent successful deploys
   * apply mutation/resistance shifts
   * clamp
4. Build `updates = [[regionId, infectionBp], ...]` only where changed > threshold
5. Bulk update `region_state`
6. Insert `ticks` record with `diff=updates`
7. WS broadcast `{t:"state", tickId, updates}`

### Keeping it fast

* Keep `neighbors` in memory (cache) rather than DB lookups per tick.
* Pre-store neighbor list in `regions` and load once at boot.
* If region count becomes huge, process in chunks and stream diffs.

---

## Deploy evaluation (server-side detailed)

You need:

* Input validation
* Anti-spam + cooldown
* Deterministic scoring
* Bounded effect sizes (so one action can’t zero a country)
* Side effects on fail (accelerate spread)

### Validation rules (baseline)

* `regionId` exists
* `mix` components are known compound ids
* percentages sum to 100 (or 10000)
* max number of components (e.g. 4)
* enforce per-player cooldown (e.g. 30s / 60s)
* enforce per-region action cap per tick (optional)

### Scoring

Store compounds in config:

* `compounds`

  * `id`
  * `tags` (antiviral, antibiotic, etc.)
  * `basePower`
  * `synergies` (optional later)

Region has `resistance` per tag.
Compute:

* `mixVector = sum(pct * compoundTagVector)`
* `effective = dot(mixVector, (1 - resistance))`
* Overmix penalty if too many components
* Randomness seeded by `(tickId, regionId, playerId, refId)` so it’s reproducible

Outcome:

* if `effective > threshold` => success
* else fail

Delta bounds:

* success delta `[-50bp .. -500bp]` depending on effective
* fail delta `[+10bp .. +200bp]` plus temporary resistance increase

Apply:

* update region infection immediately OR queue for next tick

  * MVP: apply immediately (feels responsive)
  * Later: “appliedAtTick” if you want tick-atomic consistency

**Recommendation for game feel:** apply immediately but also record `tickIdApplied = currentTickId`.

### Concurrency + idempotency

* `actions.refId` unique
* On duplicate refId, return the original action result
  This prevents double-spends from retries.

---

## Realtime design (don’t DDoS yourself)

### Clients should receive:

* tick diff every N minutes (small)
* deploy fx events (small)

### Avoid:

* sending full snapshot every tick
* broadcasting huge JSON every second

Compression:

* Use tuple arrays
* Optional: `permessage-deflate` WebSocket compression

Backpressure:

* If a client can’t keep up, drop it or send periodic snapshots only.

---

## Security / integrity (MVP sensible)

* Auth: token-based (JWT) as you’re already doing in other systems
* Rate limit deploy endpoint (IP + player)
* Validate region IDs strictly (no arbitrary polygon injection)
* Sign “deploy result” payload with server secret hash (cheap tamper-evidence)

If you go “on-chain verifiable” later, your tick log + action log becomes the audit trail.

---

## Operational notes (single VPS)

Run 2 PM2 processes:

1. `api` (HTTP + WS)
2. `tick-worker` (leader lock + tick loop)

Even if worker dies, API stays alive. If API restarts, clients reconnect.

Mongo:

* single-node replica set (good call) for reliability and future transactions.

---

## What I need from you (so the write-up matches your build)

Answer these in one message (short is fine) and I’ll tailor the final spec + endpoint contracts:

1. Region granularity: **countries only** or **countries + provinces**?
2. Tick rate: stick to **5 minutes** or faster?
3. Deploy applies **immediately** or at **next tick**?
4. Auth: wallet login (nonce/JWT) like your other API, or guest accounts?

If you don’t want to answer, I’ll assume: countries+provinces, 5 minutes, immediate apply, JWT wallet login.
