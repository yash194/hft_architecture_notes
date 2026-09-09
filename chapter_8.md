# Chapter 8 — Designing the Complete Production-Grade HFT Platform

We now have all of the individual pieces:

```text
Chapter 1  → low-latency design philosophy
Chapter 2  → CPU/cache/memory/concurrency
Chapter 3  → network/NIC/exchange path
Chapter 4  → market microstructure + matching
Chapter 5  → feed handler + production order book
Chapter 6  → strategy + queue/fill/markout
Chapter 7  → order gateway + risk + session state
```

Chapter 8 answers the systems question:

> **If we had to build one complete HFT-like production platform from scratch, exactly how should we assemble all of those pieces into machines, CPU cores, processes, memory regions, NIC queues and failure domains?**

This is the architecture I would use as the reference design for the system we eventually implement.

---

# Part I — What “production grade” means

A low-latency prototype can be:

```text
UDP socket
   ↓
order book
   ↓
strategy
   ↓
send()
```

A **production-grade** HFT platform must simultaneously achieve:

```text
LOW LATENCY
+
LOW JITTER
+
CORRECTNESS
+
DETERMINISM
+
RISK SAFETY
+
RECOVERY
+
OBSERVABILITY
+
HIGH AVAILABILITY
+
AUDITABILITY
+
DEPLOYABILITY
```

Those goals sometimes conflict.

For example:

```text
more replication
    ↑
more safety

but also potentially
    ↑
more synchronization
    ↑
more latency
```

The central architectural principle is therefore:

> **Keep safety and redundancy, but remove them from the critical path whenever they do not need to be synchronous.**

---

# Part II — The three-plane architecture

Before discussing CPUs, divide the entire platform into three planes.

## 1. Hot data plane

This is where nanoseconds and microseconds matter.

```text
Market Packet
     ↓
Feed Handler
     ↓
Book
     ↓
Features
     ↓
Strategy
     ↓
Risk
     ↓
Order Manager
     ↓
Order Encoder
     ↓
NIC
```

Properties:

```text
pinned CPUs
bounded memory
no blocking
no database
no disk
no RPC
no text logging
no dynamic allocation
minimal cross-core communication
```

---

## 2. Warm control plane

This manages trading but does not process every tick.

```text
parameter updates
risk-budget updates
strategy enable/disable
session management
recovery
kill switches
health decisions
operator commands
```

Milliseconds are often fine here.

What matters is:

```text
correct
reliable
auditable.
```

---

## 3. Cold research/operations plane

This includes:

```text
historical storage
backtesting
model training
dashboards
P&L reports
databases
Kafka
data warehouse
Python research
ML training
compliance analytics
long-term logs
```

These systems can be large distributed systems.

They simply must not sit here:

```text
Strategy
   ↓
   X
database
   X
   ↓
Order
```

---

# 2. The complete platform

The high-level deployment should look approximately like this:

```text
                           EXCHANGE
             ┌────────────────────────────────┐
             │                                │
      Market Data A                    Market Data B
             │                                │
             │                                │
      physical path A                  physical path B
             │                                │
             ▼                                ▼
        ┌─────────┐                      ┌─────────┐
        │ NIC A   │                      │ NIC B   │
        └────┬────┘                      └────┬────┘
             │                                │
             └───────────────┬────────────────┘
                             ▼
                ┌────────────────────────┐
                │   HOT TRADING ENGINE   │
                │                        │
                │ A/B arbitration        │
                │ protocol decode        │
                │ order books            │
                │ features               │
                │ strategy               │
                │ local risk             │
                │ order manager          │
                │ order session          │
                └───────────┬────────────┘
                            │
                            ▼
                         NIC TX
                            │
                            ▼
                     Exchange Gateway
                            │
                            ▼
                     Matching Engine
                            │
                            │
                       ACK / FILL
                            │
                            ▼
                   HOT TRADING ENGINE
                            │
             ┌──────────────┼───────────────┐
             │              │               │
             ▼              ▼               ▼
         Drop Copy      Flight Recorder   Metrics
             │              │               │
             └──────────────┼───────────────┘
                            ▼
                     CONTROL PLANE
                            │
                            ▼
                     COLD PLATFORM
```

---

# Part III — First major design choice: one giant process or many services?

Do **not** build the hot system like conventional microservices:

```text
Feed Service
   ↓ network RPC
Book Service
   ↓ network RPC
Feature Service
   ↓ network RPC
Strategy Service
   ↓ network RPC
Risk Service
   ↓ network RPC
Order Service
```

Architecturally neat.

Terrible for HFT.

Every arrow can introduce:

```text
serialization
queueing
kernel work
context switches
cache movement
scheduling
tail latency.
```

---

# 3. Our reference architecture: trading shards

Instead, create a number of **independent trading shards**.

Each shard owns:

```text
market-data channel(s)
instrument set
local order books
features
strategy state
risk budget
order session
TX queue
```

Conceptually:

```text
             SHARD 0

Market Feed A ─┐
               │
Market Feed B ─┤
               ▼
        A/B Arbitration
               ↓
             ITCH
               ↓
             BOOK
               ↓
           FEATURES
               ↓
           STRATEGY
               ↓
             RISK
               ↓
        ORDER MANAGER
               ↓
         OUCH / iLink
               ↓
             TX
```

Ideally all of this executes on:

# one CPU core.

That gives us an extraordinarily important property:

```text
one shard
=
one owner
=
one event loop
=
no locks.
```

---

# 4. Why this architecture is so powerful

Imagine:

```text
book
strategy
position
order state
risk
```

are all owned by one thread.

Then they can use:

```cpp
uint64_t
int32_t
normal arrays
normal structs
```

instead of:

```cpp
std::atomic<>
mutex
seqlock
cross-core queues.
```

The processing order becomes deterministic:

```text
Market Event
     ↓
Book Update
     ↓
Feature Update
     ↓
Strategy
     ↓
Risk
     ↓
Order TX
```

Then:

```text
Fill Event
     ↓
Order State
     ↓
Position
     ↓
Risk
     ↓
Strategy
```

No concurrency ambiguity.

---

# 5. This is essentially an event reactor

The shard is conceptually:

```cpp
while (running) {

    poll_market_data();

    poll_order_responses();

    process_control_messages();

    maybe_flush_tx();
}
```

But each operation is non-blocking.

The thread never sleeps during trading.

This is a specialized:

# event reactor / run-to-completion engine.

Current DPDK documentation explicitly describes this run-to-completion architecture: one logical core polls RX descriptors, processes packets, and transmits through TX descriptors. It also recommends single-core ownership of individual RX queues and NUMA-local packet buffers. ([DPDK][1])

---

# Part IV — Physical hardware design

For our CPU implementation I would start with a machine conceptually like:

```text
                 TRADING SERVER

┌──────────────────────────────────────────────┐
│                                              │
│       High-clock x86 CPU                     │
│                                              │
│       16–32 physical cores                   │
│       ECC RAM                                │
│                                              │
│     ┌────────────┐    ┌────────────┐         │
│     │ Trading    │    │ Trading    │         │
│     │ NIC A      │    │ NIC B      │         │
│     └────────────┘    └────────────┘         │
│                                              │
│             NVMe storage                     │
│        (capture/audit only)                  │
│                                              │
│       Separate management NIC                │
│                                              │
└──────────────────────────────────────────────┘
```

