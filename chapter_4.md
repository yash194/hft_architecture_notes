Yes. This is the right next chapter because **network latency only matters because the matching engine has rules that convert “arrived 100 ns earlier” into different economic outcomes**.

The previous chapter answered:

> How does my packet get from the exchange to my trading machine and how does my order get back?

This chapter answers:

> **What exactly happens when that order reaches the exchange, why does arrival order matter, how is liquidity allocated, and what does an HFT system have to model to trade intelligently?**

I’ll treat this as **Chapter 4: Exchange Microstructure and Matching Engines**. I’ll use Nasdaq equities, CME futures and Eurex examples where useful, because there is no single universal matching rule: Nasdaq uses price/time priority for its continuous book, while CME and Eurex support products with FIFO, pro-rata and hybrid allocation models. ([NASDAQ Trader][1])

---

# Chapter 4 — Exchange Microstructure and Matching Engines

## 1. Why microstructure matters to an HFT engineer

Suppose two trading firms calculate exactly the same fair value:

```text
True estimated value = 100.05
```

and both want to buy at:

```text
100.00
```

Firm A's order reaches the exchange at:

```text
09:30:00.000001000
```

Firm B arrives at:

```text
09:30:00.000001100
```

Only:

```text
100 ns
```

separate them.

If the venue uses strict price-time priority:

```text
100.00 BID QUEUE

Firm A   500 shares
Firm B   500 shares
```

then incoming sellers consume A before B.

So latency changes:

```text
arrival time
    ↓
queue position
    ↓
fill probability
    ↓
inventory
    ↓
P&L
```

That is the economic reason behind nearly everything we discussed in the previous three chapters.

Nasdaq explicitly states that displayed orders at the same price are executed in received-time order. ([NASDAQ Trader][1])

---

# 2. What “market microstructure” actually means

Market microstructure is the study of the **mechanisms through which trading happens**.

It isn't mainly:

```text
Is Apple undervalued?
Will rates fall?
```

Those are broader investment questions.

Microstructure asks questions such as:

```text
Who is currently first in the queue?

What happens when a market order arrives?

How are orders prioritized?

What does hidden liquidity do?

What happens if I cancel and replace my order?

How does an iceberg refresh?

Does my order earn or pay a liquidity fee?

How much quantity is probably ahead of me?

Can I improve queue position?

What is the probability my quote gets filled?

What information does a fill reveal?

Was I filled because my price was good,
or because everyone else knows the market is about to move?
```

For HFT, these questions are often more important than predicting a long-term price.

---

# 3. The Central Limit Order Book

Most of the systems we're discussing revolve around a:

# Central Limit Order Book — CLOB

Conceptually:

```text
                  SELL / ASK

Price        Quantity

100.05          600
100.04          400
100.03          200       ← best ask
────────────────────────
100.02          500       ← best bid
100.01          800
100.00         1200

                  BUY / BID
```

The highest buy price is:

$$
BestBid = 100.02
$$

The lowest sell price is:

$$
BestAsk = 100.03
$$

The spread is:

$$
Spread=BestAsk-BestBid
$$

therefore:

$$
100.03-100.02 = \$0.01
$$

The midpoint is:

$$
Mid=\frac{BestBid+BestAsk}{2}
$$

so:

$$
Mid=100.025
$$

---

# 4. Price levels are not necessarily single orders

Suppose best bid shows:

```text
100.02 × 500
```

That may actually be:

```text
Price = 100.02

Order A   100
Order B   250
Order C    50
Order D   100
────────────────
Total     500
```

Therefore there are two different views of the book.

### Market-by-price

```text
100.02 → 500
```

### Market-by-order

```text
100.02
    Order A → 100
    Order B → 250
    Order C → 50
    Order D → 100
```

This distinction becomes enormously important when estimating queue position.

---

# 5. Resting order versus aggressive order

This terminology is fundamental.

Suppose the current ask is:

```text
100.03
```

and I submit:

```text
BUY 100 @ 100.02
```

There is no seller available at 100.02.

My order therefore sits on the book.

It is a:

# resting order

or a **passive order**.

Now another trader sends:

```text
SELL 100 @ 100.02
```

That order crosses my resting bid and immediately trades.

It is the:

# aggressing order

or **liquidity-taking order**.

Therefore:

```text
resting order = provides liquidity

aggressing order = consumes liquidity
```

This distinction influences both fees and strategy behavior.

---

# 6. Limit orders

A limit buy says:

> Buy no higher than this price.

Example:

```text
BUY 100 @ 100.02
```

Possible outcomes depend on current asks.

If:

```text
Best Ask = 100.05
```

the order rests.

If:

```text
Best Ask = 100.02
```

it immediately executes.

If:

```text
Best Ask = 100.01
```

you may receive price improvement and execute at the available resting price, depending on venue rules.

The core concept is:

$$
P_{buy}\leq LimitPrice
$$

For sell:

$$
P_{sell}\geq LimitPrice
$$

CME similarly defines a limit order as one where the buyer defines the maximum acceptable price and the seller the minimum acceptable price. ([CME Group][2])

---

# 7. Market orders

A conceptual market order says:

> Trade now against available liquidity regardless of price.

Suppose asks are:

```text
100.03  → 100 shares
100.04  → 200
100.05  → 500
```

You submit:

```text
MARKET BUY 400
```

Execution becomes approximately:

```text
100 @ 100.03
200 @ 100.04
100 @ 100.05
```

Average price:

$$
\frac{
100(100.03)+
200(100.04)+
100(100.05)
}{400}
$$

which is:

$$
100.04
$$

This movement through multiple price levels is called:

# walking the book

and the difference between the initial best price and your realized prices contributes to:

# slippage.

CME notably implements a protected form called **Market Order with Protection**, where execution is constrained to a predefined protection range rather than permitting unlimited movement through the book. ([CME Group][2])

---

# 8. Immediate-Or-Cancel — IOC

An IOC means:

> Execute whatever is possible now and cancel the rest.

Book:

```text
ASK

100.03 × 100
100.04 × 100
```

Submit:

```text
IOC BUY 300 @ 100.04
```

You might get:

```text
100 @ 100.03
100 @ 100.04

remaining 100 → CANCEL
```

Nothing rests.

Nasdaq OUCH supports IOC as one of its Time-in-Force choices. ([NASDAQ Trader][3])

This is extremely useful to an HFT strategy that wants:

```text
take liquidity now
```

but explicitly does not want:

```text
leave a stale resting order behind.
```

---

# 9. Fill-Or-Kill — FOK

FOK means:

> Fill the entire requested quantity immediately or execute nothing.

Suppose:

```text
BUY 500 @ 100.05
```

but only:

```text
400
```

shares are available up to 100.05.

FOK result:

```text
0 shares executed
500 cancelled
```

rather than a partial fill.

CME/EBS documentation explicitly describes FOK as requiring the entire order to fill immediately or the whole order to be cancelled. ([CME Group][4])

---

# 10. FAK / IOC distinction

Different venues use slightly different terminology.

CME/EBS uses **FAK — Fill And Kill** for an instruction essentially resembling:

```text
execute available amount
cancel remainder
```

while FOK requires full execution. ([CME Group][4])

Conceptually:

| Order   |          Immediate partial fill? | Remaining quantity rests? |
| ------- | -------------------------------: | ------------------------: |
| Limit   |                         Possibly |                       Yes |
| IOC/FAK |                              Yes |                        No |
| FOK     | No, unless entire size available |                        No |

---

# 11. The matching engine

Now imagine your order actually arrives at the exchange.

Conceptually the matching engine performs:

```text
Incoming Order
     │
     ▼
validate
     │
     ▼
find instrument
     │
     ▼
does order cross opposite book?
     │
   ┌─┴─────┐
   │       │
  YES      NO
   │       │
   ▼       ▼
match     rest?
   │       │
   ▼       ▼
trades   insert into book
   │
   ▼
update book
   │
   ▼
generate acknowledgements
   │
   ▼
generate market data
```

This seems simple.

The complexity lies in answering:

> **Exactly which resting order gets matched first?**

That is the matching algorithm.

---

# 12. Price priority

Nearly all common continuous-order-book algorithms begin with:

# Better price wins.

Suppose you submit:

```text
SELL MARKET
```

against:

```text
BIDS

100.02  500
100.01  300
100.00 1000
```

