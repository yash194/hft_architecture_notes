Yes. I went through the talk structure, transcript, and slide material rather than relying on a short YouTube summary. This is **David Gross of Optiver, “When Nanoseconds Matter: Ultrafast Trading Systems in C++,” CppCon 2024**. The talk is valuable because it explains the *engineering philosophy* behind low-latency systems rather than just throwing around tricks such as “use C++,” “use lock-free queues,” or “avoid malloc.” ([CppCon][1])

[Watch the original CppCon talk](https://www.youtube.com/watch?v=sX2nF1fW7kI&utm_source=chatgpt.com)
[Official CppCon 2024 slides](https://github.com/CppCon/CppCon2024/blob/main/Presentations/When_Nanoseconds_Matter.pdf?utm_source=chatgpt.com)

What follows is best treated as a **chapter of notes on low-latency/HFT system engineering**, not merely a video summary. I mark some sections as **Beyond the talk** where I add concepts that Gross does not cover deeply but that you should understand to properly connect everything.

---

# When Nanoseconds Matter

## Detailed Notes on Ultrafast Trading Systems in C++

### David Gross — Optiver — CppCon 2024

---

# 1. The central idea of the entire talk

The most important lesson is:

> **Ultra-low latency is primarily an architecture/design problem, not a code-optimization problem.**

You cannot normally build:

```text
normal application
      ↓
profile it
      ↓
optimize some loops
      ↓
HFT-grade application
```

because fundamental decisions may already be wrong:

```text
data structures
memory layout
process architecture
communication mechanisms
networking model
ownership of data
concurrency model
CPU placement
API design
```

If those choices were made without latency in mind, fixing the system later may effectively require redesigning it.

Gross repeatedly returns to this theme. His examples start from sensible conventional implementations and progressively change them after looking at the **actual workload and actual hardware behaviour**. ([Josherich’s Blog][2])

That gives us our first mental model:

```text
                 Algorithm
                    ↓
                 C++ code
                    ↓
             Compiler output
                    ↓
      CPU instructions / branches
                    ↓
          L1 → L2 → L3 → RAM
                    ↓
             NIC / PCIe / NUMA
                    ↓
           Whole machine behavior
```

You cannot optimize only the first two levels.

---

# 2. Why trading systems care about nanoseconds

Gross begins unusually—with ancient Rome.

The point of this historical introduction is not really Roman economics. It is **uncertainty**.

Humans have always wanted mechanisms for managing uncertain future outcomes. Futures and derivatives are mechanisms for transferring and pricing that uncertainty.

Modern market makers perform a related role by continuously offering prices at which they are willing to buy and sell.

Suppose:

```text
Bid     Ask

$99     $101
```

A market maker may be willing to:

```text
buy at 99
sell at 101
```

and potentially earn some portion of the spread.

But this creates a serious problem.

Imagine unexpected information arrives:

```text
NEWS EVENT
    ↓
true value falls quickly
    ↓
your old bid = $99
    ↓
your quote is now stale
    ↓
others immediately sell to you at $99
    ↓
you lose money
```

Therefore a market maker has two major requirements.

### Requirement 1 — react quickly

New information arrives:

```text
market data
news
price change
trade
order book update
correlated instrument movement
```

The system must react before its existing orders become dangerous.

### Requirement 2 — calculate intelligently

Speed alone isn't enough.

You need:

```text
good models
good pricing
risk calculations
cross-instrument relationships
position awareness
volatility estimates
market microstructure information
```

So the objective isn't merely:

```text
FAST
```

It is:

```text
accurate + fast
```

Gross describes market making as a kind of **“loser's game”**: there is no single magical optimization or model that wins everything. You need to consistently avoid being bad at any important part of the system. ([Josherich’s Blog][2])

That becomes analogous to low-latency engineering itself:

```text
20 ns saved here
+
50 ns saved there
+
150 ns avoided there
+
less jitter
+
better cache locality
+
faster network
+
better CPU isolation
────────────────────
fast system
```

There is no single:

```cpp
make_everything_fast();
```

---

# 3. A simplified architecture of a modern trading system

One extremely useful part of the talk is the distinction between **FPGA logic and software strategies**.

Conceptually:

```text
                   EXCHANGE
                      │
                      │ Market Data
                      ▼
              ┌───────────────┐
              │      NIC      │
              └───────┬───────┘
                      │
                  FPGA path
                      │
                      ▼
              very fast reaction
                      │
                      ▼
                   orders
                      │
                      ▼
                   EXCHANGE


Meanwhile:

              SOFTWARE STRATEGY
                      │
             complex calculations
                      │
                      ▼
              updates FPGA rules
```

Why use both?

An FPGA is capable of extremely fast deterministic operations, but Gross emphasizes that the extremely fast FPGA path tends to stay **functionally simple**.

Think of logic such as:

```text
IF market_price > X
THEN cancel order
```

or:

```text
IF condition A occurs
THEN modify price to Y
```

The FPGA can essentially:

```text
receive bits
compare bits
modify bits
send bits
```

very quickly.

But complicated strategy logic might involve:

```text
volatility models
option pricing
risk calculations
position management
multiple correlated markets
model calibration
dynamic parameters
complex decision trees
```

That remains in software because software is enormously more flexible.

Therefore:

```text
             complex / adaptable logic
                     SOFTWARE
                         │
                         │ programs rules
                         ▼
                  ┌─────────────┐
Market Data ────► │    FPGA     │ ────► Orders
                  └─────────────┘
                    simple +
                  deterministic +
                  extremely fast
```

This also answers the common question:

> If HFT firms have FPGAs, why do they still care about extremely fast C++?

Because software continuously generates the **state, models and rules** from which the extremely fast hardware decisions operate.

If strategy software recalculates those rules too slowly, the FPGA ends up executing very quickly using **stale information**. ([Josherich’s Blog][2])

---

# 4. First major engineering example: the order book

This is probably the most important technical section of the talk.

An exchange order book can conceptually look like:

```text
BIDS                         ASKS

Price      Quantity          Price      Quantity

92           50               95           25
90          100               98           80
85           70              100          120
```

The highest bid is:

```text
Best Bid = 92
```

The lowest ask is:

```text
Best Ask = 95
```

These two points form the **top of book** or **touch**.

Each price and associated aggregate volume forms a **price level**.

Gross notes that the stock order books considered in the talk have roughly **1,000 levels on each side**, giving us an important scale assumption for later optimizations. ([Josherich’s Blog][2])

---

# 5. Operations the order book needs to support

Incoming exchange messages generally include operations resembling:

```text
ADD
MODIFY
DELETE
```

An ADD might mean:

```text
Buy 25 shares @ 92
```

If there are already 50 shares at that level:

```text
92 → 50
```

then:

```text
92 → 75
```

Another order might introduce a new level.

MODIFY changes an existing order.

DELETE removes an existing order.

The challenge is that exchange messages commonly identify orders using something similar to:

```text
Order ID = 82737291
```

not simply:

```text
Price = 92
```

So somewhere you generally need a structure mapping:

```text
Order ID
   ↓
price
quantity
side
possibly location/iterator
```

Therefore the complete book frequently needs both:

```text
Order ID lookup
+
ordered price-level structure
```

---

# 6. The obvious implementation: `std::map`

If you ask someone who knows algorithms:

> I need dynamically updated sorted price levels. What container should I use?

A completely reasonable answer is:

```cpp
std::map
```

because it is usually implemented as a balanced tree.

So:

```text
             90
           /    \
         85      95
        /       /  \
       ...     ... ...
```

Operations are approximately:

```text
lookup        O(log N)
insert        O(log N)
erase         O(log N)
```

This sounds excellent.

It also provides stable iterators under many operations, which allows the ID map to potentially retain references into the price-level structure.

Algorithmically this looks almost perfect.

And yet it performs badly relative to other solutions.

Why?

---

# 7. Big-O complexity is not CPU performance

A tree like `std::map` typically consists of separately allocated nodes:

```text
Node A at 0x13788210
       │
       ▼
Node B at 0x92ABC610
       │
       ▼
Node C at 0x31F91200
```

instead of:

```text
[ A ][ B ][ C ][ D ][ E ][ F ]
```

The first representation causes **pointer chasing**.

The CPU asks:

```text
Where is the next node?
```

It often cannot determine that until the current node has been loaded.

This hurts several mechanisms:

```text
cache locality
hardware prefetching
memory-level predictability
TLB locality
branch prediction
```

The cost of an algorithm is therefore better thought about as:

```text
algorithmic work
+
memory accesses
+
cache misses
+
branch misses
+
dependency chains
+
instructions
+
allocation behaviour
```

rather than merely:

```text
O(log N)
```

This gives Gross's Principle #1:

# Principle 1

## Most of the time, don't use node-based containers in latency-sensitive hot paths.

He specifically highlights the family containing structures such as:

```text
std::map
std::set
std::unordered_map
std::unordered_set
std::list
```

when implemented using individually allocated nodes.

The official slides contrast this with contiguous storage and later flat/data-oriented structures. ([studylib.net][3])

This doesn't mean:

> `std::map` is bad software.

It means:

> When every tens of nanoseconds matter, scattered pointer-heavy structures often fight the hardware.

---

# 8. Why contiguous memory is so powerful

Compare:

```text
Tree

RAM:
[A] ... [X] ... [B] ........ [C] ... [D]
```

with:

```text
Vector

RAM:
[A][B][C][D][E][F][G][H]
```

Once the CPU loads `A`, it may pull an entire cache line:

```text
[A][B][C][D][E][F][G][H]
^^^^^^^^^^^^^^^^^^^^^^^^^
one cache line
```

Now several future values may already be available.

The hardware prefetcher also observes:

```text
address
address + 8
address + 16
address + 24
...
```

and starts fetching future memory automatically.

This property becomes enormously important later.

---

# 9. Replace `std::map` with `std::vector`

Gross next tries something conceptually like:

```cpp
std::vector<std::pair<Price, Volume>>
```

and uses:

```cpp
std::lower_bound(...)
```

to locate the position.

Now lookups are:

```text
binary search → O(log N)
```

But insertion is:

```text
O(N)
```

because elements may have to shift.

Suppose:

```text
[85][90][92][95][100]
```

and you insert 91:

```text
[85][90][   ][92][95][100]
          ↑
       insertion
```

Several elements move.

According to traditional data-structure analysis:

```text
std::map should probably win
```

because:

```text
map insert     O(log N)
vector insert  O(N)
```

Yet vector performs extremely competitively because:

```text
moving contiguous bytes is cheap
```

while:

```text
random pointer chasing is expensive
```

This is one of the most important lessons in performance engineering.

---

# 10. But the vector still has a latency tail

The initial vector implementation improves performance considerably, but Gross shows something unacceptable for latency-sensitive systems:

```text
          main body
             ███
            █████
           ███████
         ██████████
                      ░
                       ░
                        ░
                         ░
                          ░

                   LATENCY TAIL
```

This is important.

In ordinary applications people often optimize:

```text
average latency
```

HFT engineers care greatly about:

```text
distribution
variance
p99
p99.9
worst cases
jitter
```

because:

```text
99 fast updates + 1 catastrophically slow update
```

can be much worse than:

```text
100 consistently moderate updates
```

when the slow update happens during an important market event.

Gross therefore repeatedly uses **full latency distributions**, not just a single benchmark mean. ([Josherich’s Blog][2])

---

# 11. The breakthrough: look at the actual data

Why is vector insertion producing a tail?

Gross examines actual market-data behaviour.

The update distribution is highly concentrated around the:

```text
TOP OF BOOK
```

So if levels are:

```text
BEST
 ↓
[92][90][85][80][75][70]...
```

most changes occur around the beginning.

But inserting or removing at the beginning of a vector means:

```text
[92][90][85][80][75]
 ↓
change
 ↓
shift many elements
```

The clever solution is almost embarrassingly simple:

# Reverse the vector.

Instead store:

```text
[70][75][80][85][90][92]
                     ↑
                    BEST
```

Now the most frequently changing area lies near the end.

An insertion near the end moves:

```text
very few elements
```

instead of:

```text
almost the entire vector
```

The talk shows that this substantially improves the latency distribution and removes much of the fat tail. ([Josherich’s Blog][2])

This leads to two crucial principles.

# Principle 2

## Understand your problem by looking at the data.

# Principle 3

## Specialized algorithms are critical for extreme performance.

The slide deck states these explicitly. ([studylib.net][3])

---

# 12. This is an incredibly important HFT lesson

Notice what happened.

Nobody invented a new revolutionary data structure.

Instead:

```text
observe business-domain property
             ↓
updates concentrate near top of book
             ↓
observe memory-layout property
             ↓
vector insertion near beginning is expensive
             ↓
change representation
             ↓
best prices placed near end
             ↓
fewer moves
             ↓
better tail latency
```

That is **data-oriented design**.

Instead of asking:

> Which general-purpose container theoretically supports my operation?

you ask:

> What does my actual data look like, and what memory representation makes the CPU's job easiest?

That distinction runs through the entire talk.

---

# 13. Do not begin profiling with random hardware counters

Once the data structure is improved, Gross profiles it more deeply.

A common mistake is:

```text
run profiler
 ↓
collect 200 counters
 ↓
L1 misses!
branch misses!
TLB misses!
cycles!
instructions!
 ↓
confusion
```

Instead, start broad.

He uses Intel's **Top-Down Microarchitecture Analysis methodology**.

At the highest level CPU pipeline activity can be divided into:

```text
┌─────────────────────────┐
│        Retiring         │ useful work completed
├─────────────────────────┤
│     Bad Speculation     │ work discarded
├─────────────────────────┤
│     Front-End Bound     │ CPU can't feed work quickly enough
├─────────────────────────┤
│      Back-End Bound     │ execution/memory side can't keep up
└─────────────────────────┘
```

Intel describes the same methodology: identify the dominant top-level bottleneck first, then drill down rather than optimizing arbitrary counters. ([Intel][4])

---

# 14. What those four CPU categories actually mean

## Retiring

Useful instructions successfully finish.

Very roughly:

```text
fetch
decode
execute
retire ✓
```

Higher retiring isn't automatically perfect, but generally you want useful work being completed.

---

## Bad speculation

The CPU predicted something and executed speculatively:

```text
IF condition...
```

It predicts:

```text
probably TRUE
```

starts executing that path—

and discovers:

```text
actually FALSE
```

Now the speculative work must be discarded.

This commonly results from:

```text
branch mispredictions
```

---

## Front-end bound

The execution engine is ready, but instructions aren't arriving quickly enough.

Possible causes include:

```text
instruction-cache misses
instruction TLB misses
decoding limitations
large code footprint
bad code layout
```

---

## Back-end bound

Instructions are available but execution cannot progress.

Possible causes:

```text
waiting for memory
cache misses
RAM access
execution-port contention
dependency chains
```

This classification helps answer:

> What type of problem should I investigate next?

---

# 15. The surprising problem: binary search

For Gross's order-book benchmark, the Top-Down analysis reports around:

```text
Bad speculation ≈ 25%
```

which is very high for this workload.

Profiling then reveals a large amount of CPU time around conditional jumps in the binary search. The slides show the top-down breakdown and then transition directly into branchless binary search. ([studylib.net][3])

Why is binary search hard for branch prediction?

Binary search repeatedly does:

```text
value < midpoint?
```

then:

```text
left or right?
```

If incoming values are difficult to predict:

```text
left
right
right
left
left
right
...
```

the CPU's branch predictor struggles.

A modern branch misprediction is not just one failed comparison.

It can mean:

```text
wrong speculative execution
        ↓
pipeline flush
        ↓
correct target discovered
        ↓
instruction stream restarted
```

which can cost many CPU cycles.

---

# 16. Branchless binary search

So Gross tries a branchless form.

Instead of something logically like:

```cpp
if (a[mid] < target)
    low = mid + 1;
else
    high = mid;
```

the idea is to express computation so the compiler can use something such as:

```text
CMOV
```

rather than unpredictable branches.

Results presented in the talk show roughly:

```text
branch misses → about half
IPC            1.4 → 1.6
```

which is a meaningful gain. ([Josherich’s Blog][2])

But there is a tradeoff.

Traditional binary search can stop or avoid some work.

A branchless implementation may perform:

```text
more instructions
more predictable work
```

So the question becomes:

```text
fewer unpredictable branches
vs.
more total work
```

There is no universal winner.

Measure.

---

# 17. Then something even stranger wins: linear search

After trying multiple sophisticated order-book implementations, Gross finds that a **linear search** performs extremely well for this workload.

That sounds wrong if you only know algorithm classes:

```text
binary search = O(log N)

linear search = O(N)
```

But the CPU sees something else.

Linear search:

```text
A → B → C → D → E → F → G
```

means:

```text
sequential memory
excellent cache locality
excellent hardware prefetching
simple loop
predictable execution
little pointer chasing
```

Meanwhile binary search does something like:

```text
500 → 250 → 375 → 312 → 343...
```

which jumps through memory and has unpredictable control flow.

Also remember:

```text
N ≈ ~1000 levels
```

and actual updates aren't uniformly distributed across all levels.

Many are near the best price.

So a linear scan frequently does **far less than N work in practice**.

Gross concludes that for the studied workload the extremely simple linear search is the best solution, with a tight latency distribution. ([Josherich’s Blog][2])

This gives two more principles.

# Principle 4

## Simplicity is the ultimate sophistication.

# Principle 5

## Mechanical sympathy.

The official slides identify these explicitly. ([studylib.net][3])

---

# 18. What “mechanical sympathy” really means

This expression is extremely important in HFT.

It means:

> Design software with an understanding of the machine that executes it.

For example:

Bad:

```text
beautiful abstraction
     ↓
lots of pointers
     ↓
many allocations
     ↓
random memory access
     ↓
branch-heavy virtual calls
```

Mechanically sympathetic:

```text
contiguous arrays
      ↓
sequential reads
      ↓
hardware prefetcher works
      ↓
cache lines used efficiently
      ↓
few unpredictable branches
```

You don't merely ask:

```text
Is my C++ logically efficient?
```

You ask:

```text
What will the CPU actually do?
```

---

# 19. Instruction cache matters too

Most programmers first learn about:

```text
data cache
```

but extremely low-latency systems also care about:

```text
instruction cache
```

Your program's machine code itself must be fetched.

Therefore the **hot path should remain small**.

Imagine:

```cpp
if (unlikely_error) {
    giant_error_handling_function();
}
```

If the error code gets inlined into the main function, your hot instruction stream gets bigger despite that path almost never executing.

Gross therefore discusses:

```cpp
[[likely]]
[[unlikely]]
```

and moving unlikely/error handling into something approximately:

```cpp
__attribute__((cold))
__attribute__((noinline))
```

The goal is:

```text
HOT CODE
[price update][search][modify][send]
```

kept together,

while:

```text
COLD CODE
[format error][logging][crash handling][rare validation]
```

is kept elsewhere.

The slide deck shows this progression explicitly. ([studylib.net][3])

---

# 20. Inlining is not automatically good

Typical performance folklore says:

```text
inline everything
```

because function calls have overhead.

But aggressive inlining can cause:

```text
larger binary
larger hot functions
instruction cache pressure
worse code locality
```

So the real choice is:

```text
call overhead
vs.
instruction-cache footprint
```

Again:

```text
measure
```

---

# 21. Lambda/functor vs `std::function`

Gross also warns about casually converting templated callable code into:

```cpp
std::function
```

A compiler knows the exact type of a lambda/functor in templated code and can often:

```text
inline
constant propagate
specialize
remove dead code
```

`std::function` introduces **type erasure**, which can obscure the actual callable behind a runtime abstraction.

Conceptually:

```text
template/lambda:

compiler
   ↓
exact function known
   ↓
deep optimization possible
```

versus:

```text
std::function
   ↓
type-erased callable
   ↓
indirection/runtime machinery
   ↓
fewer optimization opportunities
```

This does not mean "`std::function` is bad."

It means don't casually put abstraction overhead into a path where nanoseconds matter.

---

# 22. Now move outward: networking

We have optimized the in-memory order book.

But exchange data still has to arrive.

Conventional networking looks approximately:

```text
NIC
 ↓
kernel driver
 ↓
kernel network stack
 ↓
socket buffers
 ↓
syscalls
 ↓
user-space application
```

This architecture is enormously useful because the kernel provides:

```text
security
multiplexing
TCP/IP
resource management
process isolation
drivers
routing
firewalls
general-purpose abstractions
```

But HFT asks:

> Do I need all of this machinery on every packet in my hottest path?

Often the answer is no.

---

# 23. Kernel bypass

So low-latency systems often use:

```text
NIC
 ↓
userspace
```

rather than:

```text
NIC
 ↓
kernel
 ↓
userspace
```

This is **kernel bypass**.

Gross discusses Solarflare/AMD technologies including different levels of abstraction.

Conceptually:

```text
Normal sockets
     │
     ▼
 Onload
     │
     ▼
TCPDirect
     │
     ▼
   ef_vi
     │
     ▼
raw Layer-2 style access
```

As you move downward:

```text
convenience decreases
control increases
latency decreases
engineering responsibility increases
```

AMD's current documentation describes `ef_vi` as a Layer-2 API providing direct access to the network-adapter datapath, including a zero-copy interface, specifically for applications seeking extremely low send/receive latency. ([AMD Documentation][5])

Gross also mentions DPDK as an alternative family of approaches.

---

# 24. Why kernel bypass helps

Traditional packet handling may involve:

```text
interrupts
kernel scheduling
socket queues
packet structures
kernel/user transitions
copies
syscalls
```

The danger is not merely average cost.

A massive concern is:

# jitter

Instead of:

```text
810 ns
815 ns
809 ns
812 ns
```

you might obtain:

```text
1.8 μs
2.1 μs
1.9 μs
9.7 μs
2.0 μs
...
```

Low-latency trading wants:

```text
fast
AND
predictably fast
```

Gross cites measurements in the talk showing a progression from roughly microseconds through a kernel path toward the sub-microsecond region using lower-level NIC interfaces, though exact values are hardware-specific. ([Josherich’s Blog][2])

This gives Principle #6:

# Principle 6

## Efficiency often comes from removing unnecessary layers rather than adding sophisticated ones.

The corresponding slide explicitly frames the kernel as useful complexity that should not be paid for when the application does not require it. ([studylib.net][3])

---

# 25. From NIC to processes: shared memory

Imagine market data has now reached your server.

You have:

```text
Market Data Feed Handler
```

and perhaps:

```text
Strategy 1
Strategy 2
Strategy 3
...
Strategy 50
```

How should they communicate?

You could use:

```text
TCP sockets
Unix-domain sockets
message brokers
pipes
```

But if everything lives on the same machine:

> Why leave memory just to come back to memory?

Instead:

```text
                   Shared Memory
                         │
         ┌───────────────┼───────────────┐
         ▼               ▼               ▼
     Strategy A      Strategy B      Strategy C
```

Processes map the same physical memory.

Once mapped, ordinary memory operations provide the communication path without repeatedly traversing the kernel networking stack. ([Josherich’s Blog][2])

---

# 26. Why multiple processes instead of one giant process?

At first glance:

```text
one process
+
many threads
```

seems even faster because threads naturally share memory.

But Gross emphasizes operational robustness.

Imagine:

```text
Strategy 37
     ↓
crashes
     ↓
entire monolithic process crashes
     ↓
50 strategies disappear
```

Multiprocess isolation gives:

```text
Feed Handler
   │
Shared Memory
   ├── Strategy 1 process
   ├── Strategy 2 process
   ├── Strategy 3 process
   └── Strategy 50 process
```

Now:

```text
Strategy 37 crashes
```

does not necessarily kill:

```text
Strategy 1-36
Strategy 38-50
```

So architecture balances:

```text
latency
+
failure isolation
+
operational simplicity
```

This is an important reminder that real HFT engineering isn't simply code-golf for nanoseconds.

---

# 27. Shared-memory protocol design

Putting bytes in shared memory isn't sufficient.

You need a protocol.

Gross shows a header containing things conceptually such as:

```text
protocol name
magic number
buffer size
major version
minor version
queue sizes
reserved space
```

Why a magic number?

Suppose Process A expects:

```text
OrderBookProtocol
```

but accidentally maps:

```text
TradeProtocol
```

Without identifying metadata it might interpret arbitrary bytes as legitimate structures.

Why version numbers?

Because:

```text
Version 1 deployed today
Version 2 deployed six months later
```

may coexist temporarily.

The hard problem isn't:

```text
design a shared memory protocol once
```

It is:

```text
keep it working and evolvable for 5–10 years
```

The official deck shows precisely this kind of protocol-header design. ([studylib.net][3])

---

# 28. One writer, many readers

Gross strongly favors keeping a shared-memory region approximately:

```text
ONE WRITER
   │
   ├── reader
   ├── reader
   ├── reader
   └── reader
```

rather than:

```text
writer ─┐
writer ─┼── same data
writer ─┘
```

Why?

Multiple writers introduce:

```text
ownership coordination
locking
CAS loops
cache-line bouncing
ordering complexity
harder correctness proofs
```

Single-writer designs eliminate entire categories of concurrency problems.

This principle appears throughout low-latency system architecture:

```text
ownership is performance
```

---

# 29. Requirements for the queue

Gross builds a specialized shared-memory queue whose requirements are roughly:

```text
Bounded?          YES
Blocking?         NO
Consumers?        MANY
Messages?         VARIABLE LENGTH
Distribution?     FAN-OUT
Data?             POD-like bytes
```

The slides state these design choices explicitly. ([studylib.net][3])

Why bounded?

Because resizing means:

```text
allocation
coordination
possibly copying
unbounded latency
```

In HFT:

```text
fixed capacity
```

is often a desirable architectural constraint.

---

# 30. Why non-blocking?

Suppose:

```text
Producer
   ↓
Market updates
```

feeds:

```text
Strategy A
Strategy B
Strategy C
```

Strategy B develops a bug and becomes slow.

You don't want:

```text
B slow
 ↓
producer waits
 ↓
A starves
 ↓
C starves
 ↓
whole trading system delayed
```

So:

```text
readers do not control writer progress
```

If a reader cannot keep up, that reader should fail/recover rather than slowing every other strategy.

This is a very HFT-style design tradeoff.

---

# 31. The FastQueue design

The queue has two important counters:

```text
Write Counter
Read/Published Counter
```

Both are controlled by the producer.

Simplified operation:

```text
1. advance/reserve write position

WRITE COUNTER ───────────────►

2. copy message bytes

[ message being written ]

3. advance published/readable counter

READ COUNTER ────────────────►
```

Readers only consume up to the published boundary.

So:

```text
          safe readable data
<────────────────────────────>

read counter
      │
      ▼
[finished data][currently being written]
                            ▲
                            │
                      write counter
```

This allows readers to distinguish:

```text
fully committed bytes
```

from:

```text
data currently being written
```

The slides show this state transition and the two atomic counters directly. ([studylib.net][3])

---

# 32. Cache-line ownership matters enormously

The counters are atomic because producer and consumers access them concurrently.

If producer frequently writes:

```text
writeCounter
```

and many consumers repeatedly read it, the cache-coherence system has work to do.

Cache lines may transition between coherence states across CPU cores.

This can become:

```text
cache-line ping-pong
```

or coherence traffic.

Therefore Gross's biggest queue optimization is remarkably simple:

# Don't update the shared atomic for every message.

---

# 33. Reserve in chunks

Instead of:

```text
message
update writeCounter

message
update writeCounter

message
update writeCounter
```

reserve something like:

```text
100 KB
```

at once.

For approximately:

```text
100 byte messages
```

that can mean roughly:

```text
1000 messages
```

before touching the shared write counter again.

Conceptually:

```text
Before:

atomic
message
atomic
message
atomic
message
atomic
message


After:

atomic reserve 100 KB

message
message
message
...
message × ~1000

atomic reserve next block
```

This drastically reduces contested shared-memory writes. ([Josherich’s Blog][2])

---

# 34. Another subtle result: don't align everything to 64 bytes

A common low-latency mantra is:

```cpp
alignas(64)
```

everywhere.

Why?

Because typical cache lines are 64 bytes and separating independent atomics can avoid **false sharing**.

That's good for:

```text
hot shared counters
```

But it doesn't mean every object should occupy a full cache line.

Doing:

```text
[8-byte payload........................................56 wasted bytes]
```

can make your data less dense.

Instead of eight useful 8-byte values per cache line you might get one.

So Gross distinguishes:

```text
shared mutable atomics
→ separate cache lines when needed

ordinary sequential payload data
→ keep compact
```

His queue optimization finds that smaller natural/aligned payload packing can improve locality rather than forcing each record to cache-line size. ([Josherich’s Blog][2])

This is another excellent example of:

> performance rules are context-dependent.

---

# 35. Cache the reader's counter too

Suppose a consumer previously read:

```text
published bytes = 1 MB
```

and has consumed only:

```text
100 KB
```

There is still:

```text
900 KB
```

known to be available.

So why reload the shared atomic on every message?

Don't.

Cache:

```text
mCachedReadCounter
```

locally and only reload the shared value once you exhaust the already-known range.

This removes repeated shared cache-line traffic.

The slides call this **Optimization #3: Caching the Read Counter**. ([studylib.net][3])

---

# 36. Queue benchmark setup

Gross tests the queue under a setup including:

```text
AMD EPYC 9474F
low-latency tuning
isolated cores
73-byte messages
8 MB queue
```

and compares against approaches including Aeron and Disruptor-like designs. ([studylib.net][3])

The conclusion isn't:

```text
everyone should use this exact queue
```

It is:

> A very small structure specialized for exactly your communication semantics can outperform much more general systems because it doesn't pay for features you don't need.

This loops back to:

```text
simplicity
specialization
mechanical sympathy
```

---

# 37. Principle 7

# Choose the right tool for the right task.

This deserves emphasis because it prevents people from misunderstanding the talk.

Gross is **not** saying:

```text
Never use libraries.
Never use Linux.
Never use queues.
Never use std::map.
Never use general-purpose systems.
```

He is saying:

```text
Know exactly what problem you have.
Then pay only for what that problem requires.
```

The official deck explicitly labels this Principle #7. ([studylib.net][3])

---

# 38. API design can determine performance

The first queue API essentially requires:

```text
serialize data into temporary buffer
              ↓
          queue.write()
              ↓
copy temporary buffer into queue
```

That's:

```text
serialization
+
copy
```

But the better API gives the caller direct space inside the queue:

```text
queue gives writable region
           ↓
serialize DIRECTLY into it
           ↓
publish
```

So:

```text
Old

Object
 ↓ serialize
Temporary Buffer
 ↓ memcpy
Queue


Better

Object
 ↓ serialize
Queue directly
```

The slides show this as the “Going further – Zero copy” API and mention serialization formats including SBE, Cap'n Proto, FlatBuffers and Protobuf. ([studylib.net][3])

Gross reports that API-level changes like this can produce very substantial performance improvements.

This is an underappreciated lesson:

> Performance is often encoded in the API long before it appears in a profiler.

---

# 39. Batching

Another extension of this principle is:

```text
write one message
update metadata
write one message
update metadata
```

versus:

```text
write N messages
update metadata once
```

Batching amortizes:

```text
atomic operations
doorbells
cache-line transfers
syscalls
metadata updates
```

But there is an important tradeoff:

```text
larger batch
    ↓
better throughput
but
higher waiting latency
```

Therefore HFT typically uses **small intentional batching**, not unlimited throughput-oriented batching.

---

# 40. NUMA awareness

Gross briefly mentions making the queue NUMA-aware and potentially duplicating queue header data on remote NUMA nodes. ([Josherich’s Blog][2])

This is important enough to explain properly.

Modern servers may look like:

```text
NUMA NODE 0                    NUMA NODE 1

CPU cores                      CPU cores
   │                              │
L3 caches                       L3 caches
   │                              │
Local RAM                      Local RAM
```

A CPU accessing:

```text
its local memory
```

is generally cheaper than repeatedly accessing:

```text
memory owned by another NUMA node
```

Therefore:

```text
thread placement
+
memory allocation placement
+
NIC placement
```

need to agree.

A theoretically fast queue can become slower because:

```text
producer on NUMA 0
consumer on NUMA 1
shared counter on NUMA 0
```

causes repeated interconnect traffic.

---

# 41. One of the trickiest details: the C++ memory model

Gross points out that the shared-memory byte-copy algorithm has an awkward issue.

From a practical hardware algorithm perspective, the protocol can detect overwritten data.

But in formal C++ semantics, concurrently touching ordinary non-atomic memory can produce a:

```text
data race
```

and a C++ data race means undefined behavior.

He discusses work around **byte-wise atomic memcpy** and also notes similar language-model difficulties around seqlock-style designs. ([Josherich’s Blog][2])

This is deeper than it initially appears.

There are three layers:

```text
Algorithm is logically correct
        ≠
Hardware execution appears correct
        ≠
C++ language memory model guarantees correctness
```

Low-level systems programmers have to reason about all three.

---

# 42. Measuring an event-driven trading system is hard

A typical trading loop may resemble:

```cpp
while (true) {
    poll_market_data();

    if (interesting_event()) {
        calculate();
        send_order();
    }
}
```

Most of the time the application may be:

```text
polling
polling
polling
polling
EVENT
tiny amount of critical work
polling
polling
```

Now suppose a sampling profiler samples:

```text
1000 times/sec
```

It will probably observe:

```text
poll
poll
poll
poll
poll
```

and might completely miss:

```text
critical 200 ns function
```

that actually determines trading latency.

Therefore conventional CPU profiling alone isn't enough.

---

# 43. Intrusive profiling with the Time Stamp Counter

One technique is:

```text
start = TSC
run function
end = TSC
latency = end - start
```

The x86 TSC is a CPU counter usable for very low-overhead timing.

Gross shows an RAII-style profiler concept:

```text
constructor:
    record start TSC

destructor:
    record end TSC
```

This provides accurate event-level measurements.

But if you put it everywhere:

```text
function A → TSC
function B → TSC
function C → TSC
function D → TSC
...
```

you eventually spend meaningful CPU time measuring the program rather than running it.

This is the classic:

# observer effect.

---

# 44. Clang XRay

Gross then introduces a very useful mechanism:

```text
Clang XRay
```

Compile functions with instrumentation support.

Conceptually the compiler leaves patchable regions:

```text
FUNCTION ENTRY
[NOP][NOP][NOP]...
real function
[NOP][NOP]...
FUNCTION EXIT
```

Normally:

```text
almost nothing happens
```

When profiling is enabled, these areas can be patched to invoke profiling handlers.

That means:

```text
same binary
+
selectively enabled instrumentation
+
fine-grained timings
```

without constantly rebuilding the application whenever a new hotspot appears.

The slide deck shows the XRay handler and patching interface around the profiling section. ([studylib.net][3])

---

# 45. Being fast once is not enough

Suppose release 1 produces:

```text
800 ns
```

Then:

```text
release 2 → 830 ns
release 3 → 870 ns
release 4 → 940 ns
release 5 → 1.1 μs
```

Each individual regression seems tiny.

Eventually the system isn't fast anymore.

Therefore production low-latency systems need:

```text
continuous metrics
latency distributions
automated audits
performance regression tests
alerts
trend analysis
```

Gross makes a particularly good observation:

> Recording millions of performance measurements is not enough if nobody automatically checks them.

Dashboards get ignored.

Automated expectations are harder to ignore.

This produces:

# Principle 8

## Being fast is good; staying fast is harder and more important.

([studylib.net][3])

---

# 46. The final big lesson: “You are not alone”

Near the end Gross creates a memory benchmark whose working-set size increases.

When small enough:

```text
working set
    ↓
fits L1
```

Performance is very high.

Then:

```text
working set > L1
       ↓
L2
```

performance drops.

Then:

```text
working set > L2
       ↓
L3
```

another change.

Finally:

```text
working set > L3
       ↓
DRAM
```

another major drop.

So you get conceptually:

```text
throughput
   │
   │██████ L1
   │
   │    █████ L2
   │
   │         ████ L3
   │
   │              ██ RAM
   └──────────────────────── working set
```

Nothing surprising yet.

Then he runs multiple workers on separate CPU cores.

---

# 47. Separate cores don't mean independent performance

You might assume:

```text
Worker 1 → Core 1
Worker 2 → Core 2
Worker 3 → Core 3
...
```

therefore:

```text
each worker independent
```

Not true.

They can share:

```text
L3/LLC
memory controllers
NUMA links
PCIe resources
DRAM bandwidth
inter-core fabrics
```

Gross demonstrates major scaling loss around the shared LLC region despite workers not logically sharing application data. ([Josherich’s Blog][2])

That means:

```text
YOUR SERVICE
```

can slow down because:

```text
SOMEONE ELSE'S SERVICE
```

running on the same box changes the hardware environment.

---

# 48. Principles 9 and 10

The slide deck makes the final two principles explicit.

# Principle 9

## Think about the system as a whole.

# Principle 10

## Your code's performance depends on your colleagues' code too.

([studylib.net][3])

This is hugely important for low-latency infrastructure.

You can write:

```text
perfectly optimized strategy
```

then place it beside:

```text
analytics process scanning 20 GB
```

and destroy your carefully optimized L3 behaviour.

Therefore deployment topology itself is part of performance engineering.

---

# 49. All ten principles from the talk

These are worth memorizing as the framework behind everything else:

| #      | Principle                          | Practical meaning                                      |
| ------ | ---------------------------------- | ------------------------------------------------------ |
| **1**  | Avoid node containers in hot paths | Prefer contiguous/data-oriented storage                |
| **2**  | Understand the problem using data  | Real workload > theoretical assumptions                |
| **3**  | Specialized algorithms matter      | Exploit known properties of your workload              |
| **4**  | Simplicity wins                    | Minimal implementations are easier for CPUs and humans |
| **5**  | Mechanical sympathy                | Design around actual hardware behaviour                |
| **6**  | Remove unnecessary layers          | Don't pay for generality you don't need                |
| **7**  | Right tool for right task          | There is no universally fastest queue/container/API    |
| **8**  | Stay fast                          | Continuously monitor latency regressions               |
| **9**  | Optimize the whole system          | CPU/NIC/cache/NUMA/processes interact                  |
| **10** | Your neighbours matter             | Other workloads on the server affect your latency      |

The final slides tie these ideas together by describing low-latency programming itself as another “loser's game”: disciplined engineering and avoiding unnecessary complexity matter more than hunting for one clever trick. ([studylib.net][3])

---

# 50. The complete system we have built so far

The talk's concepts now combine into something like:

```text
                              EXCHANGE
                                  │
                      UDP multicast / feeds
                                  │
                                  ▼
                       ┌──────────────────┐
                       │ Low-latency NIC  │
                       │ Solarflare/etc.  │
                       └────────┬─────────┘
                                │
                         kernel bypass
                                │
                                ▼
                      ┌─────────────────────┐
                      │   MARKET DATA       │
                      │   FEED HANDLER      │
                      │                     │
                      │ fast order books    │
                      │ contiguous memory   │
                      │ specialized search  │
                      └──────────┬──────────┘
                                 │
                           Shared Memory
                                 │
                        SPMC / fan-out queue
                                 │
              ┌──────────────────┼──────────────────┐
              │                  │                  │
              ▼                  ▼                  ▼
        ┌───────────┐      ┌───────────┐      ┌───────────┐
        │ Strategy A│      │ Strategy B│      │ Strategy C│
        └─────┬─────┘      └─────┬─────┘      └─────┬─────┘
              │                  │                  │
              │ pricing/model/risk decisions       │
              └──────────────────┼──────────────────┘
                                 │
                                 ▼
                        Order / Rule Output
                                 │
                         ┌───────┴───────┐
                         │               │
                         ▼               ▼
                      SOFTWARE          FPGA
                       path           fast path
                         │               │
                         └───────┬───────┘
                                 ▼
                           Low-latency NIC
                                 │
                                 ▼
                              EXCHANGE
```

This is much closer to how you should mentally picture a real low-latency trading environment than:

```text
strategy.py
   ↓
broker API
```

---

# 51. Q&A: hybrid binary/linear search

Someone asks:

> Why not use linear search over the first few levels and binary search afterwards?

Gross says he tested approaches in this direction.

For the relatively small collections involved, plain linear search remained sufficiently good that the additional complexity wasn't worthwhile.

But if you had something like:

```text
100,000 levels
```

the tradeoff could change. ([Josherich’s Blog][2])

Important lesson:

```text
There is no universally fastest search.
```

Performance depends on:

```text
N
value distribution
memory layout
CPU architecture
branch predictability
```

---

# 52. Q&A: Struct of Arrays and SIMD

Someone asks whether separating:

```cpp
struct Level {
    Price price;
    Volume volume;
};
```

from an Array-of-Structs layout:

```text
[P,V][P,V][P,V][P,V]
```

into Structure-of-Arrays:

```text
Prices:
[P][P][P][P][P]

Volumes:
[V][V][V][V][V]
```

helps.

Gross says it can.

Why?

During search you need:

```text
prices
```

but may not need:

```text
volumes
```

AoS loads both:

```text
price + useless volume
```

SoA packs more useful prices into cache lines:

```text
P P P P P P P P
```

and can make automatic SIMD/vectorization easier.

He reports a modest benefit in the tested scenario, but SIMD/AVX results aren't universally better because startup/instruction cost can outweigh benefits on small collections. ([Josherich’s Blog][2])

Again:

```text
measure.
```

---

# 53. Q&A: persistence and crashes

Someone asks:

> What if the queue should survive a process crash?

Gross points toward **write-ahead-log-style structures**, mentioning PostgreSQL as something to study—not suggesting using PostgreSQL in the low-latency queue path itself.

Conceptually:

```text
event
 ↓
append to durable log
 ↓
update state
```

If a process crashes:

```text
read log
 ↓
reconstruct state
```

This is essentially:

```text
event sourcing / WAL thinking
```

combined with a low-latency in-memory path. ([Josherich’s Blog][2])

---

# 54. Q&A: Linux message queues

Another question asks why not simply use Linux message queues.

Gross's answer is important:

The kernel's internal queue data structure might itself be perfectly fast.

The problem is the surrounding:

```text
kernel transition
scheduling behaviour
system activity
jitter
```

For ultra-low-latency systems, avoiding that unpredictable variance is often the reason for userspace/shared-memory approaches. ([Josherich’s Blog][2])

---

# 55. Beyond the talk: things you need to know to fully understand HFT latency

The talk is excellent, but intentionally focuses on selected examples. A real HFT engineer should add the following concepts.

---

# 56. Tail latency is often more important than average latency

Suppose System A:

```text
Median  = 500 ns
p99     = 550 ns
p99.9   = 650 ns
```

System B:

```text
Median  = 400 ns
p99     = 1.5 μs
p99.9   = 20 μs
```

Which is faster?

For ordinary benchmarks:

```text
B
```

For latency-sensitive trading, quite possibly:

```text
A
```

because B occasionally stalls catastrophically.

Therefore latency should be represented as:

```text
distribution
```

not:

```text
one number
```

A useful hierarchy is:

```text
minimum
median
p90
p99
p99.9
p99.99
maximum
jitter
```

---

# 57. CPU pinning and isolation

One trading thread should generally not jump:

```text
Core 1 → Core 7 → Core 3 → Core 19
```

because migration destroys locality and introduces scheduler variability.

Instead:

```text
Market feed → Core 4
Strategy    → Core 5
Order send  → Core 6
```

with carefully selected affinity.

Linux real-time documentation explicitly recommends CPU affinity/isolation for latency-sensitive workloads and explains that migrations can cost performance through cache invalidation. ([Red Hat Documentation][6])

---

# 58. IRQ affinity

Your NIC generates hardware activity.

If the operating system places network interrupts on the same CPU as your strategy:

```text
Strategy calculating
      ↓
IRQ arrives
      ↓
strategy interrupted
      ↓
kernel handler executes
      ↓
strategy resumes
```

you just introduced jitter.

So a typical topology might resemble:

```text
Core 0      OS/background
Core 1      interrupts
Core 2      interrupts
Core 3      feed handler
Core 4      strategy
Core 5      order sender
...
```

Red Hat's low-latency guidance specifically discusses isolating interrupts and application processes onto separate CPUs. ([Red Hat Documentation][7])

---

# 59. CPU C-states

When a CPU is idle, it can enter progressively deeper sleep states:

```text
C0 → running

C1
C2
C3
...
```

Deeper sleep:

```text
saves power
```

but waking up costs latency.

For normal servers:

```text
excellent
```

For a core that needs to respond immediately to a market packet:

```text
potential problem
```

This is why latency systems frequently configure power management carefully. Red Hat's real-time documentation explicitly notes that exiting deep C-states can introduce unwanted response latency. ([Red Hat Documentation][7])

Gross's earlier “Trading at Light Speed” talk also goes deeper into C-state/P-state and shared-LLC tuning. ([YouTube][8])

---

# 60. P-states / frequency scaling

Similarly:

```text
CPU frequency
```

may dynamically change.

For normal systems:

```text
energy efficient
```

For deterministic low latency:

```text
frequency transition
→ timing variability
```

So low-latency environments often care more about:

```text
stable predictable frequency
```

than peak benchmark frequency.

---

# 61. Hyper-Threading / SMT

Two logical threads may share parts of one physical core.

Conceptually:

```text
Physical Core
 ├── Logical CPU A
 └── Logical CPU B
```

They share resources.

So placing:

```text
critical trading thread A
```

beside:

```text
heavy thread B
```

on sibling SMT threads can create contention.

Many low-latency deployments either disable SMT or control sibling placement very carefully.

---

# 62. Page faults

Suppose your hot path accesses memory whose physical page hasn't yet been materialized.

You may trigger:

```text
page fault
      ↓
kernel
      ↓
page table work
      ↓
possibly memory allocation
      ↓
resume
```

That's disastrous for deterministic latency.

Therefore systems frequently:

```text
allocate memory during startup
touch/pre-fault it
lock critical pages when appropriate
avoid runtime allocation
```

This is another reason allocation policy is architectural rather than merely a `malloc()` microbenchmark question.

---

# 63. Huge pages

Normal pages may be:

```text
4 KB
```

A large working set requires many page-table/TLB entries.

Huge pages can be:

```text
2 MB
```

or larger depending on architecture/configuration.

Fewer translations can mean:

```text
fewer TLB misses
```

but huge pages aren't automatically better everywhere. They must be benchmarked with the complete workload.

---

# 64. Preallocation

Hot-path code should ideally avoid:

```cpp
new
delete
malloc
free
```

because allocation can produce:

```text
allocator metadata work
locks/contention
cache effects
page faults
fragmentation
unpredictability
```

Instead:

```text
startup
 ↓
allocate pools
 ↓
initialize objects
 ↓
trading starts
 ↓
reuse memory
```

This is one reason fixed-capacity structures appear frequently in low-latency systems.

---

# 65. Busy polling instead of sleeping

Ordinary application:

```text
no message
 ↓
sleep
 ↓
kernel wakes thread later
```

Low latency:

```cpp
while (true) {
    if (message_available())
        process();
}
```

This deliberately burns a CPU core.

Why?

Because you're exchanging:

```text
CPU efficiency
```

for:

```text
response latency
```

In HFT that can be entirely rational.

A server may intentionally use:

```text
100% CPU
```

while doing almost no business computation.

It is actively waiting.

---

# 66. Market-data networking itself

Real exchange feeds frequently involve multicast-style dissemination.

Conceptually:

```text
            EXCHANGE
                │
          multicast feed
                │
        ┌───────┼───────┐
        ▼       ▼       ▼
      Firm A  Firm B   Firm C
```

Your system needs a **feed handler** that:

```text
receives packet
 ↓
parses exchange protocol
 ↓
checks sequence number
 ↓
detects gaps
 ↓
updates internal order book
 ↓
publishes normalized event
```

The talk mostly begins around the latter portions rather than teaching exchange protocol mechanics from first principles.

---

# 67. Sequence numbers and gap recovery

Suppose packets arrive:

```text
1001
1002
1003
1005
```

You immediately know:

```text
1004 missing
```

Updating the order book anyway may create corrupted state.

So feed handlers need:

```text
sequence tracking
gap detection
retransmission/snapshot recovery
state validation
```

Low latency cannot mean:

```text
fast but wrong.
```

Gross emphasizes the need for **accuracy and speed together**, which is exactly why these mechanisms matter.

---

# 68. Normalized market data

Different exchanges speak different protocols.

Your strategy ideally should not contain:

```text
NASDAQ packet parser
NYSE parser
CME parser
EUREX parser
...
```

Instead:

```text
Exchange Protocol
       ↓
Feed Handler
       ↓
Normalized Internal Event
       ↓
Strategies
```

Example:

```cpp
struct PriceUpdate {
    InstrumentId id;
    Price price;
    Quantity qty;
    Side side;
};
```

This normalization is convenient, but in HFT even the abstraction must be designed carefully to avoid:

```text
unnecessary copying
dynamic allocation
virtual dispatch
oversized representations
```

---

# 69. Typical hot-path pipeline

A simplified market-data-to-order path can look like:

```text
Ethernet packet arrives
      ↓
NIC DMA
      ↓
userspace poll
      ↓
decode exchange message
      ↓
update order book
      ↓
update derived market state
      ↓
strategy evaluates
      ↓
risk check
      ↓
construct order
      ↓
serialize protocol
      ↓
NIC TX
      ↓
wire
```

This is the pipeline whose latency matters.

You can think of:

```text
Ttotal =
Tnic_rx
+ Tdecode
+ Tbook
+ Tstrategy
+ Trisk
+ Tserialize
+ Tnic_tx
```

Saving 40 ns in six different stages is:

```text
240 ns
```

which can be meaningful.

---

# 70. Why “lock-free” doesn't automatically mean fast

A common mistake is:

```text
mutex bad
lock-free good
```

Not necessarily.

A lock-free queue with multiple producers constantly doing:

```text
compare_exchange
```

on the same cache line can create enormous coherence traffic.

For example:

```text
Core 1 ─┐
Core 2 ─┼── CAS on same atomic
Core 3 ─┤
Core 4 ─┘
```

may be slower than redesigning the architecture:

```text
one owner
one writer
many readers
```

The right objective is:

```text
minimize coordination
```

not:

```text
maximize clever atomic algorithms.
```

That is exactly why Gross's queue design uses one producer.

---

# 71. Why copying isn't automatically bad

Another common slogan is:

```text
zero-copy = always faster
```

Gross pushes back indirectly on this.

Small contiguous copies are extremely optimized.

Modern `memcpy` can be astonishingly fast.

Sometimes:

```text
copy 64 bytes
```

is cheaper than designing:

```text
reference-counted shared objects
pointer ownership
cross-thread lifetime tracking
heap allocation
synchronization
```

So ask:

```text
What is the complete cost?
```

not:

```text
Did I eliminate memcpy?
```

---

# 72. FPGA vs software is a latency/flexibility spectrum

You should picture:

```text
Flexibility ↑

Python models / research
        │
general C++
        │
optimized C++
        │
kernel bypass C++
        │
FPGA
        │
ASIC

Performance / determinism ↑
```

Going downward means increasingly:

```text
specialized
harder to change
more expensive to engineer
```

This is why actual firms combine technologies rather than moving the entire trading stack into hardware.

---

# 73. Measurement methodology is part of system design

For every optimization you should record:

```text
Before
After
Difference
Confidence
Workload
CPU model
Compiler
Compiler flags
Kernel
Topology
NUMA placement
Data distribution
```

Because:

```text
“linear search is faster”
```

is incomplete.

Correct statement:

```text
“linear search was faster for this order-book workload,
with this distribution, size and machine.”
```

That qualification is exactly the engineering mindset the talk promotes.

---

# 74. Performance optimization workflow derived from the talk

A robust workflow is:

```text
1. Define latency requirement
              ↓
2. Build representative workload
              ↓
3. Measure entire latency distribution
              ↓
4. Look at application/domain behaviour
              ↓
5. Identify architectural bottleneck
              ↓
6. Run broad hardware analysis
              ↓
7. Drill into dominant bottleneck
              ↓
8. Change one design assumption
              ↓
9. Benchmark
              ↓
10. Validate tails, not merely mean
              ↓
11. Test whole-system behaviour
              ↓
12. Deploy
              ↓
13. Continuously monitor regression
```

Notice that:

```text
rewrite assembly
```

doesn't even appear near the beginning.

---

# 75. The deeper lesson about Big-O

This talk should change how you think about algorithms in systems programming.

At university:

```text
O(log N) < O(N)
```

Correct asymptotically.

But actual execution time is closer to:

$$
T \approx
N_{instructions}C_{instruction}
+ N_{branchmiss}C_{branchmiss}
+ N_{L1miss}C_{L1miss}
+ N_{L2miss}C_{L2miss}
+ N_{L3miss}C_{L3miss}
+ N_{DRAM}C_{DRAM}
+\cdots
$$

Big-O tells you how some operation counts grow.

It doesn't tell you the constants.

For:

```text
N = 1000
```

a simple:

```text
O(N)
```

loop with perfect locality can beat:

```text
O(log N)
```

pointer-heavy or branch-heavy code.

As:

```text
N → enormous
```

the asymptotic advantage eventually dominates.

But HFT engineering works with **real N**, not \(N \to \infty\).

---

# 76. A useful latency hierarchy to keep in your head

Exact numbers differ by CPU generation, but conceptually think:

```text
register
   ↓
L1 cache
   ↓
L2 cache
   ↓
L3 cache
   ↓
local DRAM
   ↓
remote NUMA memory
   ↓
PCIe / NIC
   ↓
network
```

Every time your algorithm falls further down this hierarchy, cost grows.

This is why:

```text
memory layout
```

can matter more than:

```text
number of C++ statements.
```

---

# 77. What makes this talk particularly important

The examples all point to the same style of thinking:

```text
std::map
    ↓
theoretically elegant
    ↓
poor locality
    ↓
replace


vector
    ↓
better locality
    ↓
tail due to actual update pattern
    ↓
reverse vector


binary search
    ↓
good complexity
    ↓
branch misses
    ↓
branchless search


branchless search
    ↓
better
    ↓
still more complicated
    ↓
linear search wins


kernel network
    ↓
general purpose
    ↓
unnecessary machinery
    ↓
bypass


general queue
    ↓
supports many semantics
    ↓
unneeded complexity
    ↓
specialized SPMC queue


atomic update every message
    ↓
cache coherence
    ↓
reserve blocks


profiling after release
    ↓
regressions creep in
    ↓
continuous measurement


optimize one CPU
    ↓
shared L3 interference
    ↓
optimize whole server
```

That pattern is the real content of the talk.

---

# 78. What you should *not* take away

Do **not** interpret this talk as:

```text
std::map is always bad
linear search is always fastest
std::function is forbidden
FPGA is always better
kernel is slow
shared memory solves everything
write your own queue
use alignas(64) everywhere
lock-free = low latency
zero-copy = fast
```

The correct conclusion is:

```text
GENERAL-PURPOSE ABSTRACTION
           ↓
has a cost
           ↓
sometimes the cost is negligible
           ↓
sometimes it dominates your latency
           ↓
only measurement + workload knowledge tells you which
```

---

# 79. A compact HFT engineering philosophy from the talk

If I compressed the entire 90-minute presentation into one idea, it would be:

```text
KNOW YOUR DATA
      +
KNOW YOUR HARDWARE
      +
KEEP THE HOT PATH SIMPLE
      +
REMOVE UNNECESSARY WORK
      +
SPECIALIZE WHERE IT MATTERS
      +
MEASURE EVERYTHING
      +
OPTIMIZE THE WHOLE SYSTEM
      +
KEEP MEASURING FOREVER
```

That is much closer to how serious low-latency engineering works than:

```text
write C++
use lock-free structures
use FPGA
```

---

# 80. Where this fits into learning HFT architecture

This video is excellent, but it is **not a complete HFT architecture course**. It goes deeply into selected pieces:

```text
✓ market-making motivation
✓ FPGA/software relationship
✓ order books
✓ data-oriented structures
✓ branch prediction
✓ cache locality
✓ CPU profiling
✓ kernel bypass
✓ shared memory
✓ concurrent queues
✓ NUMA
✓ profiling
✓ whole-server contention
```

It does **not** deeply teach:

```text
exchange protocols
ITCH/OUCH
CME MDP
FIX/SBE
feed arbitration
packet gap recovery
hardware timestamping
PTP
colocation
cross-connects
pre-trade risk architecture
order gateways
drop copies
position systems
kill switches
reconciliation
strategy modeling
exchange matching engines
SmartNIC design
FPGA feed handlers
production failover
```

Those are the next layer.

Interestingly, Gross's earlier **“Trading at Light Speed: Designing Low Latency Systems in C++”** talk covers some complementary areas such as system design, working-set sizing, concurrent data, seqlocks, CPU C/P states, shared LLC optimization and metrics. ([YouTube][8])

---

# 81. The most important connection to your low-latency systems work

This talk gives a very useful framework for thinking about projects such as taking an existing general-purpose system and trying to make it HFT-grade.

The goal is **not**:

```text
take software
+
apply 50 optimization tricks
=
low latency
```

Instead you ask every layer:

```text
What semantic guarantees does this layer provide?

Do I actually need them?

What latency/variance do those guarantees cost?

Can the architecture remove the requirement entirely?
```

For example:

```text
Do we need multiple writers?
      ↓
No
      ↓
remove synchronization problem
```

rather than:

```text
find fastest multiple-writer lock
```

or:

```text
Do we need dynamic resizing?
      ↓
No
      ↓
bounded preallocated structure
```

rather than:

```text
optimize allocator
```

or:

```text
Do processes on same server need TCP?
      ↓
No
      ↓
shared memory
```

rather than:

```text
make TCP slightly faster
```

This distinction—**remove the work instead of optimizing the work**—is probably the single most powerful systems-design idea in the presentation.

---

## Final mental model

Whenever you design a low-latency component, mentally run this checklist:

```text
             Is this operation necessary?
                       │
                ┌──────┴──────┐
               NO             YES
               │               │
            remove it    Can it be simpler?
                               │
                        Can data be contiguous?
                               │
                       Can ownership be single?
                               │
                       Can allocation be upfront?
                               │
                       Can communication be local?
                               │
                       Are branches predictable?
                               │
                       Does data fit in cache?
                               │
                         Is NUMA correct?
                               │
                         Measure PMU
                               │
                         Measure tails
                               │
                         Measure whole box
                               │
                         Production audits
```

That is the way I would study this talk: **not as a collection of optimization recipes, but as a methodology for engineering machines where a nanosecond has economic value.**

If you're building toward understanding actual **Optiver/Hudson River Trading/Jane Street-style HFT architecture**, the logical next notes after this are **“from Ethernet packet arriving at the NIC to an order leaving the machine”**—covering NIC RX rings → kernel bypass → feed handler → order book → strategy → risk → order gateway → NIC TX, including CPU cores, caches, NUMA, FPGA/SmartNIC placement, exchange protocols and approximate latency budget at every stage.

[1]: https://cppcon.org/2024grossprerelease/?utm_source=chatgpt.com "2024 Keynote on Ultrafast Trading Systems in C++: David Gross, prerelease | CppCon"
[2]: https://www.josherich.me/podcast/when-nanoseconds-matter-ultrafast-trading-systems-in-c-david-gross-cppcon-2024 "When Nanoseconds Matter: Ultrafast Trading Systems in C++ - David Gross - CppCon 2024"
[3]: https://studylib.net/doc/28045690/when-nanoseconds-matter "Ultrafast Trading Systems in C++ | CppCon 2024"
[4]: https://www.intel.com/content/www/us/en/docs/vtune-profiler/cookbook/2023-2/top-down-microarchitecture-analysis-method.html?utm_source=chatgpt.com "Top-down Microarchitecture Analysis Method"
[5]: https://docs.amd.com/r/en-US/ug1586-onload-user/ef_vi?contentId=BOFxy7Gif8zBb26BhDBz_g&utm_source=chatgpt.com "ef_vi - ef_vi - UG1586"
[6]: https://docs.redhat.com/en/documentation/red_hat_enterprise_linux_for_real_time/9/html/optimizing_rhel_9_for_real_time_for_low_latency_operation/assembly_setting-cpu-affinity-on-rhel-for-real-time_optimizing-rhel9-for-real-time-for-low-latency-operation?utm_source=chatgpt.com "Chapter 12. Setting CPU affinity on RHEL for Real Time | Optimizing RHEL 9 for Real Time for low latency operation | Red Hat Enterprise Linux for Real Time | 9 | Red Hat Documentation"
[7]: https://docs.redhat.com/en/documentation/red_hat_enterprise_linux_for_real_time/9/pdf/optimizing_rhel_9_for_real_time_for_low_latency_operation/real-time-kernel-tuning-in-rhel-9_optimizing-rhel9-for-real-time-for-low-latency-operation?utm_source=chatgpt.com "Optimizing RHEL 9 for Real Time for low latency operation | Red Hat Enterprise Linux for Real Time | 9 | Red Hat Documentation"
[8]: https://www.youtube.com/watch?v=8uAW5FQtcvE&utm_source=chatgpt.com "Trading at light speed: designing low latency systems in C++ - David Gross - Meeting C++ 2022 - YouTube"
