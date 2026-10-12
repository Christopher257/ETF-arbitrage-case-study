# An ETF Arbitrage Case Study

*Creation/redemption arbitrage on the iShares Russell 3000 ETF (IWV). Analysis only: no trading, no money, no real AP status. Data up to 9 Oct 2026.*

## Files

- `README.md`: the light version, a quick skim.
- `etf_arbitrage.ipynb`: the full version with all code, outputs and the longer reasoning. Partly redundant with this file on purpose.

## The core idea

An ETF has a fair value: the value of the stocks inside it. The price on the screen can drift away from that. Large institutions called Authorized Participants (APs) can swap ETF shares for the stocks inside (and back) at fair value. In theory they pocket the difference, and their trading pulls the price back. A textbook free lunch.

I went looking for it with free data, to see whether the lunch survives the bill.

The real strategy trades during the day, whenever the ETF moves away from its basket. Testing that needs second-by-second prices for the ETF and all its stocks, which free data does not have. So the backtest uses the daily close as a model, to show the likely outcome and the limitations, not as the trading edge itself. It is looked at in two ways:

- **Theoretical edge:** the gap to the official NAV, as if it were known at the time of the trade. An upper bound, since the NAV is only published after the close.
- **Practical edge:** the gap to my own NAV estimate. This is the actual strategy, because in real time the NAV always has to be estimated. It needs live data, so it is logged forward instead of backtested.

## What it shows