The engine must trade:

```text
100.02
```

before:

```text
100.01
```

because 100.02 is a superior price for the seller.

Therefore:

$$
price\ priority > other\ priority
$$

Time, quantity or special allocation logic usually matters only among orders at the **same executable price**.

---

# 13. Price-time priority

Now suppose best bid is:

```text
100.02
```

with:

```text
Order A  100 shares  arrived 10:00:00.001
Order B  300 shares  arrived 10:00:00.002
Order C  200 shares  arrived 10:00:00.003
```

Incoming sell:

```text
SELL 250 @ 100.02
```

Under FIFO/price-time allocation:

```text
A receives 100
B receives 150
C receives   0
```

Remaining queue:

```text
B 150
C 200
```

Nasdaq describes this exact principle for displayed orders: at equal price, orders execute in the order received. Eurex's time-allocation method similarly sorts eligible same-price orders by priority timestamp, oldest first. ([NASDAQ Trader][1])

---

# 14. Queue position

This creates one of the most important HFT concepts.

Suppose at:

```text
100.02
```

there are already:

```text
1,000 shares
```

ahead of you.

You add:

```text
100 shares.
```

Conceptually:

```text
100.02 BID

[100][250][150][500][YOU 100]
 ←── 1,000 shares ahead ──→
```

Your:

# queue ahead

is approximately:

$$
Q_{ahead}=1000
$$

Before you are filled, roughly that quantity must:

```text
execute
```

or:

```text
cancel
```

ahead of you.

---

# 15. Why latency matters through queue position

Firm A:

```text
order arrival = t
```

Firm B:

```text
order arrival = t + 100 ns
```

Suppose nobody else is between them.

Queue:

```text
A
B
```

Now only 100 shares trade and A has size 100.

Result:

```text
A fills
B gets nothing.
```

The economic consequence of 100 ns was not:

```text
“CPU benchmark looks nicer.”
```

It was:

```text
fill versus no fill.
```

That is why HFT optimization matters.

---

# 16. Queue position is not simply displayed quantity

This is where microstructure becomes subtle.

Suppose displayed quantity ahead:

```text
1000
```

But perhaps there are also:

```text
hidden orders
reserve quantity
midpoint liquidity
priority rules
```

that your public feed doesn't reveal.

Thus your true execution priority may differ from what visible quantity alone suggests.

Your HFT strategy needs a **queue model**, not just:

```cpp
displayed_volume_ahead
```

---

# 17. Queue position estimation

Suppose you submit your bid when total displayed quantity at your level is:

```text
800
```

A simple estimate:

$$
Q_{ahead,0}=800
$$

Then observed events occur.

Trade:

```text
100 executed at level
```

Estimate:

$$
Q_{ahead}=700
$$

Then cancel:

```text
200 disappears
```

But was it:

```text
ahead of me?
```

or:

```text
behind me?
```

Public market data may not tell you depending on feed granularity.

If you have market-by-order data and exact ordering information, you can estimate much more accurately.

This is one reason MBO feeds are valuable.

---

# 18. Cancellation uncertainty

Suppose:

```text
Queue before me = 1000
Queue after me = 1000
```

Then the displayed level falls by:

```text
500
```

because of cancellations.

If your data doesn't identify individual order priority, you don't know whether those cancellations were:

```text
ahead
```

or:

```text
behind.
```

Different queue models use different assumptions.

For example:

```text
optimistic:
assume cancellations occurred ahead

pessimistic:
assume they occurred behind

probabilistic:
allocate cancellation probability based
on estimated queue distribution
```

This matters enormously for realistic backtesting.

---

# 19. Why simple backtests lie

Imagine historical data shows:

```text
Best Bid = 100.00
Best Ask = 100.01
```

Your backtest says:

```text
I would place BUY 100 @ 100.00.
```

Then a trade occurs:

```text
100 shares @ 100.00
```

Naive simulator:

```text
Great. I got filled.
```

But perhaps there were:

```text
20,000 shares ahead of you.
```

Then your fill probability was effectively zero.

Therefore HFT backtesting must model:

```text
queue position
latency
order acknowledgements
cancel latency
partial fills
matching rules
```

not simply historical prices.

---

# 20. Price improvement resets the entire game

Suppose queue:

```text
100.00
5000 shares
```

Instead of joining behind them you submit:

```text
BUY @ 100.01
```

assuming tick rules permit it and ask is still above.

Now you become:

```text
best bid
```

and potentially:

```text
first in queue
```

but you paid another tick.

So strategy faces:

$$
better\ queue\ position
\quad vs \quad
worse\ economic\ price
$$

This is one of the central trade-offs in market making.

---

# 21. Join versus improve versus cross

Given:

```text
Best Bid = 100.00
Best Ask = 100.02
```

a buyer can choose:

### Join

```text
BUY @ 100.00
```

You join the existing bid queue.

### Improve

```text
BUY @ 100.01
```

You become the new best bid.

### Cross

```text
BUY @ 100.02
```

You immediately trade against the ask.

Each has fundamentally different economics.

---

# 22. Passive versus aggressive execution

Passive:

```text
submit 100.00 bid
wait for seller
```

Possible advantages:

```text
better price
maker rebate on some venues
spread capture
```

Possible disadvantages:

```text
waiting
uncertain fill
adverse selection
queue risk
```

Aggressive:

```text
cross 100.02 ask
```

Advantages:

```text
high immediate execution probability
known position change
```

Disadvantages:

```text
pay spread
possibly pay taker fee
market impact
```

HFT strategies continuously solve this choice.

---

# 23. Maker-taker economics

Many U.S. equity venues use versions of a:

# maker-taker model

where:

```text
liquidity provider → rebate
liquidity remover → fee
```

Current Nasdaq pricing, for example, lists a $0.0030-per-share fee for many liquidity-removing executions, while rebates for adding liquidity depend on activity tiers and order characteristics. ([NASDAQ Trader][5])

Nasdaq PSX currently illustrates how venue economics can even differ materially by venue and liquidity type, with separate add rebates and remove fees. ([NASDAQ Trader][6])

Therefore strategy P&L isn't simply:

$$
SellPrice-BuyPrice
$$

More realistically:

$$
PnL =
TradingEdge
+
Rebates
-
Fees
-
AdverseSelection
-
HedgingCost
-
MarketImpact
$$

---

# 24. Why a one-cent spread does not mean one-cent profit

Suppose you:

```text
buy passively @ 100.00
sell passively @ 100.01
```

Gross spread:

```text
$0.01/share
```

Sounds excellent.

But perhaps:

```text
you only get bought when price is about to fall
```

and:

```text
you only get sold when price is about to rise.
```

This is:

# adverse selection.

It is arguably the central problem of market making.

---

# 25. Adverse selection

Suppose you quote:

```text
BUY 100.00
SELL 100.02
```

Then some trader suddenly aggressively sells to your:

```text
100.00 bid.
```

Two possibilities:

### Benign fill

They simply needed to sell.

Price remains near:

```text
100.01
```

Your fill may be good.

### Toxic fill

They know or infer that price is falling.

Immediately afterward:

```text
99.95
```

Now you own inventory bought at:

```text
100.00
```

worth approximately:

```text
99.95.
```

You lost:

```text
5 cents
```

despite supposedly capturing spread.

---

# 26. Markout

HFT firms often evaluate fill quality using:

# markout

Suppose you buy at:

$$
P_{fill}=100.00
$$

After 10 ms the mid is:

$$
P_{mid}(10ms)=99.98
$$

For a buy, simple markout:

$$
Markout = P_{mid}(t)-P_{fill}
$$

So:

$$
99.98-100.00=-0.02
$$

Bad.

If instead:

$$
P_{mid}(10ms)=100.03
$$

then:

$$
Markout=+0.03
$$

Your fill was favorable.

Professional systems measure markouts across many horizons:

```text
10 µs
100 µs
1 ms
10 ms
100 ms
1 s
```

because they reveal whether your quotes are being adversely selected.

---

# 27. Fill probability versus expected edge

Suppose quoting at best bid gives:

```text
fill probability = 80%
```

but conditional edge after fill is:

```text
-0.3 ticks
```

Bad strategy.

Another queue position gives:

```text
fill probability = 20%
```

but expected conditional edge:

```text
+0.5 ticks.
```