---

# 6. Prefer one CPU socket if it has enough capacity

Suppose you have a two-socket system:

```text
Socket 0                Socket 1

CPU cores               CPU cores
L3                      L3
RAM                     RAM
NIC                     ...
   \                    /
    \___ interconnect _/
```

Now every remote access can involve:

```text
cross-socket fabric
```

and potentially higher latency/jitter.

If one socket has enough:

```text
cores
memory bandwidth
PCIe lanes
```

I would prefer keeping the entire latency-critical trading plant on one socket.

A second socket can be used for:

```text
capture
analytics
shadow validation
monitoring
```

if necessary.

---

# 7. NUMA rule

The most important relationship should be:

```text
NIC
 │
 │ PCIe
 ▼
NUMA NODE 0
 │
 ├── packet buffers
 ├── trading core
 ├── order book
 ├── strategy state
 └── TX buffers
```

Not:

```text
NIC
 ↓
NUMA 0

Strategy
 ↓
NUMA 1

Book
 ↓
NUMA 0
```

DPDK's current guidance explicitly recommends allocating buffers for a PCIe network interface from memory local to that processor/NUMA node and keeping packet processing local where possible. ([DPDK][1])

---

# Part V — NIC architecture

I would not let one physical NIC become the only network failure domain.

Reference topology:

```text
             Exchange

          ┌─────┴─────┐
          │           │
       Feed A       Feed B
          │           │
          ▼           ▼
        NIC A       NIC B
```

Ideally these represent genuinely independent:

```text
ports
switch paths
exchange feed paths
```

rather than:

```text
two cables
into the same switch
into the same NIC
```

which only looks redundant.

CME, for example, describes MDP as a dual-feed UDP multicast market-data architecture. ([CME Group][2])

Nasdaq's current TotalView-ITCH deployment similarly publishes sequenced feeds over MoldUDP64 and maintains feed/recovery infrastructure. ([NASDAQ Trader][3])

---

# 8. Separate management networking

Trading ports should carry:

```text
market data
order entry
exchange recovery
```

Management NIC:

```text
SSH
monitoring
deployment
operator UI
configuration
```

Do not let:

```text
Grafana scrape
package downloads
SSH transfer
```

compete with exchange traffic on the latency-sensitive NIC.

---

# 9. NIC queue mapping

Suppose Shard 0 trades one exchange channel.

NIC configuration:

```text
Market Feed A
      ↓
RX Queue A0
      ↓
Core 2

Market Feed B
      ↓
RX Queue B0
      ↓
Core 2

Order Response Session 0
      ↓
RX Queue OE0
      ↓
Core 2

Orders
      ↓
TX Queue 0
      ↓
Core 2
```

Everything relevant to that shard reaches:

```text
one core.
```

DPDK recommends assigning each receive queue to one logical core rather than sharing it among cores; sharing queues introduces synchronization and harms performance. ([DPDK][1])

---

# Part VI — Exact CPU core design

Let's assume:

```text
1 socket
16 physical cores
```

as an example.

I would start approximately like:

| Core | Responsibility                               |
| ---- | -------------------------------------------- |
| 0    | OS housekeeping, interrupts, SSH, management |
| 1    | PTP/time sync + control-plane services       |
| 2    | Trading Shard 0                              |
| 3    | Trading Shard 1                              |
| 4    | Trading Shard 2                              |
| 5    | Trading Shard 3                              |
| 6    | Recovery / snapshot / feed repair            |
| 7    | Drop-copy reconciler                         |
| 8    | Firm-risk supervisor                         |
| 9    | Raw market packet capture                    |
| 10   | Order/audit recorder                         |
| 11   | Shadow/reference book                        |
| 12   | Metrics exporter                             |
| 13   | Heavy cross-instrument model if needed       |
| 14   | Spare/failover workload                      |
| 15   | Spare/diagnostic workload                    |

This is illustrative, not a universal allocation.

The principle matters more:

```text
hot cores
=
isolated

housekeeping cores
=
everything noisy.
```

---

# 10. Keep interrupts off hot cores

The Linux kernel documentation specifically recommends isolating IRQs away from CPUs intended for isolated workloads, and Red Hat's real-time tuning guidance describes CPU isolation as moving user tasks, unbound kernel work and interrupts away from selected CPUs. ([Linux Kernel Documentation][4])

We therefore want approximately:

```text
Core 0/1:
scheduler noise
kernel workers
storage interrupts
management IRQs
network management IRQs

Core 2–5:
trading loops
```

---

# 11. `nohz_full`

For a trading core spending almost all its life in user-space polling, Linux's `nohz_full` mechanism can suppress scheduling ticks while one task occupies the CPU.

Linux explicitly notes that such isolated CPUs are intended for a single running userspace task and should avoid kernel entry, IRQs and other noise. ([Linux Kernel Documentation][4])

That maps nicely onto:

```text
one trading process
+
one dedicated core
+
kernel bypass
+
busy polling.
```

But this is a **benchmarking choice**, not magic. Full-dynticks increases some kernel-entry bookkeeping costs, so the correct kernel configuration must be measured for the particular workload.

---

# 12. SMT / Hyper-Threading

Suppose:

```text
Physical Core 2

Logical CPU 2
Logical CPU 18
```

share substantial core resources.

Putting:

```text
strategy
```

on one sibling and:

```text
logging
```

on the other can cause interference.

For the hottest cores I would initially:

```text
disable SMT
```

or leave the sibling unused.

Then benchmark whether enabling it provides any advantage.

---

# Part VII — CPU power management

Low-power servers frequently do:

```text
idle
 ↓
deep sleep
 ↓
packet arrives
 ↓
wake up
```

Great for power consumption.

Bad for deterministic microsecond reaction times.

For the hot cores I would test:

```text
restrict deep C-states
stable performance governor
stable CPU frequency
avoid unpredictable frequency transitions
```

The objective isn't necessarily:

```text
highest benchmark turbo.
```

It is:

# predictable clock behavior.

A slightly slower but stable core may be superior to one that occasionally incurs large latency excursions.

---

# Part VIII — Memory architecture

For each shard, allocate everything before trading begins:

```text
Packet buffers
Order table
Price levels
Feature state
Strategy state
Own-order pool
Risk state
TX buffers
Recovery buffers
Flight recorder
Metric structures
```

Then:

```text
touch every page
```

before activation.

---

# 13. Why prefaulting matters

You don't want:

```text
market packet
      ↓
strategy
      ↓
first access to memory page
      ↓
PAGE FAULT
      ↓
kernel
```

during trading.

So initialization should:

```text
allocate
prefault
touch
lock/pin where appropriate
```

before the market opens.

---

# 14. Huge pages

DPDK commonly uses huge-page-backed memory pools, and AF_XDP permits the application to choose how its UMEM backing memory is allocated, including huge pages. ([Linux Kernel Documentation][5])

Potential advantages include:

```text
fewer TLB entries
less page-table pressure
large contiguous DMA-capable memory.
```

But use huge pages because they solve measured translation/memory problems, not because:

```text
HFT always = huge pages everywhere.
```

---

# 15. Per-shard memory ownership

I want memory to look like:

