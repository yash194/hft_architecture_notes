# Chapter 6 — Building the HFT Strategy Engine + Queue-Position / Fill-Probability Model

Chapter 5 gave us a production market-data engine capable of turning:

```text
raw packet
   ↓
ordered ITCH event
   ↓
correct local order book
   ↓
derived market state
```

into a reliable representation of the exchange.

Now we need to answer the reason the whole system exists:

> **Given the current market state, what should we do?**

A production strategy engine has to continuously solve something much richer than:

```text
if price_goes_up:
    buy()
```

The real decision is closer to:

```text
What is the instrument worth right now?
             │
             ▼
Where is price likely to move next?
             │
             ▼
How toxic is the current order flow?
             │
             ▼
If I join this queue, will I fill?
             │
             ▼
If I fill, will the fill actually be profitable?
             │
             ▼
What inventory/risk do I already have?
             │
             ▼
Should I JOIN / IMPROVE / CROSS / CANCEL?
             │
             ▼
At what PRICE?
             │
             ▼
For what SIZE?
```

That is Chapter 6.

---

# Part I — The complete strategy pipeline

The system we want to build now looks like:

```text
                     MARKET DATA ENGINE
                            │
                            ▼
                    Local Order Book
                            │
                ┌───────────┼───────────┐
                ▼           ▼           ▼
               BBO       Full Depth    Trades
                │           │           │
                └───────────┼───────────┘
                            ▼
                     FEATURE ENGINE
                            │
             ┌──────────────┼───────────────┐
             │              │               │
             ▼              ▼               ▼
        Fair Value     Queue Model      Flow/Toxicity
             │              │               │
             └──────────────┼───────────────┘
                            ▼
                  FILL PROBABILITY MODEL
                            │
                            ▼
                      EXPECTED VALUE
                            │
                     + inventory state
                            │
                     + trading costs
                            │
                     + latency state
                            │
                            ▼
                      QUOTE ENGINE
                            │
              ┌─────────────┼─────────────┐
              ▼             ▼             ▼
             JOIN        IMPROVE        CROSS
              │             │             │
              └──────┬──────┴──────┬──────┘
                     │             │
                  CANCEL        DO NOTHING
                     │             │
                     └──────┬──────┘
                            ▼
                    PRE-TRADE RISK
                            │
                            ▼
                     ORDER GATEWAY
```

The key conceptual separation is:

# Prediction and execution are different problems.

The prediction problem is:

> Where is the market likely to move?

The execution problem is:

> Given that prediction, what order should I place and where should I place it?

Many weak trading systems mix these together.

A better architecture keeps them separate.

---

# 1. Alpha versus execution policy

Suppose your model says:

```text
Expected fair value in 5 ms = 100.015
```

Current market:

```text
Bid = 100.00
Ask = 100.01
```

That does **not** automatically mean:

```text
BUY.
```

You still need to ask:

```text
Should I cross the ask at 100.01?

Should I join the bid at 100.00?

Should I improve to 100.005 if allowed?

How much queue is ahead?

What is fill probability?

What fee do I pay?

Will I be adversely selected?

How much inventory do I already own?
```

So think:

$$
\text{Trading Strategy}
=
\text{Price Prediction}
+
\text{Execution Policy}.
$$

This distinction becomes critical in HFT.

---

# Part II — What is “fair value”?

# 2. Mid-price: the simplest reference

Suppose:

```text
Best Bid = 100.00
Best Ask = 100.02
```

Mid:

$$
M =
\frac{100.00+100.02}{2}
=
100.01.
$$

This gives us the simplest reference price:

```text
100.01
```

But midpoint assumes the bid and ask sides contain equal information.

Often they don't.

---

# 3. Queue imbalance

Suppose:

```text
Best Bid:

100.00 × 1000

Best Ask:

100.01 × 100
```

There is far more displayed demand than supply.

Define top-of-book imbalance:

$$
I =
\frac{Q_b-Q_a}
{Q_b+Q_a}.
$$

Here:

$$
I=
\frac{1000-100}{1100}
=
0.818.
$$

Range:

$$
-1\le I\le1.
$$

Interpretation:

```text
I ≈ +1 → strongly bid-heavy

I ≈  0 → balanced

I ≈ -1 → strongly ask-heavy
```

Empirical research has found queue imbalance useful for predicting the direction of the next mid-price move, particularly for large-tick stocks. ([arXiv][1])

---

# 4. Why queue imbalance can contain information

Imagine:

```text
Bid queue = 10,000
Ask queue = 100
```

For price to move downward:

```text
10,000 bid units
```

need to be executed or cancelled.

For price to move upward:

```text
only 100 ask units
```

need to disappear.

This does **not** mean price must rise.

Those 10,000 bid units can cancel instantly.

But it gives the system information about the current state of the queues.

Cont and de Larrard model bid/ask queues as a stochastic queuing system and derive quantities such as the probability of the next price move conditional on queue state. ([arXiv][2])

---

# 5. Microprice

A useful refinement of midpoint is:

$$
P_{micro}
=
\frac{
P_aQ_b+P_bQ_a
}{
Q_b+Q_a
}.
$$

Notice the cross-weighting.

Suppose:

```text
Bid = 100.00, Qb = 1000
Ask = 100.01, Qa = 100
```

Then:

$$
P_{micro}
=
\frac{100.01(1000)+100.00(100)}
{1100}.
$$

Approximately:

$$
P_{micro}=100.00909.
$$

While midpoint is:

$$
100.005.
$$

So:

```text
Mid        = 100.00500
Microprice = 100.00909
```

The bid-heavy book pushes microprice toward the ask.

This is saying:

> Based purely on current best-queue pressure, the market may be more likely to move upward than downward.

---

# 6. But microprice is not true fair value

Do not build:

```cpp
fair_value = microprice;
```

and assume you've solved HFT.

Microprice ignores many things:

```text
deeper book

recent trades

recent cancellations

correlated instruments

futures

ETFs

options

news

volatility

latency

hidden liquidity

market regime
```

Microprice is a:

# feature.

Not necessarily the complete model.

---

# 7. A better conceptual fair-value model

Think:

$$
FV_t =
ReferencePrice_t
+
\alpha_t.
$$

For example:

$$
FV_t =
Microprice_t
+
\alpha_{flow}
+
\alpha_{depth}
+
\alpha_{cross}
+
\alpha_{trade}
+\cdots
$$

where:

```text
α_flow   = order-flow signal
α_depth  = deeper-book signal
α_cross  = related-instrument signal
α_trade  = recent aggressive-trade signal.
```

Your fair-value model predicts something like:

$$
E[M_{t+h}\mid X_t].
$$

The horizon \(h\) matters enormously.

---

# 8. Prediction horizon matters

You shouldn't simply predict:

```text
future price.
```

Predict:

```text
future price after 10 µs?

100 µs?

1 ms?

10 ms?

1 second?
```

A signal useful at:

```text
100 µs
```

may be useless at:

```text
100 ms.
```

If your system's total round trip is:

```text
2 µs
```

then predicting what happens:

```text
100 ns later
```

may be impossible to monetize because your order can't arrive in time.

So model horizon and system latency must be designed together.

---

# Part III — Order Flow

# 9. Static state versus flow

A snapshot tells you:

```text
Bid quantity = 1000
Ask quantity = 500
```

But it doesn't tell you **how you got there**.

Consider two markets with the same current state.

### Market A

```text
Bid went:

100 → 300 → 600 → 1000
```

while ask stayed near:

```text
500.
```

### Market B

Bid was:

```text
5000
```

and rapidly cancelled to:

```text
1000.
```

Same final snapshot:

```text
Bid = 1000
Ask = 500.
```

Completely different recent dynamics.

Therefore a strategy should consume:

```text
STATE
+
EVENT FLOW.
```

---

# 10. Order-flow events