Potentially better.

Therefore you optimize something closer to:

$$
ExpectedPnL =
P(fill)
\times
E[PnL\mid fill]
$$

not:

```text
maximize fill rate.
```

---

# 28. Why HFT cancels so often

Suppose you posted:

```text
BUY @ 100.00
```

when estimated fair value was:

```text
100.01.
```

Then correlated-market information changes and fair value becomes:

```text
99.95.
```

Your bid is now terrible.

The strategy must:

```text
CANCEL
```

before another participant sells into it.

This is why:

```text
market data latency
+
strategy latency
+
cancel latency
```

are crucial.

---

# 29. Cancel latency is often more valuable than order-entry latency

Suppose your new quote can arrive in:

```text
1.0 µs
```

but stale quotes take:

```text
10 µs
```

to cancel.

You may repeatedly become the victim of informed traders.

Many market-making systems therefore care enormously about:

```text
market event
   ↓
decision that quote is stale
   ↓
cancel packet sent
```

This is often called a form of:

# quote protection latency.

---

# 30. Cancel/replace and priority

Here is an extremely important rule:

> Modifying an existing order can change its queue priority depending on what is modified and the venue's rules.

You should never assume:

```text
modify order
=
keep exact priority.
```

Venue protocols often distinguish between changes that preserve identity/priority and changes that effectively create a new order.

For your production system, priority consequences of every modification must be encoded in a venue-specific rule module.

---

# 31. Why size increases often matter

Conceptually, consider:

```text
Order A:
100 shares
timestamp t1
```

Now you increase it:

```text
100 → 10,000.
```

If you could keep your original priority for all new shares, you'd gain an unfair queue-position advantage over people who arrived later.

Venues therefore have specific rules governing modifications and priority.

Your HFT simulator must reproduce those rules exactly for the venue you intend to trade.

---

# 32. Hidden liquidity

Not all orders are displayed.

Nasdaq explicitly states that non-displayed shares generally execute after displayed shares at the same price, ordered by received time within that category. ([NASDAQ Trader][1])

So a simplified priority structure might be:

```text
Price level 100.00

Displayed A
Displayed B
Displayed C
────────────
Hidden X
Hidden Y
```

This means visible depth is not always equal to total available liquidity.

---

# 33. Why hidden orders exist

Suppose an institution wants to sell:

```text
1,000,000 shares.
```

Showing:

```text
SELL 1,000,000
```

may reveal its intention and move the market.

Instead it may prefer some form of:

```text
hidden
reserve
iceberg
```

order.

This reduces information leakage.

---

# 34. Iceberg / reserve order

An iceberg might contain:

```text
Total = 10,000
Displayed = 500
Hidden = 9,500
```

Market sees:

```text
500
```

When the displayed 500 fills:

```text
another slice may refresh.
```

Conceptually:

```text
Total remaining = 9,500

display another 500

remaining hidden = 9,000
```

CME/EBS explicitly describes iceberg orders as exposing only a portion of the total quantity while the rest remains hidden. ([CME Group][7])

The exact priority of replenished quantities is venue-specific and critical to simulation.

---

# 35. Hidden liquidity changes observed execution behavior

Suppose ask displays:

```text
100.05 × 100
```

You send:

```text
BUY 500 @ 100.05
```

and strangely receive:

```text
500 @ 100.05.
```

Why?

Because there may have been:

```text
400 hidden
```

behind the visible 100.

This phenomenon matters for liquidity detection and execution models.

---

# 36. Pegged orders

A pegged order doesn't necessarily have one fixed price.

Instead its price follows a reference.

Possible references include:

```text
best bid
best ask
midpoint
```

Nasdaq currently supports Midpoint Peg orders that price to the NBBO midpoint. ([NASDAQ Trader][8])

If:

```text
NBBO = 100.00 × 100.02
```

midpoint:

$$
100.01
$$

A midpoint peg buy might therefore rest at:

```text
100.01
```

without the participant continuously calculating/replacing the order itself.

---

# 37. Why peg orders matter

Suppose NBBO moves:

```text
100.00 / 100.02
```

to:

```text
100.01 / 100.03.
```

Midpoint moves:

```text
100.01
```

to:

```text
100.02.
```

The exchange reprices the pegged order according to its rules.

This creates subtleties:

```text
Does repricing affect priority?
When exactly is repricing processed?
Can the order lock/cross?
What happens on odd half-tick midpoints?
```

Your strategy must understand venue-specific semantics.

---

# 38. Post-only

Suppose current ask:

```text
100.01.
```

You submit:

```text
BUY LIMIT 100.01
```

Normally that would take liquidity immediately.

But perhaps your goal is:

```text
provide liquidity only.
```

A **post-only** instruction asks the venue to avoid ordinary taker behavior according to that venue's rules.

Nasdaq supports Post Only functionality and also combines it with midpoint-pegged functionality in MPPO orders. ([NASDAQ Trader][9])

This matters when maker/taker economics are part of the strategy.

---

# 39. Why order-type complexity can become dangerous

A beginner sees:

```text
limit
market
IOC
FOK
peg
post-only
hidden
iceberg
```

and thinks:

> more order types = more opportunities.

But every order type increases the state machine:

```text
when does it reprice?
does it retain priority?
what happens during lock/cross?
is it displayed?
can it route?
how does SMP interact?
what happens at auction?
```

Production-grade HFT should support only what it understands exactly.

---

# 40. Self-match prevention

Suppose your own strategy has:

```text
BUY @ 100.00
```

resting.

Another strategy from your firm sends:

```text
SELL @ 100.00.
```

Without protection you could trade against yourself.

This creates:

```text
unnecessary fees
fake volume
operational complexity
potential regulatory concerns
```

so exchanges offer:

# Self-Match Prevention — SMP

CME allows participants to attach SMP identifiers; when matching buy and sell orders have the same SMP ID, the exchange cancels the resting or aggressing side according to the selected instruction. ([CME Group][10])

Nasdaq is introducing enhanced order-level SMP controls in September 2026, including configurable strategies such as cancelling oldest/newest or decrementing both. ([NASDAQ Trader][11])

This is a good example of why exchange functionality is not static and must be implemented from current venue specifications.

---

# 41. Continuous trading versus auctions

So far we assumed:

```text
one incoming order
↓
immediately match against book
```

That is:

# continuous trading.

But exchanges also use:

# auctions / crosses

where orders accumulate and are matched together at a single calculated price.

---

# 42. Opening auction

Before regular continuous trading starts, many orders have accumulated overnight or pre-open.

The exchange needs to determine:

```text
What price should trading begin at?
```

Rather than arbitrarily executing the first order, an opening auction combines available interest and chooses an auction clearing price based on venue rules.

Nasdaq's Opening Cross occurs at 9:30 ET and accepts special orders such as Market-On-Open and Limit-On-Open; it disseminates imbalance information before the cross. ([NASDAQ Trader][12])

---

# 43. Closing auction

Similarly, at the market close, enormous institutional interest may be tied specifically to the official closing print.

Example:

```text
Index fund must rebalance
↓
needs shares at closing price
```

A closing auction pools:

```text
buy interest
sell interest
```

and determines one closing price.

Nasdaq describes its opening and closing crosses as single-price price-discovery facilities combining cross interest with the continuous book. ([NASDAQ Trader][13])

---

# 44. Auction imbalance

Suppose close-auction interest currently is:

```text
BUY 2,000,000

SELL 1,200,000
```

Imbalance:

```text
800,000 BUY.
```

This information may be disseminated before the auction.

HFT/market makers may react by supplying offsetting liquidity.

Auction dynamics are therefore their own microstructure regime.

---

# 45. Auction price formation

A simplified auction might search candidate prices.

For each price calculate:

```text
executable buy quantity
executable sell quantity
```

Matched quantity:

$$
Q_{match}(P)=
\min(Q_{buy}(P),Q_{sell}(P))
$$

Then the engine chooses a price according to venue-specific rules involving things such as:

```text
maximize executable volume
minimize imbalance
reference-price tie breakers
```

Do not assume every venue chooses auction prices identically.

---

# 46. Market orders during auction

Auction market orders are especially aggressive because they indicate:

```text
I want execution in the auction,
price less important.
```

Limit auction orders impose a price constraint.

This affects the equilibrium clearing price.

