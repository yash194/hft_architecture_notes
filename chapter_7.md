# Chapter 7 — Production Order Gateway + Pre-Trade Risk + Order State Machine

Chapter 6 ended with the strategy producing an **intent**:

```text
BUY 100 @ 100.00
CANCEL order 712
REPLACE order 900:
    price 100.01 → 100.00
```

That is not yet an exchange order.

Between:

```text
Strategy Decision
```

and:

```text
Exchange Matching Engine
```

we still need an extremely important subsystem:

```text
                       STRATEGY
                           │
                           │ OrderIntent
                           ▼
                  ┌──────────────────┐
                  │ PRE-TRADE RISK   │
                  └────────┬─────────┘
                           │
                           ▼
                  ┌──────────────────┐
                  │ ORDER MANAGER    │
                  │ STATE MACHINE    │
                  └────────┬─────────┘
                           │
                           ▼
                  ┌──────────────────┐
                  │ VENUE ADAPTER    │
                  │ OUCH / iLink     │
                  └────────┬─────────┘
                           │
                           ▼
                     SESSION LAYER
                           │
                           ▼
                        TX RING
                           │
                           ▼
                          NIC
                           │
                           ▼
                       EXCHANGE
                           │
                    ACK / FILL / REJECT
                           │
                           ▼
                          NIC
                           │
                           ▼
                  ┌──────────────────┐
                  │ RESPONSE DECODER │
                  └────────┬─────────┘
                           │
                           ▼
                  ORDER STATE MACHINE
                           │
                  ┌────────┼─────────┐
                  ▼        ▼         ▼
               Position   Risk     Strategy
```

This chapter is about making that entire path:

> **correct, deterministic, bounded, recoverable, risk-controlled and extremely fast.**

And perhaps the most important idea in the chapter is:

# Your desired state is not the exchange's state.

If you remember only one thing from Chapter 7, remember that.

---

# Part I — Why the order gateway is difficult

## 1. A strategy sends intentions, not truth

Suppose we have:

```text
Live order:
BUY 100 @ 100.00
```

The strategy decides:

```text
CANCEL
```

Locally we now **want**:

```text
quantity = 0
```

But the exchange may still have:

```text
quantity = 100
```

because the cancel hasn't arrived yet.

During this interval:

```text
our desired state       = CANCELLED

exchange actual state   = LIVE
```

Those are different.

And the order can still execute.

---

# 2. The fundamental latency race

Timeline:

```text
T0   Strategy decides cancel

T1   Cancel enters TX queue

T2   Cancel leaves NIC

T3   Aggressive seller reaches exchange

T4   Cancel reaches exchange
```

If:

```text
T3 < T4
```

then:

```text
SELLER HITS OUR ORDER
        ↓
WE FILL
        ↓
CANCEL ARRIVES TOO LATE
```

If:

```text
T4 < T3
```

then:

```text
CANCEL REMOVES ORDER
        ↓
SELLER ARRIVES
        ↓
OUR ORDER IS GONE
```

That is the **cancel race**.

Nasdaq OUCH makes this especially concrete: a full cancel request uses the current order's `UserRefNum` and a requested total intended quantity of zero; there is no generic “too late to cancel” response because an execution can simply arrive before the cancellation takes effect. 

---

# 3. Therefore this is wrong

```cpp
send_cancel(order);

order.state = Cancelled;
order.qty   = 0;
```

You have just lied to your own trading system.

Correct concept:

```cpp
send_cancel(order);

order.state = PendingCancel;
```

The order remains **potentially executable**.

Only the exchange can confirm the eventual result.

---

# Part II — The two different states every order has

## 4. Desired state

This is what strategy currently wants.

Example:

```text
DesiredQuote:

price = 100.01
qty   = 100
```

---

# 5. Exchange-confirmed state

This is what the exchange has actually acknowledged.

Example:

```text
LiveOrder:

price = 100.00
qty   = 100
```

Therefore strategy may want:

```text
100.01
```

while exchange still has:

```text
100.00.
```

The difference drives gateway actions.

---

# 6. Quote reconciliation

Conceptually:

```text
DESIRED                         EXCHANGE

No order                        No order
             → do nothing

BUY 100 @100                    No order
             → NEW

No order                        BUY 100 @100
             → CANCEL

BUY 100 @101                    BUY 100 @100
             → REPLACE

BUY 100 @100                    BUY 100 @100
             → do nothing
```

So the order gateway is effectively a:

# desired-state reconciler.

This is very similar to control-plane architectures elsewhere in distributed systems.

---

# Part III — The basic order state machine

## 7. Do not use a boolean `is_live`

This is hopelessly insufficient:

```cpp
bool is_live;
```

Real orders have multiple intermediate states.

A much better conceptual machine is:

```text
                         NEW INTENT
                            │
                            ▼
                      ┌─────────────┐
                      │ PENDING_NEW │
                      └──────┬──────┘
                 ┌───────────┼────────────┐
                 │           │            │
              ACCEPT       REJECT       FILL*
                 │           │            │
                 ▼           ▼            │
              ┌──────┐    ┌───────┐      │
              │ LIVE │    │ DEAD  │◄─────┘
              └──┬───┘    └───────┘
                 │
        ┌────────┼───────────────┐
        │        │               │
       FILL    CANCEL          REPLACE
        │        │               │
        ▼        ▼               ▼
   PARTIAL   PENDING_CANCEL   PENDING_REPLACE
        │        │               │
        │     ┌──┴────┐       ┌──┴──────┐
        │     │       │       │         │
        │   FILL    CANCEL   FILL     REPLACED
        │     │       │       │         │
        └─────┼───────┴───────┼─────────┘
              │               │
              ▼               ▼
            LIVE/DEAD       NEW LIVE ORDER
```

`FILL*` while pending new means the exact ordering of acceptance/fill messages depends on venue semantics; your state machine must follow the actual protocol.

---

# 8. A practical enum

```cpp
enum class OrderState : uint8_t {
    Free,

    PendingNew,
    Live,
    PartiallyFilled,

    PendingCancel,
    PendingReplace,

    Rejected,
    Cancelled,
    Filled,
    Expired,

    Unknown
};
```

`Unknown` deserves special attention.

It means:

> **We cannot confidently say what liability remains at the venue.**

That is a dangerous state.

---

# 9. Terminal states

These generally imply no remaining live quantity:

```text
Rejected
Cancelled
Filled
Expired
```

But don't infer this purely from the enum.

Maintain quantity explicitly as well.

---

# Part IV — What must be stored for each order?

A hot-path order state might conceptually look like:

```cpp
struct OwnOrder {
    OrderId local_id;

    uint32_t venue_user_ref;
    uint64_t venue_order_ref;

    InstrumentId instrument;

    Price price;

    uint32_t original_qty;
    uint32_t working_qty;
    uint32_t cumulative_filled;

    Side side;
    OrderState state;

    uint64_t create_tsc;
    uint64_t send_tsc;
    uint64_t ack_tsc;

    uint32_t replace_generation;

    uint32_t strategy_id;
};
```

Not every venue exposes the same identifiers.

The point is to capture:

```text
our identity
venue identity
price
quantity
fills
lifecycle
timestamps.
```

---

# 10. Keep hot and cold fields separate

Hot:

```text
order ID
state
price
quantity
side
instrument
risk quantities
```

Cold:

```text
human-readable symbol
debug reason string
audit description
full strategy name
operator notes.
```

Do not make every order object 256 bytes because somebody wants convenient logging.

Millions of hot objects multiply every wasted byte.

---

# Part V — Order identities

## 11. You normally need multiple IDs

A production system may encounter:

```text
InternalOrderId
ClientOrderId
VenueUserReference
ExchangeOrderReference
MatchId
SessionSequence
```

These are not interchangeable.

---

# 12. Internal order ID

Generated by your own system:

```text
Strategy Order ID = 81293002
```

Used internally across:

```text
strategy
risk
gateway
logging
replay.
```

Should ideally be:

```text
numeric
unique
cheap to generate.
```

---

# 13. Client / protocol reference

OUCH 5.0 uses a `UserRefNum`; Nasdaq's current specification requires it to be day-unique and strictly increasing for each OUCH account. Nasdaq combines logical OUCH account + participant-created `UserRefNum` to identify an order for that trading day. 

This is a good example of why your gateway needs:

```text
VenueOrderIdAllocator.
```

Not:

```cpp
rand();
```

---

# 14. Exchange order reference

