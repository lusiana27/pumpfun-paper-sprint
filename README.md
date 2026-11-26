# Pump.fun Paper Sprint — Net-Flow Surge Sniper

[![make.fun](https://make.fun/badge/u-boothlord-pump-fun-paper-sprint-any-bot-can-f83c389b.svg)](https://make.fun/bounty/u-boothlord-pump-fun-paper-sprint-any-bot-can-f83c389b?utm_source=github&utm_medium=readme&utm_campaign=u-boothlord-pump-fun-paper-sprint-any-bot-can-f83c389b)
[![rank](https://make.fun/badge/u-boothlord-pump-fun-paper-sprint-any-bot-can-f83c389b/lusiana27/pumpfun-paper-sprint/rank.svg)](https://make.fun/bounty/u-boothlord-pump-fun-paper-sprint-any-bot-can-f83c389b?utm_source=github&utm_medium=readme&utm_campaign=u-boothlord-pump-fun-paper-sprint-any-bot-can-f83c389b)
[![reward](https://make.fun/badge/u-boothlord-pump-fun-paper-sprint-any-bot-can-f83c389b/lusiana27/pumpfun-paper-sprint/reward.svg)](https://make.fun/bounty/u-boothlord-pump-fun-paper-sprint-any-bot-can-f83c389b?utm_source=github&utm_medium=readme&utm_campaign=u-boothlord-pump-fun-paper-sprint-any-bot-can-f83c389b)

A paper-trading bot for the [Pump.fun Paper Sprint](https://make.fun/bounty/u-boothlord-pump-fun-paper-sprint-any-bot-can-f83c389b) bounty on make.fun: it replays 6 hours of real pump.fun bonding-curve data, and the highest net PnL wins.

## Strategy in one paragraph

Most pump.fun coins bleed slowly and die, but a real buying burst — several wallets piling in together, buys overwhelming sells — often precedes a short leg up. This bot buys only that burst, and only while the price has not run away yet (volume first, price second). It risks at most 0.05 SOL per coin, takes profit fast (+12%), cuts losers fast (-6%), and never holds longer than 150 slots. One buy per coin, ever. Full rules, sweep results and honest caveats: see [STRATEGY.md](STRATEGY.md).

## Run

```sh
node main.ts
```

Node 22. No dependencies, no install step, no network at runtime: `main.ts` speaks the `agent-fun/replay/v2` protocol on stdin/stdout through the vendored shim in `vendor/makefun-bot.ts`. It is deterministic — two executions produce byte-identical action logs.

## Results (dev window, official local grader)

| Metric | Value |
| --- | --- |
| Net PnL | **-0.0241 SOL** |
| Round trips | 97 over 97 coins |
| Win rate | 44.3% |
| Max drawdown | 0.87% |
| Guardrails | met (97 ≥ 20 coins, drawdown ≤ 50%) |

For comparison, our earlier selective-momentum bot on the same window scored -0.0568 SOL with only 22 round trips — and would fail the final window's 20-coin guardrail (~5–6 coins per 6 hours). This bot trades ~25 coins per 6-hour slice, so it clears the bar with margin:

| 6-hour slice of the dev window | Net PnL (sim) | Coins traded |
| --- | --- | --- |
| Slice 1 | +0.029 SOL | 22 |
| Slice 2 | +0.003 SOL | 24 |
| Slice 3 | -0.056 SOL | 26 |
| Slice 4 | +0.014 SOL | 27 |

The final window is fresh data and a lottery either way; see STRATEGY.md for the honest assessment.

## What it does not do

- Never pyramids, never averages down, never buys a fresh launch.
- Never holds a runner hoping for +100%.
- Never buys a coin whose burst already pushed the price up more than 5%.
- Never reads the clock, the environment, or randomness — every decision is a pure function of the batches the harness sends.

## Referral

Built with make.fun — [join with my referral link](https://make.fun/r/j11vkafs) (code `j11vkafs`).

## License

MIT — see [LICENSE](LICENSE).