```text
SHARD 0 MEMORY

RX packet pool
    │
Order book
    │
Features
    │
Strategy
    │
Own orders
    │
Risk
    │
TX packet pool
```

owned exclusively by:

```text
Core 2.
```

Shard 1 gets its own memory.

No global allocator.

No shared order table.

No central feature map.

---

# Part IX — Shared memory rules

Sometimes information must cross shards.

Examples:

```text
firm risk
cross-product price
strategy parameter update
kill switch
```

Do not respond by creating:

```text
GlobalSharedState {
    mutex everything;
}
```

Instead use one of three patterns.

---

# 16. Pattern 1 — immutable snapshot publication

Writer creates:

```text
Config B
```

while readers use:

```text
Config A.
```

Then atomically publish:

```text
active = B.
```

Excellent for:

```text
strategy parameters
limits
instrument configuration.
```

---

# 17. Pattern 2 — SPSC event queues

If one producer sends to one consumer:

```text
Risk supervisor
      ↓
   SPSC
      ↓
Trading shard
```

No multiwriter contention.

---

# 18. Pattern 3 — preallocated budgets

Instead of every strategy incrementing one global risk counter:

```text
Firm risk
    ↓
allocate budget
    ↓
Shard A = 2000
Shard B = 3000
Shard C = 5000
```

Each hot shard enforces its budget locally.

That removes global synchronization entirely from normal order entry.

---

# Part X — The complete hot event loop

This is the heart of our platform.

Conceptually:

```cpp
while (running) {

    poll_market_feed_A();
    poll_market_feed_B();

    arbitrate_sequences();

    for (market_event : ordered_events) {

        book.apply(market_event);

        auto change =
            features.update(market_event);

        if (strategy.is_relevant(change)) {

            auto desired =
                strategy.evaluate();

            auto action =
                order_manager.reconcile(desired);

            if (action &&
                risk.allow(action)) {

                order_manager.reserve(action);

                encoder.encode(action);

                tx.submit();
            }
        }
    }

    for (order_event : order_rx.poll()) {

        order_manager.apply(order_event);

        position.apply(order_event);

        risk.apply(order_event);

        strategy.on_execution(order_event);
    }
}
```

Obviously production code is more specialized.

Architecturally, though:

> **That is our HFT system.**

Everything else supports this loop.

---

# Part XI — Why market-data and private-order events belong together

Suppose:

```text
Market Event
    ↓
Strategy decides buy

then

Execution ACK arrives
    ↓
Position changes

then

next Market Event.
```

If order events are processed by a separate thread and position updates arrive late, the strategy may act with stale inventory.

Running both event classes under one deterministic owner gives:

```text
public market state
+
private order state
+
position
+
risk
```

one coherent timeline.

This is extremely valuable.

---

# Part XII — What if the order session cannot live on the same core?

Sometimes:

```text
many strategies
```

share:

```text
one exchange session.
```

Then use:

```text
Strategy Core 2 ──SPSC──┐
Strategy Core 3 ──SPSC──┤
Strategy Core 4 ──SPSC──┼──► Gateway Core
Strategy Core 5 ──SPSC──┘
```

The Gateway Core owns:

```text
session sequence
risk reservations
order IDs
protocol encoding
TX queue
private responses.
```

This costs a cache-line transfer.

But prevents session-state races.

---

# 19. Better alternative when venue/account structure allows it

Use independent order-entry sessions:

```text
Shard 0 → Session 0
Shard 1 → Session 1
Shard 2 → Session 2
Shard 3 → Session 3
```

Then:

```text
one shard
=
one session
=
one TX queue
=
one owner.
```

Venue rules, costs, risk configuration and session allocation determine whether this is feasible.

CME, for example, explicitly treats iLink sessions as distinct managed order-entry resources with their own session and risk-control configuration. ([CME Group][6])

---

# Part XIII — Network bypass choice

For our project, I would support one common internal packet API with interchangeable network backends.

```text
                  PacketSource

        ┌─────────────┼─────────────┐
        ▼             ▼             ▼
       UDP          AF_XDP         DPDK
      socket
        │             │             │
        └─────────────┼─────────────┘
                      ▼
                  PacketView
```

This lets the entire upper stack remain unchanged.

---

# 20. Stage 1 — normal sockets

Excellent for:

```text
correctness
multicast testing
protocol debugging.
```

Not your final latency target.

---

# 21. Stage 2 — AF_XDP

AF_XDP gives you:

```text
UMEM
RX ring
TX ring
FILL ring
COMPLETION ring.
```

Linux documents these as single-producer/single-consumer ownership rings, with descriptors referencing packet frames within UMEM. ([Linux Kernel Documentation][5])

This makes AF_XDP a very useful bridge between:

```text
ordinary sockets
```

and:

```text
full userspace poll-mode networking.
```

---

# 22. Stage 3 — DPDK / vendor low-level API

For the final CPU low-latency version I would use:

```text
DPDK
```

or a NIC-specific interface such as:

```text
ef_vi
```

depending on hardware.

DPDK's current PMD architecture directly polls RX/TX descriptors from userspace instead of relying on the ordinary interrupt-driven kernel path. ([DPDK][1])

---

# Part XIV — Clock and timestamp architecture

We need at least three clocks/timestamps conceptually.

```text
Exchange timestamp

NIC RX/TX hardware timestamp

CPU TSC.
```

---

# 23. PTP hardware clock

Choose NICs that expose a:

```text
PHC
=
PTP Hardware Clock.
```

Linux exposes hardware PTP clocks to user space and supports synchronizing the system with external clocks. ([Linux Kernel Archives][7])

Hardware packet timestamps can be reported close to RX/TX and are more useful than software timestamps for understanding wire-adjacent latency. ([Kernel.org][8])

---

# 24. Why we need both PTP and TSC

PTP gives us:

```text
cross-machine comparable time.
```

TSC gives us:

```text
cheap cycle-level measurement
inside one CPU.
```

Use:

```text
TSC
```

for hot profiling.

Use synchronized hardware time for:

```text
packet timelines
machine comparisons
postmortems
network latency
exchange timestamp correlation.
```

---

# Part XV — Observability architecture

A production HFT engine cannot become:

```text
fast but invisible.
```

We need extensive telemetry without making the hot path slow.

Architecture:

```text
HOT CORE
   │
   ├── counters
   ├── latency samples
   ├── flight recorder
   │
   ▼
SPSC / shared snapshot
   │
   ▼
MONITOR PROCESS
   │
   ├── metrics
   ├── dashboards
   ├── alerts
   └── persistent logs
```

---

# 25. Per-core metrics

Do not have:

```cpp
std::atomic<uint64_t>
global_packets;
```

incremented by all shards.

Each core gets:

```cpp
struct LocalMetrics {
    uint64_t packets;
    uint64_t gaps;
    uint64_t duplicates;

    uint64_t book_updates;

    uint64_t decisions;

    uint64_t orders;
    uint64_t cancels;
    uint64_t fills;

    ...
};
```

No atomics.

Monitor periodically reads snapshots.

---

# 26. Essential latency histograms

Track:

```text
NIC RX → packet loop

RX → book updated

RX → strategy decision

RX → NIC TX

strategy → NIC TX

order TX → ACK

cancel TX → cancel result

fill RX → position updated.
```

And report:

```text
p50
p90
p99
p99.9
max.
```