---

# 47. Matching algorithms beyond FIFO

Now we come to something essential for futures/options HFT.

Not every market uses strict price-time FIFO.

CME supports product-dependent algorithms including:

```text
FIFO
FIFO with LMM
FIFO with Top Order
Pro-Rata
Threshold Pro-Rata
Configurable hybrids
```

among others. ([CME Group][14])

Eurex similarly documents price/time, pro-rata and time-pro-rata allocation approaches. ([Deutsche Börse Group][15])

This radically changes strategy incentives.

---

# 48. Pro-rata matching

Suppose best offer contains:

```text
Order A = 100
Order B = 300
Order C = 600
```

Total:

$$
1000
$$

Incoming buy:

```text
BUY 500
```

Under pure proportional allocation, approximate fills:

$$
A = 500\times\frac{100}{1000}=50
$$

$$
B = 500\times\frac{300}{1000}=150
$$

$$
C = 500\times\frac{600}{1000}=300
$$

Everyone gets something proportional to their resting quantity.

Eurex describes its pro-rata method as sharing incoming quantity among orders at the best price in proportion to open quantity. ([Deutsche Börse Group][15])

---

# 49. Why pro-rata changes HFT behavior

Under FIFO:

```text
being early
```

is enormously important.

Under pro-rata:

```text
showing more quantity
```

can increase allocation.

That can create incentives for larger resting orders.

Therefore your optimization target changes from:

```text
latency → queue priority
```

toward something involving:

```text
latency
+
displayed size
+
allocation formula.
```

---

# 50. FIFO versus pro-rata

Compare.

### FIFO

At same price:

```text
A = 100 @ time 1
B = 500 @ time 2
```

Incoming:

```text
BUY 100
```

A receives:

```text
100.
```

B gets:

```text
0.
```

### Pro-rata

Total resting:

```text
600
```

Incoming:

```text
100.
```

Approximate proportional allocation:

```text
A ≈ 16.7
B ≈ 83.3
```

subject to venue rounding/priority rules.

Completely different trading incentive.

---

# 51. Time-pro-rata and hybrid algorithms

Venues may deliberately combine incentives.

Eurex documents **time-pro-rata**, where earlier orders receive a larger allocation than pure pro-rata would give, but allocation remains less extreme than strict FIFO. ([Deutsche Börse Group][15])

CME also uses hybrid algorithms where portions of an incoming order can be allocated through mechanisms such as top-order priority, LMM allocations, FIFO and pro-rata. ([CME Group][16])

Therefore:

> “I understand queue position” is not enough.

You must understand the **allocation formula for each product**.

---

# 52. Lead Market Maker priority

Some products provide special treatment for designated:

```text
LMM
```

or other market-making roles.

These participants may receive a portion of allocation according to venue rules before or alongside the normal allocation algorithm.

That creates another variable in execution probability.

---

# 53. Matching-engine configuration is instrument-specific

A production trading system should conceptually store:

```cpp
Instrument {
    tickSize;
    matchingAlgorithm;
    auctionRules;
    priceLimits;
    orderTypesSupported;
    ...
}
```

not:

```cpp
matchingAlgorithm = FIFO; // everywhere
```

CME even publishes a matching-algorithm identifier in security-definition data for products. ([CME Group][16])

That is the correct architectural approach.

---

# 54. Complex instruments and implied liquidity

Futures markets introduce another advanced concept:

# implied orders.

Suppose:

```text
Spread = Future_A - Future_B
```

There may be an outright market in:

```text
A
B
```

and also a direct order book for:

```text
A-B spread.
```

The exchange can infer synthetic liquidity.

Example:

```text
A = 101
B = 100
```

implies spread:

$$
A-B=1
$$

Synthetic/implied matching can allow an order in one market to execute using liquidity in related legs.

Eurex describes this broader concept as **Synthetic Matching**, where orders involving different simple/complex instruments can execute together. ([Deutsche Börse Group][15])

This makes futures matching engines far more complicated than a single equity order book.

---

# 55. Why implied liquidity matters for strategy

Suppose you see displayed spread liquidity:

```text
Spread A-B @ 1.0
```

But that may arise from:

```text
actual spread order
```

or:

```text
implied combination of outright orders.
```

Your execution behavior, leg risk and reaction to fills can differ.

Professional futures strategies therefore model:

```text
outright books
spread books
implied relationships
```

together.

---

# 56. Tick size

Suppose tick size:

```text
$0.01.
```

Legal prices:

```text
100.00
100.01
100.02
```

not:

```text
100.005.
```

Tick size discretizes price competition.

That has a huge effect on microstructure.

If tick size is large relative to volatility:

```text
many participants queue at same price
```

and queue position becomes extremely valuable.

If tick size is very small:

```text
participants can constantly price-improve by one tick.
```

---

# 57. Large tick versus small tick markets

### Large-tick market

Suppose fair value:

```text
100.004
```

but legal prices:

```text
100.00
100.01.
```

Spread often remains:

```text
1 tick.
```

Participants compete through:

```text
queue position.
```

### Small-tick market

Legal prices:

```text
100.000
100.001
100.002
...
```

Participants can frequently step ahead.

Competition shifts more toward:

```text
price improvement.
```

This determines what latency advantage is economically useful.

---

# 58. Spread

Bid-ask spread is:

$$
Spread=Ask-Bid
$$

but an HFT market maker should think of the spread as compensation for several risks:

$$
SpreadCompensation
\approx
AdverseSelection
+
InventoryRisk
+
Fees
+
LatencyRisk
+
OperationalRisk
+
Profit
$$

If the market becomes more toxic:

```text
spread often widens.
```

If competition increases:

```text
spread often narrows.
```

---

# 59. Microprice

Midpoint treats bid and ask equally.

But suppose:

```text
Bid = 100.00, quantity = 1000
Ask = 100.01, quantity = 100
```

The book is heavily bid-weighted.

A simple microprice formulation is:

$$
Microprice =
\frac{
AskPrice\times BidQty+
BidPrice\times AskQty
}{
BidQty+AskQty
}
$$

Plugging in:

$$
\frac{
100.01(1000)+
100.00(100)
}{
1100
}
$$

≈

$$
100.00909.
$$

Notice:

```text
Midpoint = 100.005

Microprice ≈ 100.009
```

Microprice leans toward the ask because bid pressure is much stronger.

This is one simple microstructure signal.

---

# 60. Order-book imbalance

Another common feature:

$$
I=
\frac{Q_b-Q_a}
{Q_b+Q_a}
$$

If:

```text
Qbid = 1000
Qask = 100
```

then:

$$
I=
\frac{1000-100}
{1100}
=
0.818.
$$

Strong positive imbalance.

But this does not guarantee price rise.

Why?

Because:

```text
displayed size can cancel
hidden size may exist
some orders may be stale
large displayed orders may never trade
```

HFT signals are probabilistic.

---

# 61. Order flow imbalance

Instead of only looking at static depth, consider events.

During the last 100 µs:

```text
+500 bid added
-300 ask cancelled
+400 aggressive buys
```

This may indicate upward pressure.

So strategies often care about:

```text
state
+
event dynamics.
```

That maps directly to the architecture we discussed earlier:

```text
state → seqlock/shared snapshot

events → SPMC/SPSC stream.
```

---

# 62. Trade aggressor

If a trade executes against the ask, the aggressor was typically:

```text
buyer.
```

If it executes against the bid:

```text
seller.
```

A sequence of aggressive buys can signal:

```text
buy pressure.
```

But interpreting this requires careful venue semantics and data.

---

# 63. Queue depletion

Suppose ask:

```text
100.01 × 500
```

Trades:

```text
100
100
100
100
```

Now:

```text
100 left.
```

If that last 100 disappears:

```text
best ask may move to 100.02.
```

Therefore a strategy may estimate:

```text
probability current best price gets depleted soon.
```

This is another microstructure prediction problem.

---

# 64. Cancel-to-trade dynamics

Imagine best ask:

```text
100.01 × 5000
```

Suddenly:

```text
4500 cancel
```

Leaving:

```text
500.
```

That is qualitatively different from:

```text
4500 executing.
```

Both reduce displayed quantity, but one is liquidity withdrawal while the other is aggressive consumption.

A proper event-driven feed exposes this distinction.

---

# 65. Spoofing versus legitimate order updates