After acceptance, the exchange may assign another identifier.

OUCH's Accepted message includes a day-unique exchange `Order Reference Number`. 

So your order may become:

```text
Internal ID       = 551992

UserRefNum        = 82944

Exchange OrderRef = 913587002154
```

All refer to the same logical exposure.

---

# 15. Match ID

An execution may have its own identifier.

OUCH's Executed message includes a Match Number shared between the buy and sell executions associated with the trade. 

Use match IDs for:

```text
deduplication
trade reconciliation
drop copy
broken trade handling
audit.
```

---

# Part VI — New order lifecycle

## 16. Strategy generates a new order

Chapter 6 says:

```text
BUY
100 shares
@ 100.00
```

First transform that into an internal immutable intent:

```cpp
struct NewOrderIntent {
    InstrumentId instrument;
    Side side;
    Price price;
    Quantity qty;
    TimeInForce tif;
    StrategyId strategy;
};
```

Do not let strategy directly write OUCH bytes.

---

# 17. Why strategy should not encode venue protocol

Otherwise:

```text
Strategy A knows OUCH
Strategy B knows iLink
Strategy C knows Eurex
```

Now trading logic and exchange integration are inseparable.

Better:

```text
Strategy

      ↓

Canonical OrderIntent

      ↓

Venue Adapter

      ├─ Nasdaq OUCH
      ├─ CME iLink
      └─ Eurex...
```

Logical abstraction.

Physical execution can still be fully inlined and zero-allocation.

---

# Part VII — Pre-trade risk

This is one of the most important components in the entire production system.

## 18. Why risk must occur before packet transmission

You cannot do:

```text
Strategy
   ↓
send order
   ↓
risk checks afterward.
```

Once the packet enters the exchange:

```text
too late.
```

Pre-trade controls must gate the order **before it becomes executable**.

For U.S. securities market access, SEC Rule 15c3-5 requires broker-dealers with market access to maintain automated pre-trade controls designed, among other things, to stop orders exceeding preset capital/credit thresholds and erroneous orders exceeding appropriate price/size parameters; the SEC explicitly says this applies to market-maker quotes as well. ([SEC][1])

The exact regulatory framework depends on jurisdiction/market, but from a system-design perspective the principle is universal:

> **Never give an automated strategy unrestricted packet-level market access.**

---

# 19. Risk must be fast

Enterprise-style architecture:

```text
Strategy
   ↓
HTTP
   ↓
Risk microservice
   ↓
database
   ↓
Redis
   ↓
response
```

might take:

```text
milliseconds.
```

Completely impossible for HFT.

Hot-path risk needs to look closer to:

```text
load position
load limits
integer comparisons
update counters
PASS / REJECT
```

possibly tens of instructions.

---

# 20. Layered risk architecture

Think of risk in layers:

```text
                 STRATEGY
                     │
                     ▼
             STATIC VALIDATION
                     │
                     ▼
              ORDER LIMITS
                     │
                     ▼
            POSITION / EXPOSURE
                     │
                     ▼
           PRICE / FAT-FINGER
                     │
                     ▼
             RATE / THROTTLE
                     │
                     ▼
              SESSION HEALTH
                     │
                     ▼
                  SEND
```

Some controls are:

```text
hard
```

and some are:

```text
strategy-level soft controls.
```

Keep the distinction explicit.

---

# Part VIII — Static validation

Before complex risk, reject impossible orders.

Examples:

```text
quantity > 0?

instrument valid?

side valid?

price on legal tick?

price within numeric encoding range?

time-in-force supported?

session permits this instrument?

venue currently open?

strategy permission valid?
```

These should be cheap.

---

# 21. Tick validation

Suppose instrument tick size:

```text
5 price units.
```

Price:

```text
1003
```

is invalid because:

```text
1003 % 5 != 0.
```

Hot path:

```cpp
if (price % tick_size != 0)
    reject_local();
```

Often tick size can be structured to make this even cheaper.

---

# Part IX — Maximum order size

## 22. Single-order quantity limit

Example:

```text
max_order_qty = 1000.
```

Intent:

```text
BUY 50,000.
```

Immediate local rejection.

This prevents:

```text
bug produces 100× intended quantity.
```

---

# 23. Notional limit

For equities:

$$
Notional=Price\times Quantity.
$$

Suppose:

```text
price = $500
qty   = 100,000
```

Notional:

$$
\$50,000,000.
$$

Maybe impossible under your limits even though:

```text
qty < max_qty.
```

Therefore both:

```text
quantity limits
notional limits
```

can be useful.

---

# Part X — Fat-finger price controls

## 24. Price reasonability

Suppose market:

```text
Bid = 100
Ask = 101.
```

Strategy accidentally generates:

```text
BUY @ 10000.
```

Technically valid integer.

Economically disastrous.

Use price collars:

```text
Reference = current mid/fair/venue limit

MaxBuyPrice =
    Reference + allowed_deviation

MinSellPrice =
    Reference - allowed_deviation.
```

Then:

```cpp
if (buy_price > max_buy)
    reject();
```

---

# 25. Reference price must itself be valid

Do not calculate:

```text
fat finger control
```

from a stale or invalid market book.

If:

```text
book != LIVE
```

then either:

```text
stop trading
```

or fall back to a separately valid risk reference.

Risk shouldn't depend blindly on the same possibly-corrupt state that generated the bad order.

---

# Part XI — Position risk

## 26. Current position is not enough

Suppose:

```text
Current filled position = +500
```

and you have live bids:

```text
Order A +1000
Order B +1000
Order C +1000.
```

Current position says:

```text
+500.
```

Worst plausible filled position is:

```text
+3500.
```

Therefore risk must consider:

# outstanding order exposure.

---

# 27. Potential position

Define:

$$
PotentialLong =
CurrentPosition
+
OpenBuyQuantity.
$$

And:

$$
PotentialShort =
CurrentPosition
-
OpenSellQuantity.
$$

If:

```text
long limit = +2000
```

then a new buy should account for all already-live and potentially-live buy orders.

---

# 28. Pending orders count too

This is crucial.

Suppose:

```text
Position = 0

PendingNew BUY 1000
```

Strategy wants another:

```text
BUY 1000.
```

You cannot say:

```text
first order isn't live yet
so exposure = 0.
```

It may be accepted one nanosecond later.

Risk should usually reserve exposure when the first order passes the local risk gate.

So:

```text
RISK RESERVED
```

does not mean:

```text
EXCHANGE CONFIRMED.
```

Different concepts.

---

# 29. Risk reservation

When New A passes:

```text
reserve +1000 buy exposure
```

before transmission.

Then New B sees:

```text
reserved exposure = +1000
```

and cannot accidentally ignore A.

This is another version of:

# single-writer ownership.

Ideally one component atomically owns:

```text
risk state + order submission.
```

No race between separate threads.

---

# Part XII — Worst-case exposure

## 30. The safe question

Do not ask:

> What do I think will probably happen?

Risk asks:

> **What could still legally happen at the exchange given all outstanding messages?**

That is much more conservative.

---

# 31. Example: Pending Cancel

Order:

```text
BUY 1000
state = PendingCancel.
```

Risk exposure is **not**:

```text
0.
```

It can still fill all 1000 before cancellation reaches the matching engine.

So:

```text
WorstCaseBuyExposure = 1000.
```

Only reduce exposure after definitive exchange events.

---

# Part XIII — Pending Replace is even harder

Suppose:

```text
Live BUY:
qty = 100
price = 100.
```

We send replacement:

```text
qty = 500
price = 101.
```

What exposure should risk reserve?

Not simply:

```text
100
```

because replacement may become 500.

Not simply:

```text
500
```

without understanding protocol semantics either.

For safe design, reason about the maximum quantity that could become executable across the replace operation.

---

# 32. Replace races with execution

Imagine:

```text
Original order = 500

Already filled = 0
```

You send replace.

But while the replacement is in transit:

```text
original fills 100.
```

Then replacement completes.

What quantity should remain?

Venue semantics decide.

Nasdaq's current OUCH design handles this explicitly: replace quantities represent total liability for the order/replace chain, including prior executions, so a replacement response can report fewer shares exposed if an execution occurred while the replace was in flight. The specification even gives this exact race as an example. 

This is why:

> **You cannot invent your own replace semantics.**

Implement the venue specification exactly.

---

# Part XIV — Local risk versus venue risk

## 33. Both matter

Your system:

```text
Strategy
 ↓
Our Risk
 ↓
Order
```