Average alone is almost useless.

---

# 27. Market freshness

Also measure:

```text
CurrentTime - PacketRxTime

CurrentSequence - ProcessedSequence

LastFeedEventAge.
```

A book can be:

```text
perfectly correct
```

but:

```text
100 µs behind.
```

For HFT this may be as dangerous as corruption.

---

# Part XVI — Flight recorder

Every shard should have a fixed circular event buffer.

Store compact events such as:

```text
RX sequence

BBO change

strategy fair value

desired quote

risk result

order sent

ACK

fill

cancel

health transition.
```

Example:

```cpp
struct FlightEvent {
    uint64_t tsc;
    uint64_t sequence;
    uint64_t value1;
    uint64_t value2;
    uint32_t code;
    uint16_t instrument;
};
```

No strings.

If something catastrophic happens:

```text
freeze ring
    ↓
persist off-path
    ↓
replay.
```

---

# Part XVII — Control plane

Operator actions should never manipulate trading structures directly.

Instead:

```text
Operator UI
     ↓
Control Service
     ↓
validated command
     ↓
Trading Engine
```

Commands such as:

```text
ENABLE STRATEGY

DISABLE STRATEGY

CANCEL ONLY

MASS CANCEL

UPDATE LIMIT

UPDATE PARAMETERS

KILL.
```

---

# 28. Configuration should be versioned

Never have:

```text
someone edits config.json
```

while the system is running.

Instead:

```text
StrategyConfig v142
      ↓
validated
      ↓
published
      ↓
Shard reports active v142.
```

Every trading decision can then be associated with:

```text
code version
config version
model version
risk version.
```

This is hugely useful for postmortems.

---

# 29. Parameter publication

Hot shard:

```text
reads immutable Parameters A.
```

Research/control creates:

```text
Parameters B.
```

When fully validated:

```text
atomic pointer swap
A → B.
```

No lock around every model inference.

---

# Part XVIII — Production health state

Do not model the whole trading platform with:

```cpp
bool healthy;
```

Use explicit states:

```text
OFFLINE

STARTING

SYNCHRONIZING

CANCEL_ONLY

ACTIVE

DEGRADED

RECOVERING

KILLED.
```

Then define legal transitions.

For example:

```text
STARTING
   ↓
SYNCHRONIZING
   ↓
CANCEL_ONLY
   ↓
ACTIVE.
```

Not:

```text
OFFLINE
   ↓
ACTIVE immediately.
```

---

# 30. Why `CANCEL_ONLY` matters

Suppose:

```text
market data invalid.
```

You don't want:

```text
new orders.
```

But you may desperately need:

```text
cancels.
```

So:

```text
Order Permission Matrix

              NEW   CANCEL   HEDGE

ACTIVE        YES    YES      YES

CANCEL_ONLY   NO     YES      maybe

KILLED        NO     controlled emergency only
```

Much better than simple ON/OFF.

---

# Part XIX — Startup procedure

A production trading engine should not start trading merely because:

```text
process launched successfully.
```

I would require something like:

```text
1  Verify binary/config/model versions

2  Check CPU affinity/isolation

3  Check memory/hugepage pools

4  Check NIC link status

5  Check PTP lock/clock health

6  Join Feed A

7  Join Feed B

8  Establish market-data sequence

9  Build/recover order books

10 Verify both feeds

11 Establish order-entry session

12 Recover private order state

13 Verify drop copy

14 Verify positions

15 Load firm/strategy risk budgets

16 Validate no unknown live orders

17 Warm instruction/data paths

18 Enter CANCEL_ONLY

19 Operator/automated safety checks

20 Enter ACTIVE.
```

Only step 20 permits new risk.

---

# Part XX — Warm-up

Modern CPUs and memory systems have state:

```text
instruction caches
data caches
TLBs
branch predictor
pages
NIC rings.
```

Before activation, you can replay representative synthetic data through:

```text
decoder
book
features
strategy
risk
encoder
```

without transmitting orders.

This:

```text
prefaults memory
warms code paths
validates pipeline.
```

Then reset relevant book/state and synchronize to live data before enabling.

---

# Part XXI — Market-data failure policy

## Feed A fails

```text
A = DOWN
B = healthy.
```

Result:

```text
continue using B
alert
mark redundancy degraded.
```

No need to stop immediately if B is complete and verified.

---

## Sequence missing from A but present on B

```text
A/B arbitrator repairs.
```

Normal operation.

---

## Both feeds miss sequence

```text
Book State → RECOVERING

Strategy → CANCEL_ONLY

Live packets continue buffering

Request retransmission
```

After gap repair:

```text
replay
validate
LIVE.
```

---

## Major corruption/large loss

```text
invalidate channel
     ↓
snapshot
     ↓
buffer live
     ↓
replay
     ↓
validate
     ↓
ACTIVE.
```

---

# Part XXII — Order-session failure policy

Suppose order connection dies.

Immediately:

```text
NEW disabled.
```

But keep:

```text
market data
position processing
drop copy
risk
recovery.
```

Potential exchange orders may still exist.

Do not guess.

---

# 31. Cancel-on-disconnect

Some venues provide exchange-side protection.

CME, for example, supports Cancel on Disconnect for Globex sessions, with configuration depending on session/venue; its current risk-control documentation describes resting orders being cancelled after qualifying disconnects. ([CME Group][9])

This is useful.

But never design:

```text
"socket died, therefore orders definitely gone."
```

Your venue adapter must know the exact configured semantics.

---

# Part XXIII — High availability

A truly production system should not depend on one server.

Reference:

```text
               EXCHANGE

             Feed A   Feed B
                │       │
       ┌────────┴───────┴────────┐
       │                         │
       ▼                         ▼
   HOST A                    HOST B

   ACTIVE                    STANDBY

   full books                full books
   strategy state            shadow strategy
   position                  drop-copy state
   order state               reconciliation state

       │
       ▼
   ORDER ENTRY

       Host B:
       NO ORDER TRANSMISSION
       until explicitly fenced/activated
```

---

# 32. The standby should already be hot

Bad standby:

```text
Host A dies
   ↓
boot Host B
   ↓
load software
   ↓
download market snapshot
   ↓
recover
```

Far too slow.

Better:

```text
Host B already running

receiving A/B feeds

reconstructing same books

processing same model

receiving drop-copy/reconciliation state.
```

It simply does not own the active order-entry authority.

---

# 33. Split brain is the nightmare

Suppose:

```text
Host A thinks Host B died.

Host B thinks Host A died.
```

Both become:

```text
ACTIVE.
```

Now both may send orders.

This can double exposure catastrophically.

Therefore:

> **Failover must use fencing, not only heartbeat absence.**

---

# 34. Fencing

Before Host B can trade, it must prove Host A cannot still send through the same logical trading authority.

Possible mechanisms depend on venue/infrastructure:

```text
session ownership
exchange disconnect
order-entry port fencing
external hardware/network fence
risk-controller lease
operator confirmation.
```

Principle:

```text
At most one writer
```

applies at the **host level** too.

---

# Part XXIV — Failover sequence

A conservative failover:

