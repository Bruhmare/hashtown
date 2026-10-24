# API

The game client talks to a small JSON API. Player routes need a session token from X sign-in.

| Method | Path | What it does |
| --- | --- | --- |
| GET | `/health` | Service status |
| GET | `/auth/x/start` | Starts X sign-in |
| GET | `/auth/x/callback` | X sign-in return |
| GET | `/api/me` | The signed-in player |
| GET / PUT / DELETE | `/api/save` | Read, upload (anti-cheat checked) or remove the cloud save |
| GET | `/api/exchange` | Pool size, your share, NVDA balances, next payout |
| POST | `/api/pool/deposit` | Deposit HTC into the pool |
| POST | `/api/nvda/spend` | Spend NVDA in-game (the Industrial District) |
| GET / POST | `/api/withdrawals` | List or request NVDA withdrawals |
| GET | `/api/withdrawals/:id` | Track one withdrawal |
| POST | `/api/withdrawals/:id/cancel` | Cancel a withdrawal that has not been sent |
| GET | `/api/ledger` | Your full balance history |

Admin routes for operations (treasury status, the withdrawal queue) exist and are limited to the
developer account.