Exchange/clearing infrastructure may also apply:

```text
venue controls
clearing credit controls
exchange price controls.
```

For example, CME currently provides Globex Credit Controls for pre-execution limits and other risk-management interfaces, including real-time controls and order-entry blocking. ([CME Group][2])

But:

> Exchange risk controls are not a substitute for your own hot-path controls.

They are another defence layer.

---

# Part XV — Risk hierarchy

A production system might have:

```text
STRATEGY LIMIT

position ±100
       ↓

DESK LIMIT

position ±10,000
       ↓

FIRM LIMIT

risk / notional / credit
       ↓

BROKER/CLEARING LIMIT
       ↓

EXCHANGE CONTROLS.
```

A strategy should never get anywhere close to outer emergency limits during normal operation.

---

# Part XVI — Order rate limits

## 34. Why throughput risk matters

A bug could enter:

```cpp
while (true)
    send_order();
```

Even if each order is tiny.

Consequences:

```text
exchange throttling
message fees
session disconnect
massive open exposure
system overload.
```

So enforce:

```text
orders / second
cancels / second
replaces / second
total messages / second.
```

---

# 35. Token bucket

A useful throttle model is:

```text
tokens accumulate at rate R
max bucket capacity B

every message consumes token(s).
```

Conceptually:

```cpp
if (tokens >= cost) {
    tokens -= cost;
    send();
} else {
    throttle();
}
```

Allows:

```text
average rate control
+
short bursts.
```

---

# 36. But do not blindly queue stale orders behind a throttle

Suppose strategy generates:

```text
NEW quote at 100.00
```

but throttle delays it for:

```text
500 µs.
```

By transmission time the market may be:

```text
99.90 / 99.91.
```

Sending the old order is dangerous.

So throttled intents need:

```text
expiration / revalidation.
```

Often better:

```text
drop stale desired action
```

and let strategy produce current state rather than maintaining a long FIFO of obsolete quote commands.

---

# Part XVII — Intent coalescing

Suppose while throttled:

```text
T0 desired bid = 100.00

T1 desired bid = 100.01

T2 desired bid = 99.99
```

Do not necessarily transmit:

```text
100.00
then 100.01
then 99.99.
```

The relevant current desired state is:

```text
99.99.
```

This is another advantage of desired-state reconciliation.

You can collapse intermediate obsolete intents.

---

# Part XVIII — Cancel priority

If resource constrained:

```text
cancel stale dangerous order
```

typically deserves higher priority than:

```text
add new optional quote.
```

A possible TX priority:

```text
1. Emergency kill
2. Risk cancels
3. Strategy cancels
4. Critical hedges
5. Replace
6. New passive orders
```

Exact policy depends on strategy.

But don't treat every outbound message identically.

---

# Part XIX — Encoding OUCH

OUCH is a useful concrete example because Nasdaq explicitly describes it as its low-level native protocol designed for maximum performance rather than flexibility; host-to-client responses are sequenced by a lower layer, while host-bound messages are designed to be safely resent for recovery. 

## 37. Enter Order

Current OUCH 5.0 Enter Order includes core fields such as:

```text
UserRefNum
Side
Quantity
Symbol
Price
Time In Force
Display
Capacity
...
```

plus optional appendages for additional functionality. 

Do not construct it using:

```cpp
std::ostringstream
```

or dictionaries.

---

# 38. Preallocated encoder

Conceptually:

```cpp
struct OuchEncoder {
    static size_t encode_enter(
        std::byte* out,
        const NewOrder& o);
};
```

Encoder does:

```text
write type byte
store big-endian integers
copy fixed symbol
append optional fields
return byte count.
```

No heap.

---

# 39. Endianness

OUCH numeric fields are binary big-endian. 

So:

```cpp
store_be32(buffer + offset, quantity);
store_be64(buffer + offset, price);
```

Do not rely on native x86 byte order.

---

# 40. Prebuild message templates where useful

If many fields are constant:

```text
firm
capacity
display type
session-specific flags
```

you may create a template:

```text
[constant bytes][quantity][price][ID]
```

and patch only variable fields.

This can reduce:

```text
branches
copies
encoding work.
```

Only worthwhile after profiling.

---

# Part XX — OUCH acceptance

## 41. Accepted does not simply mean “our original order exists exactly as requested”

The current OUCH Accepted message echoes accepted values and explicitly notes that accepted price/time-in-force can differ from what was entered under exchange rules. It also returns an exchange-assigned order reference and an `Order State` of live or dead. 

Therefore:

```text
REQUESTED ORDER
```

and:

```text
ACCEPTED ORDER
```

may differ.

Update your live state from:

# exchange-confirmed fields.

Not from what you hoped was accepted.

---

# 42. Accepted-but-dead

OUCH can respond:

```text
Accepted
Order State = Dead.
```

Meaning the order was accepted and automatically cancelled; the spec says no additional messages follow for that order in that case. 

So:

```text
Accepted != Live.
```

Important.

---

# Part XXI — Execution processing

## 43. Executed message

OUCH's Executed message provides:

```text
UserRefNum
incremental executed quantity
execution price
liquidity flag
match number.
```



On execution:

```text
working_qty      -= fill_qty
cumulative_fill  += fill_qty

position         += signed(fill_qty)
```

and update:

```text
P&L
risk reservations
hedge requirements
strategy inventory.
```

---

# 44. Fill must update risk before generating new decisions

Suppose:

```text
position = +900

limit = +1000
```

Execution:

```text
BUY 100.
```

Now:

```text
position = +1000.
```

If strategy processes another market event before position/risk sees the fill, it may generate:

```text
BUY 100
```

based on stale inventory.

Therefore within the logical hot path:

```text
execution
   ↓
own order state
   ↓
position
   ↓
risk
   ↓
strategy
```

must have a deterministic ordering.

---

# Part XXII — Partial fills

Order:

```text
BUY 500.
```

Execution:

```text
100.
```

State becomes:

```text
cum filled = 100
working    = 400.
```

Not:

```text
FILLED.
```

Order remains live unless another event says otherwise.

---

# 45. Multiple fills

One order can receive:

```text
fill 100
fill 50
fill 200
fill 150
```

Your state machine needs to accept:

```text
multiple execution events
```

without assuming:

```text
one order → one trade.
```

---

# Part XXIII — Cancellations

## 46. Canceled response can be partial

OUCH's Canceled message reports the incremental quantity decremented and explicitly says this does **not necessarily mean the entire order is dead**. 

Suppose:

```text
working = 500
```

Cancel response:

```text
decremented = 200.
```

Then:

```text
working = 300.
```

Still live.

Don't set:

```text
state = Cancelled
```

unless working quantity becomes zero or protocol state confirms termination.

---

# Part XXIV — Cancel request semantics

OUCH's cancel request specifies a new intended total order size; setting that to zero requests cancellation of the entire remaining balance. Superfluous full cancels can be silently ignored. 

This matters for recovery.

You cannot always rely on:

```text
every outbound request
→ exactly one response.
```

Protocol semantics differ by request and state.

Your order state machine must tolerate this.

---

# Part XXV — Replace

## 47. Replace is not “edit the existing object”

Nasdaq's OUCH Replace request identifies both:

```text
original UserRefNum
new UserRefNum.
```

A successful replace yields a new order identity and always gets a new timestamp for time priority; Nasdaq explicitly says a partial Cancel/Modify should be used where priority-retaining behavior is desired and supported. 

Therefore:

```text
REPLACE
```

conceptually means:

```text
old order dies
      ↓
new order becomes live.
```

Not:

```cpp
order.price = new_price;
```

---

# 48. Queue consequences

Chapter 4 taught:

```text
price-time priority.
```

Therefore replacing an order often means:

```text
lose queue position.
```

This matters economically.

Chapter 6's strategy should therefore compare:

```text
value of current queue position
```

versus:

```text
value of improved quote.
```

The gateway simply implements the chosen lifecycle correctly.

---

# Part XXVI — Modify versus Replace

The current OUCH 5.0 spec contains a Modify request specifically for changes that generally preserve book priority, with restricted allowed changes; increasing share amount is not allowed through that priority-preserving path. 

This is an excellent example of why your canonical order API may need to distinguish:

```text
Cancel
Replace
Modify
```

instead of one generic:

```text
update_order().
```

Their microstructure consequences differ.

---

# Part XXVII — Rejection

## 49. Local reject versus exchange reject

Keep these separate.

### Local

Our risk engine says:

```text
RejectedBeforeSend.
```

