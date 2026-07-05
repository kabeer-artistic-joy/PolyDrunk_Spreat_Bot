# PolyReact

A Polymarket trading bot for the 5-minute BTC/ETH Up/Down markets. It buys
one side cheaply, then sells for a small, fixed profit margin once the price
moves in its favor — or exits for whatever's available if that never
happens before the window closes.

It runs in two variants, selected at launch:

- **`predict`** — decides direction from the *previous* window's price
  action, and buys the instant the *new* window opens.
- **`react`** — watches the *new* window's own real-time price movement for
  30 seconds before deciding, then buys at whatever price results.

---

## ⚠️ Read this before running `--live`

This bot buys near a mid-range price and profits from a small move in its
favor — it does not need to correctly predict which side ultimately wins the
market. But if price never moves the way it needs to, and the window
approaches its close, the bot exits at whatever price is available. That
exit price could be far below the entry price, and in the worst case, close
to a full loss on that trade. Always start with `--dry-run` and let it
accumulate real results before ever using `--live`.

---

## 🧠 How each variant decides

### `predict`
Looks at the window that's about to close: how far has price moved from
that window's own open, as a percentage. If that move clears
`ENTRY_MIN_DELTA_PCT` (0.05%), the bot trusts that direction and buys that
side the instant the new window opens. If the move is too small, the window
is skipped — no trade, no signal to act on.

Skipped windows are still watched in the background (no real order placed)
to check what *would* have happened, logged separately to `shadow_log.csv`.

### `react`
Instead of looking backward, this variant watches the market currently open
right now. For `REACT_WAIT_SEC` (30 seconds), it samples the underlying
asset's price every `REACT_POLL_SEC` (2 seconds), building a real price
history for that window. At the end of that observation period, it checks
two things together:

- **Move size** — did price move at least `REACT_MIN_DELTA_PCT` (0.005%)
  from where it started?
- **Cleanliness** — how much of the total price range during that window was
  "wasted" on back-and-forth movement, versus contributing to the net move?
  A price that climbed steadily is clean; a price that spiked up, dropped,
  and spiked again before ending up in the same place is not, even if the
  net move looks identical.

Only if both checks pass does the bot buy. If either fails, the window is
skipped — this variant does not force a trade when the signal isn't there.

---

## 💰 How a trade plays out, once entered

**Buy:**
- `predict` buys at a target of $0.50, with a $0.52 ceiling, within 3 seconds
  of window open.
- `react` buys at whatever price the market shows after its 30-second
  observation, up to that price plus a small buffer ($0.02), within 3
  seconds.

**Sell:**
- `predict` sells once price reaches $0.58 (floor), aiming for $0.60.
- `react` sells once price reaches its own entry price plus $0.05 — a
  margin relative to wherever it actually bought in, not a fixed number.

In both variants, the sell order rests directly on the order book the
moment the buy confirms, rather than the bot checking the price
periodically and reacting afterward — this lets the sell fill the instant
the market crosses the trigger, without waiting on a polling cycle.

**If the price never reaches the sell trigger:** in the final 30 seconds of
the window (`FORCE_EXIT_SECONDS_LEFT`), the bot exits at whatever price is
currently available, with no minimum — this is a deliberate "get out" move,
not a profit attempt.

---

## 📋 Requirements

- Python 3.10+
- A Polymarket account with USDC on Polygon, for `--live` mode
- No separate account needed for Binance — price data comes from Binance's
  public API, no authentication required

---

## 🚀 Installation

```bash
git clone https://github.com/kabeer-artistic-joy/PolyDrunk_Spreat_Bot.git
cd PolyDrunk_Spreat_Bot
python -m venv .venv
source .venv/bin/activate   # Windows: .venv\Scripts\activate
pip install -r requirements.txt
cp .env.example .env
# Edit .env — only required for --live mode
```

---

## 🖥️ Usage

```bash
# predict variant, dry run
python spread_bot.py --dry-run --variant predict

# react variant, dry run
python spread_bot.py --dry-run --variant react

# Live, with a $2 stake per trade
python spread_bot.py --live --amount 2 --variant react
```

`--variant` defaults to `predict` if not specified. Press **Ctrl+C** to stop
and print a session summary.

---

## ⚙️ Configuration

All key values are constants near the top of `spread_bot.py`:

| Constant | Value | Applies to |
|---|---|---|
| `BUY_TARGET_PRICE` | $0.50 | predict |
| `BUY_CEILING_PRICE` | $0.52 | predict |
| `BUY_TIMEOUT_SEC` | 3.0s | predict |
| `SELL_TARGET_PRICE` | $0.60 | predict |
| `SELL_FLOOR_PRICE` | $0.58 | predict |
| `ENTRY_MIN_DELTA_PCT` | 0.05% | predict |
| `REACT_WAIT_SEC` | 30s | react |
| `REACT_POLL_SEC` | 2s | react |
| `REACT_MIN_DELTA_PCT` | 0.005% | react |
| `REACT_CLEANLINESS_MIN` | 0.5 | react |
| `REACT_BUY_TIMEOUT_SEC` | 3.0s | react |
| `REACT_BUY_CEILING_BUFFER` | $0.02 | react |
| `REACT_PROFIT_MARGIN` | $0.05 | react |
| `FORCE_EXIT_SECONDS_LEFT` | 30s | both |

---

## 📊 What `--dry-run` measures

Dry-run mode polls Polymarket's real, live order book and Binance's real
price data throughout every window — it does not simulate or assume prices.
It records, for every window:

- Whether the entry signal fired at all, and the exact numbers behind that
  decision
- Whether a real buy order would have filled, and at what price
- Whether the sell trigger was reached, and if so, whether there was enough
  real market depth to fill the position
- What would have happened at forced exit if the trigger was never reached

All of this is written to `trades_log.csv` in the same folder, live, trade
by trade — no need to stop the bot to see results. `predict`'s skipped
windows are additionally shadow-tracked to `shadow_log.csv`.

---

## 📁 Project Structure

```
PolyDrunk_Spreat_Bot/
├── spread_bot.py        # Main bot — both variants
├── requirements.txt
├── .env.example
├── .gitignore
└── README.md
```

---

## 🔍 Known limitations

- The exact field name Polymarket uses for balance in one specific fallback
  check has not been independently confirmed from documentation — if it
  fires, the bot logs the raw response so the correct field can be
  identified from real data.
- Winning the race to be among the first to fill at a given price depends on
  network latency and how quickly counterparties are present on the order
  book — this is a real constraint the bot cannot fully control for.
- `react`'s price observation uses Binance's own price feed as a proxy for
  momentum; Polymarket's own price for a given side does not always move in
  perfect lockstep with the underlying asset's price.

---

## ⚠️ Disclaimer

> This software is for educational and experimental purposes only. This is
> not financial advice. Both variants can lose close to a full stake on a
> single trade if the price never reaches the sell trigger. Always start
> with `--dry-run` and accumulate real evidence before considering `--live`.
> Never risk money you cannot afford to lose.