Large orders appearing and disappearing can reflect many legitimate reasons:

```text
model change
risk change
hedging
inventory
market move
```

but intentionally submitting orders to mislead others may violate market-manipulation rules.

Your architecture should be capable of handling rapid cancellation patterns without assuming every such event has the same economic interpretation.

---

# 66. Queue jumping

Suppose:

```text
Best bid = 100.00
Best ask = 100.02.
```

Instead of joining:

```text
100.00
```

you enter:

```text
100.01.
```

You now jump ahead of everyone at 100.00 through **price priority**.

They aren't literally moved backward within their price.

Your better price creates a new priority level.

---

# 67. Latency race when fair value changes

Imagine:

```text
old fair value = 100.00
new information arrives
new fair value = 99.95
```

Several market makers are currently bidding:

```text
100.00.
```

Everyone sees the same trigger.

Race:

```text
market event
   │
   ├── Firm A detects
   │       ↓
   │     cancel
   │
   ├── Firm B detects
   │       ↓
   │     cancel
   │
   └── Aggressor detects
           ↓
      SELL into stale bids
```

The winner may be determined by hundreds of nanoseconds.

This is the core **stale quote race**.

---

# 68. Why being first isn't always good

Suppose you are always:

```text
first bid.
```

Sounds perfect.

But then every informed seller hits you first.

You may receive:

```text
the most toxic flow.
```

Queue priority is valuable only if:

$$
ExpectedEdgeAfterFill>0
$$

So good market making isn't:

```text
always be first.
```

It is:

```text
be first when being first is valuable.
```

---

# 69. Queue value

You can think of a queue position as having an economic value.

Suppose expected fill probability at position \(q\) is:

$$
P_{fill}(q)
$$

and expected profit conditional on fill:

$$
E[\Pi \mid fill,q]
$$

Then:

$$
QueueValue(q)
=
P_{fill}(q)
\times
E[\Pi\mid fill,q].
$$

Moving forward may increase fill probability but can also increase exposure to toxic flow.

This is the type of thinking behind sophisticated quoting systems.

---

# 70. Inventory risk

Suppose your strategy keeps buying.

Position becomes:

```text
+100
+500
+1000
+5000
```

Even if each trade had positive spread capture, you now carry directional risk.

So market makers modify quotes based on inventory.

Example:

```text
position = strongly long
```

Strategy may:

```text
reduce buy quote aggressiveness
increase sell quote aggressiveness.
```

Conceptually shift reservation price:

$$
r=
FairValue-\lambda \cdot Inventory
$$

where \(\lambda\) represents risk aversion.

---

# 71. Inventory skew

Neutral:

```text
Fair = 100

Bid 99.99
Ask 100.01
```

Long inventory:

```text
Bid 99.98
Ask 100.00
```

You've shifted both quotes downward to encourage selling and discourage further buying.

Short inventory might do the opposite.

This is a central market-making mechanism.

---

# 72. Hedge versus skew

Suppose you're long an option.

You might reduce risk by:

### changing quotes

```text
inventory skew
```

or:

### trading a related instrument

```text
hedge future/stock.
```

Those mechanisms have different:

```text
latency
transaction cost
market impact
risk reduction.
```

The trading architecture must support both.

---

# 73. Order state machine

Now connect microstructure back to production engineering.

Every order must have a precise internal lifecycle.

Example:

```text
CREATED
   ↓
PENDING_NEW
   ↓
LIVE
   ├────► PARTIALLY_FILLED
   │
   ├────► PENDING_CANCEL
   │            ↓
   │         CANCELLED
   │
   └────► FILLED
```

Possible additional states:

```text
REJECTED
EXPIRED
PENDING_REPLACE
REPLACED
UNKNOWN
```

Your strategy should never casually modify order state without reconciliation to exchange acknowledgements.

---

# 74. Local desired state versus exchange-confirmed state

This is crucial.

You decide:

```text
Cancel Order A
```

At that instant:

```text
Desired state = CANCELLED
```

but exchange state may still be:

```text
LIVE
```

until the cancel reaches the venue.

During that flight time:

```text
Order A can still fill.
```

Therefore:

```text
requested cancel
≠
cancelled.
```

Production HFT systems must model this explicitly.

---

# 75. The cancel race

Timeline:

```text
T0   you decide cancel
T1   cancel leaves NIC
T2   seller order enters exchange
T3   cancel enters exchange
```

If seller reaches matching engine first:

```text
you fill.
```

If cancel reaches first:

```text
you escape.
```

This is why the HFT system must continue accepting executions against:

```text
PENDING_CANCEL
```

orders.

Anything else is a correctness bug.

---

# 76. Replace race

Similar problem:

```text
old order live
↓
replace requested
↓
fill arrives
↓
replace ack arrives
```

Order-state machines must handle every possible event ordering.

These are exactly the kinds of rare cases that break simplistic HFT prototypes.

---

# 77. Matching engine as deterministic state machine

Now we move from using a matching engine to **how one is commonly designed conceptually**.

The most important idea is:

> For a given ordered sequence of commands, the engine must produce deterministic results.

Conceptually:

```text
Input Event 1
Input Event 2
Input Event 3
...
        │
        ▼
DETERMINISTIC STATE MACHINE
        │
        ├── book updates
        ├── trades
        ├── acknowledgements
        └── market-data events
```

If two replicas process exactly the same ordered commands, they should reach the same state.

---

# 78. Why one ordered input stream is so powerful

Imagine two orders arrive “simultaneously.”

The system still needs to decide:

```text
which is first?
```

The matching engine requires a total order.

Conceptually:

```text
Gateway A ─┐
Gateway B ─┼──► SEQUENCER ─► Engine
Gateway C ─┘
```

The sequencer creates:

```text
1
2
3
4
...
```

Once ordered, the engine processes commands deterministically.

This is conceptually much easier than having 32 threads simultaneously mutate one order book.

---

# 79. Why a matching engine may prefer one writer per partition

Suppose many threads modify the same book:

```text
Thread A ─┐
Thread B ─┼──► same order book
Thread C ─┘
```

Now you need:

```text
locks
atomic operations
concurrent ordering
conflict resolution.
```

Instead:

```text
one partition
   ↓
one logical event owner
   ↓
orders processed sequentially
```

can be both:

```text
faster
more deterministic
easier to recover
easier to test.
```

This is a common matching-engine design pattern, not a claim that every exchange implements its engine identically.

It is the same principle Gross emphasized:

# single-writer ownership.

---

# 80. Partitioning the exchange

How do you scale if one engine core is insufficient?

Don't necessarily multithread one book.

Instead partition instruments:

```text
Partition 1
AAPL
MSFT
META

Partition 2
GOOG
AMZN
NVDA

Partition 3
...
```

Each partition can run an independent deterministic event loop.

This gives:

```text
parallelism across instruments
```

without:

```text
concurrency inside one book.
```

---

# 81. Matching-engine hot path

Conceptually:

```cpp
while (true) {

    cmd = nextCommand();

    switch (cmd.type) {

        NEW:
            processNew(cmd);
            break;

        CANCEL:
            processCancel(cmd);
            break;

        MODIFY:
            processModify(cmd);
            break;
    }

    publishEvents();
}
```

Obviously production engines are more sophisticated, but the critical characteristic is:

```text
tiny deterministic loop.
```

---

# 82. Data structures inside the matching engine

The engine needs fast access by:

```text
price
order ID
best bid
best ask
FIFO position.
```

A conceptual structure could be:

```text
Order ID index
     │
     ▼
Order object
     │
     ▼
Price level
```

Price level:

```text
Price = 100.00

head → Order A → Order B → Order C → tail
```

The book:

```text
Bid price levels

100.00
99.99
99.98
...
```

and:

```text
Ask price levels

100.01
100.02
100.03
...
```

Since the engine controls its universe and price grid, specialized data structures can be far better than generic containers.

---

# 83. Integer ticks inside matching engines

Suppose tick:

```text
$0.01.
```

Instead of:

```cpp
double price = 100.01;
```

use:

```cpp
int price_ticks = 10001;
```

Then comparison:

```cpp
if (buy_price >= best_ask)
```

is exact integer comparison.

This avoids floating-point ambiguity and maps naturally onto price-level indexing.

---

# 84. Direct price indexing