Exchange never sees order.

### Exchange

Order was transmitted but venue responds:

```text
Rejected.
```

These have different diagnostic meaning.

---

# 50. Exchange reject

OUCH returns a Rejected message with the order's UserRefNum and a reject reason; rejected UserRefNums cannot be reused. 

Examples can include:

```text
invalid quantity
fat finger
halted
invalid side
replace not allowed
firm not authorized
etc.
```

([NASDAQ Trader][3])

Repeated exchange rejects should be treated as a health signal.

---

# 51. Reject-rate circuit breaker

For example:

```text
10 exchange rejects
within 100 ms
```

might indicate:

```text
protocol bug
bad reference data
market state changed
strategy bug.
```

Automatically stop new order entry rather than spamming the exchange indefinitely.

---

# Part XXVIII — Cancel/Fill race in state-machine form

Suppose:

```text
Order 50 LIVE
qty 100.
```

We send cancel:

```text
state = PendingCancel.
```

Then receive:

```text
Execution qty 100.
```

Correct:

```text
working = 0
position +=100
state = Filled.
```

The later cancel may produce nothing meaningful depending on venue semantics.

Wrong:

```text
ignore fill because we already canceled.
```

That is a catastrophic accounting bug.

---

# Part XXIX — Replace/Fill race

Start:

```text
BUY 500
UserRef 100
```

We request replacement.

While request travels:

```text
100 shares execute.
```

Then replacement confirms:

```text
400 remaining.
```

Your state transitions must be able to process:

```text
Execution old ID
    ↓
Replacement ack
```

without assuming the original order remained untouched.

Nasdaq explicitly documents this in its current Replace semantics. 

---

# Part XXX — State-machine transitions should be table-driven/testable

Rather than scattering:

```cpp
if (state == ...)
```

throughout the codebase, define transitions centrally.

Conceptually:

| Current        | Event         | New State       | Action              |
| -------------- | ------------- | --------------- | ------------------- |
| PendingNew     | Accepted Live | Live            | confirm exposure    |
| PendingNew     | Rejected      | Rejected        | release risk        |
| Live           | Partial Fill  | PartiallyFilled | position update     |
| Live           | Full Fill     | Filled          | release remaining   |
| Live           | Cancel Sent   | PendingCancel   | keep liability      |
| PendingCancel  | Fill          | Filled/Partial  | position update     |
| PendingCancel  | Cancel Ack    | Cancelled/Live  | adjust working      |
| Live           | Replace Sent  | PendingReplace  | reserve replacement |
| PendingReplace | Fill          | PendingReplace  | reduce chain qty    |
| PendingReplace | Replaced      | Live new ID     | remap identity      |

Then write tests for every transition.

---

# Part XXXI — Unknown/Impossible transitions

Suppose:

```text
order.state = Rejected
```

and suddenly:

```text
Execution arrives.
```

That should be impossible under normal correct state.

Do not silently accept it.

Trigger:

```text
critical invariant violation
flight recorder freeze
strategy disable
reconciliation.
```

Unexpected order-state transitions are often evidence of:

```text
dropped response
duplicate processing
corrupted state
protocol misunderstanding.
```

---

# Part XXXII — Session layer

The order protocol alone is not enough.

You also need to know:

```text
Are we connected?

What messages have been sent?

What responses have we processed?

What happens after disconnect?
```

That is the:

# session layer.

---

# 52. OUCH sequencing model

Nasdaq's current OUCH architecture says host-to-client messages are assumed sequenced and must use a lower-level protocol that guarantees delivery, typically SoupBinTCP; conversely, host-bound messages are treated as non-guaranteed and are designed so they can be safely resent during recovery. 

That tells you something profound:

> Application-level recovery semantics matter even when TCP exists.

TCP connection reliability does not magically solve application failure.

---

# Part XXXIII — Why TCP isn't enough

Imagine:

```text
Client sends New Order
        ↓
TCP stack transmits
        ↓
exchange receives
        ↓
exchange accepts
        ↓
our process crashes
        ↓
ACK not processed.
```

When application restarts:

```text
Did exchange accept it?
```

TCP cannot answer your new process's business-state question.

Network reliability:

```text
≠
order-state recovery.
```

---

# Part XXXIV — CME FIXP session model

CME iLink 3 uses a binary FIXP session model, with session establishment including operations such as:

```text
Negotiate
Establish
Sequence
Terminate
```

and tracks UUIDs and inbound/outbound sequence numbers. CME's current certification material explicitly tests sequence synchronization and recovery after application failure. ([CME Group][4])

So whether:

```text
OUCH/Soup
```

or:

```text
iLink/FIXP
```

the broader architecture remains:

```text
SESSION
   │
   ├─ identity
   ├─ sequencing
   ├─ connection
   ├─ keepalive
   ├─ recovery
   └─ termination
```

---

# Part XXXV — Session state machine

Conceptually:

```text
DISCONNECTED
      │
      ▼
CONNECTING
      │
      ▼
NEGOTIATING
      │
      ▼
ESTABLISHING
      │
      ▼
ACTIVE
      │
      ├── heartbeat failure ─► SUSPECT
      │
      ├── protocol error ────► TERMINATING
      │
      └── socket failure ────► DISCONNECTED
```

Strategy may only send orders when:

```text
SessionState == ACTIVE.
```

---

# Part XXXVI — Sequence numbers

Maintain:

```text
next_outbound_seq
next_expected_inbound_seq.
```

Never let unrelated strategy threads generate session sequence numbers independently.

One session owner should own:

```text
encoding
sequence assignment
TX.
```

Another single-writer pattern.

---

# Part XXXVII — What happens on a response gap?

Suppose expected:

```text
100
101
102
103
```

receive:

```text
100
101
103.
```

You may now be missing:

```text
102.
```

What if 102 was:

```text
FILL BUY 10,000?
```

You cannot continue normal trading as if nothing happened.

Session state becomes:

```text
RECOVERING / UNSAFE.
```

Depending on venue:

```text
request retransmission
reconnect/resynchronize
use drop copy
query state
```

etc.

---

# Part XXXVIII — The uncertain order-state problem

This is arguably the hardest operational problem.

Suppose:

```text
Order A sent.
```

Connection drops.

Locally:

```text
state = PendingNew.
```

Possible exchange states:

```text
Never received

Received and rejected

Received and live

Received and fully filled

Received and partially filled.
```

You do not know.

Therefore:

# never guess.

---

# 53. Unsafe recovery

Bad:

```text
connection recovered
↓
mark all PendingNew as rejected
↓
start trading.
```

You could now duplicate already-live exposure.

---

# 54. Safe recovery

Conceptually:

```text
connection failure
      ↓
stop new trading
      ↓
preserve outstanding order ledger
      ↓
restore session sequencing
      ↓
recover missing responses
      ↓
use venue/order-query/drop-copy mechanisms
      ↓
reconcile live orders/fills
      ↓
recompute positions/risk
      ↓
only then resume.
```

Correctness before speed.

---

# Part XXXIX — Idempotency

If a protocol allows benign resending, identifiers must let the venue distinguish:

```text
same request again
```

from:

```text
new logical order.
```

Nasdaq's OUCH design explicitly makes host-bound messages safely resendable and ties order uniqueness to account + UserRefNum. 

General lesson:

> Every business command crossing an unreliable application boundary needs well-defined duplicate semantics.

---

# Part XL — Drop Copy

A production trading system should not rely exclusively on the primary order session for its authoritative post-trade view.

A:

# drop-copy feed

provides an independent stream of:

```text
orders
fills
cancels
possibly other activity.
```

CME explicitly lists Drop Copy as part of its production/certification ecosystem alongside iLink order routing. ([CME Group][5])

Conceptually:

```text
                      MATCHING ENGINE
                         │       │
                         │       │
                    iLink ACK   DROP COPY
                         │       │
                         ▼       ▼
                     Hot Gateway Reconciler
```

---

# 55. Why independent copy matters

Suppose primary application has a bug:

```text
misses fill.
```

Its local position says:

```text
+100.
```

Drop copy says:

```text
+200.
```

This discrepancy is:

```text
CRITICAL.
```

Stop trading and reconcile.

---

# Part XLI — Position service

The gateway should update local low-latency positions immediately.

But a separate authoritative position service can aggregate across:

```text
strategy A
strategy B
session A
session B
venue A
venue B.
```

Architecture:

```text
Hot local position
      │
      │ events
      ▼
Firm position aggregator
```

