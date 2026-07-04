# Polymarket Down-Spread Scalper Bot

A **completely separate** bot from the delta/momentum strategy bot. This one
shares no risk logic, no ATR, no delta threshold, no confidence scoring — it
tests a different, narrower hypothesis entirely.

---

## ⚠️ Read this before running `--live`

This bot's loss shape is **fundamentally different and riskier** than the
delta bot's:

- The delta bot only enters when the market price already strongly agrees
  with the signal (≥0.92), so a loss is bounded and historically rare.
- **This bot has no directional signal at all.** It always buys Down at the
  very start of the window — a coin-flip price point — and hopes the price
  later drifts into the 58-60c range so it can sell for a small profit. It
  does not matter which side wins; it only matters whether that price range
  gets touched before the forced exit.
- **If Down never reaches 58c and the window resolves the other way, the
  loss is close to your full stake per trade** — not a bounded, small
  percentage like the delta bot.
- **The core assumption ("the price almost always touches 60c") has not been
  independently verified against real trade outcomes in this codebase.**
  `--dry-run` exists specifically to test this against real, live order book
  data before any capital is at risk. Run it for a meaningful sample first.

---

## 🧠 Strategy

1. Wake 10 seconds before each new 5-minute BTC/ETH Up/Down window opens.
2. The instant the window opens, attempt to buy the **Down** token as close
   to $0.50 as possible (ceiling $0.52). If unfilled after 2 seconds, cancel.
3. If bought, watch for the Down price to reach $0.58-$0.60. Attempt to sell
   on every such opportunity (there may be more than one per window).
4. If no opportunity results in a sale by the time 60 seconds remain in the
   window, exit immediately at whatever price is available — no floor, no
   ceiling — to avoid holding into resolution uncertainty.
5. Sleep until 10 seconds before the next window, repeat.

This runs **BTC and ETH in parallel** (separate threads), each on its own
independent 5-minute cycle.

---

## 📋 Requirements

- Python 3.10+
- A Polymarket account with USDC on Polygon (for `--live` mode)
- No Binance account needed — this bot only uses Polymarket's own order book,
  not external price feeds

---

## 🚀 Installation

```bash
git clone <this-repo-url>
cd spread-bot
python -m venv .venv
source .venv/bin/activate   # Windows: .venv\Scripts\activate
pip install -r requirements.txt
cp .env.example .env
# Edit .env — only required for --live mode
```

---

## 🖥️ Usage

```bash
# Dry run — real order book data, no real orders, no funds at risk
python spread_bot.py --dry-run

# Live — real orders, real funds
python spread_bot.py --live --amount 2
```

Press **Ctrl+C** to stop and print the session summary.

---

## ⚙️ Configuration

Key constants are defined at the top of `spread_bot.py`:

| Constant | Default | Description |
|---|---|---|
| `BUY_TARGET_PRICE` | `0.50` | Price we hope to pay |
| `BUY_CEILING_PRICE` | `0.52` | Max we're willing to pay (actual limit order price) |
| `BUY_TIMEOUT_SEC` | `2.0` | Cancel the buy if unfilled after this long |
| `SELL_TARGET_PRICE` | `0.60` | Price we hope to sell at |
| `SELL_FLOOR_PRICE` | `0.58` | Minimum acceptable sell price |
| `FORCE_EXIT_SECONDS_LEFT` | `60` | Exit at any price if still holding with this much time left |

---

## 📊 What `--dry-run` actually measures

Dry-run mode is **not a simulation with assumed prices** — it polls
Polymarket's real, live public order book throughout each window and records:

- Whether a real ask at or below the buy ceiling existed within the timeout,
  and at what price
- Every distinct point where the real bid reached the sell floor, and
  whether there was enough real depth to fill your position size at that
  moment (not just whether the price was touched)
- What would have happened at forced exit if no opportunity worked out

This gives you real evidence for or against the core hypothesis before any
capital is committed — the same way the delta bot's threshold changes were
validated against real trade data before being trusted.

---

## 📁 Project Structure

```
spread-bot/
├── spread_bot.py        # Main bot
├── requirements.txt      # Python dependencies
├── .env.example          # Environment variable template
├── .gitignore
└── README.md
```

Trade-by-trade results are logged live to `trades_log.csv` in this folder as
they happen — no need to stop the bot to see results.

---

## 🔍 Known uncertainties in this implementation

Documented honestly rather than glossed over:

- **Order-fill detection in live mode** relies on `get_order()` returning
  `None` once an order leaves the open-orders index, which is inferred
  behavior (observed in a different context on the delta bot), not something
  confirmed in Polymarket's documentation for this exact case. Watch your
  first few live fills closely against your actual Polymarket account.
- **Winning the race for the first 1-2 seconds of liquidity** depends on
  network latency to Polymarket's servers and how fast market makers seed
  that opening liquidity — this is a real competitive/infrastructure factor
  this code cannot fully control for.
- **Gamma API market listing may lag slightly behind the exact time
  boundary** — the bot retries for up to 3 seconds to find the window's
  market before giving up on that cycle.

---

## ⚠️ Disclaimer

> This software is for educational and experimental purposes only. This is
> **not financial advice**. This specific strategy has a loss shape that can
> approach full stake per losing trade — materially different from, and
> potentially riskier than, other bots in this project. Its core assumption
> has not been validated against real outcome data as of this writing.
> Always start with `--dry-run` and accumulate real evidence before
> considering `--live`. Never risk money you cannot afford to lose.