From Chapter 5 we have events such as:

```text
Add bid
Add ask

Cancel bid
Cancel ask

Execute bid
Execute ask

Trade buy
Trade sell

Price level created
Price level depleted.
```

Each contains information.

For example:

```text
ask cancellations
```

remove sell-side liquidity and may be bullish.

While:

```text
aggressive selling
```

consumes bid liquidity and may be bearish.

---

# 11. Order Flow Imbalance — OFI

A classic result by Cont, Kukanov and Stoikov is that short-horizon price changes are strongly related to **order-flow imbalance**, incorporating limit-order additions, cancellations and market-order consumption around the best quotes. They found a roughly linear relationship between OFI and short-horizon price changes, with the impact coefficient inversely related to market depth. ([arXiv][3])

A commonly used top-of-book event contribution is:

$$
e_n =
\mathbf 1_{P^b_n\ge P^b_{n-1}}q^b_n
-
\mathbf 1_{P^b_n\le P^b_{n-1}}q^b_{n-1}
$$

$$
-
\mathbf 1_{P^a_n\le P^a_{n-1}}q^a_n
+
\mathbf 1_{P^a_n\ge P^a_{n-1}}q^a_{n-1}.
$$

Then over some event/time window:

$$
OFI =
\sum_n e_n.
$$

The formula looks complicated, but the intuition is straightforward.

---

# 12. OFI intuition

Think:

### Bullish contributions

```text
bid liquidity added
ask liquidity removed
aggressive buying consumes ask
```

### Bearish contributions

```text
ask liquidity added
bid liquidity removed
aggressive selling consumes bid.
```

So approximately:

$$
OFI =
DemandIncrease-SupplyIncrease.
$$

Large positive:

```text
buy-side pressure.
```

Large negative:

```text
sell-side pressure.
```

---

# 13. Why OFI can be better than trade imbalance alone

Suppose:

```text
no trades occur.
```

But suddenly:

```text
80% of best ask cancels
```

and:

```text
large new bid liquidity appears.
```

The market has changed materially.

Trade-only analysis sees:

```text
nothing.
```

OFI sees:

```text
significant positive pressure.
```

That's why order-book events matter.

---

# 14. Multi-level imbalance

Top-level imbalance:

$$
I_1=
\frac{Q^b_1-Q^a_1}
{Q^b_1+Q^a_1}.
$$

But you can extend:

$$
I_k=
\frac{
\sum_{i=1}^k w_iQ^b_i
-
\sum_{i=1}^k w_iQ^a_i
}{
\sum_{i=1}^k w_iQ^b_i
+
\sum_{i=1}^k w_iQ^a_i
}
$$

with:

```text
w1 > w2 > w3...
```

because nearer levels generally matter more.

For example:

```text
w1 = 1
w2 = 0.5
w3 = 0.25.
```

Again, coefficients should come from data, not intuition alone.

---

# 15. Depth slope

Consider ask side:

### Book A

```text
100.01 × 100
100.02 × 100
100.03 × 100
```

### Book B

```text
100.01 × 100
100.02 × 5000
100.03 × 10000
```

Same best ask quantity.

But Book B contains a wall of liquidity behind it.

So useful depth features include:

```text
quantity by level
cumulative depth
depth gradient
distance-weighted depth
```

rather than only:

```text
best quantity.
```

---

# 16. Trade imbalance

Over a short horizon:

$$
TI=
\frac{
V_{buy}-V_{sell}
}{
V_{buy}+V_{sell}
}.
$$

Where:

```text
Vbuy  = aggressive buy volume
Vsell = aggressive sell volume.
```

Strong positive:

```text
buyers repeatedly crossing ask.
```

Potentially bullish.

But interpretation depends on whether passive sell liquidity replenishes.

---

# 17. Replenishment is information

Imagine ask:

```text
100.01 × 500.
```

Aggressive buyers consume:

```text
500.
```

But immediately another:

```text
500
```

appears at the same price.

Then another 500 gets consumed.

Then another appears.

This can indicate:

```text
strong hidden/replenishing seller.
```

So strategy shouldn't merely see:

```text
buy trades = bullish.
```

It should potentially see:

```text
aggressive buys
+
ask refuses to move
=
sell-side resilience.
```

Microstructure signals are contextual.

---

# 18. Event rates

Over a short rolling window, estimate:

$$
\lambda_{add,bid}
$$

$$
\lambda_{cancel,bid}
$$

$$
\lambda_{trade,bid}
$$

and equivalent ask-side rates.

For example:

```text
Bid:
add rate    = 20k events/s
cancel rate = 10k events/s

Ask:
add rate    = 5k
cancel rate = 30k
```

That tells a very different story from one static snapshot.

---

# 19. Event time versus clock time

A feature window could mean:

```text
last 1 ms
```

or:

```text
last 100 events.
```

These behave differently.

During quiet markets:

```text
1 ms → 2 events.
```

During a burst:

```text
1 ms → 5000 events.
```

Event-time windows can therefore normalize some aspects of activity.

Production systems often maintain both:

```text
clock-time features
event-time features.
```

---

# 20. Exponentially weighted features

Instead of storing the last 10,000 events, maintain:

$$
x_t =
\lambda x_{t-1}
+
(1-\lambda)e_t.
$$

For example:

```text
EWMA buy pressure
EWMA cancel pressure
EWMA volatility
EWMA trade direction.
```

Advantages:

```text
O(1) update
constant memory
no dynamic window structure.
```

Excellent for a hot path.

---

# Part IV — Queue Position

This is where the strategy becomes fundamentally different from ordinary trading.