Hot strategy does not wait synchronously for aggregator.

---

# Part XLII — Kill switches

## 56. What a kill switch means

Emergency action:

```text
STOP NEW ORDERS

and usually

CANCEL LIVE ORDERS.
```

Triggers:

```text
position breach

runaway message loop

market data invalid

order session corruption

abnormal losses

latency explosion

manual operator action

risk service failure.
```

---

# 57. Multi-layer kill design

Do not have only:

```text
one boolean in strategy.
```

Use several independent controls:

```text
STRATEGY KILL
      ↓
GATEWAY KILL
      ↓
FIRM RISK KILL
      ↓
EXCHANGE / CLEARING CONTROLS.
```

A broken strategy shouldn't be able to re-enable its own trading.

---

# Part XLIII — Disable new order entry first

Emergency sequence:

```text
1. Block NEW/REPLACE locally

2. Allow/priority-route CANCELS

3. initiate mass cancellation

4. reconcile resulting fills/cancels

5. verify zero/known live orders.
```

If you simply shut down the process:

```text
live orders may remain at exchange.
```

That's extremely dangerous.

---

# 58. Exchange-side mass cancel

Current OUCH 5.0 supports Mass Cancel requests, including criteria such as symbol/side/group, and also supports Disable Order Entry / Enable Order Entry requests at the account/firm level. 

This gives you another safety layer beyond sending:

```text
10,000 individual cancels.
```

But your own system must still reconcile the resulting order states.

---

# Part XLIV — Mass cancel is not magic either

Suppose mass cancel sent at:

```text
T0.
```

Some orders fill at:

```text
T0 + 100 ns
```

before it takes effect.

Therefore after kill:

```text
you may still receive fills.
```

Do not shut off inbound processing after initiating emergency cancellation.

Keep processing until state is reconciled.

---

# Part XLV — Order entry disable

A particularly useful concept is:

```text
block NEW
```

while preserving:

```text
ability to cancel/manage existing risk.
```

This is much safer than:

```text
disconnect immediately
```

because disconnect may make controlling existing exposure harder.

Current OUCH explicitly includes account-level Disable Order Entry functionality. 

---

# Part XLVI — Self-match prevention

You should also prevent strategies from unintentionally trading against your own resting orders.

System-level approach:

```text
Own bid
+
Own aggressive sell
      ↓
SMP logic
```

may:

```text
cancel old
cancel new
decrement
```

depending on venue functionality.

Nasdaq has been actively updating OUCH 5.0 for order-by-order self-match-prevention capabilities in 2026, which is a reminder that venue protocol/risk semantics evolve and adapters must be versioned and tested against current specifications. ([NASDAQ Trader][6])

Do not hard-code eternal assumptions about exchange protocols.

---

# Part XLVII — TX architecture

Now we reach the actual outgoing network path.

```text
Canonical Order
      ↓
Risk PASS
      ↓
Protocol Encoder
      ↓
TX Buffer
      ↓
TX Descriptor
      ↓
NIC DMA
      ↓
Wire.
```

---

# 59. Preallocated TX buffers

Do not:

```cpp
std::vector<uint8_t> packet;
packet.push_back(...);
```

per order.

Use:

```text
preallocated packet buffers
fixed queue
known maximum message size.
```

For DPDK:

```text
rte_mbuf
```

or equivalent venue/NIC API storage.

---

# Part XLVIII — TX ring ownership

Ideally:

```text
One gateway core
      │
      ▼
One TX queue.
```

Not:

```text
Strategy 1 ─┐
Strategy 2 ─┼── same TX ring
Strategy 3 ─┘
```

requiring synchronization.

Strategies submit:

```text
OrderIntent
```

to gateway owner.

Or if everything is on the same run-to-completion core:

```text
Strategy
  ↓
Risk
  ↓
Encode
  ↓
TX
```

with no handoff.

---

# Part XLIX — The run-to-completion ideal

For one ultra-fast strategy:

```text
Core 4

Market RX
    ↓
Book
    ↓
Strategy
    ↓
Risk
    ↓
Own Order State
    ↓
Encode
    ↓
TX
```

No:

```text
mutex
queue
cross-core transfer.
```

Then private execution responses may also be processed on that same owner or carefully coordinated.

This is the mechanically simplest low-latency model.

---

# Part L — When separate gateway core makes sense

If:

```text
many strategies
many exchange sessions
complex throttles
shared firm risk
```

you may use:

```text
Strategy cores
     │
   SPSC
     │
     ▼
Gateway core
```

Gateway serializes:

```text
risk
session sequencing
encoding
TX.
```

The queue adds latency but simplifies ownership.

As always:

```text
measure.
```

---

# Part LI — Cancel-path optimization

The fastest new-order path is useful.

But a market maker's:

# cancel path

may be even more economically important.

Optimize separately:

```text
Market event
    ↓
strategy marks quote dangerous
    ↓
risk/validation
    ↓
cancel encode
    ↓
TX.
```

Cancellation generally needs fewer decision checks than creating arbitrary new exposure.

---

# Part LII — Pre-generated cancel templates

If venue cancel message is simple, live order state may already contain enough information to build it with minimal operations.

You can maintain:

```text
UserRefNum
session routing
order group
```

directly in hot order state.

Then:

```text
CancelIntent
      ↓
few stores
      ↓
TX.
```

Do not search a complex order database on the critical cancel path if you already have the live-order handle.

---

# Part LIII — Risk path design

## 60. Keep risk state compact

Example:

```cpp
struct InstrumentRisk {
    int64_t position;

    uint32_t open_buy_qty;
    uint32_t open_sell_qty;

    int64_t long_limit;
    int64_t short_limit;

    uint32_t max_order_qty;

    Price max_buy_price;
    Price min_sell_price;
};
```

Indexed by:

```text
InstrumentId.
```

No hash lookup if IDs are dense.

---

# 61. Risk check

Conceptually:

```cpp
bool allow_buy(
    const InstrumentRisk& r,
    Price price,
    Qty qty)
{
    if (qty == 0 ||
        qty > r.max_order_qty)
        return false;

    if (price > r.max_buy_price)
        return false;

    const auto potential =
        r.position +
        r.open_buy_qty +
        qty;

    if (potential > r.long_limit)
        return false;

    return true;
}
```

Real risk is richer, but notice:

```text
loads
integer arithmetic
branches.
```

No network.

---

# Part LIV — Firm-wide risk complicates single-core design

Suppose:

```text
Strategy A trades AAPL
Strategy B trades AAPL
```

on separate cores.

Both share:

```text
firm position limit.
```

If each independently reads:

```text
position = +900
limit = +1000
```

both may simultaneously send:

```text
BUY 100.
```

Now firm reaches:

```text
+1100.
```

This is a classic distributed reservation problem.

---

# 62. Solutions

Possible architecture:

```text
central risk owner
```

or:

```text
preallocated risk budgets.
```

Example:

```text
Firm limit = 1000.

Strategy A budget = 400
Strategy B budget = 300
Strategy C budget = 300.
```

Each strategy can locally enforce its allocated budget without shared atomics on every order.

Higher-level risk periodically redistributes limits.

This is a very HFT-style design:

> Move coordination off the hot path by partitioning ownership.

---

# Part LV — Budget partitioning

Firm-level:

```text
Total allowed long exposure = 10,000.
```

Assign:

```text
Core A = 3,000
Core B = 2,000
Core C = 5,000.
```

Hot path checks only:

```text
local budget.
```

Slow risk layer adjusts budgets when needed.

This avoids:

```text
global atomic CAS on every order.
```

Same mechanical-sympathy philosophy again.

---

# Part LVI — Throttle architecture

Similarly avoid one contended global:

```cpp
atomic<int> messages_this_second;
```

across 50 cores.

Possible:

```text
per-session throttle
per-core budget
periodic central aggregation.
```

The session gateway naturally owns its own protocol rate.

---

# Part LVII — Persistence and audit

You do need an audit trail.

But do not synchronously write a database transaction before every order unless the venue/regulatory design specifically requires that architecture.

The hot path should publish compact events:

```text
OrderIntent
RiskDecision
OrderSent
Ack
Fill
Cancel
Reject.
```

to a recorder.

---

# 63. Order journal

Conceptually:

```text
Event:
sequence
timestamp
order ID
type
price
qty
state
```

stored in:

```text
preallocated memory ring
```

and drained by another component.

For CME specifically, customer systems are subject to front-end audit-trail certification requirements for iLink 3. ([CME Group][7])