Suppose permitted price tick index runs:

```text
0 ... 1,000,000.
```

A specialized system may use array-like structures:

```text
levels[price_tick]
```

rather than balanced trees.

This gives:

```text
O(1) level lookup
excellent predictable memory
```

at the cost of memory or requiring compressed/page-based structures.

This is the same performance philosophy we saw in the Gross talks.

---

# 85. Order-ID lookup

Cancellation arrives as:

```text
Cancel OrderID 817231
```

Engine needs:

```text
order = lookup(817231)
```

immediately.

Therefore order-ID indexing must be extremely fast.

Possible techniques include:

```text
preallocated arrays
flat hash tables
dense IDs
custom pools
```

depending on the venue.

Generic pointer-heavy hash maps may not be ideal for the hottest path.

---

# 86. Matching an aggressive limit order

Book:

```text
ASK

100.01  100
100.02  200
100.03  300
```

Incoming:

```text
BUY 400 @ 100.02
```

Matching:

```text
100 @ 100.01
200 @ 100.02
```

300 filled.

Remaining:

```text
100.
```

But your limit is:

```text
100.02
```

and next ask:

```text
100.03.
```

Cannot cross.

So remaining 100 rests:

```text
BID 100 @ 100.02
```

assuming time-in-force allows resting.

---

# 87. Matching pseudocode

Conceptually:

```cpp
while (incoming.qty > 0 &&
       incoming.price >= bestAsk.price) {

    resting = bestAsk.front();

    tradeQty =
        min(incoming.qty,
            resting.qty);

    execute(incoming,
            resting,
            tradeQty);

    if (resting.qty == 0)
        remove(resting);
}

if (incoming.qty > 0)
    addToBook(incoming);
```

This is almost trivial.

But add:

```text
hidden orders
pro-rata
SMP
icebergs
price bands
auctions
special priorities
pegs
spread matching
```

and the state machine gets far richer.

---

# 88. Execution price

Under normal resting/aggressing matching, execution generally occurs at the resting order's price.

Example:

Resting:

```text
SELL @ 100.00
```

Incoming:

```text
BUY LIMIT @ 100.05
```

You usually don't pay:

```text
100.05
```

just because that was your maximum.

You trade at:

```text
100.00
```

subject to venue rules.

That's why limit price means:

```text
worst acceptable price
```

not:

```text
requested exact price.
```

---

# 89. Outputs of a matching-engine command

One input:

```text
NEW BUY 500
```

can produce several outputs:

```text
ACK
TRADE 1
TRADE 2
BOOK REMOVE
BOOK MODIFY
BOOK ADD
POSITION/CLEARING EVENT
MARKET DATA
```

Therefore:

```text
one order command
```

does not correspond to:

```text
one response.
```

Your participant order manager must handle multiple executions.

---

# 90. Matching engine and market-data generator

Conceptually:

```text
Command
  ↓
Matching Engine
  ↓
State transitions
  ├────► private execution/ack
  └────► market-data events
```

Your feed handler later reconstructs the exchange book using those public events.

So:

```text
exchange authoritative book
       ↓ encode
market-data events
       ↓ network
your reconstructed book
```

You are essentially running a distributed replica of part of the exchange's state.

---

# 91. Why market data can lag order acknowledgement

Different output pipelines may exist:

```text
Matching Engine
    │
    ├──── private response
    │
    └──── public market data
```

They can have different:

```text
serialization
network path
queueing
```

Therefore you may receive:

```text
private fill acknowledgement
```

before the corresponding public book update.

Your architecture must tolerate that.

---

# 92. Event sourcing analogy

A matching engine can be understood conceptually as:

```text
command log
    ↓
deterministic state machine
    ↓
current order book
```

If you persist:

```text
ordered command/event stream
```

you can potentially rebuild state by replaying it.

That gives you:

```text
recovery
auditability
reproducibility.
```

Do not put a relational database transaction in the matching hot path; persistence is usually designed around append/replay-style mechanisms when ultra-low latency is required.

---

# 93. Hot standby

Suppose primary matching-engine process fails.

You do not want:

```text
entire market lost.
```

A common conceptual architecture is:

```text
                   ordered events
                        │
             ┌──────────┴──────────┐
             ▼                     ▼
          Primary               Standby
             │                     │
        process events          same events
             │                     │
        current state           same state
```

If processing is deterministic:

```text
same ordered input
       ↓
same state.
```

Then failover becomes substantially easier to reason about.

Again, exact venue architectures vary.

---

# 94. Journaling

A durable ordered journal can sit conceptually beside the engine:

```text
Sequenced command
       │
       ├──► matching engine
       │
       └──► journal
```

Important design decision:

```text
Must durability complete before acknowledgement?
```

Stronger durability:

```text
safer
```

but potentially:

```text
higher latency.
```

Exchanges make carefully engineered trade-offs here; their precise internals are generally proprietary.

---

# 95. Snapshot + journal recovery

Suppose engine has processed:

```text
10 billion commands.
```

Replaying everything from day one would be impractical.

Instead:

```text
snapshot at sequence N
          +
journal N+1...
```

Recovery:

```text
load snapshot
    ↓
replay recent journal
    ↓
resume.
```

This is analogous to the market-data recovery architecture we discussed earlier.

---

# 96. Determinism requirements

For deterministic replication, avoid hidden nondeterminism such as:

```text
unordered thread races
wall-clock calls affecting matching
random iteration orders
uncontrolled floating-point variability
```

Any rule requiring time should use an explicitly sequenced/time-stamped input rather than asking random workers for:

```cpp
now()
```

inside business logic.

---

# 97. Fairness and sequencing

An exchange cannot simply give some participants random preference.

Its documented matching rules define priority.

The technical infrastructure must therefore turn incoming traffic into a deterministic order according to the exchange's fairness/access model.

You can think of the core contract as:

```text
same rules
+
well-defined input ordering
+
deterministic allocation.
```

---

# 98. Gateway versus matching-engine latency

Your order path is:

```text
NIC
 ↓
exchange gateway
 ↓
sequencing/routing
 ↓
matching engine
```

Therefore when an HFT firm optimizes:

```text
my NIC TX by 50 ns
```

it is competing to enter the venue's priority ordering earlier.

But the participant usually cannot optimize:

```text
the exchange's internal engine.
```

That part is shared external infrastructure.

---

# 99. Price-time priority gives latency economic value

Now the full causal chain should be obvious:

```text
faster feed receive
      ↓
faster signal
      ↓
faster order construction
      ↓
faster NIC TX
      ↓
earlier exchange arrival
      ↓
earlier priority timestamp
      ↓
better queue position
      ↓
higher fill probability
      ↓
different P&L
```

This is why optimizing:

```text
cache line
branch
PCIe transaction
```

can matter financially.

---

# 100. But pro-rata changes the causal chain

For pro-rata:

```text
fast arrival
```

still matters for some rules and getting your order present before the aggressor arrives, but allocation depends much more heavily on:

```text
your displayed quantity relative to total.
```

So:

```text
latency optimization
```

cannot be separated from:

```text
matching algorithm.
```

The microstructure determines the value of hardware speed.

---

# 101. Market impact

Suppose you submit:

```text
BUY 50,000
```

against:

```text
100.01 × 1,000
100.02 × 2,000
100.03 × 5,000
...
```

Your aggressive order consumes many levels.

Other participants observe:

```text
large buyer
```

and may react.

So your own trading changes the market.

This is:

# market impact.

Therefore execution strategy must solve:

$$
NeedToTrade
\quad vs \quad
InformationLeakage
\quad vs \quad
MarketImpact.
$$

---

# 102. Signaling risk

Even passive orders reveal information.

Suppose you suddenly place:

```text
BUY 100,000
```

at best bid.

Others may infer:

```text
someone strongly wants to buy.
```

Your order becomes a signal.

That's why iceberg/hidden/minimum-quantity and execution tactics exist.

---

# 103. Latency arbitrage conceptually

Suppose instrument A and B are tightly related.

A moves:

```text
100 → 101.
```

B's correct fair value should move too.

But a market maker in B still displays a stale offer.

Fast participant:

```text
sees A move
   ↓
computes B fair value
   ↓
buys stale B offer
```

Slow market maker:

```text
tries cancel
```

but loses race.

This is one reason cross-market latency is valuable.

---

