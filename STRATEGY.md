# Strategy — Net-Flow Surge Sniper

## Rules (all in `main.ts`)

**Entry — one buy per coin, at most 0.05 SOL:**

- The coin is at least 100 slots old (~20 seconds). Fresh launches are skipped:
  they are mostly insiders exiting.
- Its real SOL reserve is between 1 and 60 SOL at decision time.
- Buy SOL over the last 80 slots is at least 3x buy SOL over the 80 slots
  before that, and at least 1 SOL in absolute terms — an accelerating burst,
  not background noise.
- Net flow over the last 80 slots (buys minus sells) is at least +0.5 SOL,
  so the burst is net buying, not a dump disguised as volume.
- At least 3 distinct wallets bought inside the window.
- The price rose at most 5% over the window — volume first, price second;
  a move that already happened is not chased.
- Size is min(0.05 SOL, 1% of the coin's real SOL reserve), never below
  0.01 SOL; at most 5 open positions; at most 16 orders per reply.

**Exit — sell the whole position when any of these hits:**

- Take-profit: price ≥ +12% over the filled entry price.
- Stop-loss: price ≤ -6% under the filled entry price.
- Max hold: 150 slots (~30 seconds) since the buy filled.
- A coin that was exited is ignored for 300 slots (cooldown).
- Slippage allowance 10,000 bps on every order (a fill is never refused for
  the price moving during the 2-slot latency).

**Determinism:** every decision is a pure function of the batches the
harness sends. No randomness, no clock, no network. Two local replay runs
produced byte-identical action logs, and the official local grader's L1
regrade reproduced the run byte for byte.

## Fee math

Per side: ~95 bps protocol + ~30 bps creator = 1.25% of trade value, plus
5,000 + 100,000 lamports tx fee per order. A 0.05 SOL round trip costs
roughly 0.00125 SOL in curve fees + 0.00021 SOL in tx fees (~2.9% of the
stake) — which is why the take-profit is only +12% and the hold is short:
the edge, if any, is measured in tens of slots, not hours.

## Validation — read this before believing anything

Tool: a local replay simulator (constant-product bonding curve, 2-slot
latency, exact-in fee split per the harness source, 105,000 lamports tx fee,
5 open positions, 1% reserve cap at decision time). It is a proxy, not the
grader: it ignores the harness's seeded 2% order failures and evaluates
exits every slot instead of every batch. Calibrated against the official
grader on a survivor-momentum bot: sim -0.1839 vs official -0.1793 (delta
0.005 SOL). The official `grade-local` numbers below are the authority.

Dev dataset: 24h, 1,222 coins, 128,058 events, 35 curve completions. Note:
all 35 completions happen inside a single slot (0 to 85 SOL in one slot, in
a handful of large buys) — they cannot be traded (2-slot latency) and no
strategy can ride them. The final window is 6 hours, so the guardrail check
below also splits the dev data into four 6-hour slices.

### Shipped variant (net-flow surge, window 80, TP+12/SL-6, hold 150)

| Window | Net PnL (SOL) | Round trips | Win rate | Max drawdown |
| --- | --- | --- | --- | --- |
| Full 24h (sim) | +0.0001 | 88 | 45% | 0.7% |
| **Full 24h (grade-local, official)** | **-0.0241** | **97** | **44.3%** | **0.87%** |

Grade-local detail: 90% interval -0.153 … +0.092 SOL; fees 0.103 SOL curve
+ 0.021 SOL tx; at 2x fees -0.147 SOL; top 3 coins made 21% of the gains
(without them: -0.08 SOL); 0 rejected, 3 failed (seeded); deterministic
(action_log_sha256 `674103bd…`, L1 regrade reproduced byte for byte);
guardrails met (97 ≥ 20 coins with a round trip ≥ 0.01 SOL, drawdown ok).

Per 6-hour slice (sim, same rules):

| Slice | Net PnL (SOL) | Distinct coins |
| --- | --- | --- |
| 1 | +0.029 | 22 |
| 2 | +0.003 | 24 |
| 3 | -0.056 | 26 |
| 4 | +0.014 | 27 |

Every slice clears the 20-coin guardrail. Two of four slices are positive,
two negative — the per-slice mean is about -0.003 SOL, i.e. the final window
is close to a coin flip around a small loss, with a wide spread.

### What else was tried (full-window simulator, same 24h)

| Variant | Full PnL (SOL) | Trips |
| --- | --- | --- |
| Survivor momentum, continuous confirmation, price exits (old-bot family) | -0.075 … -0.115 | 29–44 |
| Survivor momentum, value exits | -0.173 | 41 |
| Threshold momentum (enter at first crossing of 10/20/30 SOL real) | -0.18 … -1.20 | 73–267 |
| Volume surge, window 100, no net-flow filter | -0.195 | 117 |
| Volume surge, window 100, net ≥ +1 SOL | -0.072 | 87 |
| Volume surge, window 100, net ≥ +0.5, TP+12/SL-6, hold 100 | -0.052 | 98 |
| **Volume surge, window 80, net ≥ +0.5, TP+12/SL-6, hold 150 (SHIPPED)** | **+0.000** | **88** |
| Volume surge, window 75 / 85, same exits | +0.013 / -0.004 | 97 / 91 |
| Volume surge, window 90 / 95 / 100 / 110, same exits | -0.059 … -0.052 | 94–104 |
| Launch sniper (kit pilot, age ≤ 3 slots) | -0.423 (official) | 343 |
| Momentum pilot (kit, pyramid + trail) | -9.45 (official) | 77 |
| Reversion pilot (kit, dip buying) | -4.37 (official) | 423 |
| Equal-weight baseline (kit, buy-and-hold) | -0.023 (official) | 5 |

The window-75–85 plateau is a real region, not a single lucky cell: three
consecutive windows and two hold values (125, 150 slots) all land between
+0.013 and -0.012 SOL in simulation, with win rates 42–47%.

### Stress test (rebuilt windows)

`grade-local --stress 20` reruns the bot on 20 rebuilt copies of the dev
window (blocks of trades the window itself shows) and reports the median,
10th/90th percentile and how many copies lose money — the check for whether
the dev score leans on this window's particular coins.

Result (official, process driver, 20 copies):

| Metric | Value |
| --- | --- |
| Median of rebuilt copies | -0.18 SOL |
| 10th / 90th percentile | -0.27 / -0.11 SOL |
| Copies that lost money | 20 of 20 |
| Dev score vs copies | above 20 of 20 |

The grader also printed both warnings: the dev score is above 90% of the
rebuilt copies, and half or more of the copies lost money. Read plainly:
**the dev score (-0.024 SOL) is far better than the typical rebuilt window
(median -0.18 SOL) and depends heavily on this window's particular coins.**
A fresh window should be expected to land nearer -0.18 over 24 hours
(~-0.045 per 6 hours) than near the dev score. That does not change the
choice — every family tested is worse on the same measure, and the old bot's
-0.057 dev score would face the same test — but the prize case must be
stated with the stress median, not the dev point estimate: this entry is a
long shot whose best realistic final outcome is a small loss, with a
minority chance of a small win if the final window's coins behave like the
dev window's best slices.

### Comparison with the old bot (honest)

The previous entry on the sibling competition (selective momentum,
submission 01a21360-c424) scored **-0.0568 SOL official** with 22 round
trips and an 18.2% win rate. This bot beats it on the dev window
(**-0.0241 vs -0.0568**, +0.0327 SOL) with 97 round trips and a 44.3% win
rate — and, unlike the old bot, it meets the final window's guardrail: the
old bot trades only ~5–6 coins per 6 hours, below the 20-coin bar, so it
would rank below every guardrail-meeting entry in the final even before PnL
is considered. This bot trades 22–27 coins per 6-hour slice.

## Lessons (honest)

1. **No variant was positive out of sample.** The shipped one is ~breakeven
   on the dev window in simulation (-0.024 official) with a 90% interval
   that spans -0.15 to +0.09 SOL. A fresh 6-hour window could land either
   side; the honest expectation is a small loss with a real chance of a
   small win.
2. **The edge, if any, lives in the net-flow filter.** Requiring buys minus
   sells ≥ +0.5 SOL over the burst window cut the loss from -0.195 to
   -0.072 (window 100) — it removes bursts that are mostly one big seller
   exiting into wash volume.
3. **Short holds win.** Every family tested lost more the longer it held:
   survivor holds of 5,000+ slots and threshold holds of 1,000 slots were
   2–5x worse than 100–250 slot holds. Mean reversion to zero dominates
   after the first minute.
4. **Concentration is far lower than the old bot's.** Top 3 coins made 21%
   of the gains here vs 91% there — the result depends less on a single
   lottery ticket, which is why the per-slice PnLs are comparatively stable.
5. **The simulator is faithful but not exact.** It runs ~0.01–0.02 SOL
   optimistic versus the official grader (failed txs, exit timing on
   batches vs slots, entry-price reference). Simulator numbers were used
   only to rank variants; every claim above the line uses grade-local.
