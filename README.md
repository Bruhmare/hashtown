# HashTown

**Play:** https://www.hashtown.io · **X:** [@hashtown_gpu](https://x.com/hashtown_gpu) · **Chain:** Robinhood Chain (4663)

HashTown is a 2D top-down pixel-art GPU mining tycoon that plays like a small RPG town. You walk a
character through the streets, step inside real shops, talk to the keepers, carry hardware back to your
garage, place it on a grid, wire up power and watch the rigs light up and mine **HTC**. Signed-in players
deposit HTC into the **Exchange pool** and receive a share of real **tokenized NVDA** every hour, paid
from an on-chain treasury and withdrawable to their own wallet.

> This repository is the public technical overview. The game client and the API are closed source.
> Everything described here is running in production.

| Doc | Contents |
| --- | --- |
| [ARCHITECTURE.md](ARCHITECTURE.md) | System diagram and how each subsystem behaves |
| [API.md](API.md) | The HTTP surface the client talks to |
| [ON-CHAIN.md](ON-CHAIN.md) | Treasury, token and asset addresses, verifiable on-chain |

---

## Stack

| Layer | Tech |
| --- | --- |
| Client | React 19, TypeScript 6, Vite 8, a hand-written Canvas 2D engine (no game framework) |
| Art | Procedurally generated pixel sprites, tiles and interiors at 16 px, plus hand-drawn NPC portraits |
| API | Node 20, Express 4, `pg`, `jsonwebtoken`, ethers v6 |
| Database | PostgreSQL |
| Chain | Robinhood Chain mainnet (Arbitrum Orbit L2, chain ID 4663), ERC-20 transfers |
| Auth | Sign in with X (OAuth 2.0 + PKCE), 30-day signed session tokens |
| Hosting | Vercel (client, edge-cached static build) · Railway (API + managed Postgres) |

## Client engine

The game is a single `requestAnimationFrame` loop that owns input, simulation and rendering; React only
draws the overlays (HUD, shops, exchange desk, inventory, dialogs). The engine never imports React, and
React never mutates game state directly.

- **One state owner.** A single store module holds all game state and exposes actions. UI and engine
  subscribe to it; nothing else writes. That one seam is what the cloud-save layer plugs into.
- **Data-driven content.** Every GPU, generator, panel, turbine, rack and drink is a row in one catalog
  (tier, level lock, power draw, hash rate, footprint, animation frames). Balance changes are a data
  edit, not a code change, and the same catalog is exported for the server to validate saves against.
- **Pixel-perfect scaling.** The camera zoom is quantised to 1/16 steps so 16 px tiles always land on
  whole screen pixels at any window size. No shimmer, no blurry seams.
- **World simulation.** A real-time day/night cycle (multiply-blended dusk, night and dawn with light
  glows), drifting wind that changes turbine output, solar that follows the sun, and a road network with
  cars that turn at junctions, follow each other and knock the player over if they step into traffic.
- **Walkable interiors.** Every shop is a real room you walk into through its door, with its own keeper,
  dialogue and stock. A late-night dealer only opens between 22:00 and 04:00 in game time.
- **Power model.** Rigs draw watts; generators, solar and wind supply them. Solar and wind are grid-tied
  and pooled, and running on clean power earns an eco bonus on the hash rate.
- **No offline mining.** The simulation tick is skipped while `document.hidden`, so mining only happens
  while you are actually playing. The server's mining allowance follows recorded play time to match.
- **Versioned saves.** Saves carry a schema version with forward migrations, so renamed items and
  reworked systems never break an old save.

## Backend

### Auth and cloud saves

Sign in with X runs the OAuth 2.0 authorization-code flow with PKCE and returns to whichever allowed
origin started it. The API issues a signed session token; one save per user lives in Postgres. The client
syncs every 30 s, on tab hide and on demand. Logging in on a device with existing local progress shows a
cloud-vs-local chooser instead of silently overwriting either.

### Server-side save validation

The browser owns the game loop, so the server treats every uploaded save as untrusted input. Each upload
is diffed against the last accepted save and the item catalog, and rejected (`409`) if it describes a
state that could not have been reached legitimately. An honest player who trips a check recovers on the
next sync, because deltas are always measured from the last accepted state. Only accepted saves can
deposit into the pool, which is the boundary that protects real funds. Save routes are rate limited.

### Exchange pool

HTC deposited into the pool is debited from the saved game server-side, and contributions never reset.
Every hour the period's NVDA is split pro rata by `contribution / totalPool`. Distributions are keyed by
period, so a crashed or repeated run can never pay the same hour twice.

### Treasury scheduler

Payouts are funded by a public treasury wallet. Every top-up is detected as new money and scheduled as
its own tranche, paid in equal hourly slices over the next 24 hours, so a deposit at any time of day
streams out evenly instead of landing in one block. Tranches stack, advance inside the same database
transaction as each distribution, and shrink proportionally if the treasury ever comes up short. NVDA
already owed to players is excluded from what counts as available, and a payout that cannot read the
treasury is retried rather than skipped.

### Withdrawals

Withdrawals are fully server-authoritative. A request validates the player, the amount, the EIP-55
checksummed address, the network, per-request and daily limits, treasury liquidity and an idempotency
key, then moves funds `available → pending` in a single atomic statement. A background worker drives the
state machine:

```
REQUESTED → VALIDATING → QUEUED → PROCESSING → SUBMITTED → CONFIRMED
                                        └──────────────→ FAILED / CANCELLED  (funds returned)
```

A withdrawal that already has a transaction hash is never re-broadcast, and a submission whose outcome is
unknown is parked for manual resolution instead of guessed at. That is the rule that makes double sends
impossible across crashes and restarts.

### Ledger

Every balance change (deposit, payout, spend, withdrawal, refund) is an append-only ledger row. Balances
can be rebuilt from the ledger at any time, and players can read their own history.

### Chain providers

The exchange talks to the chain through a provider interface with two implementations: a sandbox chain
backed by Postgres (fake hashes, simulated reverts, timeouts and empty-treasury cases for testing) and
the real ERC-20 provider on ethers. The real provider only starts when every required setting is present
and explicitly enabled, and moving a database from sandbox to real is a one-way migration: a sandbox boot
against a real database halts the exchange instead of paying out.

## Testing

| Suite | What it covers |
| --- | --- |
| Exchange (58 checks) | Idempotency, double clicks, treasury budget, refunds, timeouts, limits, admin gating |
| Restart | In-flight withdrawals surviving an API restart mid-state-machine |
| Payout schedule (15 checks) | Tranche maths, stacking top-ups, deficits, rescheduling, offline |
| Save validation | Accept/reject cases for every guard, including recovery after a rejection |
| Real-mode end to end (37 checks) | Full flow against a fork of Robinhood Chain mainnet |
| Live smoke (14 checks) | Production config and guards, read-only, moves no tokens |
| Live withdrawal (9 checks) | A real mainnet NVDA withdrawal from pool deposit to on-chain confirmation |

Gameplay is verified with Playwright driving the real client (keyboard and mouse input, CDP screencast
capture), which is also how the trailer footage was recorded.

## Numbers

- 13 GPU tiers, from the Starter card (500 HTC) to the Hashcore Titan (10M HTC, 2,500 HTC/min)
- Generators from 500 W up to industrial scale, plus solar and wind
- Base plots from 10×10 to 30×30, plus the Industrial District
- 1 hour real-time day/night cycle
- A payout costs about 75k gas on Robinhood Chain, which is a fraction of a cent, and confirms in seconds