```text
1 Host A health lost

2 Firm risk disables further A authority

3 Confirm/fence old order session

4 Determine what live orders remain

5 Reconcile via private recovery/drop copy

6 Confirm Host B market book synchronized

7 Confirm positions

8 Establish Host B order session

9 Load remaining risk budget

10 Enter CANCEL_ONLY

11 clean/cancel unknown exposure

12 finally ACTIVE.
```

This is slower than blindly failing over in 1 ms.

It is also vastly safer.

---

# Part XXV — Drop-copy reconciliation

Run a separate component:

```text
PRIMARY ORDER SESSION
        │
        ▼
    Gateway State
        │
        │ compare
        ▼
      Reconciler
        ▲
        │
      DROP COPY
```

Verify:

```text
orders
fills
match IDs
positions
```

agree.

If not:

```text
TRADING → CANCEL_ONLY / KILL.
```

Never allow:

```text
"position discrepancy but strategy still seems fine"
```

to persist.

---

# Part XXVI — Firm risk supervisor

Each shard handles:

```text
fast local limits.
```

Separate core/process manages:

```text
aggregate position

gross exposure

cross-strategy risk

daily loss

firm message rates

strategy budgets.
```

Architecture:

```text
Shard 0 ──►
Shard 1 ──► Firm Risk
Shard 2 ──►
Shard 3 ──►
```

But normal orders do not synchronously ask it:

```text
"Can I send?"
```

Instead firm risk publishes budgets.

---

# 35. Example

Firm allows:

```text
+10,000 / -10,000
```

inventory.

Allocate:

```text
Shard 0 ±2000

Shard 1 ±2000

Shard 2 ±3000

Shard 3 ±3000.
```

Hot check:

```text
local only.
```

If firm exposure changes dramatically:

```text
Risk Supervisor
      ↓
new budgets
      ↓
shards.
```

---

# Part XXVII — Kill system

I would have at least four independent kill mechanisms.

```text
1 Strategy self-kill

2 Gateway kill

3 Firm risk kill

4 Exchange/clearing-side control
```

Any higher authority can disable a lower one.

Strategy itself cannot override them.

---

# 36. Kill sequence

```text
KILL

 ↓

Block NEW

 ↓

Block REPLACE increasing risk

 ↓

Prioritize CANCEL

 ↓

Mass cancel if available

 ↓

Continue processing ACK/FILL

 ↓

Reconcile

 ↓

Confirm exposure

 ↓

Remain disabled.
```

Do not:

```text
KILL
 ↓
exit(0)
```

because live exchange orders may remain.

---

# Part XXVIII — Storage architecture

Two kinds of data are important.

## Raw market capture

```text
raw Ethernet/UDP packets
hardware timestamps
feed ID
```

Used for:

```text
replay
research
debugging
compliance.
```

---

## Private trading event journal

```text
strategy decision
risk result
order sent
ACK
fill
cancel
reject
parameter version.
```

Used for:

```text
reconstruction
audit
P&L
postmortem.
```

Neither path should synchronously write disk on the hot core.

---

# 37. Capture pipeline

```text
HOT SHARD
   │
   │ compact SPSC
   ▼
CAPTURE CORE
   │
   ▼
large memory buffer
   │
   ▼
NVMe
```

If capture cannot keep up:

```text
metrics/alert
```

and policy determines whether trading can legally/safely continue.

---

# Part XXIX — Reference-process architecture

On each host:

```text
┌─────────────────────────────────────────┐
│              HOT PROCESSES              │
│                                         │
│ trader_shard_0                          │
│ trader_shard_1                          │
│ trader_shard_2                          │
│ trader_shard_3                          │
└─────────────────────────────────────────┘

┌─────────────────────────────────────────┐
│              WARM PROCESSES             │
│                                         │
│ recovery_service                        │
│ dropcopy_reconciler                     │
│ firm_risk                               │
│ control_agent                           │
│ time_sync                               │
└─────────────────────────────────────────┘

┌─────────────────────────────────────────┐
│              COLD PROCESSES             │
│                                         │
│ packet_recorder                         │
│ audit_recorder                          │
│ metrics_exporter                        │
│ shadow_book                             │
└─────────────────────────────────────────┘
```

---

# 38. Why separate processes for shards?

Advantages:

```text
failure isolation

separate address spaces

separate CPU affinity

separate memory pools

separate order sessions.
```

A crash in:

```text
Shard 0
```

does not necessarily kill:

```text
Shard 1.
```

The cost is shared cross-shard state.

That's why we design around independent ownership/budgets.

---

# Part XXX — Project C++ architecture

I would structure our code approximately as:

```text
hft/

├── common/
│   ├── types/
│   ├── fixed_point/
│   ├── endian/
│   ├── time/
│   └── memory/
│
├── net/
│   ├── packet_view/
│   ├── udp/
│   ├── af_xdp/
│   ├── dpdk/
│   └── replay/
│
├── marketdata/
│   ├── moldudp/
│   ├── itch/
│   ├── sbe/
│   ├── arbitrator/
│   └── recovery/
│
├── book/
│   ├── order_pool/
│   ├── order_index/
│   ├── price_levels/
│   └── book_builder/
│
├── features/
│   ├── bbo/
│   ├── microprice/
│   ├── ofi/
│   ├── volatility/
│   └── queue/
│
├── strategy/
│   ├── fair_value/
│   ├── fill_model/
│   ├── markout/
│   ├── inventory/
│   └── quote_engine/
│
├── risk/
│   ├── limits/
│   ├── exposure/
│   ├── throttle/
│   └── kill/
│
├── orders/
│   ├── state_machine/
│   ├── order_pool/
│   ├── reconcile/
│   └── position/
│
├── venue/
│   ├── nasdaq_ouch/
│   ├── cme_ilink/
│   └── simulator/
│
├── control/
│   ├── config/
│   ├── health/
│   └── commands/
│
├── telemetry/
│   ├── counters/
│   ├── histograms/
│   ├── flight_recorder/
│   └── audit/
│
├── sim/
│   ├── exchange/
│   ├── network/
│   ├── latency/
│   └── replay/
│
└── apps/
    ├── trader/
    ├── replay/
    ├── benchmark/
    └── simulator/
```

One shared codebase.

Live and replay use the same core logic.

---

# Part XXXI — No separate “backtest strategy”

This is very important.

Do not have:

```text
Python strategy implementation

and

unrelated C++ production implementation.
```

They eventually diverge.

Our architecture should instead allow:

```text
Historical Packet Source
             │
             ▼
       SAME Feed Handler
             │
             ▼
         SAME Book
             │
             ▼
      SAME Features
             │
             ▼
       SAME Strategy
             │
             ▼
         SAME Risk
             │
             ▼
      Simulated Exchange
```

versus production:

```text
Live NIC
   │
   ▼
SAME ENGINE
   │
   ▼
Real Exchange.
```

That's enormously powerful.

---

# Part XXXII — The deterministic simulator

The simulator becomes a first-class component, not a toy.

It needs:

```text
Exchange event queue

matching engine

market-data generator

order-entry gateway

latency model

network loss/reorder model

clock model

queue position

fees

order session.

```

Everything operates on deterministic virtual time.

---

# 39. Example simulated timeline

```text
1000 ns   Exchange market event

1300 ns   Market packet reaches us

1600 ns   Strategy decision

1800 ns   Order leaves NIC

2200 ns   Exchange receives order

2300 ns   Matching occurs

2500 ns   Private ACK created

2800 ns   ACK reaches us.
```