# 104. Why market makers invest heavily in feed speed

Market maker's loss:

```text
stale quote gets hit.
```

So they need the fastest possible:

```text
information
→ fair-value update
→ cancel/replace.
```

Aggressive latency-arbitrage participant needs:

```text
information
→ opportunity recognition
→ aggressive order.
```

Thus:

```text
maker cancel
```

and:

```text
taker hit
```

race each other.

---

# 105. Queue-reactive strategy

A sophisticated market maker doesn't simply say:

```text
fair value = 100
quote 99.99 / 100.01.
```

It also considers:

```text
queue ahead
queue behind
recent cancels
recent aggressive flow
expected fill probability
inventory
volatility
fees
hedge price
latency
```

Conceptually:

$$
QuoteDecision =
f(
FairValue,
QueueState,
OrderFlow,
Inventory,
Risk,
Fees,
Latency
)
$$

---

# 106. Production HFT model of one quote

Your in-memory quote state might conceptually contain:

```cpp
struct QuoteState {
    Price desired_price;

    Price live_price;

    Quantity desired_qty;
    Quantity live_qty;

    OrderId exchange_order_id;

    OrderStatus status;

    Quantity queue_ahead_estimate;

    Timestamp exchange_ack_time;
    Timestamp last_market_event;

    double expected_fill_probability;
    double expected_markout;
};
```

Obviously you would use more mechanically sympathetic types in a real hot path, but these are the semantics your system needs.

---

# 107. Quote decision state machine

Conceptually:

```text
market update
   ↓
recalculate fair value
   ↓
compare desired quote with live quote
   │
   ├── unchanged → do nothing
   │
   ├── no longer safe → cancel
   │
   ├── price should move → replace
   │
   ├── desired size changed → modify
   │
   └── no live order → new order
```

The fastest message is:

```text
the message you do not need to send.
```

Avoid unnecessary quote churn.

---

# 108. Messaging-rate limits

Exchanges don't allow infinite gratuitous messages.

They may impose:

```text
technical rate limits
messaging efficiency programs
fees
throttles
```

CME's SMP documentation, for example, notes that certain unsolicited SMP cancellations count under its messaging efficiency framework. ([CME Group][10])

Therefore strategy optimization includes:

```text
latency
+
message efficiency.
```

---

# 109. Exchange throttling

If your strategy wants to send:

```text
1,000,000 cancel/replaces/sec
```

but venue/session permits much less, your local gateway may queue.

Then your beautifully optimized strategy creates:

```text
orders waiting in local throttle queue.
```

That can destroy latency.

Therefore the HFT order gateway needs local knowledge of:

```text
session throughput limits
burst limits
venue message budgets.
```

---

# 110. Never let strategy blindly write to NIC

A production architecture usually needs a disciplined boundary:

```text
Strategy intent
     ↓
Order state validation
     ↓
Risk
     ↓
Session/throttle logic
     ↓
Encoder
     ↓
TX
```

Even when those are fused into one process/core for latency, the logical responsibilities remain distinct.

---

# 111. A complete matching-engine mental model

You should now picture the exchange side approximately as:

```text
Participants
    │
    │ NEW/CANCEL/REPLACE
    ▼
┌──────────────────────────┐
│ Order Entry Gateways     │
│ protocol/session checks  │
└────────────┬─────────────┘
             │
             ▼
       ORDER SEQUENCER
             │
             ▼
┌──────────────────────────┐
│ MATCHING PARTITION       │
│                          │
│ instrument book          │
│ order-id index           │
│ price levels             │
│ FIFO/pro-rata rules      │
│ hidden/reserve logic     │
│ SMP                      │
│ order-type state         │
│ auction state            │
└────────────┬─────────────┘
             │
       ┌─────┴────────┐
       │              │
       ▼              ▼
Private Responses   Market Data
ACK/FILL/CANCEL     Add/Modify/Trade
       │              │
       ▼              ▼
 participant       multicast
```

Alongside it:

```text
journal
replication
standby
snapshots
clearing events
surveillance
risk
monitoring
```

The exact architecture of a specific exchange may differ and is generally proprietary; this is a common systems-design model for understanding the responsibilities.

---

# 112. Now connect both sides of the system

The entire loop is:

```text
              EXCHANGE MATCHING ENGINE
                       │
                  market data
                       │
                       ▼
                 YOUR NIC RX
                       │
                  feed handler
                       │
                       ▼
                 LOCAL BOOK
                       │
                       ▼
                  STRATEGY
                       │
             ┌─────────┼─────────┐
             │         │         │
          fair      queue     inventory
          value     model       risk
             │         │         │
             └─────────┼─────────┘
                       ▼
                 quote decision
                       │
                       ▼
                  risk checks
                       │
                       ▼
                 YOUR NIC TX
                       │
                       ▼
                EXCHANGE GATEWAY
                       │
                       ▼
                MATCHING ENGINE
                       │
                       ▼
             Queue priority / fill
                       │
                       ▼
                  market changes
                       │
                       └──────────────► repeat
```

This is the core feedback loop of electronic market making.

---

# 113. What your HFT simulator must eventually reproduce

If you want to build a **production-grade HFT-like system**, your simulator cannot merely implement:

```text
if price predicted up:
    buy
```

It eventually needs to reproduce four separate dimensions:

| Dimension          | Simulator must model                               |
| ------------------ | -------------------------------------------------- |
| **Network**        | market-data latency, order latency, cancel latency |
| **Exchange**       | matching algorithm, order priority, partial fills  |
| **Microstructure** | queue position, hidden/reserve assumptions, fees   |
| **Own system**     | pending states, risk, throttles, order lifecycle   |

Otherwise your backtest and production system will describe different worlds.

---

# 114. Minimal matching engine you should build first

For your own low-latency project, don't begin with every order type.

Version 1 should implement:

```text
single instrument partition

limit orders

price-time priority

NEW

CANCEL

partial fills

full fills

best bid / best ask

order IDs

sequence numbers

execution reports

market-data events
```

Then add:

```text
MODIFY/REPLACE
IOC
FOK
market-with-protection style behavior
```

Only afterwards:

```text
hidden/reserve
pegs
SMP
auctions
pro-rata
```

This gives you a sane correctness path before optimizing.

---

# 115. Recommended matching-engine architecture for your project

For the first serious implementation:

```text
             ORDER GATEWAY
                  │
                  ▼
             SequencedCommand
                  │
                  ▼
       ┌─────────────────────┐
       │ MATCHING CORE       │
       │ one writer/thread   │
       │                     │
       │ no allocations      │
       │ integer prices      │
       │ object pools        │
       │ deterministic       │
       └─────────┬───────────┘
                 │
        ┌────────┴────────┐
        ▼                 ▼
ExecutionEvents      MarketDataEvents
```

Data structures conceptually:

```text
ordersById
+
bidLevels
+
askLevels
+
FIFO lists per level.
```

All memory allocated at startup.

---

# 116. Don't optimize too early—but design for optimization

Version 1 should prioritize:

```text
correct semantics
determinism
replayability.
```

But avoid architectural traps such as:

```text
database per order
heap allocation per event
mutex around every book
JSON messages
shared mutable state across many threads.
```

Then benchmark and optimize:

```text
container layout
cache locality
branch behavior
batching
NUMA
kernel bypass.
```

Exactly following the Gross philosophy.

---

# 117. Testing the matching engine

Correctness tests should include scenarios such as:

```text
same-price FIFO
multiple price levels
partial fill
full fill
incoming order remainder
cancel first order
cancel middle order
cancel unknown ID
replace order
IOC partial
FOK unavailable quantity
self-match rule
hidden replenishment
auction
```

More importantly:

# invariant testing.

For every state:

$$
Bid_{best}<Ask_{best}
$$

outside of transient matching operations.

And:

```text
sum(order quantities at level)
=
level aggregate quantity.
```

And:

```text
every live order ID appears exactly once.
```

And:

```text
filled + cancelled + live quantity
=
original order quantity
```

where applicable.

---

# 118. Deterministic replay test

Record commands:

```text
1 NEW...
2 NEW...
3 CANCEL...
4 NEW...
...
```

Run engine.

Hash final state:

```text
H1.
```

Restart.

Replay identical commands.

Hash:

```text
H2.
```

Require:

$$
H1=H2.
$$