Production design must therefore satisfy both:

```text
latency
```

and:

```text
regulatory/audit correctness.
```

---

# Part LVIII — A flight recorder is invaluable

Maintain last:

```text
100k / 1M
```

compact gateway events in memory:

```text
market sequence
strategy action
risk result
order send
session sequence
ACK
fill
cancel
rejection.
```

When something impossible occurs:

```text
freeze recorder.
```

Now you can reconstruct:

```text
the exact race.
```

---

# Part LIX — Deterministic replay

Your replay system should feed:

```text
market events
private order responses
timer events
risk updates
parameter changes.
```

into the same order state machine.

Then:

```text
same inputs
     ↓
same state transitions
     ↓
same positions/orders.
```

This is fundamental for debugging race conditions.

---

# Part LX — Gateway event sequencing

There are two independent streams:

```text
MARKET EVENTS

PRIVATE ORDER EVENTS.
```

Both affect strategy.

The system needs a deterministic policy for how they are observed.

Example:

```text
T0 market update
T1 fill
T2 market update
```

The replay engine must reproduce the same ordering visible to the strategy.

Otherwise:

```text
replay != production.
```

Hardware timestamps/sequence numbers can help establish ordering boundaries, but cross-channel “simultaneity” needs explicit system design.

---

# Part LXI — Strategy does not own order truth

This boundary should be strict:

```text
Strategy says:
"I want bid 100 @ 100"

Gateway says:
"Current exchange-confirmed state is..."
```

The strategy must not mutate:

```text
order.state.
```

Only gateway/order manager does.

Strategy mutates:

```text
desired state.
```

This clean separation prevents enormous classes of bugs.

---

# Part LXII — Desired quote model

For a market maker:

```cpp
struct DesiredQuote {
    bool enabled;

    Price price;
    Quantity qty;
};
```

Strategy writes:

```text
desired bid
desired ask.
```

Gateway reconciles:

```text
desired vs current.
```

This is often cleaner than strategy generating a long series of imperative commands.

---

# Part LXIII — Example reconciliation

Exchange:

```text
Live bid:
100 @100.00
```

Strategy:

```text
Desired:
100 @100.01.
```

Gateway sees:

```text
difference in price
```

then checks:

```text
replacement permitted?
risk?
throttle?
queue priority cost?
```

and sends appropriate venue command.

While replacement pending:

```text
desired may change again.
```

Gateway stores newest desired state.

Once current in-flight operation resolves:

```text
reconcile again.
```

---

# Part LXIV — This prevents replace storms

Suppose market moves rapidly:

```text
100.01
100.02
100.03
100.04
```

while first replace is still pending.

Naive:

```text
send four replaces.
```

State becomes complicated and messaging explodes.

Desired-state reconciler:

```text
current exchange = 100.00

pending →100.01

latest desired =100.04
```

When first operation resolves:

```text
reconcile directly toward 100.04.
```

Exact behaviour depends on venue capability, but this architecture makes coalescing possible.

---

# Part LXV — Pending-operation policy

For each order define whether venue permits:

```text
cancel while replace pending?

replace while replace pending?

new cancel after partial fill?

chained replaces?
```

OUCH, for example, allows replace chains, but the semantics around IDs and cumulative liability are specific. 

Don't invent a generic order state machine and assume it fits every exchange.

Build:

```text
common conceptual state
+
venue-specific transition rules.
```

---

# Part LXVI — Broken trades

OUCH supports a Broken Trade message after an execution, indicating the execution will no longer clear. 

That means even:

```text
FILLED
```

isn't always the absolute end of post-trade semantics.

You may need:

```text
position correction
P&L correction
risk adjustment
trade reconciliation.
```

Another reason post-trade/drop-copy systems are necessary.

---

# Part LXVII — Risk on broken trades

If:

```text
BUY 100 filled
```

position changes:

```text
+100.
```

Later trade break:

```text
position back to 0.
```

Risk and strategy need a consistent correction event.

Do not “delete the previous fill.”

Model:

```text
TradeBreak
```

as a new state transition.

Append-only event reasoning is much safer.

---

# Part LXVIII — System health gating

Order entry should depend on an explicit health vector.

Example:

```cpp
struct TradingHealth {
    bool market_data_live;
    bool order_session_live;
    bool position_consistent;
    bool risk_live;
    bool latency_healthy;
    bool operator_enabled;
};
```

Then:

```cpp
can_trade =
    market_data_live &&
    order_session_live &&
    position_consistent &&
    risk_live &&
    latency_healthy &&
    operator_enabled;
```

No mysterious global:

```cpp
bool trading = true;
```

---

# Part LXIX — Different health levels

Possible modes:

```text
ACTIVE

CANCEL_ONLY

HEDGE_ONLY

DISABLED

RECOVERING.
```

This is more useful than:

```text
ON/OFF.
```

Example:

```text
market data invalid
```

might imply:

```text
CANCEL_ONLY.
```

Whereas:

```text
primary quoting strategy dead
but position known
```

might permit:

```text
HEDGE_ONLY.
```

---

# Part LXX — Latency health

Chapter 6 established that stale decisions can be economically invalid.

Track:

```text
market event age
risk evaluation time
gateway queue delay
TX submission delay
ACK latency.
```

If:

```text
gateway queue age > safety threshold
```

do not send the stale intent.

---

# Part LXXI — ACK latency

Measure:

$$
T_{ack}
=
T_{ack\ received}
-
T_{order\ sent}.
$$

Distribution:

```text
p50
p99
p99.9.
```

Sudden increase may indicate:

```text
network issue
exchange load
session issue
local backlog.
```

This can feed system health.

---

# Part LXXII — Cancel ACK latency

Measure separately:

$$
T_{cancel}
=
T_{cancel-confirmed}
-
T_{cancel-requested}.
$$

Because for market making:

```text
cancel responsiveness
```

is itself a core risk metric.

---

# Part LXXIII — Decision-to-wire latency

Timestamp:

```text
T0 strategy action chosen

T1 risk passed

T2 message encoded

T3 TX submitted

T4 NIC hardware TX timestamp.
```

Then:

$$
T_{strategy→risk}
$$

$$
T_{risk→encode}
$$

$$
T_{encode→NIC}
$$

can all be measured.

This lets you know what to optimize.

---

# Part LXXIV — Exchange acceptance latency

If exchange response contains an exchange-side timestamp and your clocks are sufficiently synchronized, you may also estimate parts of:

```text
wire + venue processing.
```

Be careful about clock domains.

Never subtract timestamps from unrelated clocks without synchronization/known offset.

---

# Part LXXV — C++ architecture

A useful project structure could be:

```text
order/

  intent/
    order_intent.hpp
    desired_quote.hpp

  risk/
    instrument_limits.hpp
    exposure.hpp
    throttle.hpp
    risk_engine.hpp

  state/
    own_order.hpp
    order_pool.hpp
    order_state_machine.hpp
    positions.hpp

  venue/
    venue_adapter.hpp

    nasdaq/
      ouch_encoder.hpp
      ouch_decoder.hpp
      ouch_session.hpp

    cme/
      ilink_encoder.hpp
      ilink_decoder.hpp
      fixp_session.hpp

  transport/
    tx_queue.hpp
    rx_private.hpp

  recovery/
    order_reconciler.hpp
    drop_copy.hpp

  control/
    kill_switch.hpp
    trading_health.hpp

  audit/
    order_journal.hpp
    flight_recorder.hpp
```

---

# Part LXXVI — Hot path composition

Conceptually:

```cpp
void on_strategy_intent(
    const OrderIntent& intent)
{
    if (!health.can_submit_new())
        return;

    auto& risk =
        risk_state[intent.instrument];

    if (!risk_engine.allow(intent, risk))
        return;

    auto handle =
        order_manager.reserve(intent);

    auto msg =
        encoder.encode_new(handle, intent);

    session.send(msg);

    order_manager.mark_sent(handle);
}
```

Real code must ensure:

```text
risk reservation
order state reservation
sequence assignment
TX submission
```

have well-defined failure semantics.

---

# Part LXXVII — What if TX submission fails?

Suppose risk reserved:

```text
+100.
```

Order state:

```text
PendingNew.
```

But NIC TX call says:

```text
not submitted.
```

Now the order never became network-visible.

You need:

```text
release reservation
transition local order state
possibly retry
```

depending on whether submission definitely failed.

Again:

# every boundary needs precise ownership semantics.

---

