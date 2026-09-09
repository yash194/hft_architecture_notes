Yes. This talk is especially useful when read **before/alongside the 2024 “When Nanoseconds Matter” talk**, because the two focus on different layers.

The 2024 talk asks: **“How do I make specific hot-path components absurdly fast?”**

This 2022 talk asks the more fundamental question:

> **“How should I design the entire software system so that low latency is even possible?”**

The talk is **David Gross, “Trading at Light Speed: Designing Low Latency Systems in C++,” Meeting C++ 2022**. Its official outline moves through latency requirements, data modelling, instrument stores, working-set-size analysis, seqlocks, SPMC queues, machine tuning, C/P states, shared LLC effects and production metrics. ([YouTube][1])

[Trading at light speed: designing low latency systems in C++ — David Gross](https://www.youtube.com/watch?v=8uAW5FQtcvE&utm_source=chatgpt.com)

What follows is not just a transcript summary. It is a **complete set of study notes**, with additional explanations where Gross assumes knowledge and a separate **Beyond the Talk** section filling in the architecture pieces you need for HFT.

---

# Trading at Light Speed

## Designing Low-Latency Trading Systems in C++

### David Gross — Optiver — Meeting C++ 2022

---

# PART I — WHAT THE TALK IS REALLY ABOUT

## 1. Low latency is a design property, not an optimization property

This is the central idea of the entire presentation.

A common software-development approach is:

```text
Build application
      ↓
make it correct
      ↓
deploy
      ↓
profile
      ↓
find slow functions
      ↓
optimize them
```

That works for many applications.

It is often fundamentally inadequate for ultra-low-latency systems.

Why?

Because by the time you profile the application, you may already have chosen:

```text
wrong process architecture
wrong ownership model
wrong concurrency model
wrong data structures
wrong memory representation
wrong networking mechanism
wrong CPU topology
wrong APIs
wrong latency targets
```

You cannot fix those with:

```cpp
-O3
```

or:

```cpp
inline
```

or a faster loop.

Gross's explicit position is that performance must be designed into the system from the beginning rather than added afterwards. 

The deeper distinction is:

```text
PERFORMANCE STRATEGY
        ≠
PERFORMANCE TACTICS
```

We'll return to that.

---

# 2. Why trading systems need low latency

Gross starts from automated trading.

Imagine you are providing liquidity:

```text
             ORDER BOOK

Buy                     Sell

99.00                    101.00
98.50                    101.50
```

You currently have:

```text
Buy order @ 99
```

Then new information enters the market suggesting the asset is now worth:

```text
95
```

Your order at:

```text
99
```

is suddenly dangerous.

Someone faster than you can sell to you at 99 when the thing may already be worth 95.

So:

```text
Market changes
      ↓
you learn about change
      ↓
you calculate reaction
      ↓
you cancel/change order
```

Every delay in that chain increases your exposure to stale pricing.

The talk uses this basic market-making situation to explain why response latency has direct economic consequences. ([Electronic Trading Hub][2])

---

# 3. Latency is not the only requirement

A trading system cannot simply be:

```text
FAST
```

It must be:

```text
FAST
+
CORRECT
+
ROBUST
+
SCALABLE
+
MAINTAINABLE
```

And it may need to run many sophisticated strategies.

This becomes important when Gross discusses FPGAs.

Hardware can perform extremely simple reactions at extraordinary speed.

Software can implement much richer calculations.

So modern low-latency systems aren't necessarily trying to move **everything** onto the fastest possible hardware.

Rather:

```text
simple critical reactions
        ↓
FPGA / hardware

complex strategy/model logic
        ↓
C++
```

The talk notes that extremely competitive trigger-to-target responses can reach tens-of-nanoseconds territory through hardware-based paths, while software remains essential for complex strategies and system coordination. 

---

# 4. The most important performance metric: trigger-to-target

Gross introduces a useful way of defining trading-system speed:

# Trigger-to-target latency

Think:

```text
EXCHANGE
   │
   │ sends market event
   ▼
TRIGGER
   │
   │ market participant reacts
   │
   ▼
TARGET
   │
   │ sends order
   ▼
EXCHANGE
```

Therefore:

$$
T_{trigger\rightarrow target}
=
T_{reaction}
$$

It captures something close to:

> **How long does the system take to observe meaningful market information and produce a response?**

The important thing is that you define latency around a **business event**, not around some arbitrary function.

Gross uses trigger-to-target as a core notion for discussing low-latency systems. ([Electronic Trading Hub][2])

---

# 5. Latency should be expressed as a distribution

Suppose someone tells you:

```text
Our system latency = 2 μs
```

That isn't enough.

You should ask:

```text
mean?
median?
p90?
p99?
p99.9?
maximum?
under what load?
```

Consider:

### System A

```text
median      1.5 μs
p99         1.7 μs
p99.9       1.9 μs
```

### System B

```text
median      1.2 μs
p99         4 μs
p99.9      20 μs
```

System B has a better median.

But A may be much more useful for trading.

Why?

Because trading often cares about:

```text
predictability
+
tail latency
```

as much as raw average speed.

This is why Gross frames performance requirements using both event throughput and response-latency distributions rather than vague requirements such as “make it fast.” ([HFT Tech Advisory][3])

---

# PART II — PERFORMANCE STRATEGY VS PERFORMANCE TACTICS

# 6. “Premature optimization is the root of all evil” is often misunderstood

People sometimes use Knuth's famous statement as justification for:

```text
Ignore performance while designing.
We'll fix it later.
```

Gross argues that this confuses two different things.

### Tactical optimization

Examples:

```text
replace division with multiplication
force inline a function
use branch hint
write SIMD
rearrange one loop
```

Doing these before identifying a performance problem can indeed be premature optimization.

But:

### Strategic performance design

is something entirely different.

For example:

```text
We must process 1,000,000 events/sec.

p99 response must remain below X.

No allocation is allowed on critical path.

Strategy thread owns its state.

Market-data path stays on one NUMA node.

Working data must remain cache resident.
```

These decisions determine what architecture you build.

Gross explicitly distinguishes performance **strategy**—the system-level design needed to satisfy latency/throughput requirements—from implementation-level **tactics**. ([Electronic Trading Hub][2])

---

# 7. Example: architectural performance vs micro-optimization

Imagine two designs.

### Design A

```text
Market data
    ↓
network socket
    ↓
shared thread pool
    ↓
JSON serialization
    ↓
message broker
    ↓
strategy service
    ↓
database lookup
    ↓
order service
```

Then somebody starts optimizing:

```cpp
inline calculate_price()
```

This is almost meaningless.

The architecture already contains huge latency.

Instead:

### Design B

```text
NIC
 ↓
feed handler
 ↓
cache-resident state
 ↓
strategy
 ↓
risk
 ↓
order encode
 ↓
NIC
```

Now micro-optimizations actually matter.

This leads to a useful rule:

> First remove architectural latency. Then optimize computational latency.

---

# PART III — HOW FAST IS “FAST”?

# 8. Latency has changed historically

Gross shows that financial-market response times have steadily fallen.

Conceptually:

```text
human traders
    ↓
milliseconds
    ↓
microseconds
    ↓
hundreds of nanoseconds
    ↓
tens of nanoseconds for specialized paths
```

That does **not** mean every trading strategy must run in 20 ns.

Instead there is a spectrum:

```text
                      flexibility ↑

complex C++ strategy
       │
optimized C++
       │
specialized software
       │
FPGA logic
       │
specialized hardware

                      latency ↓
```

The faster you go, the more functionality you generally have to sacrifice or specialize.

---

# 9. Why C++ remains important when FPGA exists

This is crucial.

You might think:

> “If FPGA can react in tens of nanoseconds, why are Optiver engineers discussing C++?”

Because FPGA logic cannot conveniently replace every aspect of a complex trading strategy.

Imagine calculating something involving:

```text
100 instruments
+
risk state
+
volatility
+
positions
+
option Greeks
+
correlations
+
model parameters
+
market conditions
```

Software is much easier to:

```text
change
test
deploy
debug
model
extend
```

So an HFT stack frequently becomes:

```text
                    Market
                      │
                      ▼
               ┌────────────┐
               │    FPGA    │
               └─────┬──────┘
                     │
        extremely simple reaction path
                     │
                     ▼
                   Market


Meanwhile

               C++ Strategy
                    │
          sophisticated models
                    │
                    ▼
          configure/update state
                    │
                    ▼
                  FPGA
```

This hybrid idea also appears more explicitly in Gross's later 2024 presentation.

---

# PART IV — WHY A NORMAL PROFILER CAN FAIL YOU

# 10. The “underwhelming profiling result”

Around the next stage of the talk, Gross profiles a representative application.

One might expect:

```text
Function A: 50%
Function B: 20%
Function C: 10%
```

and then simply optimize Function A.

Instead the profile is not particularly helpful.

There may be no gigantic hotspot.

Why?

Because performance can be poor due to the **data model itself**.

Imagine:

```text
function A touches random memory
function B touches random memory
function C touches random memory
function D touches random memory
```

Every individual function is merely:

```text
“a little slow”
```

because the fundamental problem is:

```text
your working data does not fit efficiently into the cache hierarchy.
```

This is the bridge into perhaps the most important part of the 2022 talk:

# Data modelling for performance.

The official talk outline places “An Underwhelming Profiling Result” immediately before “Data Model for Performance” and “Instrument Store,” reflecting this shift from code hotspots to memory behaviour. ([YouTube][1])

---

# PART V — DATA MODEL FOR PERFORMANCE

# 11. CPU performance is mostly about feeding the execution units

A simplified modern CPU looks like:

```text
               CPU CORE

                registers
                    │
                   L1
                    │
                   L2
                    │
                shared L3
                    │
                  DRAM
```

Very approximately:

```text
closer to CPU
      ↓
faster
smaller
more expensive

farther away
      ↓
slower
larger
cheaper
```

The exact latency numbers depend heavily on processor generation, but the hierarchy is the important point.

A program whose important data resides in L1/L2 behaves very differently from one repeatedly fetching from DRAM.

---

# 12. Why object-oriented design can accidentally fight the CPU

Suppose you model financial instruments using:

```cpp
class Instrument {
    MarketData* marketData;
    RiskData* riskData;
    Model* model;
    Statistics* statistics;
};
```

The object itself may be tiny.

But accessing it becomes:

```text
Instrument
   │
   ├────► MarketData somewhere
   │
   ├────► RiskData somewhere else
   │
   ├────► Model somewhere else
   │
   └────► Statistics somewhere else
```

Memory accesses become:

```text
RAM address A
RAM address Z
RAM address Q
RAM address M
```

The CPU cannot exploit locality effectively.

This is pointer chasing.

---

# 13. Data locality

Compare:

```text
node-based structure

[A] ─────► [B] ─────► [C] ─────► [D]

random addresses
```

with:

```text
contiguous structure

[A][B][C][D][E][F][G]
```

The second representation helps because CPUs fetch data in **cache lines**, not byte-by-byte.

If the cache line contains:

```text
[A][B][C][D]
```

and you access A, B, C and D soon afterwards, a single cache-line fetch provides significant useful work.

This is why:

```cpp
std::vector
```

frequently appears in low-latency code.

---

# 14. The Instrument Store

Gross uses an **Instrument Store** as the central example.

A trading strategy needs fast access to state related to instruments:

```text
Instrument ID
   │
   ├── price
   ├── market state
   ├── risk information
   ├── configuration
   └── strategy state
```

The critical design goals are:

```text
fast lookup
+
compact objects
+
good locality
+
stable references where needed
```

A write-up of the talk notes that the per-instrument core data discussed is deliberately compact—under roughly 1 KB—because size and locality are central to the design. ([Free Packets Blog][4])

---

# 15. Why `std::unordered_map` can be surprisingly expensive

Suppose you use:

```cpp
std::unordered_map<InstrumentId, Instrument>
```

On paper:

```text
lookup = O(1)
```

Excellent.

But what does the CPU see?

Potentially:

```text
hash computation
      ↓
bucket access
      ↓
pointer
      ↓
node
      ↓
Instrument object
```

and a large amount of metadata.

So the theoretical:

```text
O(1)
```

hides:

```text
cache misses
pointer chasing
large working set
extra allocations
TLB pressure
```

Contemporary summaries of Gross's talk specifically highlight avoiding large node-heavy containers and preferring vector-like storage for compact hot data. ([GitHub][5])

---

# 16. But `std::vector` has another problem: pointer stability

Suppose:

```cpp
std::vector<Instrument> instruments;
```

and another structure stores:

```cpp
Instrument* ptr;
```

Then the vector grows beyond capacity.

It reallocates:

```text
OLD:

[A][B][C][D]
 ^
 pointer


NEW:

                     [A][B][C][D][E][...]
```

The original pointer may become invalid.

For trading systems, many components may retain references to instrument state.

Therefore we want contradictory properties:

```text
contiguous-ish memory
+
stable addresses
```

---

# 17. Stable-vector/chunked storage idea

Gross presents a stable-vector-style structure built from bounded/chunked storage rather than using one endlessly reallocating dynamic vector. Discussion around the talk notes a design based on fixed-capacity chunks (`boost::container::static_vector` appears in the explanation), giving a compromise between pointer stability and locality. ([Ruclips][6])

Conceptually:

```text
Chunk 1:
[A][B][C][D][E]

Chunk 2:
[F][G][H][I][J]

Chunk 3:
[K][L][M][N][O]
```

You get:

```text
better locality than individual heap nodes
+
stable locations once chunks exist
+
controlled allocation pattern
```

This is a classic low-latency compromise.

---

# 18. Why chunk size matters

Suppose each chunk contains:

```text
4 objects
```

You may create lots of chunks.

That creates:

```text
more metadata
more pointers
less locality
```

Suppose each chunk contains:

```text
1,000,000 objects
```

Now you may waste enormous amounts of memory.

So chunk size is a performance parameter.

The important point isn't:

> “Always use chunk size 64.”

It is:

> **Choose representation from the actual workload and memory hierarchy.**

This is data-oriented design.

---

# PART VI — WORKING SET SIZE

# 19. What is Working Set Size?

This is one of the most useful concepts in the talk.

Suppose a process allocates:

```text
20 GB
```

But during the critical trading loop, it repeatedly touches only:

```text
4 MB
```

Then the important number is not necessarily:

```text
20 GB
```

It is closer to:

```text
4 MB working set
```

Brendan Gregg defines Working Set Size as the memory an application actually needs/touches to keep doing its work, distinguishing it from virtual size and resident set size. ([Brendan Gregg][7])

---

# 20. Why WSS matters enormously for HFT

Suppose your CPU has an effective cache footprint sufficient for your critical set.

### Strategy A

```text
hot working set = 1 MB
```

may mostly operate from fast caches.

### Strategy B

```text
hot working set = 200 MB
```

constantly evicts itself.

Then:

```text
same C++ code
same CPU
same compiler
same algorithmic complexity
```

can produce wildly different latency.

So sometimes you should not ask:

> Which function is slow?

Ask:

> **How much memory must the processor keep hot to execute this strategy?**

---

# 21. Why a profiler might show no hotspot

Imagine:

```text
100 different functions
```

and every function touches a different random object.

No individual function dominates.

But collectively:

```text
L3 misses
+
DRAM accesses
+
TLB misses
```

slow down everything.

So:

```text
bad data architecture
```

can masquerade as:

```text
lots of slightly slow code.
```

That is the deeper meaning of the “underwhelming profiling result” section.

---

# 22. WSS estimation

Gross points to tools for working-set analysis. Brendan Gregg's `wss` tooling uses Linux page-reference/idle-page mechanisms to estimate how much memory is actually touched over an interval. The tools themselves warn that measurement can perturb latency, so they are more appropriate for controlled analysis than blindly running against an ultra-sensitive production path. ([GitHub][8])

For example:

```text
Process RSS = 4 GB

Observed during 100 ms:
Referenced = 12 MB
```

Then approximately:

```text
active WSS ≈ 12 MB
```

rather than:

```text
4 GB
```

---

# PART VII — CONCURRENCY IN TRADING SYSTEMS

# 23. The next architectural problem: multiple cores

A real trading machine has many components:

```text
market-data handler
strategy
pricing
risk
order gateway
logging
monitoring
recovery
```

You could run everything on one core.

But then:

```text
feed parsing
   ↓
strategy
   ↓
risk
   ↓
order encoding
   ↓
logging
```

all compete sequentially.

Instead we use multiple cores/processes.

Now the problem becomes:

# How do those components share information without destroying latency?

The talk divides concurrent information conceptually into **state** and **events**, then treats them differently. ([Podwise][9])

---

# 24. State vs event

This distinction is extremely important.

## State

State means:

> “What is the latest value?”

Examples:

```text
current position
latest theoretical price
current volatility
latest risk limit
latest market midpoint
```

If the value changes:

```text
100 → 101 → 102
```

and a consumer sees only:

```text
102
```

that may be fine.

---

## Event

An event means:

> “Something happened, and I may need every occurrence.”

Examples:

```text
Trade #1
Trade #2
Trade #3

Order Ack #1
Order Ack #2
```

Losing:

```text
Trade #2
```

can change semantics.

Therefore:

```text
STATE
```

and:

```text
EVENT STREAM
```

should not automatically use the same communication primitive.

This is a powerful architecture lesson well beyond HFT.

---

# PART VIII — SEQLOCK FOR SHARING STATE

# 25. The basic state-sharing problem

Suppose:

```cpp
struct MarketState {
    double bid;
    double ask;
    double volatility;
};
```

Writer updates:

```text
bid = 100
ask = 101
vol = 0.2
```

Reader wants a consistent snapshot.

But reader might observe:

```text
new bid
old ask
new volatility
```

because the writer is in the middle of the update.

One solution:

```cpp
std::mutex
```

But then readers and writers can block one another.

For a heavily read, lightly written tiny object, Gross presents a **seqlock-style mechanism**.

---

# 26. How a sequence lock works

Add:

```text
version
+
data
```

Suppose version starts:

```text
42
```

even.

Writer begins:

```text
42 → 43
```

Odd means:

```text
WRITE IN PROGRESS
```

Writer modifies data.

Then:

```text
43 → 44
```

Even means:

```text
WRITE COMPLETE
```

So:

```text
       Writer

version = 42

version = 43   ← odd
write data
version = 44   ← even
```

Linux's sequence-counter documentation describes exactly this principle: readers accept a snapshot only when the sequence value is even and unchanged before/after the copy; writers increment at both sides of the update. ([Linux Kernel Archives][10])

---

# 27. Reader algorithm

Reader conceptually does:

```text
v1 = version

if odd:
    retry

copy data

v2 = version

if v1 != v2:
    retry

otherwise:
    snapshot is valid
```

Example:

```text
version = 42

Reader:
v1 = 42

copy bid
copy ask

writer begins
version = 43

reader copies volatility

v2 = 43

42 != 43
        ↓
discard
        ↓
retry
```

The reader never needed to block the writer.

---

# 28. Why seqlocks fit some trading state

Suppose:

```text
1 writer
50 readers
```

and updates are small.

A mutex requires significant coordination.

Seqlock-style operation makes reads optimistic:

```text
Assume state is stable.
Copy it.
Verify afterwards.
```

If writers are relatively rare, most reads succeed immediately.

Gross uses this as a low-latency state-sharing pattern. Summaries of the talk reproduce the single-writer sequence/version design and its optimistic reader retry logic. ([Tencent Cloud Developer Community][11])

---

# 29. But seqlocks have important limitations

The Linux documentation emphasizes several caveats.

They work best when:

```text
writes are short
writers are controlled
readers can retry
data can safely be copied
```

A writer that gets interrupted while the sequence is odd can make readers spin repeatedly. Sequence counters also require careful writer serialization and are unsuitable for arbitrary pointer-rich structures where objects can disappear while a reader follows them. ([Linux Kernel Archives][10])

This matters tremendously in C++.

A slide-sized pseudocode version is educational.

A production implementation must correctly address:

```text
memory ordering
compiler reordering
CPU ordering
object lifetime
alignment
atomic semantics
```

We'll return to this in **Beyond the Talk**.

---

# PART IX — SHARING EVENTS

# 30. State sharing is solved. What about events?

For state:

```text
latest value wins
```

works.

For events:

```text
Event 1
Event 2
Event 3
```

the consumer usually needs:

```text
all three
```

Therefore a queue/ring buffer is more suitable.

---

# 31. SPSC queue

Start with the simplest:

# Single Producer Single Consumer

```text
Producer
   │
   ▼
┌─────────────────────────┐
│ A │ B │ C │ D │   │   │
└─────────────────────────┘
                  │
                  ▼
               Consumer
```

Typical metadata:

```text
write index
read index
```

Producer owns:

```text
write index
```

Consumer owns:

```text
read index
```

This ownership structure is excellent because there is little coordination.

The talk uses this as the conceptual starting point before moving to single-producer/multiple-consumer fan-out. ([Tencent Cloud Developer Community][11])

---

# 32. Why fan-out is common in trading

A market-data feed may need to go to:

```text
Strategy A
Strategy B
Strategy C
Risk Engine
Recorder
```

Conceptually:

```text
              Producer
                  │
                  ▼
            shared events
          ┌───────┼───────┐
          ▼       ▼       ▼
        A         B       C
```

Each consumer should receive the stream independently.

This is:

# SPMC

## Single Producer, Multiple Consumers

---

# 33. Why a shared consumer read index is undesirable

Suppose all consumers coordinate around:

```text
readIndex
```

Then:

```text
Consumer A
Consumer B
Consumer C
```

all modify or observe common metadata.

This creates:

```text
atomic contention
cache-line movement
coordination
```

But in a broadcast system the producer doesn't necessarily need every reader's exact position on every operation.

Each consumer can maintain:

```text
its own position.
```

That greatly simplifies the architecture.

---

# 34. SPMC Queue V1

Gross's first version uses a structure roughly based on:

```text
pending index
published index
ring data
```

Producer:

```text
reserve region
      ↓
write message
      ↓
publish region
```

So:

```text
published region       currently writing

[ A ][ B ][ C ][ D ][ ????? ]
                  ▲       ▲
                  │       │
               index   pending
```

Readers only consume data known to be complete.

Summaries preserving Gross's pseudocode show two atomic indices in this first SPMC design: one representing the current published position and another representing pending writer progress. ([Tencent Cloud Developer Community][11])

---

# 35. Why V1 isn't good enough

There is still a central problem:

```text
many consumers repeatedly read
the same producer-owned metadata
```

The producer repeatedly modifies those shared cache lines.

That means cache-coherence traffic.

Imagine:

```text
Core 1 ──┐
Core 2 ──┼── shared atomic cache line
Core 3 ──┤
Core 4 ──┘
```

The line continually participates in the cache-coherence protocol.

That can become more expensive than the queue's actual useful work.

So Gross changes the design.

---

# PART X — SPMC QUEUE V2

# 36. Move synchronization closer to the data

Instead of coordinating everything around a few global indices, V2 splits the ring into **blocks**.

Each block has approximately:

```text
version
size
data
```

while a smaller global header tracks block progression.

A talk summary reproduces the structure conceptually as:

```cpp
Block {
    atomic version;
    atomic size;
    data...
}

Header {
    atomic blockCounter;
    Block blocks[];
}
```

([Tencent Cloud Developer Community][11])

---

# 37. Each block behaves somewhat like a seqlock

Producer chooses a block.

Then conceptually:

```text
version odd
      ↓
write payload
      ↓
version even
      ↓
publish advancement
```

Consumer checks the version.

This is similar to the earlier state-sharing idea.

Instead of making readers contend against one global state word for every message, coordination is distributed among blocks.

---

# 38. Why this helps cache coherence

Compare:

### V1

```text
Every message
   ↓
touch same global atomic
   ↓
many readers read same atomic
   ↓
cache line moves constantly
```

### V2

```text
Message 1 → Block 1 metadata
Message 2 → Block 2 metadata
Message 3 → Block 3 metadata
...
```

Synchronization traffic becomes more distributed.

This is a recurring performance technique:

> **Remove unnecessary shared writable state.**

---

# 39. False sharing

Suppose:

```text
Cache line = 64 bytes

[ProducerCounter][ConsumerCounter]
```

Producer updates the first variable.

Consumer updates the second.

Logically:

```text
independent variables
```

Physically:

```text
same cache line
```

The hardware coherence protocol operates on the **cache line**, not your C++ variable.

So the entire line bounces between cores.

That's:

# False sharing.

This is why low-latency structures often use:

```cpp
alignas(64)
```

or padding around hot independent atomics.

The code examples associated with Gross's SPSC/SPMC discussion use cache-line alignment around these shared counters for exactly this reason. ([GitHub][5])

---

# 40. But alignment isn't magic

Don't conclude:

```cpp
alignas(64)
```

every object.

That can produce:

```text
8 useful bytes
+
56 wasted bytes
```

which reduces cache density.

The later 2024 Gross talk goes even deeper into this.

Use cache-line separation for:

```text
independently modified shared variables
```

not automatically for every object.

---

# PART XI — THE BIG CONCURRENCY LESSON

# 41. The fastest synchronization is synchronization you architect away

Typical question:

> “What is the fastest lock?”

Better question:

> **“Can I redesign ownership so I don't need a lock?”**

Instead of:

```text
4 threads
   ↓
modify same object
   ↓
complex atomic protocol
```

use:

```text
1 owner
   ↓
publishes snapshots/events
   ↓
readers
```

This is one of the most important patterns in low-latency architecture.

You will repeatedly encounter:

```text
SPSC queues
SPMC fan-out
single writer state
core ownership
partitioned data
```

because:

```text
less sharing
=
less coordination
=
less cache coherence
=
more predictable latency
```

---

# PART XII — MACHINE TUNING

# 42. Fast code on a badly configured machine is still slow

Around 47 minutes Gross switches from application architecture to machine configuration.

This is another key insight:

```text
Application performance
≠
code performance alone
```

It also depends on:

```text
scheduler
CPU affinity
power management
NUMA placement
networking
cache sharing
page configuration
interrupts
```

The official outline explicitly includes system tuning, C/P states and shared LLC optimization. ([YouTube][1])

---

# 43. CPU isolation

Imagine your strategy runs on:

```text
Core 7
```

Then Linux schedules:

```text
monitoring daemon
```

onto Core 7.

Now:

```text
strategy
  ↓
context switch
  ↓
daemon
  ↓
context switch
  ↓
strategy resumes
```

Problems:

```text
latency spike
cache pollution
TLB effects
branch predictor effects
```

Low-latency machines often isolate selected cores for critical application threads.

Red Hat's real-time documentation describes CPU isolation as removing ordinary userspace threads, unbound kernel threads and interrupts from selected CPUs, while considering physical topology and NUMA placement. ([Red Hat Documentation][12])

---

# 44. CPU affinity

You also usually don't want:

```text
Strategy thread:

Core 4
 ↓
Core 12
 ↓
Core 7
 ↓
Core 19
```

because migration loses warm private-core state.

Instead:

```text
Feed handler → Core 4

Strategy     → Core 5

Order sender → Core 6
```

with deliberate affinity.

---

# 45. IRQ isolation

Even if your application has a dedicated core, hardware interrupts may still run there.

For example:

```text
Strategy on Core 5
       ↓
NIC interrupt arrives
       ↓
kernel interrupt handler
       ↓
strategy interrupted
```

So low-latency deployments often arrange:

```text
housekeeping / IRQ cores

separate from

critical trading cores
```

This is part of complete CPU isolation. ([Red Hat Documentation][12])

---

# PART XIII — C-STATES

# 46. What is a C-state?

CPU cores consume power.

When idle, they can enter progressively deeper sleep states.

Conceptually:

```text
C0 = running

C1
 ↓
C2
 ↓
C3
 ↓
...
deeper sleep
```

Deeper states generally save more power.

But:

```text
deeper sleep
       ↓
longer wake-up
```

Suppose your strategy thread waits.

CPU sleeps.

Market packet arrives.

Now:

```text
packet
 ↓
wake CPU
 ↓
restore execution state
 ↓
run strategy
```

The wake-up latency becomes part of your response time.

That's bad for deterministic HFT.

---

# 47. Why low-latency systems burn power

A normal server tries to optimize:

```text
performance per watt
```

An HFT machine may care more about:

```text
predictable wake-up latency
```

So you may intentionally:

```text
keep cores active
busy poll
limit deep C-states
```

and willingly consume electricity to remove unpredictable wakeup delay.

This is a fundamental philosophical difference from ordinary cloud computing.

---

# PART XIV — P-STATES

# 48. What is a P-state?

P-states relate more to processor performance/frequency levels.

Conceptually:

```text
low frequency
   ↕
medium frequency
   ↕
high frequency
```

Changing processor frequency is excellent for power efficiency.

But for latency measurements:

```text
variable frequency
      ↓
variable cycle duration / throughput behaviour
      ↓
variable latency
```

So low-latency machines often manage frequency behaviour carefully.

The goal isn't necessarily:

```text
highest advertised turbo
```

but often:

```text
stable predictable performance
```

---

# PART XV — SHARED LAST-LEVEL CACHE

# 49. L1/L2 may be private, LLC is shared

On many CPUs:

```text
Core 0:
L1
L2

Core 1:
L1
L2

Core 2:
L1
L2

        ↓

      Shared
       LLC
```

The last-level cache is commonly shared among cores. Intel documentation, for example, describes its Smart Cache/LLC as shared between processor cores on applicable architectures. ([Intel Caching Service][13])

This has a huge implication.

---

# 50. Your strategy does not own the LLC

Imagine:

```text
Strategy Core
   │
   └── uses 5 MB hot data

Analytics Core
   │
   └── scans 50 MB repeatedly
```

Both share LLC.

Analytics can evict strategy data.

Then:

```text
strategy working set
        ↓
evicted
        ↓
DRAM accesses
        ↓
latency rises
```

even though:

```text
analytics and strategy
never share a C++ variable.
```

This is **hardware-level interference**.

Gross explicitly includes shared-LLC optimization as a system-tuning topic. ([Podwise][9])

---

# 51. This changes the meaning of “isolated core”

A thread can be:

```text
scheduler isolated
```

but not:

```text
hardware isolated.
```

It may still share:

```text
LLC
memory controller
DRAM channels
interconnect
PCIe
```

with other cores.

Therefore:

> Core pinning is necessary but not sufficient.

This becomes an even larger theme in Gross's 2024 talk.

---

# PART XVI — NUMA

Gross only touches system-topology issues compared with how deep one can go, so this is a good place to expand.

# 52. NUMA architecture

A dual-socket server might be:

```text
             SOCKET 0

CPU cores ─── Local RAM
    │
    └──── NIC


             SOCKET 1

CPU cores ─── Local RAM
```

CPU 0 accessing:

```text
RAM attached to socket 0
```

is cheaper than repeatedly accessing remote memory attached to socket 1.

So:

```text
thread affinity
memory affinity
NIC affinity
```

should agree.

Bad:

```text
NIC on NUMA 0
Feed Handler on NUMA 1
Memory on NUMA 0
Strategy on NUMA 1
```

Good topology tries to minimize cross-node movement.

---

# PART XVII — METRICS

# 53. “Fast once” is not enough

Suppose version 1:

```text
p99 = 1.2 μs
```

Three months later:

```text
1.4 μs
```

Six months later:

```text
1.8 μs
```

One year later:

```text
2.5 μs
```

No single commit introduced a catastrophic regression.

Instead:

```text
30 ns
+ 20 ns
+ 50 ns
+ 40 ns
+ ...
```

gradually destroyed the system.

Gross therefore argues that performance measurement must be an ongoing production discipline, not something done once while developing. ([Podwise][9])

---

# 54. Measure the production system

Microbenchmarks are useful.

But:

```text
microbenchmark
```

doesn't include:

```text
actual market data
actual cache pressure
actual neighboring processes
actual NUMA traffic
actual queue load
actual network load
actual branch distributions
```

So:

```text
development benchmark
+
production measurement
```

are both necessary.

---

# 55. Timestamp important events

You can instrument a market event as:

```text
T0: packet reaches system

T1: feed decode complete

T2: market state updated

T3: strategy starts

T4: decision ready

T5: risk complete

T6: order encoded

T7: NIC transmit
```

Then:

$$
T_{decode}=T_1-T_0
$$

$$
T_{strategy}=T_4-T_3
$$

$$
T_{total}=T_7-T_0
$$

This gives you an actual **latency budget**.

The latter part of Gross's talk explicitly focuses on collecting timestamps/metrics and continually monitoring system performance. ([Ruclips][6])

---

# 56. Don't record only averages

Bad metric:

```text
average = 1.2 μs
```

Better:

```text
p50
p90
p99
p99.9
max
```

Even better:

```text
histogram
```

because latency distributions can have:

```text
multiple modes
long tails
outliers
load-dependent changes
```

For HFT:

```text
tail behaviour
```

may be more operationally meaningful than mean.

---

# 57. Record system state alongside latency

Suppose latency suddenly doubles.

You want to know:

```text
CPU frequency?
CPU temperature?
interrupt count?
context switches?
LLC misses?
page faults?
NUMA traffic?
queue occupancy?
packet rate?
working-set change?
```

So a real latency-monitoring system needs:

```text
application metrics
+
hardware metrics
+
OS metrics
```

not merely:

```text
function timings.
```

---

# PART XVIII — THE ARCHITECTURE IMPLIED BY THE TALK

If we combine the sections, we get something approximately like this:

```text
                         EXCHANGE
                            │
                            │ market data
                            ▼
                     ┌────────────┐
                     │    NIC     │
                     └─────┬──────┘
                           │
                           ▼
                   MARKET DATA CORE
                           │
                     parse messages
                           │
                     update market state
                           │
                           ▼
                  ┌─────────────────┐
                  │ INSTRUMENT STORE│
                  │                 │
                  │ compact         │
                  │ cache-friendly  │
                  │ stable storage  │
                  └───────┬─────────┘
                          │
             ┌────────────┴─────────────┐
             │                          │
         STATE DATA                   EVENTS
             │                          │
         Seqlock                    SPMC Queue
             │                          │
      ┌──────┼──────┐            ┌──────┼──────┐
      ▼      ▼      ▼            ▼      ▼      ▼
 Strategy  Risk   Model        Algo A  Algo B Recorder
      │
      ▼
 decision
      │
      ▼
    risk
      │
      ▼
 order encode
      │
      ▼
     NIC
      │
      ▼
   EXCHANGE
```

Underneath all of this:

```text
CPU pinning
C-state configuration
P-state configuration
NUMA placement
LLC considerations
working-set control
continuous metrics
```

That is the actual system-design message of the talk.

---

# PART XIX — A VERY IMPORTANT CONNECTION BETWEEN WSS AND CONCURRENCY

One of the deeper connections Gross doesn't spell out at great length is:

```text
more threads
≠
automatically more performance
```

Imagine one strategy has:

```text
4 MB working set
```

One instance:

```text
4 MB
```

Four instances:

```text
16 MB
```

Eight:

```text
32 MB
```

If your effective LLC capacity is limited, eventually:

```text
WSS_total > useful LLC
```

Then performance may suddenly collapse.

So scalability might look:

```text
performance

│
│             /
│           /
│         /
│       /
│     /
│   /
│ /
│
│          ← cache capacity exceeded
│            performance stops scaling
└──────────────────────── cores
```

This is why **working-set size and CPU-core count must be considered together**.

---

# PART XX — WHAT THE TALK ASSUMES BUT DOESN'T EXPLAIN ENOUGH

Now we move into **Beyond the Talk**.

These aren't necessarily claims made explicitly by Gross in this 2022 presentation. They are the concepts you need in order to understand the architecture properly.

---

# 58. TLB — Translation Lookaside Buffer

Your program accesses:

```text
virtual address
```

but RAM uses:

```text
physical address.
```

The CPU must translate:

```text
virtual → physical
```

Page tables perform this mapping.

But walking page tables frequently would be expensive.

So CPUs cache translations in the:

# TLB

Conceptually:

```text
Virtual Address
      ↓
     TLB
      ↓
Physical Address
```

If translation is missing:

```text
TLB miss
   ↓
page-table walk
   ↓
extra memory operations
```

Therefore huge working sets can hurt not only caches but also:

```text
TLB locality.
```

---

# 59. Huge pages

Normal pages are commonly:

```text
4 KB
```

Suppose your application actively uses:

```text
1 GB
```

That's about:

```text
262,144 × 4 KB pages.
```

Using larger pages drastically reduces the number of translations required.

That can reduce:

```text
TLB pressure
```

for certain workloads.

But huge pages are not a universal “speed switch.” They affect fragmentation, allocation and memory management and must be benchmarked.

---

# 60. Page faults

A devastating event in a hot path can be:

```text
access memory
    ↓
page not present/materialized
    ↓
page fault
    ↓
kernel involvement
```

Compared with nanosecond goals, this is enormous.

So low-latency applications often:

```text
allocate memory before trading
touch/pre-fault memory
lock critical memory where appropriate
avoid runtime allocation
```

---

# 61. Memory allocation

Consider:

```cpp
auto x = new Order();
```

That simple-looking call can involve:

```text
allocator metadata
free-list manipulation
locks
new pages
page faults
cache pollution
```

So the common HFT design is:

```text
STARTUP
   ↓
allocate required memory
   ↓
construct pools
   ↓
pre-touch
   ↓
TRADING STARTS
   ↓
reuse fixed memory
```

This is why bounded containers and ring buffers are so common.

---

# 62. Busy polling

Normal application:

```text
no work
 ↓
sleep
 ↓
scheduler wake-up
```

Low-latency application:

```cpp
while (running) {
    poll();
}
```

CPU appears:

```text
100% utilized
```

even though the thread is often merely waiting.

Why?

Because the design exchanges:

```text
power consumption
```

for:

```text
wake-up latency.
```

That connects directly with Gross's C-state discussion.

---

# 63. Kernel bypass

Normal network receive:

```text
NIC
 ↓
interrupt / driver
 ↓
kernel networking
 ↓
socket buffer
 ↓
syscall
 ↓
application
```

Ultra-low latency often tries:

```text
NIC
 ↓
userspace polling
 ↓
application
```

via technologies such as:

```text
Solarflare/OpenOnload/ef_vi
DPDK
vendor-specific userspace NIC APIs
```

This reduces:

```text
syscalls
scheduler interactions
copies
kernel-path variability
```

Gross's 2024 talk explores this more explicitly.

---

# 64. NIC DMA

The NIC doesn't normally ask the CPU to copy every incoming byte manually.

Instead it can use:

# DMA — Direct Memory Access

Conceptually:

```text
Ethernet
   ↓
NIC
   ↓
DMA
   ↓
RAM buffer
```

Then CPU processes that memory.

So physical location of:

```text
NIC
memory
CPU core
```

matters.

Especially with NUMA.

---

# 65. RX/TX rings

NICs often communicate through descriptor rings:

```text
RX ring

[descriptor][descriptor][descriptor][descriptor]
```

Each descriptor points to a packet buffer.

Application/driver processes entries sequentially.

This means even network processing becomes another:

```text
ring-buffer / ownership / cache-locality
```

problem.

The same design patterns keep reappearing.

---

# 66. Hardware timestamps and PTP

For serious latency measurement, ordinary application clocks may not be enough.

Trading systems may use:

```text
NIC hardware timestamping
PTP — Precision Time Protocol
exchange timestamps
TSC calibration
```

to distinguish:

```text
wire time
NIC time
software time
```

Without synchronized clocks you can easily misattribute latency.

---

# 67. `rdtsc` / TSC

For extremely short code regions, a common technique is to read the CPU Time Stamp Counter.

Conceptually:

```cpp
start = rdtsc();

operation();

end = rdtsc();
```

Then:

```text
cycles = end - start
```

But using TSC correctly requires thinking about:

```text
instruction ordering
serialization
core migration
frequency interpretation
measurement overhead
```

This becomes more prominent in Gross's 2024 talk.

---

# 68. Hardware performance counters

Timing tells you:

```text
Something got slower.
```

PMU counters help tell you:

```text
WHY.
```

Useful counters include:

```text
instructions retired
CPU cycles
IPC
branch misses
L1 misses
L2 misses
LLC misses
TLB misses
frontend stalls
backend stalls
```

Example:

```text
Before:

IPC = 1.8
LLC miss rate = 2%

After code change:

IPC = 0.7
LLC miss rate = 18%
```

Now you know where to investigate.

---

# 69. IPC

IPC =

$$
\frac{\text{instructions retired}}
{\text{CPU cycles}}
$$

Higher isn't always universally better, but collapsing IPC is often a useful clue.

For example:

```text
CPU waiting on DRAM
```

can produce very low IPC.

So an algorithm with fewer source-code operations can still be slower if the CPU spends most of its time stalled.

---

# 70. Branch prediction

Gross doesn't focus heavily on branch prediction in this talk; he goes much deeper in 2024.

Modern CPUs speculate:

```cpp
if (condition)
    A();
else
    B();
```

They predict which branch will execute.

Correct:

```text
excellent
```

Wrong:

```text
speculative work discarded
pipeline redirected
latency paid
```

This is why unpredictable binary search eventually becomes important in his 2024 order-book example.

---

# 71. Instruction cache

People focus heavily on data cache.

But machine instructions themselves also live in caches.

Huge hot-path code can cause:

```text
instruction-cache misses
```

So:

```text
tiny critical path
```

is desirable.

This is another reason:

```text
simple code
```

often performs disproportionately well.

---

# 72. Compiler visibility

C++ gives the compiler many opportunities to optimize when types and functions are statically visible.

This is why low-latency code often uses:

```text
templates
constexpr
static dispatch
small types
inlining where justified
```

rather than large runtime abstraction layers.

But:

```text
more templates
more inline
```

can increase code size.

Again:

```text
trade-off.
```

---

# 73. Serialization design

Suppose an internal event is:

```text
PriceUpdate
```

You could represent it using:

```text
JSON
```

But JSON brings:

```text
parsing
text conversion
variable-length fields
allocations
large message size
```

Low-latency systems frequently prefer compact binary schemas.

Examples:

```text
SBE
FlatBuffers
Cap'n Proto
custom fixed binary formats
```

depending on requirements.

---

# 74. Zero copy

You often hear:

```text
zero-copy
```

as if it were always ideal.

Not necessarily.

For tiny messages:

```text
memcpy 64 bytes
```

can be incredibly cheap.

Trying to eliminate that copy may introduce:

```text
shared ownership
pointers
lifetimes
reference counting
cache-line contention
```

which is worse.

Always compare:

```text
complete design cost
```

rather than counting copies.

---

# 75. Exchange market-data protocols

The talk assumes “market data” at a fairly high level.

In a real HFT system:

```text
exchange
   ↓
binary protocol
   ↓
feed handler
```

The feed handler must perform:

```text
packet receive
sequence check
decode
instrument mapping
book update
gap handling
normalization
publication
```

Examples of exchange protocol families include:

```text
ITCH
OUCH
SBE-based feeds
CME MDP
proprietary exchange protocols
```

depending on venue.

---

# 76. Packet sequence numbers

Imagine receiving:

```text
100
101
102
104
```

Where is:

```text
103?
```

You have a packet gap.

If you simply keep updating the book, your market state may become wrong.

So:

```text
sequence tracking
gap detection
recovery feeds
snapshots
```

are essential.

Fast but corrupted market state is useless.

---

# 77. Feed arbitration

Serious systems may receive equivalent market data through multiple network paths.

Example:

```text
Feed A ─┐
        ├──► Feed Arbitrator ─► Book
Feed B ─┘
```

If A's packet arrives first:

```text
use A
```

If B's arrives first:

```text
use B
```

If one feed loses a packet:

```text
other feed may fill gap
```

This can reduce both latency and failure risk.

---

# 78. Pre-trade risk

Between:

```text
strategy decision
```

and:

```text
order transmission
```

you still need risk controls.

Examples:

```text
maximum order size
maximum position
credit limits
price collars
duplicate order protections
kill switch
```

The challenge is:

```text
risk must be correct
+
risk must also be extremely fast.
```

So HFT risk checks are frequently:

```text
precomputed
local
cache resident
incremental
```

rather than database queries.

---

# 79. Order gateway

After strategy and risk:

```text
internal order
      ↓
exchange-specific order protocol
      ↓
encode
      ↓
send
```

The order gateway handles things such as:

```text
sequence numbers
sessions
exchange protocol
acknowledgements
rejections
recovery
```

This is another major architecture component mostly outside the scope of Gross's talk.

---

# PART XXI — C++ MEMORY MODEL WARNING

# 80. Do not copy seqlock pseudocode blindly

This is important.

The simple form:

```cpp
version++;
memcpy(data);
version++;
```

illustrates the algorithm.

It is **not automatically a correct portable C++ concurrent algorithm**.

You need to reason about:

```text
atomicity
acquire/release
compiler reordering
hardware reordering
data races
object lifetime
```

The Linux kernel operates under its own concurrency primitives and assumptions. ISO C++ has its own formal memory model.

Gross's later 2024 talk explicitly returns to language-level difficulties around seqlock-like algorithms.

So when building such a queue yourself:

```text
algorithm correctness
        +
hardware correctness
        +
C++ memory-model correctness
```

all matter.

---

# PART XXII — WHY SINGLE WRITER IS SUCH A POWERFUL PATTERN

Suppose you need shared positions.

### Approach A

```text
Strategy A ─┐
Strategy B ─┼── update position
Strategy C ─┘
```

Now you need:

```text
mutex/CAS
```

and cache-line bouncing.

Instead:

### Approach B

```text
A ─┐
B ─┼── events ─► Position Owner
C ─┘                │
                    ▼
                 snapshot
```

Now one thread owns mutation.

Readers consume published state.

You transformed:

```text
concurrency problem
```

into:

```text
message-passing problem.
```

This is an enormous architectural simplification.

---

# PART XXIII — PERFORMANCE BUDGETING

# 81. Design latency like a financial budget

Suppose your requirement is:

```text
Market-to-order p99 < 2 μs
```

Break it down:

```text
NIC / receive           250 ns

decode                  200 ns

book update             250 ns

strategy                500 ns

risk                    200 ns

order encode            200 ns

send                    250 ns

other                    50 ns
─────────────────────────────
total                  1.9 μs
```

Now every subsystem has a contract.

Instead of:

```text
“make strategy faster”
```

you can ask:

```text
Why did book-update increase
from 250 ns to 400 ns?
```

This is a far better engineering process.

---

# PART XXIV — throughput vs latency

These are not identical.

Suppose System A processes:

```text
10 million events/sec
```

in huge batches.

But each event waits:

```text
500 μs
```

before processing.

System A has:

```text
excellent throughput
bad latency.
```

System B processes:

```text
2 million events/sec
```

but each event begins immediately and finishes in:

```text
1 μs.
```

For HFT:

```text
System B may be preferable.
```

This is why batching must be treated carefully.

---

# 82. Batching trade-off

Batch size:

```text
1
```

means:

```text
low waiting latency
high per-message overhead
```

Batch size:

```text
1000
```

means:

```text
excellent amortization
but message #1 waits for others
```

So low-latency systems often use:

```text
small strategic batching
```

where it saves expensive operations without introducing meaningful queueing delay.

---

# PART XXV — QUEUEING LATENCY

Even a 10 ns queue can be disastrous if messages wait behind other messages.

Suppose processing time:

```text
100 ns/event
```

There are:

```text
50 events ahead of yours
```

Then:

$$
T_{queue}\approx50 \times 100ns
=5\mu s
$$

So:

```text
queue implementation latency = 10 ns
```

is almost irrelevant.

The actual problem is:

```text
queue depth.
```

Therefore you need metrics for:

```text
occupancy
backlog
producer-consumer lag
drop/overwrite events
```

not merely push/pop benchmark times.

---

# PART XXVI — THE REAL LATENCY EQUATION

A more complete model is:

$$
T_{total}
=
T_{wire}
+
T_{NIC}
+
T_{queue}
+
T_{decode}
+
T_{state}
+
T_{strategy}
+
T_{risk}
+
T_{encode}
+
T_{TX}
+
T_{jitter}
$$

But each term also depends on:

```text
cache state
branch predictor
CPU frequency
NUMA
interrupts
context switches
page state
coherence traffic
neighbor workloads
```

So the system is not truly:

```text
pipeline of independent constants.
```

It is a coupled hardware/software system.

That is perhaps the deepest idea tying this talk to Gross's 2024 presentation.

---

# PART XXVII — WHAT TO TAKE FROM THE SPMC/SEQLOCK SECTION

Do **not** memorize:

```text
SPMCQueueV2
```

and assume:

> “I should implement this exact queue.”

The underlying lessons are much more useful:

```text
1. Distinguish state from events.

2. Give data one owner where possible.

3. Avoid blocking readers.

4. Minimize globally shared writable metadata.

5. Align hot atomics carefully.

6. Spread coherence traffic where possible.

7. Design structures around actual access patterns.

8. Keep messages bounded and compact.

9. Measure queue latency under real fan-out.

10. Don't ignore formal memory-model correctness.
```

That is what transfers to other systems.

---

# PART XXVIII — WHAT TO TAKE FROM THE DATA-MODEL SECTION

Again, don't memorize:

```text
stable vector good
unordered_map bad.
```

The actual lesson is:

```text
DATA REPRESENTATION
      ↓
determines
      ↓
CACHE BEHAVIOUR
      ↓
determines
      ↓
CPU STALLS
      ↓
determines
      ↓
LATENCY
```

Traditional algorithm thinking asks:

```text
How many operations?
```

Low-latency systems thinking additionally asks:

```text
Where are the bytes?
```

---

# PART XXIX — FULL DESIGN PHILOSOPHY FROM THE TALK

The entire talk can be compressed into this chain:

```text
Define actual latency requirement
              ↓
Design architecture around it
              ↓
Keep critical data compact
              ↓
Keep working set cache-friendly
              ↓
Prefer locality
              ↓
Separate state from events
              ↓
Use ownership to reduce synchronization
              ↓
Use specialized communication patterns
              ↓
Tune machine topology
              ↓
Control CPU power behaviour
              ↓
Understand shared cache interference
              ↓
Instrument production
              ↓
continuously detect regressions
```

That is the actual lesson.

---

# PART XXX — 2022 TALK VS THE 2024 TALK YOU ASKED ABOUT BEFORE

These two talks complement each other remarkably well.

| 2022 — Trading at Light Speed | 2024 — When Nanoseconds Matter        |
| ----------------------------- | ------------------------------------- |
| **System design**             | **Hot-path optimization**             |
| Define latency requirements   | Optimize actual latency distributions |
| Performance strategy          | Mechanical sympathy                   |
| Data model                    | Order-book representation             |
| Instrument store              | Vector vs tree                        |
| Working-set size              | Cache/prefetch behavior               |
| Seqlock                       | More advanced shared-memory work      |
| SPMC communication            | FastQueue design                      |
| C/P states                    | PMU profiling                         |
| Shared LLC                    | Multi-worker LLC interference         |
| Production metrics            | Continuous performance audits         |
| How to design fast            | How to squeeze out remaining ns       |

The 2022 talk tells you:

> **Build the right machine.**

The 2024 talk tells you:

> **Now make every part of that machine brutally efficient.**

---

# PART XXXI — COMBINE BOTH TALKS INTO ONE HFT MENTAL MODEL

After both talks, you should picture a low-latency trading box approximately like:

```text
                         EXCHANGE
                            │
                     multicast feed
                            │
                            ▼
                   ┌────────────────┐
                   │ Low-latency NIC│
                   └───────┬────────┘
                           │
                     RX / kernel bypass
                           │
                           ▼
                     FEED HANDLER
                           │
                     decode packets
                           │
                    sequence checking
                           │
                   update order books
                           │
                           ▼
                 CACHE-RESIDENT STATE
                           │
                 ┌─────────┴─────────┐
                 │                   │
              Seqlock             SPMC Queue
               state                events
                 │                   │
          ┌──────┴──────┐      ┌─────┴─────┐
          ▼             ▼      ▼           ▼
      Strategy A   Strategy B  Risk      Recorder
          │
          │ model calculation
          │
          ▼
       decision
          │
          ▼
   PRE-TRADE RISK
          │
          ▼
    ORDER GATEWAY
          │
       encode
          │
          ▼
         NIC
          │
          ▼
       EXCHANGE
```

Running underneath:

```text
CPU affinity
CPU isolation
IRQ isolation
NUMA affinity
memory preallocation
huge pages where justified
C-state configuration
P-state configuration
busy polling
shared LLC management
hardware timestamps
continuous performance metrics
```

And potentially beside it:

```text
                   FPGA

Market Feed ─────────┬─────────► FPGA reaction
                     │
                     └─────────► Software Strategy

Software updates models/rules
               │
               ▼
             FPGA
```

That is a much more realistic mental model of HFT than simply:

```text
C++ trading algorithm.
```

---

# PART XXXII — THE QUESTIONS YOU SHOULD NOW ASK ABOUT ANY LOW-LATENCY SYSTEM

When looking at Redis, PostgreSQL, Kafka, your own messaging system, or an HFT component, don't initially ask:

```text
Can I rewrite this in C++?
```

Ask:

### Data

```text
What is the hot working set?

Is memory contiguous?

How many pointer indirections exist?

How many cache lines are touched per operation?

Does it fit L1/L2/L3?
```

### Ownership

```text
Who writes this state?

Can we use a single writer?

Why are multiple threads mutating it?
```

### Communication

```text
Is this state or an event?

Do I need every event?

Do I need acknowledgments?

Do I need blocking?

Do I need multiple producers?
```

### Memory

```text
Are allocations happening in the hot path?

Can capacity be fixed?

Are pages prefaulted?

Where is memory allocated NUMA-wise?
```

### CPU

```text
Which core runs this?

What shares its LLC?

Can it migrate?

Can interrupts run there?

Can it enter deep sleep?
```

### Network

```text
Why does this packet enter the kernel?

Do we need TCP?

Do we need a copy?

Where is the NIC relative to the CPU?
```

### Measurement

```text
What precisely is the trigger?

What precisely is the target?

What is p50?

What is p99?

What is p99.9?

What happens during bursts?

Which hardware counters change?
```

That's how these two Gross talks should change the way you reason about systems.

---

# PART XXXIII — THE SINGLE MOST IMPORTANT CONCEPT FROM THIS TALK

If the 2024 talk's biggest lesson was:

> **Mechanical sympathy: optimize according to how the CPU actually behaves.**

then the 2022 talk's biggest lesson is:

> **Shape the system's data and ownership architecture so that the CPU has very little difficult work to do in the first place.**

There is a huge difference between:

```text
Optimize synchronization.
```

and:

```text
Architect away synchronization.
```

Between:

```text
Optimize memory allocator.
```

and:

```text
Don't allocate while trading.
```

Between:

```text
Optimize hashmap.
```

and:

```text
Make the hot data directly indexable and contiguous.
```

Between:

```text
Optimize inter-process TCP.
```

and:

```text
Use shared memory because both processes are on the same machine.
```

Between:

```text
Optimize scheduler behaviour.
```

and:

```text
Don't let the scheduler touch this core.
```

The highest-level optimization is often:

# Remove the need for the expensive operation.

---

# Final picture

After studying this talk properly, your low-latency hierarchy should be:

```text
                     LOW LATENCY

                         ▲
                         │
                Algorithm tricks
                         │
             Branch/SIMD optimization
                         │
                Data structures
                         │
                  Data layout
                         │
             Ownership/concurrency
                         │
              Process architecture
                         │
                Memory topology
                         │
                CPU topology
                         │
               NIC/network path
                         │
              Hardware/platform
                         │
                 Measurement
                         │
             Performance requirement
                         │
                         ▼

                   SYSTEM DESIGN
```

The mistake most people make is starting at:

```text
algorithm tricks.
```

Gross is telling you to start almost at the bottom:

```text
What latency do I need?

What hardware do I have?

What data must stay hot?

Who owns it?

How should components communicate?

How will I prove that it remains fast?
```

Once those answers are right, **then** optimizing individual nanoseconds becomes worthwhile.

For your goal of understanding actual HFT architecture, I would put this talk among the most important ones because it bridges the gap between **“I know C++ optimization”** and **“I understand why an HFT system is architected the way it is.”** The next major missing layer is the **network/exchange path itself**: Ethernet frame → NIC RX ring → feed arbitration → exchange decoder → order book → strategy → risk → order gateway → TX ring → exchange, including how ITCH/OUCH/SBE, multicast, sequence recovery, kernel bypass, DMA, PCIe and hardware timestamping fit together. That is the natural next chapter after these two Gross talks.

[1]: https://www.youtube.com/watch?v=8uAW5FQtcvE&utm_source=chatgpt.com "Trading at light speed: designing low latency systems in C++ - David Gross - Meeting C++ 2022 - YouTube"
[2]: https://electronictradinghub.com/trading-at-light-speed-designing-low-latency-systems-in-c-david-gross-meeting-c-2022/ "Trading at light speed: designing low latency systems in C++ - David Gross - Meeting C++ 2022 - Electronic Trading Hub"
[3]: https://hftadvisory.substack.com/p/trading-at-light-speed-designing-low-latency-systems-in-c-david-gross-meeting-c-2022?utm_source=chatgpt.com "Trading at light speed: designing low latency systems in C++ – David Gross – Meeting C++ 2022"
[4]: https://blog.freepackets.net/page/2/ "blog.freepackets.net"
[5]: https://github.com/wanghenshui/cppweeklynews/blob/dev/posts/095.md "cppweeklynews/posts/095.md at dev · wanghenshui/cppweeklynews · GitHub"
[6]: https://ruclips.net/video/8uAW5FQtcvE/trading-at-light-speed-designing-low-latency-systems-in-c-david-gross-meeting-c-2022.html?utm_source=chatgpt.com "Trading at light speed: designing low latency systems in C++ - David Gross - Meeting C++ 2022"
[7]: https://www.brendangregg.com/wss.html?utm_source=chatgpt.com "Working Set Size Estimation"
[8]: https://github.com/brendangregg/wss?utm_source=chatgpt.com "GitHub - brendangregg/wss: Working Set Size tools · GitHub"
[9]: https://podwise.ai/dashboard/episodes/6458389 "Trading at light speed: designing low latency systems in C++ - David Gross - Meeting C++ 2022 | Meeting Cpp | Podwise"
[10]: https://cdn.kernel.org/doc/html/latest/locking/seqlock.html?utm_source=chatgpt.com "Sequence counters and sequential locks — The Linux Kernel documentation"
[11]: https://developer.cloud.tencent.com/article/2213621?utm_source=chatgpt.com "C++ 中文周刊 第95期-腾讯云开发者社区-腾讯云"
[12]: https://docs.redhat.com/en/documentation/red_hat_enterprise_linux_for_real_time/10/html/optimizing_rhel_for_real_time_for_low_latency_operation/isolating-cpus-using-tuned-profiles-real-time?utm_source=chatgpt.com "Chapter 26. Isolating CPUs using tuned-profiles-real-time | Optimizing RHEL for Real Time for low latency operation | Red Hat Enterprise Linux for Real Time | 10 | Red Hat Documentation"
[13]: https://cdrdv2-public.intel.com/634648/634648-004.pdf?utm_source=chatgpt.com "11th Generation Intel® Core™ Processor Desktop"