Instead of:

```text
sleep_for(300ns)
```

simulation uses:

```text
event timestamp = current + latency.
```

This lets us simulate billions of events without real-time waiting.

---

# Part XXXIII — Latency budget

We should define an engineering target **before optimizing**.

For our CPU prototype, an illustrative tick-to-order CPU-side **p99 budget** could be:

| Stage                      | Example p99 budget |
| -------------------------- | -----------------: |
| NIC/RX visibility          |             300 ns |
| transport + A/B sequencing |             200 ns |
| ITCH/SBE decode            |             150 ns |
| book + features            |             300 ns |
| strategy                   |             500 ns |
| risk + order state         |             200 ns |
| encode + TX submit         |             200 ns |
| engineering reserve        |             650 ns |
| **Total target**           |         **2.5 µs** |

These are **design targets for our project, not claims about a particular exchange or firm's latency**. Actual numbers vary enormously with NIC, CPU, protocol, feed rate, strategy complexity and measurement boundary.

The point is to have a budget.

---

# 40. Why have reserve?

If every component uses exactly its maximum:

```text
300
+200
+150
+300
+500
+200
+200
=
1850 ns
```

then one small regression breaks the system.

Keep:

```text
engineering margin.
```

And target:

```text
p99
```

not only mean.

---

# Part XXXIV — Latency SLOs

Define service objectives such as:

```text
RX → Book
p99 < X

RX → Strategy
p99 < Y

RX → TX
p99 < Z

Cancel Decision → TX
p99 < C

ACK → Position
p99 < P
```

And importantly:

```text
p99.9 / max
```

limits.

A regression becomes a release blocker.

---

# Part XXXV — Performance CI

Every code change should run:

```text
unit tests
protocol golden tests
state-machine tests
replay tests
benchmark tests.
```

For important modules:

```text
Book add p99

Book cancel p99

ITCH decode p99

Strategy p99

End-to-end replay p99.
```

If new commit adds:

```text
15% p99 latency
```

someone must explain why.

---

# Part XXXVI — Correctness testing pyramid

I would use six layers:

```text
1 Unit tests

2 Property/fuzz tests

3 Reference-vs-fast differential tests

4 Deterministic historical replay

5 Exchange simulator failure injection

6 Hardware-in-the-loop/network tests
```

---

# 41. Differential testing

Maintain:

```text
SimpleBook
```

using readable STL containers.

And:

```text
FastBook
```

using specialized arrays/pools.

Feed both:

```text
same 10 billion events.
```

Require:

```text
BBO same
levels same
orders same
sequence same.
```

This is how you safely optimize.

---

# 42. Failure injection

Simulator should deliberately cause:

```text
packet loss

A/B feed asymmetry

market-data gap

duplicate packet

out-of-order packet

order ACK loss

fill while cancel pending

fill while replace pending

order-session disconnect

reconnect

drop-copy discrepancy

PTP drift

risk breach

pool exhaustion

process failure.
```

Production-grade means you know what happens in each case.

---

# Part XXXVII — Soak testing

Don't only benchmark for:

```text
30 seconds.
```

Run:

```text
8 hours

24 hours

multi-day.
```

Look for:

```text
memory leaks

pool fragmentation

counter overflow

slow drift

rare latency spikes

clock issues

queue buildup.
```

Markets run for hours, not benchmark seconds.

---

# Part XXXVIII — Burst testing

Generate:

```text
normal market
 ↓
100× microburst
 ↓
normal.
```

Ask:

```text
Did RX ring overflow?

Did sequence lag explode?

Did strategy become stale?

How quickly did we recover backlog?

Did p99.9 stay acceptable?
```

A feed handler designed for average rate is not production grade.

Nasdaq's FPGA market-data offering specifically highlights unshaped burst delivery/deterministic handling as a differentiator from more buffered software dissemination. ([NASDAQ Trader][10])

---

# Part XXXIX — Deployment model

For the hot path I would begin with:

# bare metal Linux.

Not:

```text
Kubernetes pod
inside VM
inside cloud host.
```

because we want direct control over:

```text
CPU affinity
NUMA
NIC queues
PCIe
huge pages
interrupts
BIOS
frequency
PTP.
```

Containers can be engineered well, but they add another layer we don't currently need.

---

# 43. Reproducible machine configuration

Production deployment should version:

```text
BIOS config

kernel version

kernel boot parameters

NIC firmware

NIC driver

DPDK version

compiler

CPU microcode

application binary

strategy config.
```

Because:

```text
"same source code"
```

does not necessarily mean:

```text
"same latency behavior."
```

---

# Part XL — Binary deployment

Release artifact:

```text
Trader build 1.47.2

git commit:
abc...

compiler:
...

model:
fairvalue-219

config:
prod-2026-09-09-v18

SHA:
...
```

Never:

```text
ssh server
git pull
cmake
make
./trader
```

in a serious production environment.

Build once.

Test artifact.

Deploy identical artifact.

---

# Part XLI — Shadow mode

Before enabling real order flow:

```text
live market data
     ↓
production strategy
     ↓
real decisions
     ↓
DO NOT TRANSMIT
```

Record:

```text
what orders would have been sent.
```

Compare against:

```text
simulator
previous release
risk expectations.
```

Then graduate:

```text
SHADOW
  ↓
tiny risk
  ↓
limited instruments
  ↓
full rollout.
```

---

# Part XLII — Software version rollback

Suppose version B has:

```text
excellent average latency
```

but unexpected:

```text
20 µs p99.99 spikes.
```

Rollback must be simple.

Keep:

```text
version A package
config
model
```

ready.

But stateful order-session transitions mean rollback is not simply:

```text
kill B
start A.
```

You must:

```text
disable
reconcile orders
transfer/recover session
start A
resynchronize
activate.
```

---

# Part XLIII — Security architecture

Trading network should not be an ordinary corporate server.

Separate:

```text
Exchange Network
Management Network
Data/Research Network.
```

Hot machine should have:

```text
minimal services
minimal open ports
strict operator access
locked-down privileges
versioned deployment
credential management.
```

No:

```text
Chrome
email client
random development tools
public internet traffic.
```

The objective is both security and deterministic operation.

---

# Part XLIV — Hardware acceleration path

Once CPU architecture is correct, we can selectively move the simplest urgent operations into FPGA/SmartNIC hardware.

For example:

```text
                  CPU

complex pricing
inventory
volatility
cross-product model
strategy parameters
       │
       │ rules
       ▼
      FPGA
       │
Market │
Data ──┼─► parse
       │   simple book
       │   trigger check
       │   risk gate
       │   cancel/order
       ▼
    Exchange
```