# Part LXXVIII — NIC submission versus exchange receipt

Even:

```text
NIC accepted TX descriptor
```

doesn't prove:

```text
exchange received order.
```

It only proves:

```text
local transmission path accepted it.
```

Until venue response/recovery proves outcome, order is:

```text
in-flight.
```

This is why gateway state exists.

---

# Part LXXIX — Order pool

Like Chapter 5's public order book, your own-order manager should use:

```text
preallocated object pool.
```

No:

```cpp
new OwnOrder
```

per quote.

Allocate at startup:

```text
MaxOutstandingOrders
+
MaxPendingRequests
+
safety margin.
```

Pool exhaustion:

```text
critical risk event.
```

Never silently fall back to heap allocation in production.

---

# Part LXXX — Fixed-capacity lookup

Need:

$$
LocalOrderID\rightarrow OwnOrder.
$$

and possibly:

$$
UserRefNum\rightarrow OwnOrder.
$$

Use:

```text
dense arrays
flat hash
generation-indexed pool
```

rather than node-heavy dynamic structures if profiling justifies it.

---

# Part LXXXI — Reuse and generations

If slot 500 originally holds:

```text
Order A
```

then later:

```text
Order B,
```

a stale asynchronous reference to slot 500 must not accidentally mutate B.

Use:

```text
index + generation
```

when references escape the immediate owner.

---

# Part LXXXII — One owner of order state

Do not allow:

```text
Strategy Core
Risk Core
Gateway Core
DropCopy Core
```

all to mutate:

```text
OwnOrder.
```

Choose one owner.

Others send events.

This is perhaps the most important concurrency rule in Chapter 7.

---

# Part LXXXIII — Private-event fan-in

If one gateway owner handles state, inputs may include:

```text
StrategyIntent
VenueAck
VenueFill
DropCopyFill
RiskKill
OperatorCommand.
```

All become:

```text
typed events
```

processed by one state machine.

This produces deterministic ownership.

---

# Part LXXXIV — Separate authoritative and advisory streams

Primary venue response:

```text
hot authoritative order lifecycle.
```

Drop copy:

```text
independent reconciliation.
```

Do not blindly process the same fill twice.

Match using:

```text
Order ID
Match ID
session IDs
```

and use drop copy to confirm/detect discrepancy.

---

# Part LXXXV — Recovery reconciliation

After reconnect:

```text
Local:
Order A Live
Order B PendingCancel
Order C PendingNew
```

Venue-recovered:

```text
A Live
B Filled
C Live.
```

Reconcile:

```text
A → keep Live

B → update fill/position

C → establish Live state.
```

Only when:

```text
orders + fills + positions
```

agree can trading resume.

---

# Part LXXXVI — Never infer position solely from live orders

Position comes from:

```text
executions.
```

Live orders represent:

```text
future possible exposure.
```

Risk combines both.

Keep distinct:

```text
Position
OpenExposure
ReservedExposure.
```

---

# Part LXXXVII — Risk accounting model

A useful conceptual model:

$$
CurrentPosition
$$

plus:

$$
LiveExposure
$$

plus:

$$
PendingExposure
$$

gives:

$$
WorstCasePosition.
$$

For buy side:

$$
WorstLong =
Position
+
LiveBuy
+
PendingBuy.
$$

For short side:

$$
WorstShort =
Position
-
LiveSell
-
PendingSell.
$$

Then evaluate new intent against hard limits.

---

# Part LXXXVIII — Netting can be dangerous

Suppose:

```text
Live BUY 1000

Live SELL 1000.
```

Net open exposure seems:

```text
0.
```

But both aren't guaranteed to fill simultaneously.

One could fill and the other cancel.

Therefore hard risk often needs:

```text
gross directional potential
```

not naive netting.

Example:

```text
Potential long = position + open buys

Potential short = position - open sells.
```

---

# Part LXXXIX — Replace reservations

During replacement:

```text
old order may still fill
```

and:

```text
new state may become active.
```

Risk reservation logic must match protocol-specific maximum liability.

This is one of the areas where a generic abstraction can be actively dangerous.

Encode venue semantics in unit-tested risk transition functions.

---

# Part XC — Risk property tests

Generate random:

```text
New
Fill
Cancel
Replace
Reject
Disconnect
Recovery.
```

Assert:

```text
risk never understates possible exposure.
```

That's a great property.

For every event sequence:

$$
ReservedExposure
\ge
ActualPossibleExchangeExposure.
$$

A conservative simulator can verify this.

---

# Part XCI — Order state fuzzing

This state machine is perfect for stateful fuzzing.

Generate sequences such as:

```text
New
Ack
Fill
CancelRequest
Fill
CancelAck
```

and:

```text
New
Replace
FillOld
Replaced
FillNew
```

and:

```text
New
Disconnect
Reconnect
Recover.
```

Assert:

```text
no negative working quantity

cum_fill ≤ valid total liability

no terminal order becomes live unexpectedly

risk consistent

IDs unique

positions reconcile.
```

---

# Part XCII — Golden protocol tests

For OUCH/iLink encoding:

```text
known fields
      ↓
expected exact bytes.
```

Test both directions.

This catches:

```text
wrong offset
endianness
field width
schema version errors.
```

Protocol encoding should have byte-perfect tests.

---

# Part XCIII — Certification environments

Actual exchanges generally require conformance/certification before production connectivity. CME's current AutoCert framework explicitly tests order routing, session recovery, sequence handling, primary gateway behaviour and audit-trail capabilities for iLink 3. ([CME Group][8])

For your own HFT-like project, build an equivalent:

# exchange-certification simulator.

Make the client prove:

```text
connect
new
cancel
replace
partial fill
reject
sequence gap
disconnect
reconnect
recover
kill.
```

---

# Part XCIV — Local exchange simulator

Your Chapter 4 matching engine can now become:

```text
Simulated Exchange
```

supporting:

```text
ITCH/market-data output

OUCH-like order input

ACK

FILL

CANCEL

REPLACE

REJECT

disconnects

artificial latency.
```

This lets you test the complete system:

```text
Feed Handler
→ Strategy
→ Risk
→ Gateway
→ Exchange
→ Fill
→ Position.
```

---

# Part XCV — Simulate network races

Test:

### Cancel wins

```text
cancel arrival = 100 ns
aggressor arrival = 200 ns
```

No fill.

### Fill wins

```text
aggressor = 100 ns
cancel = 200 ns.
```

Fill.

### Replace race

```text
fill old
then replace processed.
```

This is how you verify your state machine before touching a real venue.

---

# Part XCVI — Failure injection

Your integration tests should deliberately:

```text
drop ACK

duplicate ACK

delay cancel

disconnect after send

drop private sequence

corrupt message

reorder simulated response

fill during PendingCancel

fill during PendingReplace

return exchange reject

exhaust throttle

trigger kill switch.
```

If these scenarios aren't tested, they will eventually happen in the worst possible production moment.

---

# Part XCVII — Production order path

The final new-order hot path should look like:

```text
              Strategy Candidate
                      │
                      ▼
               Desired Quote
                      │
                      ▼
                  Reconcile
                      │
                      ▼
             Static Validation
                      │
                      ▼
              Pre-Trade Risk
                      │
                      ▼
             Reserve Exposure
                      │
                      ▼
              Reserve Order ID
                      │
                      ▼
             Assign Session Seq
                      │
                      ▼
              Encode OUCH/iLink
                      │
                      ▼
                TX descriptor
                      │
                      ▼
                     NIC
```

Normal path should contain approximately:

```text
array lookups
integer arithmetic
bounded branches
fixed-size stores.
```

---

# Part XCVIII — Response hot path

```text
                    NIC RX
                       │
                       ▼
                 Session Decode
                       │
                       ▼
                 Sequence Check
                       │
                       ▼
                 Venue Message
                       │
          ┌────────────┼─────────────┐
          ▼            ▼             ▼
         ACK          FILL          CANCEL
          │            │             │
          └────────────┼─────────────┘
                       ▼
                Order State Machine
                       │
                       ▼
                   Risk State
                       │
                       ▼
                    Position
                       │
                       ▼
                    Strategy
```

No:

```text
database lookup
HTTP request
JSON
mutex to remote service.
```

---

# Part XCIX — Full end-to-end architecture

Now all seven chapters fit together:

