# avellaneda-stoikov-market-maker

An FE570 Market Microstructure final project. An Avellaneda-Stoikov market
maker for the [SHIFT](https://github.com/hanlonlab/shift-python/wiki) exchange
simulator, wrapped in the microstructure controls a naive AS loop needs to
survive a live simulated order book.

## What it does

`market_maker.py` runs a quoting loop over a basket of equities (AAPL, AMZN,
BRKb, GOOG, MSFT). Each pass it:

1. reads the best bid/ask and its current inventory from SHIFT;
2. updates a volatility estimate `σ` (EWMA of squared log-returns in the live
   loop; a GARCH(1,1) estimator is also implemented);
3. computes the Avellaneda-Stoikov **reservation price** and **optimal
   half-spread**

   ```
   r      = mid − q·γ·σ²·T
   spread = γ·σ²·T + (2/γ)·ln(1 + γ/κ)
   ```

   with `q` inventory in lots, `γ` risk aversion, `κ` order-book liquidity (the
   per-symbol min/max spread clamp usually governs the quoted half-spread in
   practice);
4. sizes each side by inventory (lean away from the side you're long) and
   submits non-crossing limit orders;
5. logs every snapshot, quote, order and fill to a multi-tab Excel report.

## Microstructure controls added on top of the textbook model

| Control | Why |
|---|---|
| Quote TTL + requote thresholds (mid move / σ change) | don't spam the book; requote only when something moved |
| Non-crossing guard + top-of-book clamp | SHIFT rejects crossed quotes; stay at or inside the touch |
| Per-symbol `γ` and min/max spread clamp | one risk-aversion doesn't fit a $100 and a $500 name |
| Inventory-scaled sizing + reduce-only band + hard share cap | bound worst-case inventory |
| Periodic + terminal forced liquidation | AS assumes a finite horizon; flatten toward it |

## Repo structure

```
avellaneda-stoikov-market-maker/
├── README.md   requirements.txt   .gitignore   .env.example
├── market_maker.py       # the strategy (needs a SHIFT install to run)
├── analyze_run.ipynb     # post-run analysis of an Excel report
└── data/
    └── shift_run_report_20251214_170837.xlsx   # one 60-minute sample run
```

## How to run

**The analysis** needs only the sample report:

```bash
python -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt
jupyter notebook analyze_run.ipynb
```

**The strategy** needs the SHIFT simulator, its Python client (from the SHIFT
distribution, not PyPI), and an `initiator.cfg`. Set credentials in the
environment (`cp .env.example .env`), then `python market_maker.py`.

## Results

The included run is a 60-minute session over five names, on a SHIFT account
that already carried positions and realized P&L from earlier testing.
`analyze_run.ipynb` separates that pre-existing tests from what this session
 did. By that measure: four of the five symbols never
traded at all this session (0 fills, 0 P&L change); `BRKb` held a flat
300-share position for the *entire* hour; present before the very first
logged tick, so inherited, not built up here, and also saw no fills.
`MSFT` is the one symbol anything happened on: one buy, one sell, a real
**+$16** on that single round trip, out of 4,428 quotes logged that hour
(a 0.05% quote-to-fill rate). The SHIFT feed was nearly static throughout, so
`σ` mostly sat at its floor.

Net: this run demonstrates the quoting and inventory-control *machinery*, not
a P&L result. With no symbol's inventory varying enough to show a slope
(`BRKb` sits at a constant 3 lots the whole run), the reservation-price-vs-
inventory skew isn't visible in this particular session. The one AS mechanic this run does show cleanly is `BRKb`'s quoted
spread widening as its `σ` estimate rises. A run against a livelier SHIFT
session with more fills, more inventory actually moving, is a natural next
step.

## Notes

- No credentials in the code. `market_maker.py` reads `SHIFT_USERNAME` /
  `SHIFT_PASSWORD` from the environment and exits if they are unset.
- Built on a foundational FE570 market-making bot developed with a classmate.

