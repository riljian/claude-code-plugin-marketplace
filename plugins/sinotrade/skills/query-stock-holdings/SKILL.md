---
name: query-stock-holdings
description: >-
  Query and present the Sinotrade stock account's current holdings (open
  positions) — share quantity, average cost, last price, and unrealized P/L —
  as a readable table with totals. Use this whenever the user asks about what
  stocks they currently hold or own, their portfolio / positions / inventory,
  how a holding is doing, or their unrealized gain/loss. Triggers on phrasings
  like "我的庫存"、"我有哪些股票"、"我的持股"、"我的部位"、"現在帳上有什麼"、
  "我的台積電賺多少"、"my holdings"、"what positions do I have"、"show my
  portfolio"、"how much am I up on my stocks", even when the user doesn't say
  the exact word "庫存" or "position". For *realized* profit/loss over a past
  date range, use the profit/loss tools instead — this skill is for currently
  open positions.
---

# Query stock holdings (查詢庫存股)

Report the account's current open stock positions. The job: pull the raw
positions, fold cash dividends into each one's cost, resolve company names, then
present everything as a table with totals.

## Step 1 — Pull the positions

Call `portfolio_position_unit` with **`unit: "Share"`**.

Always use `"Share"`, never `"Common"`. `"Common"` reports quantities in board
lots (整股, 1 lot = 1000 shares) and will misrepresent any odd-lot (零股)
holdings — a position of 1,500 shares would otherwise look like "1 lot" and
silently drop the 500 odd shares. `"Share"` reports every position as a plain
share count, so the numbers are always exact regardless of how the user bought
them.

Omit the account fields (`account_id`, `account_type`, …) to target the default
account — that's the normal case. Only pass them if the user explicitly asks
about a specific account.

Each returned position carries:

| field         | meaning                                                                   |
| ------------- | ------------------------------------------------------------------------- |
| `id`          | position id — pass as `detail_id` to `portfolio_position_detail` (Step 2) |
| `code`        | security code, e.g. `"2330"` — **no company name**                        |
| `quantity`    | shares held (because `unit: "Share"`)                                     |
| `price`       | your average cost per share                                               |
| `last_price`  | latest market price per share                                             |
| `direction`   | `Buy` (long) is the usual case                                            |
| `yd_quantity` | shares carried from previous days                                         |

The position also carries a `pnl` field, but it is **not** dividend-adjusted, so
don't display it — this skill computes its own P/L in Step 4.

If the list is empty, just tell the user the account currently has no open
stock holdings — there's nothing to look up or tabulate.

## Step 2 — Fold dividends into each position's cost

`price × quantity` from Step 1 is only the raw purchase cost; it ignores the
cash dividends the holding has paid out, which the user wants counted in cost.
For each position, call `portfolio_position_detail` with `detail_id` set to that
position's `id`. It returns the per-lot detail rows behind the position — **sum
their `ex_dividends`** to get the position's total dividend adjustment.

The cost for the position is then:

```
成本 / cost = price × quantity + Σ ex_dividends
```

Sum `ex_dividends` exactly as returned, keeping its sign — don't flip it. There
is one detail call per position and they're independent, so issue them in
parallel in a single turn rather than serially.

## Step 3 — Resolve the names you're not sure of

Positions come back as bare codes. To show the user a usable table you need
company names, but **do not guess names from codes** — a wrong mapping is worse
than no name, and Taiwan has thousands of listed symbols.

For a code whose company you genuinely know (e.g. `2330` = 台積電, `2317` = 鴻海),
fill the name in directly; there's no need to spend a round-trip confirming it.
For any code you're not confident about, call `data_contracts` with that `code`
and `security_type: "STK"`, and use the `name` field it returns. When unsure,
prefer looking it up — accuracy matters more here than saving a call. These
lookups are independent, so fire them in parallel in a single turn rather than
one at a time.

## Step 4 — Present the table

Compute these per position:

- **市值 / market value** = `last_price × quantity`
- **成本 / cost** = `price × quantity + Σ ex_dividends` (from Step 2)
- **損益 / P&L** = `市值 − 成本`
- **報酬率 / return %** = `損益 ÷ 成本 × 100%`

Then render a table and a totals row. Respond in the **same language the user
asked in** (Traditional Chinese for a Chinese question, English for an English
one). For a Chinese question, use this shape:

```
| 代號 | 名稱   | 股數  | 均價   | 現價   | 市值      | 成本      | 損益      | 報酬率  |
| ---- | ------ | ----- | ------ | ------ | --------- | --------- | --------- | ------- |
| 2330 | 台積電 | 2,500 | 580.00 | 612.00 | 1,530,000 | 1,420,000 | +110,000  | +7.75%  |
| ...  |        |       |        |        |           |           |           |         |
| 合計 |        |       |        |        | <總市值>  | <總成本>  | <總損益>  | <總報酬率> |
```

(In the example, 成本 1,420,000 = 均價 580 × 2,500 股 minus the dividends summed
in Step 2, which is why it's below 均價 × 股數.)

- Totals: **總市值** = Σ market value, **總成本** = Σ cost, **總損益** = Σ P&L
  (= 總市值 − 總成本), and overall **報酬率** = 總損益 ÷ 總成本.
- Show gains/losses with an explicit sign (`+`/`-`) so direction is obvious at
  a glance; a brief 🔴/🟢 or +/- convention is fine.
- Use thousands separators for share counts and TWD amounts, and round prices to
  the precision Shioaji returns (typically 2 decimals).

If the user asked about one specific stock ("我的台積電賺多少？"), still pull the
full position list, then present just that holding's row (plus its P/L in
prose). If they hold none of it, say so plainly.