![Image](https://images.openai.com/static-rsc-4/8NMT9JW006GzUZUjuSwYmZ5UDtzHITt2OrP5TC-UkKhbb5eZkxVlZU1UEsp5awSphv4cUPedINekgcY0ajMIpm3wRaj9gbWDKPrzUp1OrbCu1sl_iWwSkud3l13jud-O_TF23cT3o9i9fr4xvw5xoOBlgxmwwMbn7Vi812nD_5veHRQPzdfc5pWKHxM1svbB?purpose=fullsize)

![Image](https://static.wixstatic.com/media/0cb9bd_104beb4ddff34e73b06b76295f07cd42~mv2.jpg/v1/fill/w_1000%2Ch_579%2Cal_c%2Cq_85%2Cusm_0.66_1.00_0.01/0cb9bd_104beb4ddff34e73b06b76295f07cd42~mv2.jpg)

Those vendor diagrams are useful illustrations of the general CPU+FPGA split, not endorsements of a specific implementation.

---

# 44. What belongs in FPGA first?

Do not begin by moving:

```text
entire sophisticated strategy
```

into hardware.

Start with highly deterministic operations such as:

```text
packet filtering

feed decoding

A/B arbitration

top-of-book reconstruction

simple stale-quote triggers

hard risk bounds

preloaded cancel messages.
```

Complex model state remains in CPU software.

---

# 45. Nasdaq illustrates this direction

Nasdaq now offers FPGA-based OUCH gateway technology on its order-entry infrastructure, and its FPGA TotalView-ITCH feed retains ITCH/MoldUDP64 semantics while emphasizing deterministic handling of burst traffic. ([NASDAQ Trader][11])

That doesn't mean:

```text
FPGA always wins.
```

It demonstrates that hardware acceleration fits naturally at precisely the network/protocol boundary we have been designing around.

---

# Part XLV — The full active/standby deployment

Now put everything together.

```text
                              EXCHANGE

                    Feed A             Feed B
                       │                  │
             ┌─────────┴──────────────────┴─────────┐
             │                                      │
             ▼                                      ▼
     ┌─────────────────┐                   ┌─────────────────┐
     │     HOST A      │                   │     HOST B      │
     │     ACTIVE      │                   │    STANDBY      │
     │                 │                   │                 │
     │ NIC A + NIC B   │                   │ NIC A + NIC B   │
     │                 │                   │                 │
     │ Shard 0         │                   │ Shadow Shard 0  │
     │ Shard 1         │                   │ Shadow Shard 1  │
     │ Shard 2         │                   │ Shadow Shard 2  │
     │ Shard 3         │                   │ Shadow Shard 3  │
     │                 │                   │                 │
     │ local risk      │                   │ shadow risk     │
     │ order session   │                   │ recovery state  │
     │                 │                   │                 │
     └────────┬────────┘                   └────────┬────────┘
              │                                     │
              │                              NO TX AUTHORITY
              │
              ▼
        ORDER ENTRY
              │
              ▼
       MATCHING ENGINE
              │
              ▼
         DROP COPY
              │
         ┌────┴────┐
         ▼         ▼
       Host A    Host B

             │
             ▼
      FIRM RISK / FENCER
```

That is the basic two-host production topology I would eventually target.

---

# Part XLVI — Failure-domain reasoning

Every component should answer:

> If this dies, what breaks?

For example:

| Failure                  | Expected behavior                     |
| ------------------------ | ------------------------------------- |
| Feed A                   | Continue B, degraded                  |
| Feed B                   | Continue A, degraded                  |
| Both feeds               | Cancel-only + recover                 |
| One strategy shard       | Disable its session/instruments       |
| Capture service          | Hot path continues temporarily, alarm |
| Firm risk                | No new risk / cancel-only             |
| Order session            | No new orders, reconcile              |
| PTP                      | Alarm/degrade depending policy        |
| NIC A                    | Feed B path survives                  |
| Host A                   | Fence + Host B recovery               |
| Book invariant violation | Kill affected shard                   |
| Drop-copy mismatch       | Firm-level halt/reconcile             |
| Pool exhaustion          | Fail closed                           |
| Excessive event age      | Stop submitting stale orders          |

Production architecture is largely the art of making this table unambiguous.

---

# Part XLVII — One subtle principle: degrade locally

Suppose:

```text
Shard 2
```

has corrupted state.

Don't automatically kill:

```text
entire global trading firm
```

if it is provably isolated.

Prefer:

```text
Shard 2 → CANCEL_ONLY
```

while:

```text
Shard 0
Shard 1
Shard 3
```

remain active.

But only if risk/ownership isolation is real.

This is another benefit of sharding.

---

# Part XLVIII — Another subtle principle: no silent fallback

Suppose huge-page pool runs out.

Bad:

```text
fallback to normal malloc.
```

Suppose DPDK fails.

Bad:

```text
fallback to normal kernel socket.
```

Suppose strategy config invalid.

Bad:

```text
use yesterday's automatically.
```

Those “helpful” fallbacks silently change latency/correctness assumptions.

A production HFT system should prefer:

# explicit failure.

```text
cannot maintain required invariant
    ↓
stop taking new risk.
```

---

# Part XLIX — Project build roadmap

Now we can turn the chapters into a real engineering project.

I would build it in this order:

### Milestone 1 — deterministic exchange simulator

Implement:

```text
matching engine
price-time FIFO
new/cancel/replace
ACK/fill
virtual clock.
```

No networking.

---

### Milestone 2 — historical market-data replay

Build:

```text
PacketSource
↓
MoldUDP
↓
ITCH
↓
Book.
```

Reference implementation first.

---

### Milestone 3 — optimized production book

Add:

```text
preallocated pools
flat order index
compact price levels
zero allocation.
```

Differential-test against reference.

---

### Milestone 4 — strategy engine

Implement first:

$$
FV =
Microprice + \beta OFI
$$

plus:

```text
inventory skew
queue model
fill model
markout.
```

---

### Milestone 5 — own-order gateway

Add:

```text
PendingNew
Live
PendingCancel
PendingReplace
Fill
Reject
recovery.
```

Connect strategy to simulated exchange.

At this point:

# full trading loop works offline.

---

### Milestone 6 — latency simulation

Add:

```text
market-data latency
strategy latency
order latency
ACK latency
jitter.
```

Now backtesting becomes HFT-like.

---

### Milestone 7 — normal UDP network

Run:

```text
real multicast/replayed multicast
```

over normal Linux sockets.

Verify protocol and operational behavior.

---

### Milestone 8 — flight recorder + observability

Before kernel bypass, build:

```text
metrics
latency histograms
health machine
packet capture
audit event stream.
```

Otherwise you will optimize a black box.

---

### Milestone 9 — AF_XDP

Replace socket backend.

Upper stack unchanged.

Benchmark.

---

### Milestone 10 — DPDK / low-level NIC path

Implement:

```text
RX queue
TX queue
NUMA-local pools
busy polling
hardware timestamps.
```

Now start serious nanosecond profiling.

---

### Milestone 11 — CPU/system tuning

Add:

```text
core isolation
IRQ affinity
NUMA binding
power-state tuning
huge pages
prefaulting
SMT policy
PTP.
```

Measure each change independently.

---

### Milestone 12 — multi-shard

Move from:

```text
one trading core
```

to:

```text
4 independent trading shards.
```

Measure LLC/memory/NIC interference.

---

### Milestone 13 — high availability

Add:

```text
Host A
Host B

shadow books

drop-copy reconciliation

fencing

failover.
```

---

### Milestone 14 — FPGA/SmartNIC experimentation

Only now.

At this stage we actually know:

```text
which 200 ns matter.
```

Then hardware optimization becomes rational instead of speculative.

---

# Part L — The minimum viable “production-like” system

You do **not** need two data-center racks and an FPGA to build a serious educational version.

The first credible version can run on one Linux server:

```text
┌─────────────────────────────┐
│ Linux Bare Metal            │
│                             │
│ Core 0 housekeeping         │
│                             │
│ Core 1 Feed+Book+Strategy   │
│        +Risk+Gateway        │
│                             │
│ Core 2 Simulator/Exchange   │
│                             │
│ Core 3 Recorder             │
│                             │
│ Core 4 Metrics              │
│                             │
└─────────────────────────────┘
```

With:

```text
DPDK/AF_XDP loopback or two NIC ports

binary market-data protocol

realistic matching engine

queue simulation

real order state machine

risk

hardware timestamps

replay.
```

That alone would be a very serious systems project.

---

# Part LI — What would make this project technically impressive?

Not merely:

```text
"we made an HFT simulator."
```

The interesting engineering claims would be things such as:

```text
Zero-allocation tick-to-order hot path

One-core deterministic run-to-completion architecture

MoldUDP/ITCH-like redundant feed arbitration

MBO order-book reconstruction

Queue-aware fill simulation

Real own-order asynchronous state machine

Latency-aware strategy

Pre-trade worst-case exposure accounting

DPDK/AF_XDP network backend

Hardware-timestamped latency distributions

Deterministic packet replay

Automatic feed/session recovery

Dual-host active/standby design

Continuous p99/p99.9 regression testing.
```

That is a strong systems project even independently of whether the strategy makes money.

---

# Part LII — The deepest architectural idea

You have now seen the same design principle recur at almost every level:

```text
NIC Queue
    ↓
one owner

Order Book
    ↓
one owner

Strategy State
    ↓
one owner

Order Session
    ↓
one owner

Risk Budget
    ↓
one owner

Trading Shard
    ↓
one owner

Active Host
    ↓
one owner.
```

Whenever possible:

> **Partition ownership instead of synchronizing mutation.**

That eliminates:

```text
locks
atomics
coherence
race conditions
recovery ambiguity
```

simultaneously.

That is perhaps the single most transferable design lesson across the entire HFT architecture.

---

# Part LIII — The second deepest idea

Don't:

```text
OPTIMIZE WORK.
```

until you have asked:

```text
CAN I REMOVE THE WORK?
```

Examples from our final design:

```text
Don't optimize mutex
→ single writer

Don't optimize allocator
→ preallocate

Don't optimize TCP between local components
→ same process

Don't optimize SPSC queue
→ same core

Don't optimize string lookup
→ numeric instrument ID

Don't optimize full book scan
→ incremental feature update

Don't optimize config lock
→ immutable snapshot

Don't optimize global risk atomic
→ preallocated risk budget

Don't optimize scheduler
→ keep scheduler off core.
```

That is the unifying principle behind the Gross talks and our entire platform.

---

# Part LIV — Final architecture to remember

If you retain only one diagram from all eight chapters, make it this:

```text
                         EXCHANGE
                            │
               ┌────────────┴────────────┐
               │                         │
            FEED A                    FEED B
               │                         │
               ▼                         ▼
             NIC A                     NIC B
               │                         │
               └────────────┬────────────┘
                            ▼
                   A/B ARBITRATION
                            │
                      Sequence Check
                            │
                            ▼
                     Protocol Decoder
                      ITCH / SBE
                            │
                            ▼
                      LOCAL ORDER BOOK
                            │
                            ▼
                     DERIVED FEATURES
                  BBO / OFI / Microprice
                            │
                            ▼
                       FAIR VALUE
                            │
                            ▼
                     QUEUE / FILL MODEL
                            │
                            ▼
                  MARKOUT / TOXICITY
                            │
                            ▼
                      INVENTORY SKEW
                            │
                            ▼
                       STRATEGY
                            │
                     Desired Quote
                            │
                            ▼
                      LOCAL RISK
                            │
                            ▼
                    ORDER STATE MACHINE
                            │
                            ▼
                     OUCH / iLink
                            │
                            ▼
                         NIC TX
                            │
                            ▼
                    EXCHANGE GATEWAY
                            │
                            ▼
                    MATCHING ENGINE
                            │
                       ACK / FILL
                            │
                            ▼
                       PRIVATE RX
                            │
                            ▼
                      ORDER STATE
                            │
                            ▼
                        POSITION
                            │
                            └──────────────► STRATEGY
```

Under the entire structure:

```text
CPU isolation
NUMA placement
cache locality
preallocated memory
kernel bypass
PTP
hardware timestamps
flight recorder
p99 telemetry
deterministic replay
risk
kill switch
recovery
active/standby fencing.
```

And beside it:

```text
RESEARCH / CONTROL / STORAGE
```

but never inside the tick-to-order path.

---

## Where we are after Chapter 8

At this point, we have enough architecture knowledge to stop adding conceptual boxes and start **engineering the actual platform**.

The most useful next step is therefore not “Chapter 9: another HFT concept.” It should be a **concrete system specification** for the project: exact requirements, latency SLOs, C++ interfaces, data structures, processes, CPU mapping, simulator contract, module-by-module implementation order, and acceptance tests.

From that specification we can start implementing **Milestone 1 — the deterministic exchange simulator + matching engine**, and progressively replace simulated components until the same engine runs through a real kernel-bypass networking path.

[1]: https://doc.dpdk.org/guides/prog_guide/ethdev/ethdev.html?utm_source=chatgpt.com "3.1. Poll Mode Driver — Data Plane Development Kit 26.07.0 documentation"
[2]: https://www.cmegroup.com/market-data/distributor/market-data-platform.html?redirect=%2Fbandwidthstats&utm_source=chatgpt.com "Data Directly from the Market Data Platform - CME Group"
[3]: https://www.nasdaqtrader.com/Trader.aspx?id=FeedMIPs&utm_source=chatgpt.com "UDP/IP Addresses - U.S. Equity Market Data Feeds"
[4]: https://docs.kernel.org/admin-guide/cpu-isolation.html?utm_source=chatgpt.com "CPU Isolation — The Linux Kernel documentation"
[5]: https://docs.kernel.org/networking/af_xdp.html?utm_source=chatgpt.com "AF_XDP — The Linux Kernel documentation"
[6]: https://www.cmegroup.com/globex/files/iLinkSessionIDPolicy.pdf?utm_source=chatgpt.com "CME Globex iLink Session Guidelines"
[7]: https://cdn.kernel.org/doc/html/latest/driver-api/ptp.html?utm_source=chatgpt.com "PTP hardware clock infrastructure for Linux — The Linux Kernel documentation"
[8]: https://kernel.org/doc/html/latest/networking/timestamping.html?utm_source=chatgpt.com "Timestamping — The Linux Kernel documentation"
[9]: https://www.cmegroup.com/solutions/market-access/globex/trade-on-globex/pre-trade-risk-management.html?redirect=%2Fglobex%2Ftrading-cme-group-products%2Frisk-management-tools.html&utm_source=chatgpt.com "Pre-Trade Risk Management - CME Group"
[10]: https://classic.nasdaqtrader.com/content/ProductsServices/DataProducts/TotalView/FPGAITCHFAQ.pdf?utm_source=chatgpt.com "INVESTMENT INTELLIGENCE"
[11]: https://www.nasdaqtrader.com/Trader.aspx?id=ouch&utm_source=chatgpt.com "OUCH"
