# HashTown

**Play:** https://www.hashtown.io · **X:** [@hashtown_gpu](https://x.com/hashtown_gpu)

HashTown is a 2D pixel-art GPU mining tycoon. You walk around a small town, buy hardware in real
shops, wire up rigs in your garage and mine **HTC**, the in-game coin. Signed-in players put their HTC
into the **Exchange pool** and receive a share of real **tokenized NVDA** on Robinhood Chain every hour,
which they can withdraw to their own wallet.

This repository documents how the game and its backend work. The source code is kept private.

| Doc | What's in it |
| --- | --- |
| [ARCHITECTURE.md](ARCHITECTURE.md) | How the game, saves, pool and payouts fit together |
| [API.md](API.md) | The endpoints the game client talks to |
| [ON-CHAIN.md](ON-CHAIN.md) | Token, treasury and payout addresses you can check yourself |

## In numbers

- 13 GPUs, from the Starter card (500 HTC) to the Hashcore Titan (10M HTC, 2,500 HTC/min)
- Generators, solar and wind power with a day/night cycle and drifting wind
- Base plots from 10×10 up to 30×30, plus the Industrial District
- Hourly NVDA payouts from the treasury, spread evenly over 24 hours after each top-up