```text
                      EXCHANGE
                         │
                         │ MARKET DATA
                         ▼
                ┌────────────────┐
                │      NIC       │
                └───────┬────────┘
                        │
                   RX / DMA
                        │
                        ▼
              ┌──────────────────┐
              │  FEED HANDLER    │
              │ A/B / sequence   │
              │ ITCH / SBE       │
              └────────┬─────────┘
                       │
                       ▼
              ┌──────────────────┐
              │   ORDER BOOK     │
              │ MBO / MBP / BBO  │
              └────────┬─────────┘
                       │
                       ▼
              ┌──────────────────┐
              │ FEATURE ENGINE   │
              │ OFI / microprice │
              │ queue / flow     │
              └────────┬─────────┘
                       │
                       ▼
              ┌──────────────────┐
              │ STRATEGY ENGINE  │
              │ fair value       │
              │ fill probability │
              │ markout          │
              │ inventory        │
              └────────┬─────────┘
                       │
                   ORDER INTENT
                       │
                       ▼
              ┌──────────────────┐
              │ PRE-TRADE RISK   │
              │ qty / position   │
              │ price / exposure │
              │ throttle         │
              └────────┬─────────┘
                       │
                       ▼
              ┌──────────────────┐
              │ ORDER MANAGER    │
              │ state machine    │
              │ ID / liability   │
              └────────┬─────────┘
                       │
                       ▼
              ┌──────────────────┐
              │ VENUE ADAPTER    │
              │ OUCH / iLink     │
              │ FIXP/session     │
              └────────┬─────────┘
                       │
                       ▼
                   TX / DMA
                       │
                       ▼
                      NIC
                       │
                       ▼
                  EXCHANGE
                       │
                Matching Engine
                       │
          ┌────────────┴────────────┐
          │                         │
      ACK/FILL                   Market Data
          │                         │
          └──────────────┬──────────┘
                         │
                         ▼
                       LOOP
```

Alongside:

```text
Drop Copy
Risk Supervisor
Kill Switch
Audit Journal
Flight Recorder
Position Aggregator
Monitoring
Replay
Recovery
Shadow Engine
```

---

# Part C — What absolutely must stay off the hot path

Avoid:

```text
SQL database

Redis lookup

REST API

Kafka roundtrip

text logging

filesystem

dynamic config parsing

remote risk RPC

garbage collection

unbounded queue

malloc/new per order.
```

Those systems can exist elsewhere in the platform.

They simply shouldn't sit between:

```text
strategy decision
```

and:

```text
NIC transmission.
```

---

# 101. What the hot path should roughly be

```text
Strategy says BUY
     ↓
load local risk
     ↓
5–20 checks
     ↓
reserve exposure
     ↓
allocate pool slot
     ↓
assign numeric ID
     ↓
write protocol bytes
     ↓
TX.
```

That is the design target.

---

# 102. Most important correctness invariant

At any instant your system must be able to answer:

> **What is the maximum exposure that could still exist at the exchange given everything we have sent and everything the exchange has confirmed?**

If your system cannot answer that confidently:

```text
do not send new risk.
```

That single principle drives:

```text
PendingNew

PendingCancel

PendingReplace

reserved exposure

recovery

kill switches.
```

---

# 103. Why Chapter 7 is not just “send packets fast”

A naive gateway is:

```text
Strategy
↓
serialize
↓
socket.send().
```

A production gateway is:

```text
Desired State

   ↓

Risk Reservation

   ↓

Order Lifecycle

   ↓

Protocol Identity

   ↓

Session Sequencing

   ↓

Network Transmission

   ↓

In-flight Liability

   ↓

Venue Confirmation

   ↓

Execution Accounting

   ↓

Position

   ↓

Reconciliation

   ↓

Recovery

   ↓

Audit.
```

Sending bytes is arguably the easiest part.

---

# 104. Recommended implementation progression

For your actual HFT-like project, I would build Chapter 7 in this sequence:

### Phase 1 — Order state machine

Implement entirely offline:

```text
New
Accepted
Rejected
PartialFill
FullFill
Cancel
CancelAck
Replace
ReplaceAck.
```

No network.

Brutal unit/property testing.

### Phase 2 — Risk engine

Implement:

```text
max qty
position limit
open-order exposure
pending exposure
price collar
instrument permissions.
```

Prove risk never under-reserves exposure.

### Phase 3 — Simulated exchange gateway

Connect to your Chapter 4 matching engine:

```text
Strategy
→ Risk
→ Gateway
→ Simulated Exchange.
```

Add network-delay simulation.

### Phase 4 — Failure/race simulation

Test:

```text
fill-before-cancel
cancel-before-fill
fill-during-replace
disconnect-after-send
missing ACK
recovery.
```

### Phase 5 — Binary venue adapter

Build OUCH-like encoder/decoder with byte-exact golden tests.

### Phase 6 — Session layer

Add:

```text
connection
sequences
heartbeats
disconnect
recovery.
```

### Phase 7 — TX/RX optimized path

Move to:

```text
DPDK / ef_vi / AF_XDP
```

or an appropriate low-latency transport.

Then benchmark:

```text
strategy decision → NIC TX

NIC RX response → position update.
```

---

# 105. What you know after Chapter 7

You now approximately understand the major architecture of a production HFT engine:

```text
Hardware
✓ CPU
✓ cache
✓ NUMA
✓ NIC
✓ PCIe
✓ FPGA

Networking
✓ Ethernet
✓ UDP
✓ multicast
✓ kernel bypass
✓ RX/TX rings
✓ DMA

Market Data
✓ A/B feeds
✓ sequence recovery
✓ ITCH
✓ SBE
✓ feed handler

Market State
✓ order books
✓ MBO / MBP
✓ BBO
✓ queue position

Microstructure
✓ FIFO
✓ pro-rata
✓ maker/taker
✓ auctions
✓ adverse selection

Strategy
✓ fair value
✓ microprice
✓ OFI
✓ fill probability
✓ markout
✓ inventory
✓ quote decisions

Execution
✓ New
✓ Cancel
✓ Replace
✓ Modify
✓ IDs
✓ order lifecycle

Risk
✓ fat finger
✓ quantity
✓ position
✓ outstanding exposure
✓ pending exposure
✓ throttles

Order Connectivity
✓ OUCH
✓ iLink/FIXP
✓ session sequence
✓ ACK/fill/reject

Production Safety
✓ kill switch
✓ mass cancel
✓ drop copy
✓ reconciliation
✓ deterministic replay
✓ recovery
✓ audit.
```

At this point the architecture loop is **complete**.

The next useful chapter is no longer another major missing HFT box. It should be **Chapter 8 — Putting Everything Together: Designing Our Own Production-Grade HFT Platform**, where we choose an exact hardware/software topology and turn Chapters 1–7 into one concrete system: CPU/core allocation, processes, memory ownership, NIC queues, C++ modules, thread model, IPC, startup/recovery sequence, latency budget, testing stack, observability and an implementation roadmap from a laptop simulator to a kernel-bypass multi-core production prototype.

[1]: https://www.sec.gov/rules-regulations/staff-guidance/trading-markets-frequently-asked-questions/divisionsmarketregfaq-0?utm_source=chatgpt.com "SEC.gov | Responses to Frequently Asked Questions Concerning Risk Management Controls for Brokers or Dealers with Market Access"
[2]: https://www.cmegroup.com/solutions/market-access/globex/trade-on-globex/pre-trade-risk-management.html?utm_source=chatgpt.com "Pre-Trade Risk Management - CME Group"
[3]: https://www.nasdaqtrader.com/content/technicalsupport/specifications/TradingProducts/OUCH5.0.pdf?utm_source=chatgpt.com "North American Trading Services"
[4]: https://www.cmegroup.com/tools-information/webhelp/cme-customer-center/Content/order-entry-create.html?utm_source=chatgpt.com "Creating iLink Order Entry Session"
[5]: https://www.cmegroup.com/notices/ebs/2021/08/20210805.html?utm_source=chatgpt.com "20210805 - CME Group"
[6]: https://www.nasdaqtrader.com/TraderNews.aspx?id=ETU2026-4&utm_source=chatgpt.com "Equity Technical Update #2026 - 4 Nasdaq Updates Specifications for Order-by-Order Self Match Prevention"
[7]: https://www.cmegroup.com/notices/electronic-trading/2021/01/20210118.html?utm_source=chatgpt.com "20210118 - CME Group"
[8]: https://www.cmegroup.com/tools-information/webhelp/autocert-ilink-3/Content/ACPiLink3UserManual.pdf?utm_source=chatgpt.com "AutoCert+ iLink"