![Image](https://www.researchgate.net/publication/277023747/figure/fig1/AS%3A359491863367680%401462720947929/Orders-happened-in-the-best-bid-queue.png)

![Image](https://images.openai.com/static-rsc-4/o-j9kxv18Wkhft0hkVIETOxZcDIrOIqDXbQXhlAQ8vlT-9nSpSw0TT3BWOHArwQueS_om_GR6Wa0sm3fUc7j5cHJKWN99tqslkFNildH-faMONsPOdhNvDc2X1P5kECZmUrnLjk5IpRfjESOpd-fapoXkY29-l-N4_-ibSphMYpzX1P9cNV4MxB8nV3EU-k_?purpose=fullsize)

# 21. Why price is only half the passive-order problem

Suppose best bid:

```text
100.00 × 10,000.
```

You send:

```text
BUY 100 @ 100.00.
```

Your price is correct.

But you may be:

```text
position 10,001 in queue.
```

If only:

```text
1000 shares
```

sell into that price before market rises:

```text
you get no fill.
```

Therefore the true state of a passive quote is:

```text
price
+
queue position.
```

Research specifically studying order position treats it as a key variable in limit-order execution and models how it evolves under market orders, cancellations and new limit orders. ([arXiv][4])

---

# 22. Queue ahead

Define:

$$
Q_{ahead}
$$

as displayed quantity with priority ahead of our order.

If:

```text
Existing orders:

A 200
B 500
C 300

OUR ORDER 100
```

then:

$$
Q_{ahead}=1000.
$$

And:

```text
Qbehind
```

is quantity behind us.

---

# 23. What changes queue ahead?

Suppose:

```text
Qahead = 1000.
```

### Market sell executes 200

If execution occurs from head:

$$
Q_{ahead}\to800.
$$

### Order ahead cancels 100

$$
Q_{ahead}\to700.
$$

### New order joins behind us

$$
Q_{ahead}
$$

doesn't change.

### Someone price-improves ahead of the entire level

Our level may no longer be best bid.

Our queue position at our level is unchanged, but:

```text
fill probability changes dramatically.
```

---

# 24. MBO lets you model this much more precisely

If the public feed provides individual order IDs and exact order events, your reconstructed queue can be:

```text
Price 100.00

A ──► B ──► C ──► D
```

Now when:

```text
Cancel C
```

occurs, you know exactly which displayed order disappeared.

This is better than MBP, where you may only know:

```text
quantity dropped by 100.
```

and not whether the cancellation came from ahead of you or behind you.

---

# 25. But your own exact queue position may still be inferred

This distinction is important.

In your own exchange simulator:

```text
you know exact priority.
```

On a real venue:

```text
your private order ID/session
```

and:

```text
public anonymous market-data order representation
```

may not be directly linkable.

Therefore a production strategy often maintains:

# an estimated queue position.

It may initialize queue ahead using:

```text
visible quantity present
around the time the exchange accepted our order.
```

Then evolve the estimate using public events.

So distinguish:

```text
simulator exact queue position

versus

production estimated queue position.
```

---

# 26. Queue-position uncertainty

Representing only:

```cpp
uint32_t queue_ahead;
```

may be too confident.

A better model can maintain:

```text
lower bound
expected value
upper bound
```

or:

```text
distribution.
```

For example:

```text
Qahead expected = 800
Qahead p10      = 500
Qahead p90      = 1200.
```

Then quote decisions can incorporate uncertainty.

---

# 27. Hidden liquidity cannot be reconstructed perfectly

Even with excellent MBO data:

```text
hidden/reserve liquidity
```

can exist.

So:

$$
VisibleQueue
\neq
TrueExecutableLiquidity.
$$

Your queue model should be calibrated empirically against actual fill outcomes rather than assuming visible book reconstruction is complete truth.

---

# Part V — Fill Probability

# 28. Queue position is useful because it determines execution probability

Suppose:

### Order A

```text
Qahead = 100
```

### Order B

```text
Qahead = 10,000.
```

Same price.

Same market.

Clearly:

$$
P(fill_A)>P(fill_B).
$$

But fill probability also depends on:

```text
trade arrival rate
cancellation rate
imbalance
volatility
price-change probability
time horizon
order-flow regime.
```

Optimal placement literature emphasizes queue length/position, market-order frequency, cancellations and price dynamics as determinants of limit-order execution. ([PubsOnLine][5])

---

# 29. Define the question properly

Don't ask:

```text
Will this order fill?
```

Ask:

$$
P(
Fill\ before\ time\ h
\mid
X_t
).
$$

For example:

```text
P(fill within 100 µs)
P(fill within 1 ms)
P(fill within 10 ms).
```

That's directly useful to the strategy.

---

# 30. Simplest empirical fill model

Feature vector:

$$
X=
[
Q_{ahead},
Q_b,
Q_a,
I,
OFI,
Spread,
Volatility,
TradeRate,
CancelRate,
\dots
].
$$

Target:

```text
1 = order filled within 5 ms
0 = did not fill within 5 ms.
```

Train logistic regression:

$$
P(fill\mid X)
=
\sigma(
\beta_0+\beta^TX
).
$$

Where:

$$
\sigma(z)=\frac{1}{1+e^{-z}}.
$$

This is a surprisingly useful first production model because it is:

```text
fast
interpretable
easy to calibrate
easy to deploy.
```

---

# 31. Hazard model

A more natural execution formulation is a hazard rate:

$$
\lambda_{fill}(t\mid X).
$$

If we simplify it as constant over a short horizon:

$$
P(fill\le h)
=
1-e^{-\lambda_{fill}h}.
$$

Higher:

```text
market-order intensity
```

may increase \(\lambda\).

More:

```text
queue ahead
```

typically decreases effective fill probability.

A survival model is often more informative than a simple binary classifier because you care about:

# when the order fills.

---

# 32. Queue-depletion model

Suppose:

$$
Q_0=1000
$$

shares are ahead.

Incoming market sells and cancellations remove volume from ahead.

Think of:

$$
Q_{ahead}(t)
$$

as a stochastic process.

Your order fills when:

$$
Q_{ahead}(t)\le0
$$

and enough subsequent aggressive flow reaches your own quantity.

So execution becomes a:

# first-passage problem.

This is why limit-order queue modelling naturally connects to stochastic queueing theory. Guo, Ruan and Zhu study exactly the dynamics of order positions and related queues, while queue-reactive models represent order-book event intensities as functions of the current book state. ([arXiv][4])

---

# 33. Queue-reactive model

The Huang–Lehalle–Rosenbaum queue-reactive framework treats the limit order book as a Markov queuing system during periods in which a reference price remains fixed.

The important intuition is:

> **Order arrivals and cancellations depend on the current state of the queues.**

So instead of:

$$
\lambda_{cancel}=constant
$$

you model:

$$
\lambda_{cancel}
=
f(Q_b,Q_a,\text{level},\dots).
$$

Likewise:

$$
\lambda_{market}
=
g(Q_b,Q_a,\dots).
$$

This makes simulated order flow react to book state rather than being independent random noise. ([arXiv][6])

---

# 34. Why this matters for your simulator

Bad simulator:

```text
market orders ~ Poisson(constant λ)
cancels       ~ Poisson(constant μ)
```

regardless of market state.

Real markets behave more like:

```text
queue becomes very imbalanced
        ↓
order/cancel behavior changes

spread widens
        ↓
limit order arrival behavior changes

best queue becomes tiny
        ↓
price-change probability changes.
```

A queue-reactive simulator captures part of this dependence.

---

# Part VI — Fill quality is more important than fill quantity

# 35. High fill probability is not automatically good

Suppose quote A:

```text
fill probability = 90%
```

but after fills:

```text
market moves against us by 2 ticks.
```

Quote B:

```text
fill probability = 30%
```

but market moves in our favor by:

```text
0.5 ticks.
```

Which is better?

Potentially B.

Because:

$$
P(fill)
$$

is only half the equation.

You need:

$$
E[PnL\mid fill].
$$

---

# 36. Adverse selection

Suppose we quote:

```text
Bid 100.00.
```

Someone sells to us.

Immediately:

```text
Bid 99.98
Ask 99.99.
```

Our fill likely wasn't random.

It may have occurred because the seller reacted to information before we cancelled.

This is:

# adverse selection.

Order-flow imbalance has been explicitly studied as a measure relevant to adverse selection in limit-order executions. ([arXiv][3])

---

# 37. Markout

For a buy fill at:

$$
P_f
$$

define:

$$
Markout(h)
=
M_{t+h}-P_f.
$$

For a sell:

$$
Markout(h)
=
P_f-M_{t+h}.
$$

Therefore positive always means:

```text
good for us.
```

Suppose we buy:

$$
P_f=100.00.
$$

After 1 ms:

$$
M=99.98.
$$

Then:

$$
Markout_{1ms}
=
-0.02.
$$

Bad fill.

---

# 38. Use multiple markout horizons

Measure:

```text
10 µs
50 µs
100 µs
500 µs
1 ms
5 ms
10 ms
100 ms
1 s.
```

Why?

A fill can be:

```text
bad at 100 µs
good at 100 ms.
```

Maybe you were picked off momentarily but the market reverted.

Different strategies care about different horizons.

---

# 39. Conditional markout

Don't merely calculate:

$$
E[Markout].
$$

Condition on features:

$$
E[
Markout
\mid
Imbalance,
OFI,
QueuePosition,
Volatility,
\dots
].
$$

For example:

```text
When OFI strongly negative
and we get a bid fill
→ markout = terrible.
```

That tells your strategy:

```text
do not quote bid in that regime.
```

---

# Part VII — Expected Value of a Passive Quote

# 40. The core market-making equation

For a passive bid at price \(P_b\), over horizon \(h\), think:

$$
EV_{bid}
=
P(fill)
\times
E[
M_{t+h}-P_b
\mid fill
]
$$

plus economics such as rebates and minus costs/risk.

A richer approximation:

$$
EV_{bid}
=
P(fill)
\left[
E(M_{t+h}-P_b\mid fill)
+
Rebate
-
HedgeCost
\right]
-
RiskPenalty.
$$

For an ask:

$$
EV_{ask}
=
P(fill)
\left[
E(P_a-M_{t+h}\mid fill)
+
Rebate
-
HedgeCost
\right]
-
RiskPenalty.
$$

This is more meaningful than saying:

```text
spread = 1 tick
therefore market making earns 1 tick.
```

---

# 41. Why conditional future price is crucial

Suppose unconditional fair value:

```text
100.01.
```

Bid:

```text
100.00.
```

Looks like:

```text
+1 cent edge.
```

But conditional on someone selling to us:

$$
E[M_{t+1ms}\mid bid\ fill]
=
99.98.
$$

Then the true conditional trading edge is:

$$
99.98-100.00=-0.02.
$$

So the quote is terrible.

This is precisely adverse selection.

---

# 42. Expected value of taking

For an aggressive buy at current ask \(A\):

$$
EV_{take,buy}
=
E[M_{t+h}]
-
A
-
TakerFee
-
Impact.
$$

If:

```text
fair value = 100.03
ask        = 100.01
fee        = 0.003
```

then before other costs:

$$
100.03-100.01-0.003=0.017.
$$

Potentially attractive.

There is no queue uncertainty because you're crossing available liquidity, though fill/impact uncertainty can still exist for larger orders.

---

# 43. JOIN / IMPROVE / CROSS is an expected-value problem

Current:

```text
Bid = 100.00
Ask = 100.02
```

Possible actions:

### JOIN

```text
BUY @100.00
```

Advantages:

```text
better price
```

Disadvantages:

```text
queue ahead
lower fill probability.
```

### IMPROVE

```text
BUY @100.01
```

Advantages:

```text
first/new best queue
higher fill probability.
```

Disadvantage:

```text
one tick worse price.
```

### CROSS

```text
BUY @100.02
```

Advantages:

```text
immediate execution.
```

Disadvantages:

```text
pay spread
taker cost.
```

Compare:

$$
EV_{join},
EV_{improve},
EV_{cross},
EV_{nothing}.
$$

Pick the highest feasible risk-adjusted value.

That's a much better framework than fixed rules.

---

# Part VIII — Inventory

# 44. Why market makers accumulate inventory

Suppose:

```text
Bid gets hit
Bid gets hit
Bid gets hit
```

but ask doesn't fill.

Position:

```text
0
+100
+200
+300
+500
...
```

You are becoming:

# long.

Even if each trade looked individually profitable, now:

```text
price falls
```

and your inventory loses value.

Inventory therefore changes your willingness to quote.

---

# 45. Reservation price

A useful concept is:

# reservation price.

This is not necessarily your prediction of objective market value.

It is the price at which **you**, given your inventory and risk preferences, are indifferent to buying/selling more.

If neutral:

```text
fair value = 100.00

reservation ≈ 100.00.
```

If heavily long:

```text
reservation < fair value.
```

You want to encourage selling inventory and discourage buying more.

---

# 46. Avellaneda–Stoikov model

The classic Avellaneda–Stoikov market-making model formalizes this trade-off between spread capture and inventory risk.

Under its assumptions, a finite-horizon reservation price takes the form:

$$
r_t
=
s_t
-
q_t\gamma\sigma^2(T-t),
$$

where:

```text
s      = reference/mid price
q      = inventory
γ      = inventory risk aversion
σ      = volatility
T-t    = remaining horizon.
```

If:

$$
q>0
$$

(long inventory), then:

$$
r<s.
$$

If:

$$
q<0
$$

(short inventory), then:

$$
r>s.
$$

The original model derives quotes by combining this inventory-adjusted valuation with a model of order-arrival intensity as a function of quote distance. ([math.nyu.edu][7])

---

# 47. Optimal spread in the simple A–S model

With a common exponential arrival-intensity assumption:

$$
\lambda(\delta)
=
Ae^{-k\delta},
$$

a form of the optimal total spread is:

$$
\delta_a+\delta_b
=
\gamma\sigma^2(T-t)
+
\frac{2}{\gamma}
\ln
\left(
1+\frac{\gamma}{k}
\right).
$$

Then approximately:

$$
BidQuote
=
r-\frac{Spread}{2},
$$

$$
AskQuote
=
r+\frac{Spread}{2}.
$$

Important:

> Do not treat these equations as a production HFT formula.

Their value is primarily conceptual.

The model makes simplifying assumptions about price dynamics, fills and order arrivals that real electronic markets violate. The original paper explicitly models stochastic price diffusion and Poisson-style market-order arrival intensity. ([math.nyu.edu][7])

---

# 48. What to take from Avellaneda–Stoikov

Not:

```text
copy formula
become Optiver.
```

Take:

```text
fair value
   ↓
inventory-adjusted reservation price
   ↓
quote distances
   ↓
fill intensity versus edge trade-off.
```

That framework remains extremely useful.

Production systems simply use richer models for every component.

---

# 49. Practical inventory skew

Suppose:

```text
raw fair value = 100.00
```

and inventory penalty:

$$
\lambda q=0.02.
$$

If long:

```text
reservation = 99.98.
```

Instead of:

```text
99.99 / 100.01
```

you may quote:

```text
99.97 / 99.99.
```

Now:

```text
buy side less attractive
sell side more attractive.
```

This tends to reduce long inventory.

---

# 50. Inventory penalty need not be linear

You could use:

$$
Penalty(q)=\lambda q
$$

or stronger nonlinear risk:

$$
Penalty(q)
=
\lambda q
+
\eta q^3.
$$

Why nonlinear?

Small inventory:

```text
normal.
```

Near risk limit:

```text
get very aggressive about reducing it.
```

So reservation price may shift sharply as inventory approaches limits.

---

# 51. Inventory limits should remain hard limits

A strategy model may say:

```text
expected quote value positive.
```

But if:

```text
position = maximum long limit
```

risk layer says:

```text
NO MORE BUYING.
```

Do not let probabilistic strategy decisions override hard risk constraints.

We will build that formally in Chapter 7.

---

# Part IX — Combining prediction and inventory

# 52. Production reservation price

A practical conceptual structure:

$$
R_t =
M_t
+
\alpha_t
-
InventoryPenalty_t.
$$

Where:

```text
M_t     = market reference
α_t     = predicted short-term move
Penalty = inventory/risk adjustment.
```

Example:

```text
Microprice       = 100.005
Flow alpha       = +0.007
Cross alpha      = +0.003
Inventory penalty=  0.005
```

Then:

$$
R
=
100.005
+0.007
+0.003
-0.005
=
100.010.
$$

This is the center around which you evaluate quotes.

---

# 53. Do not double count correlated signals

Suppose:

```text
microprice
```

already encodes top-level imbalance.

Then your model also adds:

```text
large coefficient × same imbalance.
```

You may count the same signal twice.

Features should be:

```text
trained/validated jointly
```

rather than manually summing every attractive indicator.

---

# Part X — Latency-aware fair value

# 54. Your decision is already old when the order reaches the exchange

This is one of the most important production concepts.

At:

```text
T0
```

you receive market event.

At:

```text
T1
```

strategy calculates.

At:

```text
T2
```

order leaves NIC.

At:

```text
T3
```

exchange processes it.

You should ideally estimate:

$$
FV_{T3}
$$

not merely:

$$
FV_{T0}.
$$

Your market information ages while you compute and transmit.

---

# 55. Latency-adjusted alpha

Suppose signal decays:

$$
\alpha(\tau)
=
\alpha_0e^{-\tau/\theta}.
$$

If:

```text
α0 = 1 tick
half-life very short
```

and your system takes:

```text
5 µs
```

most alpha may be gone before arrival.

Therefore:

```text
predictive power
```

and:

```text
system latency
```

cannot be evaluated independently.

---

# 56. The value of a faster system

Imagine signal alpha at observation:

```text
+1.0 tick.
```

After 1 µs:

```text
+0.8 tick.
```

After 5 µs:

```text
+0.2 tick.
```

System A:

```text
arrival = 1 µs.
```

System B:

```text
arrival = 5 µs.
```

Same model.

Completely different expected P&L.

This is how nanoseconds acquire economic value.

---

# Part XI — Toxicity and cancellation

# 57. Quote protection

Suppose we have a live bid:

```text
100.00.
```

New event makes reservation value:

```text
99.97.
```

Our bid is now:

```text
3 ticks too high.
```

We don't ask:

```text
Would I submit this quote now?
```

Only.

We ask:

> **Is the existing live quote dangerous enough that it must be cancelled immediately?**

This is quote protection.

---

# 58. Cancellation score

You can define something conceptually like:

$$
Danger_{bid}
=
P_{bid}
-
FV_{arrival}
+
ToxicityPenalty.
$$

If:

```text
Danger > threshold
```

cancel.

Likewise ask:

$$
Danger_{ask}
=
FV_{arrival}
-
P_{ask}
+
ToxicityPenalty.
$$

---

# 59. Cancellation often gets higher priority than new quoting

Suppose one event implies:

```text
Cancel stale bid
```

and:

```text
submit new ask.
```

If your system has limited messaging/TX resources, stale-risk cancellation may be more urgent.

Production systems can prioritize:

```text
CANCEL
```

ahead of:

```text
new opportunistic order.
```

Because stale live exposure is already at the exchange.

---

# 60. Don't cancel on every microscopic signal change

Suppose reservation price jitters:

```text
100.001
100.002
100.0015
100.0022
...
```

If you cancel/replace every event:

```text
massive message rate
lose queue priority
throttle yourself
```

and often gain nothing.

Use:

# hysteresis.

---

# 61. Requote threshold

Current quote:

```text
100.00.
```

Desired theoretical:

```text
100.001.
```

No action.

Then:

```text
desired = 100.011.
```

Now one tick difference may justify replacement.

Conceptually:

```cpp
if (abs(desired_ticks - live_ticks)
      >= requote_threshold)
{
    replace();
}
```

But threshold can depend on:

```text
queue value
volatility
inventory
toxicity
message budget.
```

---

# 62. Queue value makes cancellation expensive

Suppose you are:

```text
first in a valuable bid queue.
```

New desired theoretical price moves down slightly.

Cancelling means losing:

```text
excellent queue position.
```

So compare:

$$
ValueOfKeepingQueue
$$

against:

$$
RiskOfStaleQuote.
$$

That is a much richer decision than:

```text
desired != current
→ replace.
```

---

# 63. Queue preservation

Suppose:

```text
current bid = 100.00
Qahead = 0
```

and fair value falls:

```text
0.1 tick.
```

Maybe keep quote.

But same price with:

```text
Qahead = 20,000
```

has little queue value.

You may cancel much more readily.

Thus:

```text
cancel threshold
```

should depend on queue position.

---

# Part XII — Fill Probability + Markout = Order Score

# 64. Build an order score

For each candidate quote \(a\):

$$
Score(a)
=
P(fill_a)
\times
ExpectedPnL(a\mid fill)
-
Risk(a)
-
OpportunityCost(a).
$$

Candidate actions:

```text
join bid
improve bid
cross ask
join ask
improve ask
cross bid
cancel
nothing.
```

Then choose feasible action with highest score.

---

# 65. Example

Current:

```text
Bid  = 100.00 × 5000
Ask  = 100.01 × 300
```

Your estimated future fair value:

```text
100.008.
```

Candidate bid join:

```text
price = 100.00
Qahead = 5000
Pfill = 8%
conditional markout = +0.004
```

Candidate ask cross:

```text
buy @100.01
immediate
expected future mid =100.008
```

Cross EV:

```text
-0.002 before costs.
```

So don't cross.

Joining may have:

$$
0.08\times0.004
=
0.00032
$$

expected gross value per share before other costs.

Maybe positive.

---

# 66. Another example: toxic bid

Current:

```text
Bid = 100.00
Ask = 100.01.
```

Raw fair value:

```text
100.005.
```

Looks safe.

But:

```text
OFI strongly negative
aggressive sells accelerating
bid cancellations surging.
```

Conditional model estimates:

$$
E[M_{1ms}\mid bid\ fill]
=
99.985.
$$

Then expected fill quality:

$$
99.985-100.00=-0.015.
$$

Cancel the bid.

This is why:

```text
static spread
```

does not determine quote profitability.

---

# Part XIII — Quote size

# 67. Price is not the only control variable

You also choose:

$$
Quantity.
$$

Quote size can depend on:

```text
expected edge
inventory
volatility
queue
market depth
fill probability
risk limits.
```

---

# 68. Simple size model

Conceptually:

$$
Size
=
BaseSize
\times
SignalFactor
\times
InventoryFactor
\times
LiquidityFactor.
$$

For example:

```text
Base      = 100
Signal    = 1.5
Inventory = 0.5
Liquidity = 1.0
```

Then:

```text
75 shares.
```

In production use integer lot constraints and venue rules.

---

# 69. Size and queue economics interact

In strict FIFO:

```text
increasing size
```

does not move you earlier.

It just leaves more quantity at your position.

In pro-rata markets:

```text
size
```

may directly affect allocation.

So Chapter 4's matching algorithm must feed into Chapter 6's sizing model.

---

# Part XIV — Feature Engine Architecture

# 70. Never recompute every feature from the whole book

Bad:

```text
event arrives
 ↓
scan 1000 bid levels
scan 1000 ask levels
calculate everything.
```

Instead:

```text
event says ask L1 quantity changed
```

update only:

```text
top imbalance
microprice
related EWMA
queue state.
```

Incremental state is essential.

---

# 71. Feature state

Conceptually:

```cpp
struct FeatureState {
    int64_t microprice2;

    int32_t imbalance_q15;

    int64_t ofi_fast;
    int64_t ofi_slow;

    int64_t trade_imbalance_fast;

    uint32_t volatility_fast;

    uint32_t bid_add_rate;
    uint32_t ask_add_rate;

    uint32_t bid_cancel_rate;
    uint32_t ask_cancel_rate;
};
```

Notice:

```text
no strings
no vectors
no allocations.
```

---

# 72. Fixed-point features

You don't necessarily need:

```cpp
double imbalance;
```

For:

$$
I\in[-1,1]
$$

you could store fixed point:

```text
-32768 ... +32767.
```

Then:

```text
I = +16384
```

represents roughly:

```text
+0.5.
```

Whether this helps depends on CPU/model architecture.

Modern CPUs handle floating point extremely well.

Don't remove floating point merely because:

```text
integers feel more HFT.
```

Measure.

---

# 73. The strategy model should be tiny on the hot path

Offline training can be huge.

Online inference should ideally be:

```text
small.
```

Examples:

### Linear

$$
\alpha=\beta_0+\sum_i\beta_ix_i.
$$

Excellent latency.

---

### Logistic

$$
P(up)=\sigma(\beta^Tx).
$$

Still very small.

---

### Lookup table

```text
imbalance bucket
×
spread bucket
×
volatility bucket
→ probability.
```

Extremely fast.

---

### Small boosted-tree model

Possible.

But branches can hurt if highly irregular.

---

### Tiny neural network

Potentially feasible:

```text
32 inputs
16 hidden units
1 output.
```

But only if the predictive improvement justifies its latency/WSS.

---

# 74. Don't put a Python model in the hot path

Training:

```text
Python
PyTorch
XGBoost
pandas
```

is perfectly fine.

Deployment:

```text
C++ coefficients
generated header
ONNX/custom inference
precomputed tables
```

depending on model.

The production strategy should not make:

```text
Python RPC
```

for every market tick.

---

# 75. Offline versus online

The system architecture is:

```text
                    OFFLINE

Raw packets
    ↓
replay / feature extraction
    ↓
dataset
    ↓
Python research
    ↓
model training
    ↓
validation
    ↓
export parameters

                    ONLINE

Market event
    ↓
same feature definitions
    ↓
C++ inference
    ↓
quote decision.
```

The hardest issue here is:

# feature parity.

---

# 76. Feature parity

If training defines imbalance as:

```python
(qb - qa) / (qb + qa)
```

but C++ production accidentally uses:

```cpp
(qb - qa) / max(qb, qa)
```

you aren't deploying the model you trained.

Therefore feature definitions should ideally be shared/generated/tested.

Replay production packets through both:

```text
Python feature pipeline
C++ feature pipeline.
```

Require identical outputs within tolerances.

---

# 77. No look-ahead leakage

Suppose training feature at time \(t\):

```text
OFI in previous 1 ms
```

but implementation accidentally includes an event that occurred after \(t\).

Your backtest becomes impossibly good.

For HFT, even:

```text
1 µs
```

of look-ahead leakage can invalidate the research.

Everything must be defined according to:

```text
what information was actually available
at the decision timestamp.
```

---

# Part XV — Strategy State

# 78. Per-instrument state

Conceptually:

```cpp
struct StrategyState {
    FairValueState fair_value;
    FeatureState features;

    QuoteState bid_quote;
    QuoteState ask_quote;

    Position position;

    Timestamp last_decision;
};
```

Indexed:

```cpp
states[stock_locate];
```

just like the market-data engine.

---

# 79. Own quote state

For each side:

```cpp
struct QuoteState {
    Price desired_price;
    Quantity desired_qty;

    Price live_price;
    Quantity live_qty;

    OrderId own_order_id;

    QuoteStatus status;

    uint32_t estimated_queue_ahead;

    int32_t fill_probability_q15;

    Timestamp last_action;
};
```

Status may be:

```text
NONE
PENDING_NEW
LIVE
PENDING_CANCEL
PENDING_REPLACE.
```

The strategy must distinguish:

```text
what I want
```

from:

```text
what the exchange has confirmed.
```

---

# 80. Strategy cannot assume cancel instantly worked

If:

```text
status = PENDING_CANCEL
```

then the order can still fill.

Therefore strategy/risk position must be prepared for:

```text
execution arrives
```

before:

```text
cancel acknowledgement.
```

This becomes Chapter 7's order-state machine.

---

# Part XVI — Event-driven evaluation

# 81. Don't run strategy at arbitrary 1 ms intervals if events matter at microseconds

Normal quantitative strategy:

```text
every 100 ms:
    calculate().
```

HFT strategy:

```text
market event
    ↓
update relevant feature
    ↓
decide whether event materially changes action.
```

This is:

# event-driven trading.

---

# 82. But don't evaluate full strategy after every event either

Suppose an order is added:

```text
500 price levels away.
```

None of your features care.

Why recompute?

Chapter 5 gave us:

```text
feature dependency masks.
```

Example:

```text
BBOChanged
Top5Changed
TradeOccurred
OwnQueueChanged
CrossInstrumentChanged.
```

Then:

```cpp
if (change_mask &
    strategy.dependency_mask)
{
    evaluate();
}
```

This can dramatically reduce hot-path work.

---

# 83. Fast path versus slow strategy updates

Fast path:

```text
book event
→ alpha
→ quote safety
→ cancel/new.
```

Slow path:

```text
model parameter updates
risk coefficients
volatility regime
cross-instrument recalibration
configuration.
```

Slow calculations can run elsewhere and publish compact parameters.

---

# Part XVII — Cross-instrument fair value

# 84. Many serious HFT strategies don't price an instrument from itself alone

Suppose trading:

```text
S&P 500 ETF
```

while futures move first.

Or:

```text
option
```

whose underlying future moves.

Then:

$$
FV_X
=
f(
X\ book,
Y\ book,
Z\ book,\dots
).
$$

Example:

$$
FV_{ETF}
=
\beta F_{future}
+
basis.
$$

Now architecture becomes:

```text
Future feed
    │
    ├────► Strategy X
ETF feed ─┘
```

This introduces synchronization and stale-state issues.

---

# 85. Cross-market timestamping

Suppose:

```text
Future state timestamp = t
ETF state timestamp = t - 200 µs.
```

Using them as if simultaneous can be wrong.

Store:

```text
last update timestamp per input.
```

Strategy can reject stale cross-instrument state.

---

# 86. Synthetic fair value

Example:

$$
FV_A=
w_1P_B+w_2P_C+b.
$$

A production hot path can calculate this as:

```cpp
fv =
    w1 * stateB.microprice +
    w2 * stateC.microprice +
    intercept;
```

Potentially only a few multiply-adds.

The complex model fitting happens offline.

---

# Part XVIII — Volatility

# 87. Volatility changes optimal quoting

During calm market:

```text
mid moves rarely.
```

Tight quote may be safe.

During high volatility:

```text
market moves several ticks rapidly.
```

Tight quote becomes dangerous.

So quote width often increases with volatility.

Avellaneda–Stoikov also captures this concept: higher \(\sigma\) increases inventory-risk effects and optimal spread under the model. ([math.nyu.edu][7])

---

# 88. Short-horizon volatility estimator

You don't need a GARCH model on every packet.

Maintain something like:

$$
v_t=
\lambda v_{t-1}
+
(1-\lambda)(\Delta M_t)^2.
$$

Then:

```text
fast EWMA variance.
```

Different horizons:

```text
fast volatility
slow volatility.
```

Regime:

$$
VolRatio=
\frac{Vol_{fast}}
{Vol_{slow}}.
$$

Large ratio can indicate sudden market stress.

---

# Part XIX — Regime detection

# 89. The same model may not work everywhere

Possible regimes:

```text
quiet
normal
volatile
auction
news burst
spread widening
locked/crossed
recovery
halt.
```

Your strategy can adjust:

```text
quote width
size
fill model
cancel threshold.
```

Do not treat market dynamics as stationary.

---

# 90. Confidence gating

Suppose model was trained mostly on:

```text
1-tick spread normal market.
```

Current state:

```text
10-tick spread
massive volatility.
```

Instead of extrapolating blindly:

```text
reduce size
widen quotes
or stop trading.
```

Production strategies need:

# abstention.

Doing nothing is a valid trading action.

---

# Part XX — Backtesting the Strategy Properly

# 91. Price-only backtest is useless for this system

Wrong:

```text
If bid touched my order:
    fill me.
```

You need:

```text
market-data replay
+
queue reconstruction
+
exchange simulation
+
system latency
+
own order state.
```

---

# 92. Simulation architecture

```text
Historical Raw Feed
        │
        ▼
Same Feed Handler
        │
        ▼
Same Order Book
        │
        ▼
Same Strategy
        │
        ▼
Simulated Network Delay
        │
        ▼
Simulated Exchange
        │
        ▼
Matching Engine
        │
        ▼
Own Ack/Fill
        │
        ▼
Strategy State
```

This is why building your own matching engine in Chapter 4 was useful.

---

# 93. Latency model in simulation

A decision at:

$$
t_0
$$

does not reach exchange immediately.

Use:

$$
t_{exchange}
=
t_0
+
L_{strategy}
+
L_{gateway}
+
L_{network}.
$$

Then the exchange simulator inserts your order into the book state at:

$$
t_{exchange}.
$$

All market events occurring before that time happen first.

That's how you model queue races.

---

# 94. Separate market-data and order latency

Use:

```text
L_md
L_strategy
L_order
L_ack.
```

For example:

```text
exchange event
      ↓ L_md
strategy sees event
      ↓ L_strategy
order generated
      ↓ L_order
exchange receives it
      ↓
matching
      ↓ L_ack
strategy receives response.
```

A single:

```text
latency = 5 µs
```

is too crude.

---

# 95. Latency should be a distribution

Not:

```text
L_order = exactly 2 µs.
```

But something such as empirical:

```text
p50   1.8 µs
p99   2.4 µs
p99.9 5.5 µs.
```

Why?

Those rare latency spikes may generate many of your worst adverse-selection fills.

---

# 96. Queue-aware simulation

When your simulated order arrives at best bid:

```text
current queue = 5000.
```

Set:

```text
Qahead = 5000
```

under FIFO.

Then:

```text
executions ahead
cancellations ahead
```

reduce it.

Orders joining later:

```text
go behind.
```

Only once the queue ahead is depleted can your order fill.

This is far more realistic than:

```text
price touched → fill.
```

---

# 97. Cancellation modelling

Suppose aggregate data doesn't tell whether cancellation was ahead or behind.

You need a model.

Examples:

```text
all cancellations ahead
```

optimistic.

```text
all cancellations behind
```

pessimistic.

Or probabilistic:

$$
P(cancel\ ahead)
=
\frac{Q_{ahead}}
{Q_{ahead}+Q_{behind}}.
$$

This is still an assumption, but at least explicit.

MBO data improves the situation substantially.

---

# 98. Market impact problem

Historical replay assumes your order didn't affect history.

But in real trading:

```text
your order
```

changes the book.

Small passive HFT orders may have low impact.

Larger aggressive trading may materially affect:

```text
price
other participants
future order flow.
```

Therefore backtest accuracy degrades as strategy size grows.

This is an inherent counterfactual problem, not merely a software bug.

---

# Part XXI — Training fill and markout models

# 99. Fill dataset

For each hypothetical/actual quote, store features at placement:

```text
timestamp
price
side
queue ahead
imbalance
OFI
spread
depth
volatility
trade rates
cancel rates
latency state.
```

Labels:

```text
fill within 100 µs?
fill within 1 ms?
fill within 10 ms?

time to fill
quantity filled.
```

---

# 100. Markout dataset

For each actual/simulated fill:

```text
fill price
fill side
```

then calculate:

```text
10 µs markout
100 µs
1 ms
10 ms
100 ms.
```

Features:

```text
same market state at order/fill.
```

Then train:

$$
E[Markout_h\mid X].
$$

---

# 101. Avoid survivorship in fill-quality research

If you study markouts only for:

```text
orders that filled
```

that's correct for conditional fill quality.

But if deciding whether to quote, you also need:

```text
orders that didn't fill.
```

Otherwise you cannot estimate:

$$
P(fill).
$$

You need both models:

```text
fill probability
+
conditional fill P&L.
```

---

# 102. Calibration matters

If model says:

```text
Pfill = 70%
```

then among orders given approximately 70% probability, roughly 70% should fill under the defined horizon/population.

A model with great classification rank but poor probability calibration can produce bad expected-value decisions.

Evaluate:

```text
reliability curves
Brier score
log loss
```

not only AUC.

---

# Part XXII — Production Decision Engine

# 103. Final model architecture

For each side:

```text
Market State
     │
     ▼
Fair Value Model
     │
     ▼
Reservation Price
     │
     ├──── Candidate join
     ├──── Candidate improve
     └──── Candidate cross
             │
             ▼
      Fill Probability
             │
             ▼
       Markout Model
             │
             ▼
         Fees/Rebate
             │
             ▼
       Inventory Cost
             │
             ▼
         ACTION SCORE
```

Then:

```text
best positive score
```

wins subject to risk.

---

# 104. Candidate generation

Don't search every possible price in the market.

Generate a small candidate set.

For a bid:

```text
NO ORDER
current live price
best bid
one tick improve
cross ask
maybe second level.
```

Then score maybe:

```text
4–6 actions.
```

Much cheaper than an enormous optimization problem.

---

# 105. Candidate pruning

Suppose spread = 1 tick.

Then:

```text
price improvement
```

may immediately cross.

No need to consider it separately.

Suppose hard inventory long limit reached.

Remove:

```text
all buy candidates.
```

Prune before scoring.

---

# 106. Fast action evaluation

Conceptually:

```cpp
Action choose_bid_action(
    const MarketState& m,
    const StrategyState& s)
{
    Candidate candidates[4];

    int n =
        generate_candidates(m, s, candidates);

    Action best = Action::None;
    Score best_score = 0;

    for (int i = 0; i < n; ++i) {
        auto score =
            evaluate(candidates[i], m, s);

        if (score > best_score) {
            best_score = score;
            best = candidates[i].action;
        }
    }

    return best;
}
```

No heap.

No dynamic container.

Small bounded loop.

---

# 107. Don't calculate expensive probabilities for obviously bad candidates

For example:

```text
candidate violates risk limit
```

reject before model.

Or:

```text
candidate price > reservation + maximum edge bound.
```

reject.

Fast predicates should eliminate candidates early.

---

# Part XXIII — C++ Hot Path

# 108. Possible per-instrument state

```cpp
struct InstrumentStrategy {
    Price microprice;

    Price fair_value;
    Price reservation_price;

    int32_t imbalance;
    int64_t ofi_fast;

    Volatility volatility;

    Position position;

    QuoteState bid;
    QuoteState ask;

    Timestamp last_market_update;
};
```

Store only hot fields here.

Cold fields:

```text
symbol
model name
human-readable config
historical metrics
```

elsewhere.

---

# 109. AoS versus SoA

If one strategy processes:

```text
one instrument completely
```

per event:

```cpp
struct InstrumentState { ... };
InstrumentState states[N];
```

AoS may be ideal.

If a model loops over:

```text
fair values of 1000 instruments
```

then SoA:

```cpp
Price fair_values[N];
Position positions[N];
```

may be better.

Data layout should match access pattern.

Same Gross lesson again.

---

# 110. Parameter updates without locks

Research system may publish new coefficients.

Don't have strategy hot path take:

```cpp
std::mutex config_mutex;
```

every event.

Use:

```text
immutable parameter block A

build new block B off-thread

atomically switch active pointer/generation.
```

Strategy reads one stable block.

Parameter changes are rare.

Market ticks are frequent.

Optimize for the frequent path.

---

# 111. No logging in decision engine

Record compact:

```text
decision type
instrument
sequence
fair value
live quote
new quote
score
reason code.
```

into an asynchronous flight-recorder ring if needed.

Do not generate:

```text
"Cancelled AAPL bid because..."
```

inside the event path.

---

# Part XXIV — What to Measure in Production

# 112. Strategy metrics

You eventually need:

```text
gross P&L
net P&L

maker P&L
taker P&L

fees
rebates

fill rate
cancel rate

queue ahead at placement
queue ahead at fill

markout

inventory

spread capture

hedging cost.
```

---

# 113. Model metrics

For fair-value model:

```text
prediction correlation
MSE
directional accuracy
PnL by score bucket.
```

For fill model:

```text
calibration
fill rate by predicted bucket
time-to-fill error.
```

For markout:

```text
predicted vs actual
by horizon.
```

---

# 114. Latency-conditioned P&L

This is extremely valuable.

Bucket decisions by:

```text
market event → order arrival latency.
```

Example:

| Latency | Fill rate | 1 ms Markout |
| ------- | --------: | -----------: |
| 0–1 µs  |       35% |  +0.20 ticks |
| 1–2 µs  |       31% |  +0.12 ticks |
| 2–5 µs  |       20% |  -0.02 ticks |
| >5 µs   |       12% |  -0.30 ticks |

Now you can literally calculate:

> **How much money is 1 µs worth?**

That is the bridge between systems optimization and trading economics.

---

# 115. Queue-conditioned P&L

Likewise:

| Queue ahead | Fill probability |           Markout |
| ----------- | ---------------: | ----------------: |
| 0–500       |             High | potentially toxic |
| 500–2000    |           Medium |          moderate |
| >2000       |              Low |       maybe safer |

You may discover:

```text
best queue position
```

is not necessarily:

```text
best expected P&L.
```

---

# Part XXV — Failure / Safety Conditions

# 116. Strategy must automatically stop quoting when inputs are unsafe

Examples:

```text
book not LIVE

sequence gap

stale cross-market input

order session disconnected

latency spike above bound

inventory inconsistent

risk engine unhealthy

own order state unknown.
```

Correct response can be:

```text
cancel live orders
stop new quoting.
```

Do not let:

```text
strategy model score = positive
```

override system-health state.

---

# 117. Latency kill condition

Suppose normal:

```text
market→decision = 800 ns.
```

Suddenly:

```text
30 µs.
```

The strategy may be producing perfectly good decisions about:

```text
30-µs-old markets.
```

Which are terrible decisions.

A production system should monitor:

```text
event age.
```

If too old:

```text
don't act on it.
```

---

# Part XXVI — A sensible first strategy to build

Do **not** begin your project with:

```text
deep reinforcement learning
transformer
1000 features
multi-exchange arbitrage.
```

Start with something interpretable.

### Version 1 market maker

Fair value:

$$
FV=Microprice+\beta OFI.
$$

Inventory adjustment:

$$
R=FV-\lambda q.
$$

Quote:

```text
bid = floor_to_tick(R - half_spread)
ask = ceil_to_tick(R + half_spread)
```

Only quote if:

```text
book LIVE
spread acceptable
inventory within limits.
```

Queue model:

```text
visible queue ahead.
```

Fill simulator:

```text
FIFO event-level MBO.
```

Cancellation:

```text
cancel if quote deviates by ≥1 tick
or toxicity threshold breached.
```

That is already enough to teach you an enormous amount.

---

# 118. Version 2

Add:

```text
multi-level imbalance
trade imbalance
cancel/add rates
volatility
```

Train simple:

```text
linear fair-value model.
```

Then train:

```text
logistic fill model.
```

Now quote using:

$$
EV =
Pfill
\times
ExpectedMarkout.
$$

---

# 119. Version 3

Add:

```text
conditional markout model
queue-value preservation
requote hysteresis
latency distributions
maker/taker fees
inventory-dependent sizing.
```

Now your simulator becomes much more realistic.

---

# 120. Version 4

Add:

```text
correlated future/ETF
cross-instrument fair value
```

and measure:

```text
signal decay versus latency.
```

This begins to resemble the design problem real latency-sensitive strategies face.

---

# 121. Version 5

Only then consider:

```text
gradient boosting
small neural networks
Hawkes/event intensity models
queue-reactive simulation
reinforcement learning.
```

By then you will understand what problem the model is actually solving.

---

# Part XXVII — The deepest conceptual model

You should now stop thinking:

```text
HFT strategy predicts price.
```

A better mental model is:

```text
                CURRENT MARKET

                      │
                      ▼
             Future Price Distribution
                      │
                      │
           ┌──────────┴──────────┐
           │                     │
           ▼                     ▼
     Passive order          Aggressive order
           │                     │
           ▼                     ▼
     queue dynamics         immediate cost
           │                     │
           ▼                     │
     fill probability            │
           │                     │
           ▼                     │
   conditional future price      │
           │                     │
           └──────────┬──────────┘
                      ▼
                Expected P&L
                      │
               inventory/risk
                      │
                      ▼
                    ACTION
```

The passive-order problem contains a selection mechanism:

> You don't randomly receive fills.

You receive fills under **particular market states that caused someone to trade against you**.

That's why:

$$
P(fill)
$$

and:

$$
E[PnL\mid fill]
$$

must be modelled separately.

This is one of the most important concepts in the entire HFT architecture.

---

# 122. How Chapter 6 connects everything so far

We now have:

```text
EXCHANGE
   ↓
NETWORK
   ↓
NIC
   ↓
FEED HANDLER
   ↓
ORDER BOOK
   ↓
────────────────────────────────
          CHAPTER 6
────────────────────────────────
   ↓
FEATURES
   ↓
FAIR VALUE
   ↓
QUEUE MODEL
   ↓
FILL PROBABILITY
   ↓
MARKOUT / TOXICITY
   ↓
INVENTORY
   ↓
EXPECTED VALUE
   ↓
QUOTE DECISION
────────────────────────────────
   ↓
ORDER GATEWAY
   ↓
EXCHANGE.
```

The Gross talks explained:

```text
how to make the computation fast.
```

Chapter 3 explained:

```text
how the information reaches us.
```

Chapter 4 explained:

```text
how exchange priority creates economic value.
```

Chapter 5 explained:

```text
how to reconstruct the exchange state.
```

Chapter 6 finally explains:

> **How the reconstructed state becomes an economically rational trading action.**

---

# 123. The production hot path we now want

Ultimately:

```text
ITCH Event
     │
     ▼
Order Book Update
     │
     ▼
Incremental Features
     │
     ▼
Fair Value
     │
     ▼
Inventory Adjustment
     │
     ▼
Queue / Fill Model
     │
     ▼
Markout / Toxicity Estimate
     │
     ▼
Candidate Scores
     │
     ▼
JOIN / IMPROVE / CROSS /
CANCEL / NOTHING
```

And ideally this entire path has:

```text
no allocation

no lock

no blocking syscall

no string operation

bounded loops

small working set

predictable latency.
```

---

# 124. Where Chapter 7 starts

We now have a strategy saying:

```text
CANCEL order 7182

and

NEW SELL 100 @ 100.02.
```

But this is still only an **intent**.

We haven't solved:

```text
Is order 7182 actually still live?

What if it fills while cancellation is in flight?

How do we generate OUCH/iLink bytes?

What client-order ID do we use?

What if acknowledgement is lost?

What happens on disconnect?

How do we enforce position limits?

How do we throttle message rate?

What if NEW is accepted but our connection dies
before ACK?

How do we reconcile?

How does TX-ring ownership work?

How does a kill switch cancel thousands
of live orders safely?
```

That is exactly what **Chapter 7 — Production Order Gateway + Pre-Trade Risk + Order State Machine** should solve.

And Chapter 7 is important because after it, we will have closed the full loop:

```text
packet in
   ↓
book
   ↓
strategy
   ↓
decision
   ↓
risk
   ↓
order
   ↓
packet out
   ↓
exchange
   ↓
ack/fill
   ↓
position
   ↓
strategy again.
```

At that point, we will have the architecture for an actual end-to-end **HFT-like production low-latency trading engine**, rather than isolated pieces.

[1]: https://arxiv.org/abs/1512.03492?utm_source=chatgpt.com "Queue Imbalance as a One-Tick-Ahead Price Predictor in a Limit Order Book"
[2]: https://arxiv.org/abs/1104.4596?utm_source=chatgpt.com "Price dynamics in a Markovian limit order market"
[3]: https://arxiv.org/abs/1011.6402?utm_source=chatgpt.com "The Price Impact of Order Book Events"
[4]: https://arxiv.org/abs/1505.04810?utm_source=chatgpt.com "Dynamics of Order Positions and Related Queues in a Limit Order Book"
[5]: https://pubsonline.informs.org/doi/pdf/10.1287/educ.2013.0113?utm_source=chatgpt.com "Theory Driven by Influential Applications : Optimal Placement in a Limit Order Book"
[6]: https://arxiv.org/abs/1312.0563?utm_source=chatgpt.com "Simulating and analyzing order book data: The queue-reactive model"
[7]: https://math.nyu.edu/inmemoriam/avellaneda/HighFrequencyTrading.pdf?utm_source=chatgpt.com "High-frequency trading in a limit order book"