Run it 1,000 times.

Still require equality.

This is one of the most valuable tests you can build.

---

# 119. Performance test

Then benchmark:

```text
NEW order latency
CANCEL latency
crossing order latency
partial-fill latency
multi-level sweep latency
```

Record:

```text
p50
p90
p99
p99.9
max.
```

Not just:

```text
orders/sec.
```

Remember your actual objective:

# predictable event latency.

---

# 120. Bursty matching-engine benchmark

Don't send perfectly uniform:

```text
one order every 1 µs.
```

Real markets burst.

Test:

```text
100,000 commands
within a short burst.
```

Measure:

```text
queue build-up
tail latency
engine recovery after burst.
```

This connects directly to the microburst discussion from the previous chapter.

---

# 121. Strategy metrics you should later collect

Once your own HFT strategy sits above this engine simulator, measure:

```text
fill ratio
fill probability by queue position
cancel-to-fill ratio
average queue ahead
time at best price
maker versus taker share
gross spread capture
fees/rebates
markout by horizon
inventory
hedge cost
realized P&L
adverse-selection loss
```

These metrics will tell you whether your low-latency engineering has actual trading value.

---

# 122. Latency metrics by business outcome

This is even more valuable:

```text
orders sent within 1 µs
    ↓
what fill rate?

orders sent within 2 µs
    ↓
what fill rate?

orders sent within 5 µs
    ↓
what markout?
```

Then you can quantify:

$$
EconomicValueOfLatency.
$$

Without this, reducing:

```text
500 ns → 300 ns
```

is just a technical achievement.

With this analysis you can tell whether it affects P&L.

---

# 123. What should be on the hot path

For a simple market-making engine:

```text
market event
   ↓
book update
   ↓
derived state
   ↓
signal/fair value
   ↓
queue assessment
   ↓
inventory adjustment
   ↓
desired quote
   ↓
compare desired/live order
   ↓
risk
   ↓
encode
   ↓
TX
```

Hot path should not contain:

```text
database
JSON
logging formatting
network RPC
garbage collector
dynamic configuration lookup
filesystem
UI update.
```

---

# 124. What belongs off-path

Separately:

```text
statistics
historical storage
P&L reports
dashboards
parameter management
research features
model training
full risk aggregation
compliance logs.
```

These consume events asynchronously.

Never make:

```text
strategy
```

wait for them.

---

# 125. Your complete knowledge map so far

After the four chapters, your architecture knowledge now looks like:

```text
PHYSICAL
fiber
cross-connect
colocation
PHY

        ↓

NETWORK
Ethernet
IP
UDP
multicast
A/B feeds

        ↓

NIC
RX/TX rings
DMA
PCIe
kernel bypass

        ↓

MARKET DATA
MoldUDP64
ITCH
SBE
sequence recovery
feed arbitration

        ↓

LOCAL MARKET STATE
order books
MBO/MBP
working set
cache design

        ↓

MICROSTRUCTURE
price priority
time priority
pro-rata
queue position
hidden liquidity
pegs
auctions
fees
adverse selection

        ↓

STRATEGY
fair value
imbalance
microprice
inventory
queue model

        ↓

EXECUTION
new
cancel
replace
IOC
FOK
post-only

        ↓

RISK
position
price checks
size checks

        ↓

ORDER GATEWAY
OUCH
iLink
FIXP

        ↓

EXCHANGE
gateway
sequencer
matching engine
partitions

        ↓

MATCHING
FIFO
pro-rata
hybrid
auction

        ↓

FEEDBACK
execution ack
market data
position change
```

This is now approaching an actual HFT platform mental model rather than just low-latency programming.

---

# 126. The most important lesson of this chapter

The first Gross talk taught:

> **Design performance into the architecture.**

The second taught:

> **Understand the CPU and remove unnecessary work.**

The networking chapter taught:

> **Information has to travel through a physical packet-processing pipeline.**

This microstructure chapter adds:

> **Latency only has value because exchange rules turn arrival time, price, size and order type into execution priority.**

You should never optimize latency in isolation from:

```text
matching algorithm
queue structure
fees
fill probability
adverse selection
inventory risk.
```

For a FIFO market:

```text
100 ns faster
```

might dramatically improve queue position.

For a pro-rata market:

```text
showing the correct size
```

may matter just as much.

For a toxic market:

```text
getting filled more often
```

may actually make you poorer.

That relationship between **systems engineering and market economics** is what distinguishes an HFT system from an ordinary fast server.

---

# Where we should go next

For your goal—**actually designing an HFT-like production-grade system**—the next chapter should now move from theory to **implementation architecture**:

# Chapter 5 — Building the Market-Data Feed Handler + Production Order Book in C++

That should go all the way from:

```text
RX descriptor
↓
raw Ethernet/UDP packet
↓
MoldUDP/sequence parser
↓
A/B arbitration
↓
ITCH decoder
↓
order-ID store
↓
price levels
↓
best bid/ask
↓
derived features
↓
zero-allocation event API
```

including concrete **C++ structures, memory layout, object pools, flat arrays, ring buffers, endianness handling, compiler/cache considerations, failure recovery, testing and benchmarking**.

Then Chapter 6 should build the **strategy + queue-position/fill model**, and Chapter 7 the **order gateway + pre-trade risk + session state machine**.

Once those three are added to what you've learned so far, we'll be able to draw and then implement a credible end-to-end **low-latency trading platform prototype** rather than just discuss HFT architecture.

[1]: https://www.nasdaqtrader.com/trader.aspx?id=tradingusequities&utm_source=chatgpt.com "The NASDAQ Stock Market"
[2]: https://www.cmegroup.com/content/dam/cmegroup/globex/files/GlobexRefGd.pdf?utm_source=chatgpt.com "CME Globex Reference Guide"
[3]: https://www.nasdaqtrader.com/content/technicalsupport/specifications/TradingProducts/OUCH5.0.pdf?utm_source=chatgpt.com "North American Trading Services"
[4]: https://www.cmegroup.com/trading/market-tech-and-data-services/files/ebs-dealing-rules-general-terms-effective-2025-06-09.pdf?utm_source=chatgpt.com "2.4.6. Order types for EBS Market on Globex include the following:"
[5]: https://nasdaqtrader.com/Trader.aspx?id=PriceListTrading2&utm_source=chatgpt.com "Price List - Trading"
[6]: https://www.nasdaqtrader.com/trader.aspx?id=psx_pricing&utm_source=chatgpt.com "Nasdaq PSX Fee Schedule"
[7]: https://www.cmegroup.com/tools-information/webhelp/ebs-workstation-quick-guide/Content/IcebergOrders.html?utm_source=chatgpt.com "Iceberg Orders"
[8]: https://www.nasdaqtrader.com/content/ProductsServices/Trading/MidpointFactSheet.pdf?utm_source=chatgpt.com "Midpoint Peg Functionality"
[9]: https://www.nasdaqtrader.com/content/productsservices/trading/MPPO_factsheet.pdf?utm_source=chatgpt.com "GLOBAL TRADING AND MARKET SERVICES"
[10]: https://www.cmegroup.com/solutions/market-access/globex/trade-on-globex/faq-self-match.html?utm_source=chatgpt.com "FAQ: Globex Self-Match Prevention Functionality - CME Group"
[11]: https://www.nasdaqtrader.com/TraderNews.aspx?id=ETA2026-45&utm_source=chatgpt.com "Equity Trader Alert #2026 - 45 Launch of Order-by-Order Self Match Prevention (SMP) Functionality"
[12]: https://nasdaqtrader.com/Trader.aspx?id=OpenClose&utm_source=chatgpt.com "The NASDAQ Opening and Closing Crosses"
[13]: https://www.nasdaqtrader.com/content/ProductsServices/Trading/Crosses/openclose_faqs.pdf?utm_source=chatgpt.com "The Nasdaq Opening and Closing Crosses"
[14]: https://www.cmegroup.com/education/matching-algorithm-overview?utm_source=chatgpt.com "Matching Algorithm Overview"
[15]: https://www.eurex.com/ex-en/trade/order-book-trading/matching-principles?utm_source=chatgpt.com "Matching principles"
[16]: https://www.cmegroup.com/tools-information/lookups/advisories/electronic-trading/20080522.html?utm_source=chatgpt.com "20080522 - CME Group"