- **The gap is small but real.** Over 26 years the ETF closes a median of 3.2 bps away from its fair value (1 bp = 0.01%). Since 2021 the average is 2.9 bps. In crises it grows: up to 186 bps in 2000 (the fund's first weeks) and 173 bps in September 2008.
- **The exact hedge locks in the profit on paper.** Hedge with the exact stocks and the profit is fixed at entry. That means trading about 2,600 stocks, and each trade pays a bid-ask spread. Depending on the spread scenario, this costs between 7% and 116% of the gap.
- **An ETF hedge works only if the stocks are never sold.** Selling the redeemed stocks (B1) pays the same spreads and adds noise. Handing them back in kind to create the hedge ETFs (B2) trades no stock and keeps the most edge.
- **Verdict:** not a proven money machine. The result depends on numbers that free daily data cannot measure, namely actual quotes. That needs a follow-up study.

## Conclusion

Even a "failsafe" strategy has its caveats in reality. On paper the hedge removes all risk. In practice, data quality and market realities (spreads, execution, which prices you can actually see) add costs of the same size as the edge. The edge is thin enough that the small print decides the result.

## A worked example

Take the discount case. The ETF trades at 99, its stocks are worth 100.

1. Buy the ETF at 99. At the same time, short the stocks at 100. Market moves now cancel out.
2. If the gap closes before the swap settles: close both legs, profit 1.
3. If it stays: redeem the ETF for the stocks, hand the stocks over to close the short, profit 1.
4. If it gets worse: still redeem, still profit 1. The position looks worse on paper for a while.

Why the same 1 every time? The short is closed by delivering the redeemed stocks, not by buying stocks at a new price. So the profit depends only on the two entry prices: `P&L = F0 - P0`, where `F0` is the stock basket price and `P0` the ETF price at entry.

Why hedge at all? The calculation shows that the ETF and its stocks are priced differently, but not which of the two is wrong. Buying only the ETF is a bet that the ETF is the mispriced side. Holding both sides earns the difference, whichever side moves.

Real day, 8 Oct 2026: the ETF closed about 5 bps above its stocks. Hedged, that is a locked-in 5.1 bps. Unhedged, the same position ended at +67.8 bps, because the market rose 63 bps while waiting. On a day with a 63 bps fall it would have been about -58 bps.

## The flow

1. **Data.** Official NAV (fair value per share) and share count since 2000 from iShares, closing prices from Yahoo, the latest holdings (exact shares per stock) from iShares.
2. **The gap.** `gap = NAV / close - 1`, per day.
3. **Costs.** What would trading it cost? Shown as three scenarios plus a "how wrong can I be" breakeven.
4. **Timing.** The official NAV only shows up after the close, so the trade decision uses my own estimate: yesterday's basket at today's prices.
5. **Design A.** Hedge with the exact stocks.
6. **Design B.** Hedge with IWB and IWM instead (Russell 1000 and Russell 2000, which together are the Russell 3000), closed either by selling stocks (B1) or in kind (B2).
7. **Verdict.** Net edge per design, per scenario.

## Core maths and code

Fair value and the gap:

```python
gap_bps = (nav / close - 1) * 1e4                      # > 0 discount (buy, redeem), < 0 premium (create)
```

Own NAV estimate: yesterday's published basket, today's prices, plus cash, divided by ETF shares:

```python
estimated_nav = ((quantity * price_today).sum() + cash) / etf_shares
```

Check: the same formula at the basket's own date reproduces the official NAV to 0.04 bps. The one logged out-of-sample day came within about 1 bp. One day is too few to judge accuracy.

Design A cost, the main item that eats the edge. Every stock is traded once and pays half its spread. The cost is one sided, so it does not average out across stocks:

```python
stock_cost_bps = (weights * full_spread_bps).sum() / 2
profit_bps     = average_gap_bps - stock_cost_bps - etf_cost_bps
```

Design B2 (in kind): the hedge is sized by index ratio (96.2% IWB, 3.8% IWM, from IWV's holdings). The profit is the IWV gap minus the hedge ETFs' own gaps:

```
P&L  ~  gap_IWV - (0.962 * gap_IWB + 0.038 * gap_IWM)  +  NAV tracking noise
```

## Assumptions

- **Spreads are assumed.** I tried estimating them from daily high/low prices (Corwin-Schultz). It gave about 50 bps for Apple, so I dropped it. Stocks get a full spread by size tier (top 100, 101 to 500, 501 to 1,500, rest):

| Scenario | Spreads in bps (top 100 / 101-500 / 501-1,500 / rest) |
|---|---|
| Optimistic | 0.1 / 0.5 / 2 / 5 |
| Realistic | 1 / 3 / 10 / 30 |
| Conservative | 3 / 8 / 25 / 80 |

- ETFs (IWV, IWB, IWM) get 0, 1 or 3 bps full spread in the same three scenarios.
- Trades happen at the close and the profit is the gap to the official NAV.
- B2 assumes AP status in all three funds.
- Only the stock leg and the ETF legs are costed. Issuer fees (B2 would pay three), market impact and borrow fees would come on top, so these costs are a lower bound.

## Outcomes

Net edge per trade, in bps, since 2021 (edge is the average gap, then the costs come off):

| Design | Optimistic | Realistic | Conservative | Noise per trade (std) |
|---|---|---|---|---|
| A: exact stocks | 2.7 | 1.2 | -2.0 | about 0 (exact) |
| **B2: ETF hedge, in kind** | **2.8** | **1.8** | **-0.2** | 0.2 to 1.4 |
| B1: ETF hedge, stocks sold | 2.7 | 0.2 | -5.0 | 3.4 to 4.7 |

What to take from it:

- The designs differ in cost. The mechanism is the same.
- A's stock leg alone costs 0.2 / 1.3 / 3.4 bps against a 2.9 bps gap. The stocks ranked below 500 are 9% of the weight and 46% of the realistic cost.
- B1's noise comes from the closing prices of the hedge ETFs, which sit a few bps away from their own NAVs, as IWV does.
- B2 holds up best. It also needs the most things to be true at once: AP status in three funds and good estimates of three NAVs.

## What this is not

- This is not a backtest of a tradable strategy. The decision signal (own estimate) has one logged day, because iShares only offers the latest holdings snapshot for download.
- Yahoo's daily data is not precise enough to settle spreads or to build the three NAV estimates that B2 needs. A follow-up with intraday quotes would.
- The spread numbers are scenarios, and the notebook says so wherever they are used.
- Sharpe ratio and drawdown are left out. The hedged profit is fixed at entry, so its volatility is close to zero by construction and a Sharpe ratio would say nothing. The real risks are costs and execution.

## Run it

```bash
pip install -r requirements.txt
```

```bash
jupyter notebook etf_arbitrage.ipynb
```

The first run downloads everything into `data/`, which takes about 6 minutes because the stock prices are rate limited. Later runs read from `data/`. That folder is not in the repository, because the iShares and Yahoo data is for personal use only, so a fresh clone gets the latest data and slightly different numbers. Set `REFRESH = True` in the first code cell to download again.

`iwv_estimates.csv` is the live log of my own NAV estimates, one row per day the notebook is run after a close. It cannot be downloaded again, so it is kept in the repository.
