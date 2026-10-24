# Architecture

```
 Browser (game)                 HashTown API                     Robinhood Chain
 ───────────────                ────────────                     ───────────────
 canvas engine, shops,   ──►    X sign-in, cloud saves    ──►    treasury wallet
 rigs, mining loop              anti-cheat, Exchange pool         tokenized NVDA (ERC-20)
 local save + sync              payout scheduler, withdrawals     player wallets
                                Postgres
```

## The game

The game runs in the browser: a canvas engine renders the town, shops, interiors, traffic and the
player's base, and mines HTC while the tab is open. There is no offline mining; the miner pauses when
the tab is hidden. Guests play with a local save.

## Accounts and cloud saves

Players sign in with X (OAuth 2.0 with PKCE). The save is then stored on the server and synced every
30 seconds and whenever the tab is hidden. Signing in on a device that already has progress offers a
choice between the cloud save and the local one.

## Anti-cheat

Because the browser runs the game, the server never takes a save on trust. Every upload is checked
against the last accepted save and the item catalog: wealth, mining speed against play time and
purchases all have to add up. A save that fails is rejected; the game keeps running locally and syncs
again once a valid state arrives. Only accepted saves can deposit into the pool.

## The Exchange pool

Signed-in players deposit HTC into the pool. Every hour the period's NVDA is split between players by
their share of the pool. Each payout period is recorded once, so a retry never pays twice.

## Treasury payouts

The NVDA comes from a public treasury wallet. Every top-up of the treasury is paid out in equal hourly
slices over the next 24 hours: send 24 NVDA at any time and the next 24 payouts carry 1 NVDA each. NVDA
already owed to players never counts as available, and a payout that cannot read the treasury is
retried, never skipped.

## Withdrawals

Withdrawals are handled by the server only. A request checks the player, the amount, the address, the
limits, the treasury balance and a request ID that stops double submits, then moves the amount from
available to pending in one step. A worker carries it through
`requested → queued → submitted → confirmed`, and a failure returns the funds. Every balance change is
written to an append-only ledger.
