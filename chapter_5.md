# Chapter 5 — Building a Production Market-Data Feed Handler + Order Book in C++

This chapter is where the previous architecture becomes **actual software**.

So far, we have treated this block as one box:

```text
NIC
 ↓
Feed Handler
 ↓
Order Book
 ↓
Strategy
```

Now we are going to open that box.

A production-quality market-data path is closer to:

```text
                         NIC
                          │
                    RX descriptor
                          │
                          ▼
                  raw Ethernet frame
                          │
             ┌────────────┴────────────┐
             │ Ethernet / VLAN / IP    │
             │ UDP framing parser      │
             └────────────┬────────────┘
                          │
                          ▼
                    MoldUDP64
                          │
                  session + sequence
                          │
              ┌───────────┴───────────┐
              │                       │
           Feed A                  Feed B
              │                       │
              └──────────┬────────────┘
                         ▼
                  A/B ARBITRATOR
                         │
                    gap detector
                         │
           ┌─────────────┴──────────────┐
           │                            │
         normal                      recovery
           │                            │
           ▼                            │
                     ordered messages ◄─┘
                         │
                         ▼
                    ITCH decoder
                         │
                         ▼
                   Order-ID store
                         │
                         ▼
                   Price levels
                         │
                         ▼
                  Local order book
                         │
              ┌──────────┴─────────┐
              │                    │
          raw book             derived state
                              bid/ask/mid/etc.
                                   │
                                   ▼
                                Strategy
```

The most important principle for the entire chapter is:

> **The feed handler is not just a parser. It is a deterministic state-reconstruction engine.**

It receives unreliable network packets and must produce a **correct, ordered, low-latency representation of exchange state**.

Nasdaq TotalView-ITCH itself is a sequenced order-level feed, typically transported using a higher-level delivery protocol such as MoldUDP64. Nasdaq deliberately assigns each instrument a small `Stock Locate` integer intended to support rapid array-based lookup. ([NASDAQ Trader][1])

---

## 1. First define exactly what we are building

For this chapter, use Nasdaq TotalView-ITCH over MoldUDP64 as the concrete example.

Not because every exchange works exactly this way, but because it exposes nearly every important concept:

```text
UDP multicast
A/B feeds
MoldUDP64 sequencing
retransmission
ITCH binary messages
order IDs
full order depth
cancel/delete/replace
trade messages
snapshot recovery
```

Later the same architecture can be adapted to:

```text
CME MDP / SBE
Eurex EOBI
NYSE Pillar
other proprietary feeds
```

We want an implementation with these properties:

```text
Correct
Deterministic
Zero allocation during steady-state
Bounded memory
No locks on primary book-update path
No blocking syscalls on primary hot path
Cache-conscious
NUMA-conscious
Replayable
Recoverable after packet loss
Observable
Benchmarkable
```

That last group is what separates:

```text
“my ITCH parser works”
```

from:

```text
“this could become a production trading component.”
```

---

# 2. The central architecture

I recommend dividing the code logically into these components:

```text
┌────────────────────────────────────────────┐
│                PacketSource                │
│ DPDK / ef_vi / AF_XDP / replay file       │
└───────────────────┬────────────────────────┘
                    │ PacketView
                    ▼
┌────────────────────────────────────────────┐
│              NetworkParser                 │
│ Ethernet → VLAN → IP → UDP                 │
└───────────────────┬────────────────────────┘
                    │ UDP payload
                    ▼
┌────────────────────────────────────────────┐
│             MoldUDPDecoder                 │
│ session / firstSeq / messageCount          │
└───────────────────┬────────────────────────┘
                    │ Sequenced packet
                    ▼
┌────────────────────────────────────────────┐
│            FeedArbitrator                  │
│ A/B dedupe + gap detection + ordering      │
└───────────────────┬────────────────────────┘
                    │ strictly ordered ITCH
                    ▼
┌────────────────────────────────────────────┐
│               ItchDecoder                  │
│ A/E/C/X/D/U/P/...                          │
└───────────────────┬────────────────────────┘
                    │ typed event
                    ▼
┌────────────────────────────────────────────┐
│               BookBuilder                  │
│ order index + price levels                 │
└───────────────────┬────────────────────────┘
                    │ book delta
                    ▼
┌────────────────────────────────────────────┐
│             FeatureEngine                  │
│ BBO / mid / imbalance / queue data         │
└───────────────────┬────────────────────────┘
                    │
                    ▼
                 Strategy
```

However, these are **logical boundaries**.

They do not necessarily imply:

```text
8 processes
8 threads
8 queues
```

That would often be a latency disaster.

A very fast implementation may compile all of this into one processing loop on one pinned core.

---

# 3. Logical modularity versus physical separation

This distinction is extremely important.

You want your source code to look conceptually modular:

```cpp
network.decode(...)
mold.decode(...)
sequencer.accept(...)
itch.decode(...)
book.apply(...)
features.update(...)
strategy.on_event(...)
```

But the generated hot path should ideally resemble:

```text
packet
  ↓
few function calls/inlines
  ↓
book mutation
  ↓
strategy
```

not:

```text
packet
 ↓
thread queue
 ↓
parser
 ↓
thread queue
 ↓
sequencer
 ↓
thread queue
 ↓
book
 ↓
IPC
 ↓
strategy
```

Modularity:

```text
good.
```

Runtime boundaries:

```text
only when justified.
```

---

# 4. The three principal processing architectures

There are three useful architectures.

### Architecture A — Run to completion

```text
ONE CORE

RX
↓
decode
↓
sequence
↓
book
↓
features
↓
strategy
↓
risk
↓
TX
```

This is the lowest-coordination design.

DPDK explicitly supports this run-to-completion model: one logical core polls an RX descriptor ring, processes packets and transmits through a TX ring. ([DPDK][2])

---

### Architecture B — Feed core + strategy cores

```text
Feed core
   │
RX → book
   │
   ├────► Strategy 1
   ├────► Strategy 2
   └────► Strategy 3
```

Useful when:

```text
strategy computation is heavy
many strategies consume same feed
```

But now you need:

```text
shared-state mechanism
or
SPMC queue.
```

The Gross 2022 chapter fits here.

---

### Architecture C — staged pipeline

```text
RX
 ↓
Parser
 ↓
Book
 ↓
Strategy
```

different cores.

Useful for throughput-heavy workloads.

Usually the least attractive for the absolute lowest tick-to-trade latency because every stage transition can mean:

```text
cache-line transfer
queue
atomic metadata
waiting
```

DPDK itself describes both run-to-completion and pipeline models. ([DPDK][2])

---

# 5. Start at the actual RX ring

With DPDK, your application does not normally manipulate hardware RX descriptors manually.

You call something such as:

```cpp
rte_eth_rx_burst(...)
```

and the PMD handles descriptors.

Conceptually:

```text
                  RAM

   RX DESCRIPTOR RING

 [desc 0] ────────► [packet buffer]
 [desc 1] ────────► [packet buffer]
 [desc 2] ────────► [packet buffer]
 [desc 3] ────────► [packet buffer]
     ▲
     │
     NIC DMA
```

DPDK's `rte_eth_rx_burst()` scans completed descriptors, returns associated `rte_mbuf` objects to the caller, and replenishes consumed descriptors with fresh buffers from a memory pool. ([DPDK][3])

![Image](https://codilime.com/img/codilime_nic-packet-flow-p3-2x-1-1-1-1.png)

![Image](https://images.openai.com/static-rsc-4/LsBR3UQZe58ZpYuiVNlcKpJ7PXGKlzcLxTHL7tJ66L-dAb1p2Igj_0TYuXQSjXNVCLJSYlTriWq5lp745q_XXDXaOhCSwUMy_fRbvMoKzSxei8IaNufyaV6N-e3hr9Fg9eOG_7QZrD-uStBieeZhaQGBrvjVzBjwDBJkZJcG8XoVnwuh5MJI5NApnHnKzPY8?purpose=fullsize)

The important ownership transition is:

```text
available buffer
    │
    ▼
NIC owns it
    │
packet DMA occurs
    ▼
application owns packet
    │
process
    ▼
buffer returned/recycled
    │
    └──────────► NIC may use again
```

---

# 6. What a DPDK receive loop roughly looks like

Conceptually:

```cpp
while (running) {
    rte_mbuf* packets[32];

    uint16_t n =
        rte_eth_rx_burst(port_id,
                         queue_id,
                         packets,
                         32);

    for (uint16_t i = 0; i < n; ++i) {
        process_packet(packets[i]);
        rte_pktmbuf_free(packets[i]);
    }
}
```

That example is educational rather than final production code.

But notice what is missing:

```text
recv()
epoll()
interrupt wait
kernel socket queue
```

The CPU continuously polls.

DPDK's PMDs directly poll RX/TX descriptors in userspace and normally avoid packet-arrival interrupts. ([DPDK][2])

---

# 7. Polling burst size is a latency/throughput parameter

Why:

```cpp
packets[32]
```

instead of one packet?

Batching allows work such as:

```text
descriptor processing
mempool operations
doorbells
prefetching
```

to be amortized.

But suppose:

```text
packet 1 arrives
```

and your code waits until:

```text
packet 32 arrives.
```

That would add latency.

Fortunately `rte_eth_rx_burst()` returns what is currently available; it does not require filling the requested maximum.

Still, your own processing policy matters.

You can:

```text
receive N
process N
```

or:

```text
receive a small burst
process immediately
```

or dynamically adjust based on backlog.

DPDK explicitly exposes burst sizes so applications can choose these policies. ([DPDK][2])

---

# 8. A better abstraction: `PacketView`

You don't want the rest of the feed handler dependent on:

```cpp
rte_mbuf*
```

because then:

```text
live DPDK input
```

and:

```text
historical replay
```

require different code paths.

Use a minimal internal representation:

```cpp
struct PacketView {
    const std::byte* data;
    std::uint32_t length;

    std::uint64_t rx_timestamp;
    std::uint8_t  feed_id;
};
```

Then:

```text
DPDK
AF_XDP
ef_vi
PCAP replay
unit tests
```

can all produce the same:

```text
PacketView.
```

That gives you deterministic replay without putting a heavyweight abstraction in the hot path.

---

# 9. Avoid `std::function` for hot callbacks

You might design:

```cpp
std::function<void(const PacketView&)> handler;
```

Don't default to that in the hot path.

Prefer static dispatch:

```cpp
template<typename Handler>
void poll_packets(Handler& handler);
```

Then the compiler sees the concrete handler type and can potentially inline through:

```text
RX
→ parser
→ decoder
→ book.
```

This connects directly to Gross's `std::function` discussion.

---

# 10. Parse the network layers

Your received bytes may look like:

```text
┌─────────────────┐
│ Ethernet header │
├─────────────────┤
│ VLAN optional   │
├─────────────────┤
│ IPv4            │
├─────────────────┤
│ UDP             │
├─────────────────┤
│ MoldUDP64       │
├─────────────────┤
│ ITCH            │
└─────────────────┘
```

Your parser wants to transform:

```text
byte pointer + length
```

into:

```text
MoldUDP payload pointer + length.
```

---

# 11. Don't build C++ objects for every network header

A naive parser could construct:

```cpp
EthernetHeader eth;
IPv4Header ip;
UDPHeader udp;
```

copy each into temporary objects, then decode.

You often do not need that.

The raw packet already contains those bytes.

Treat it as:

```text
immutable byte span.
```

Something conceptually like:

```cpp
std::span<const std::byte>
```

is ideal for expressing this.

---

# 12. But don't blindly cast packet bytes to packed structs

This is tempting:

```cpp
auto* header =
    reinterpret_cast<const MyHeader*>(data);
```

Potential problems include:

```text
alignment
strict aliasing
object lifetime
padding
compiler assumptions
unaligned accesses
endianness.
```

Many network programs use packed structures carefully, but for robust portable parsing I prefer explicit load helpers.

For example:

```cpp
inline std::uint32_t load_be32(
    const std::byte* p)
{
    return
        (std::uint32_t(std::to_integer<unsigned char>(p[0])) << 24) |
        (std::uint32_t(std::to_integer<unsigned char>(p[1])) << 16) |
        (std::uint32_t(std::to_integer<unsigned char>(p[2])) <<  8) |
        (std::uint32_t(std::to_integer<unsigned char>(p[3])));
}
```

For performance-critical builds, small `memcpy` + `std::byteswap` can also compile very efficiently on modern compilers.

Measure both.

---

# 13. Bounds checking versus speed

You cannot simply assume:

```text
every packet is valid.
```

Malformed/truncated packets can otherwise cause:

```text
out-of-bounds read
crash
corruption.
```

But checking every individual field repeatedly also adds work.

A good design validates structure progressively.

For example:

```text
packet >= Ethernet minimum?
      ↓
EtherType expected?
      ↓
packet >= IP header?
      ↓
IP length valid?
      ↓
UDP length valid?
      ↓
Mold payload within UDP payload?
```

Once you prove:

```text
ITCH message has exactly expected length
```

you don't need an individual bounds check for every field inside that fixed message.

So:

```text
validate at boundaries
```

rather than:

```text
defensively branch before every byte access.
```

---

# 14. Production feeds are known traffic

A general-purpose packet analyzer must accept:

```text
IPv4
IPv6
TCP
UDP
ICMP
VLAN
fragmentation
thousands of protocols.
```

Your HFT channel may know:

```text
Ethernet + optional known VLAN
IPv4
UDP
known destination multicast IP
known port
known source
```

Specialize for that.

NIC hardware filters can often ensure only the expected feed reaches the RX queue.

Then the fast path is optimized for:

```text
expected packet shape.
```

Rare anomalies go to a cold error path.

---

# 15. Keep packet validation, but make failures cold

Conceptually:

```cpp
if (likely(valid_market_packet(packet))) {
    process_fast(packet);
} else {
    handle_bad_packet_cold(packet);
}
```

Bad packet handling might:

```text
increment counters
capture bytes
alert
mark feed unhealthy
```

but it should not bloat the hot instruction path.

---

# 16. MoldUDP64 framing

Once you reach the UDP payload, Nasdaq MoldUDP64 begins.

Its downstream header contains:

```text
Session         10 bytes
Sequence Number  8 bytes
Message Count    2 bytes
```

The sequence number is the sequence of the **first message** in that packet; subsequent contained messages are implicitly sequential. A message count of zero is a heartbeat, while `0xFFFF` denotes end of session. ([NASDAQ Trader][4])

Conceptually:

```text
MoldUDP64 packet

┌────────────────────┐
│ Session            │
├────────────────────┤
│ First sequence     │ = 1000
├────────────────────┤
│ Message count      │ = 3
├────────────────────┤
│ Message len        │
├────────────────────┤
│ ITCH seq 1000      │
├────────────────────┤
│ Message len        │
├────────────────────┤
│ ITCH seq 1001      │
├────────────────────┤
│ Message len        │
├────────────────────┤
│ ITCH seq 1002      │
└────────────────────┘
```

Nasdaq's current MoldUDP64 documentation uses precisely this session/first-sequence/message-count structure. ([NASDAQ Trader][4])

---

# 17. The feed handler tracks message sequence, not merely packet sequence

This distinction matters.

Imagine:

```text
Packet A
firstSeq = 100
count = 5
```

It contains:

```text
100
101
102
103
104
```

Next packet:

```text
firstSeq = 105
```

Correct.

Your state is:

```cpp
std::uint64_t expected_sequence;
```

After A:

```text
expected_sequence = 105.
```

---

# 18. A packet parser should expose message slices

Something conceptually like:

```cpp
struct MoldPacket {
    SessionId session;
    std::uint64_t first_sequence;
    std::uint16_t count;

    const std::byte* messages;
    std::uint32_t messages_length;
};
```

Then iterate:

```cpp
for (...) {
    auto len = load_be16(ptr);
    ptr += 2;

    MessageView msg {
        .data = ptr,
        .length = len
    };

    ptr += len;
}
```

Nasdaq's MoldUDP formats use per-message length prefixes after the packet header. ([NASDAQ Trader][5])

---

# 19. Never allocate a `Message` per ITCH message

Avoid:

```cpp
auto msg = new ItchMessage(...);
```

for every update.

Your traffic may contain millions of events.

Instead:

```text
packet bytes
    ↓
temporary view
    ↓
decode fields
    ↓
mutate persistent book state.
```

Lifetime of the decoded message itself can be:

```text
a few tens of nanoseconds.
```

It doesn't need heap ownership.

---

# 20. Sequence logic is more subtle than three `if`s

Start with the simple case.

Let:

```text
S = packet first sequence
C = count
E = expected sequence

packet covers [S, S+C)
```

There are several cases.

---

## Case 1 — exact next packet

```text
S == E
```

Example:

```text
Expected = 100
Packet   = 100..104
```

Process immediately.

Then:

```text
Expected = 105.
```

---

# 21. Case 2 — entire duplicate

Example:

```text
Expected = 105
Packet   = 100..104
```

Then:

$$
S+C\le E
$$

Everything has already been processed.

Drop it.

This is exactly what happens when Feed B delivers a duplicate after Feed A won the race.

---

# 22. Case 3 — overlapping duplicate

This case is easily missed.

Suppose:

```text
Expected = 103
Packet   = 100..105
```

You already processed:

```text
100
101
102
```

but haven't processed:

```text
103
104
105.
```

Do **not** discard the entire packet.

Skip the duplicate prefix and consume:

```text
103+
```

So your logic should reason about intervals, not merely:

```cpp
if (packet.seq < expected)
    drop();
```

That would be incorrect.

---

# 23. Case 4 — forward gap

Suppose:

```text
Expected = 100
Packet first = 105.
```

Missing:

```text
100..104.
```

You cannot safely apply 105 yet.

Why?

Sequence 100 might delete the order that message 105 modifies.

So:

```text
105 must wait.
```

---

# 24. The arbitrator's core algorithm

Conceptually:

```cpp
packet_start = p.first_sequence;
packet_end   = p.first_sequence + p.count;

if (packet_end <= expected) {
    // complete duplicate
    discard(p);
}

else if (packet_start <= expected) {
    // packet contains expected message
    skip(expected - packet_start);
    process_remaining(p);
    expected = packet_end;
    drain_pending();
}

else {
    // gap
    hold_out_of_order(p);
    request_or_wait_for_missing_data();
}
```

This sequencing code is arguably more important for correctness than your book container choice.

---

# 25. Feed A/B arbitration

Suppose:

```text
Feed A: seq 100 arrives 700 ns after exchange send
Feed B: seq 100 arrives 900 ns after exchange send
```

Process A immediately.

Do not wait for B.

Then when B comes:

```text
duplicate → discard.
```

Next sequence:

```text
Feed A loses 101
Feed A gives 102

Feed B gives 101
```

Now:

```text
B repairs A's gap.
```

The arbitrator's job is therefore:

```text
accept earliest valid copy
+
preserve strict sequence order.
```

---

# 26. Don't add an arbitrary “wait for B” sleep

Bad:

```text
detect A gap
sleep 10 microseconds
wait for B
```

A sleep introduces:

```text
scheduler interaction
jitter
guaranteed delay.
```

The arbitrator should be event driven:

```text
poll both feeds
hold future sequences
process missing sequence immediately
when either source provides it.
```

---

# 27. Three practical A/B architectures

### One core polls both feeds

```text
Core 4:

poll A
poll B
arbitrate
decode
book
```

Advantages:

```text
no cross-core handoff
single owner
easy sequencing.
```

Disadvantage:

```text
one core must sustain both feed rates.
```

This is excellent when capacity permits.

---

### Two RX cores + arbitrator

```text
Core 3 → Feed A ─┐
                 ├──► Core 5 arbitrator
Core 4 → Feed B ─┘
```

Higher throughput.

But now:

```text
two queues
cache transfers
more synchronization.
```

---

### Hardware arbitration

```text
Feed A ─┐
        ├──► FPGA / SmartNIC → ordered stream
Feed B ─┘
```

Lowest software burden, much harder engineering.

---

# 28. The packet-lifetime problem during gaps

This is a subtle production issue.

Suppose seq 100 is missing but you receive:

```text
101
102
103
104
...
```

You need to hold them.

If the data lives in DPDK `rte_mbuf`s, you can retain those buffers.

But then:

```text
held packets
    ↓
not returned to mempool
    ↓
RX descriptor replenishment consumes others
    ↓
large gap
    ↓
mempool exhaustion
    ↓
packet drops
```

So recovery buffering must be:

# bounded.

---

# 29. Two designs for gap buffering

### Design A — retain NIC packet buffers

```text
gap slot
  │
  └── rte_mbuf*
```

Advantages:

```text
zero copy
simple.
```

Risks:

```text
holds scarce RX buffers.
```

Good if gap windows are tiny and tightly bounded.

---

### Design B — copy into recovery slab

```text
rte_mbuf
  ↓ small memcpy
preallocated recovery buffer
  ↓
free rte_mbuf immediately.
```

One extra copy.

But:

```text
network pool remains healthy
recovery capacity is independent
bounded memory.
```

This is an excellent example of why:

> zero-copy is not automatically the best architecture.

---

# 30. Gap-buffer structure

Don't use:

```cpp
std::map<uint64_t, Packet>
```

in the hot path unless measured and justified.

Sequence numbers increase monotonically.

That property allows a bounded ring.

Suppose recovery window:

```text
8192 sequence slots.
```

Then:

```cpp
slot = sequence & (8192 - 1);
```

Each slot can store:

```text
sequence
status
packet reference/offset
```

Before using a slot, verify stored sequence matches, because the ring wraps.

This converts sequence lookup into:

```text
array indexing.
```

Exactly the kind of specialization HFT likes.

---

# 31. But Mold packets contain multiple sequences

You have two main choices.

### Packet-level gap buffer

Store entire packets with:

```text
first sequence
last sequence.
```

Then search/drain packets.

Less metadata.

---

### Message-level reorder ring

Split packet into:

```text
seq → MessageView
```

and place each sequence into a fixed ring.

Easy ordered draining.

But if views reference original packet memory you still have packet-lifetime ownership problems.

For a first serious implementation:

> use a bounded packet-level recovery buffer and handle overlap carefully.

It's easier to keep correct.

---

# 32. Retransmission

If neither feed delivers the missing sequence, MoldUDP64 defines a rerequest mechanism.

The client supplies:

```text
session
first requested sequence
number of requested messages
```

and the rerequest server sends a normal downstream packet back by unicast. ([NASDAQ Trader][6])

So:

```text
multicast
    ↓
gap 5001
    ↓
check B
    ↓
still absent
    ↓
request 5001+
    ↓
unicast recovery data
```

---

# 33. Keep receiving live traffic during recovery

Never do:

```text
gap
 ↓
stop live input
 ↓
recover
 ↓
rejoin live.
```

You'll usually create more gaps.

Instead:

```text
             live packets
                 │
                 ▼
           recovery buffer
                 ▲
                 │
        retransmitted missing
                 │
                 ▼
        ordered sequence stream
```

Recovery and live reception happen concurrently at the **logical** level.

They may still run on one core.

---

# 34. Feed-state machine

Use explicit feed state.

For example:

```text
STARTING
   │
   ▼
SYNCING
   │
   ▼
LIVE
   │
   ├──── gap small ──► RECOVERING
   │                      │
   │                      └────► LIVE
   │
   ├──── major loss ─► STALE
   │
   └──── session end ─► CLOSED
```

Do not encode this as scattered booleans such as:

```cpp
bool gap;
bool snapshot;
bool live;
bool reconnecting;
```

because you eventually create impossible states such as:

```text
live = true
snapshot = true
gap = false
stale = true.
```

Use one explicit state machine.

---

# 35. Book validity is separate from network connectivity

You can have:

```text
NIC link = UP
```

while:

```text
book = INVALID.
```

For example:

```text
sequence gap unresolved.
```

Or:

```text
book = VALID
```

while:

```text
one redundant feed is DOWN
```

because Feed B still gives a complete stream.

So separate health domains:

```text
PhysicalLinkState
FeedAState
FeedBState
SequenceState
BookState
TradingState
```

---

# 36. Strategy must know book validity

Never let:

```text
book recovery
```

silently produce:

```text
apparently normal best bid/ask.
```

Use:

```cpp
enum class BookState {
    Empty,
    Synchronizing,
    Live,
    Stale,
    Recovering,
    Invalid
};
```

Strategy:

```cpp
if (book.state != BookState::Live)
    cancel_or_do_not_quote();
```

That single design choice can prevent catastrophic trading on stale state.

---

# 37. Large-gap recovery: GLIMPSE snapshot

For a large loss or late startup, requesting hundreds of thousands of individual messages may be inappropriate.

Nasdaq's GLIMPSE snapshot provides order-book state together with a TotalView-ITCH sequence number identifying the point represented by the snapshot. Clients then continue real-time ITCH processing from that sequence. ([NASDAQ Trader][7])

Conceptually:

```text
start buffering live ITCH
        │
        ▼
request snapshot
        │
        ▼
build snapshot book
        │
snapshot says sequence N
        │
        ▼
replay buffered N+1...
        │
        ▼
LIVE
```

---

# 38. Snapshot installation should not mutate the live book halfway

Bad:

```text
clear live book
↓
begin receiving snapshot
↓
add 10%
↓
strategy can see partial snapshot
↓
add 20%
...
```

Use two state instances:

```text
Book A = current/invalid
Book B = snapshot under construction
```

When B is complete and caught up:

```text
atomic/logical switch
A ← B.
```

This is:

# double-buffered state reconstruction.

---

# 39. ITCH decoding begins only after ordering

This ordering is important:

Bad:

```text
parse ITCH
update book
then discover sequence gap.
```

Correct:

```text
Mold sequencing
       ↓
establish correct message order
       ↓
ITCH semantic processing
       ↓
book mutation.
```

You may parse enough of the packet for arbitration, but irreversible state mutation occurs only after sequence correctness is established.

---

# 40. ITCH messages relevant to the displayed book

For a basic TotalView book builder, the important lifecycle looks like:

```text
Add
 ↓
Execute / Cancel
 ↓
Delete

or

Add
 ↓
Replace
 ↓
new order ID
```

Current ITCH message families include:

```text
A/F  Add Order
E    Order Executed
C    Order Executed With Price
X    Order Cancel
D    Order Delete
U    Order Replace
```

The specification describes `X` as partial cancellation, `D` as full removal and `U` as replacing the old order with a new reference number. ([NASDAQ Trader][1])

---

# 41. Create typed decoded events

Do not push raw protocol bytes deep into your book.

Decode immediately into small internal values.

For example:

```cpp
struct AddOrder {
    std::uint64_t order_id;
    std::uint32_t quantity;
    std::uint32_t price;
    std::uint16_t stock_locate;
    Side side;
};
```

But you don't necessarily need to construct this object in memory.

The decoder can directly call:

```cpp
book.on_add(
    stock,
    order_id,
    side,
    price,
    qty
);
```

This can be entirely inlined.

---

# 42. Dispatch by ITCH message type

First byte:

```text
'A'
'F'
'E'
'C'
'X'
'D'
'U'
'P'
...
```

Conceptually:

```cpp
switch (type) {
    case 'A': decode_add(...);     break;
    case 'F': decode_add_mpid(...);break;
    case 'E': decode_execute(...); break;
    case 'C': decode_exec_price(); break;
    case 'X': decode_cancel(...);  break;
    case 'D': decode_delete(...);  break;
    case 'U': decode_replace(...); break;
    ...
}
```

A switch over a small byte domain is generally an excellent starting point.

Benchmark before attempting elaborate lookup tables.

---

# 43. Validate message length before decoding

Because ITCH message types have known formats.

Conceptually:

```cpp
case 'X':
    if (unlikely(length != CANCEL_LENGTH))
        malformed_message(...);

    decode_cancel(...);
```

Then the decoder can read fixed offsets safely.

Protocol validity failures should trigger:

```text
metrics
capture
feed-health action
```

not undefined behavior.

---

# 44. Endianness

Nasdaq binary ITCH integers use unsigned big-endian network byte order. ([NASDAQ Trader][8])

x86 is normally little-endian.

So bytes:

```text
00 00 01 00
```

represent:

```text
256
```

but reading them natively as a little-endian `uint32_t` would be wrong.

Therefore decode explicitly.

---

# 45. Six-byte timestamp

ITCH uses a 6-byte timestamp representing nanoseconds since midnight for many messages. ([NASDAQ Trader][1])

C++ doesn't have:

```cpp
uint48_t
```

so implement:

```cpp
inline std::uint64_t load_be48(
    const std::byte* p)
{
    std::uint64_t x = 0;

    for (int i = 0; i < 6; ++i) {
        x =
            (x << 8) |
            std::to_integer<unsigned char>(p[i]);
    }

    return x;
}
```

Then store internally as:

```cpp
std::uint64_t exchange_ns;
```

Don't convert to:

```text
hours/minutes/seconds string
```

on the hot path.

---

# 46. Instrument lookup should be an array

Nasdaq's `Stock Locate` is especially interesting.

Nasdaq says it is a low-valued integer intentionally intended to function as an array index for fast instrument lookup. It is assigned daily, remains stable intraday, but shouldn't be assumed to remain the same on subsequent days. ([NASDAQ Trader][1])

That invites:

```cpp
InstrumentState instruments[65536];
```

Then:

```cpp
auto& instrument =
    instruments[stock_locate];
```

No:

```text
hash("AAPL")
string compare
tree traversal.
```

Beautiful mechanical sympathy.

---

# 47. Reference-data initialization

ITCH's Stock Directory messages tell you:

```text
Stock Locate → symbol / metadata
```

So startup looks approximately:

```text
session begins
    ↓
Stock Directory messages
    ↓
instrument[locate] initialized
    ↓
order traffic begins.
```

Do not persist yesterday's:

```text
Stock Locate 1234 = AAPL
```

and assume it remains valid today.

Nasdaq explicitly warns that locate assignments can change between trading days. ([NASDAQ Trader][1])

---

# 48. Now the difficult part: the order store

An Add Order gives you:

```text
Order ID
stock
side
price
quantity.
```

Later you may receive:

```text
Cancel Order ID
```

without repeating every piece of original data.

So you need:

$$
OrderID
\rightarrow
OrderState.
$$

Conceptually:

```text
Order ID 987123
       │
       ▼
┌─────────────────┐
│ Stock = 123     │
│ Side = Buy      │
│ Price = 1000100 │
│ Qty = 500       │
└─────────────────┘
```

---

# 49. Why this lookup is hot

Every:

```text
execute
cancel
delete
replace
```

requires locating the order.

So the structure gets touched millions of times.

This is one of your most important data structures.

---

# 50. Don't immediately reach for `std::unordered_map`

A standard implementation is convenient:

```cpp
std::unordered_map<OrderId, Order>
```

but traditionally this can mean:

```text
bucket lookup
  ↓
pointer
  ↓
node allocation
  ↓
node memory
```

plus allocation/deallocation churn.

That's exactly what the Gross talks warn about.

But also:

> don't write a broken custom hash table merely to avoid `std::unordered_map`.

For your first correct implementation, a well-tested flat/open-addressing hash table is preferable.

After profiling, you can build a specialized fixed-capacity table if needed.

---

# 51. Fixed-capacity flat order index

A production-oriented conceptual representation:

```cpp
struct OrderIndexEntry {
    std::uint64_t order_id;
    std::uint32_t slot;
};
```

stored contiguously:

```text
[entry][entry][entry][empty][entry][entry]
```

Lookup:

```text
hash(order_id)
    ↓
array index
    ↓
linear/SIMD probing
```

Benefits:

```text
contiguous
predictable
no node allocations
prefetch-friendly.
```

---

# 52. Load factor matters

If the table is almost completely full:

```text
95%
```

collision probe chains can become expensive.

For low tail latency, you typically prefer spare capacity.

Rather than asking:

```text
Can this table store 10 million orders?
```

ask:

```text
What does p99 lookup probing look like
at realistic peak occupancy?
```

Provision around **peak live-order count**, not average live-order count.

---

# 53. Order pool

Instead of allocating:

```cpp
new OrderNode
```

on every Add message, allocate a pool at startup.

Conceptually:

```text
ORDER POOL

slot 0
slot 1
slot 2
...
slot N
```

A free-list gives available indexes.

```text
Add
 ↓
take free slot
 ↓
initialize
```

Delete:

```text
remove from structures
 ↓
return slot to free list.
```

No steady-state heap interaction.

---

# 54. Use indexes rather than pointers where helpful

An order node might be:

```cpp
struct OrderNode {
    std::uint64_t id;

    std::uint32_t quantity;
    std::uint32_t price;

    std::uint32_t prev;
    std::uint32_t next;

    std::uint16_t stock;
    std::uint8_t side;
};
```

Where:

```text
prev / next
```

are pool indexes.

Use a special sentinel such as:

```cpp
constexpr uint32_t INVALID =
    UINT32_MAX;
```

Benefits include:

```text
32-bit links rather than 64-bit pointers
compact representation
stable references
easy serialization/replay
pool ownership.
```

---

# 55. Why 32-bit indexes can be better than pointers

Pointer:

```text
8 bytes
```

on 64-bit systems.

Index:

```text
4 bytes.
```

If you have:

```text
20 million orders
```

saving a few bytes per order can materially reduce:

```text
working-set size
cache pressure
memory bandwidth.
```

Again:

> bytes are performance.

---

# 56. Hot/cold field splitting

Do you need the instrument symbol:

```text
"AAPL"
```

when canceling order 847219?

No.

Do you need:

```text
MPID attribution string
```

for every strategy calculation?

Probably not.

Separate:

### Hot state

```text
order id
price
qty
side
instrument id
links
```

from:

### Cold state

```text
symbol
human-readable names
attribution
audit metadata
debugging fields.
```

The hot structure should contain only what the critical path consumes.

---

# 57. Production order book has two different jobs

This is an important design choice.

### Job 1 — reconstruct L2 price levels

Strategy wants:

```text
Best Bid
Best Ask
volume at prices.
```

### Job 2 — reconstruct MBO/FIFO queue

Strategy wants:

```text
individual order ordering
queue position.
```

Those have different storage requirements.

---

# 58. Minimal L2 builder

To maintain aggregated levels, every order needs:

```text
order ID
stock
side
price
remaining qty.
```

When Add:

```text
levelQty[price] += qty
```

Cancel:

```text
levelQty[price] -= cancelled
```

Execute:

```text
levelQty[price] -= executed
```

Delete:

```text
levelQty[price] -= remaining.
```

You don't necessarily need a FIFO linked list.

---

# 59. MBO queue builder

For queue-position modelling you additionally maintain exact order order:

```text
Price 100.00

head
 ↓
[Order 7]
    ↓
[Order 29]
    ↓
[Order 104]
    ↓
tail
```

So each level keeps:

```cpp
struct PriceLevel {
    Price price;
    Quantity total_quantity;
    uint32_t head;
    uint32_t tail;
    uint32_t order_count;
};
```

Each order has:

```text
prev
next.
```

Now you can model:

```text
who is ahead of whom.
```

---

# 60. Order-book visualization

A price level is not merely one number; it can be a queue of individual orders.

![Image](https://images.openai.com/static-rsc-4/o-j9kxv18Wkhft0hkVIETOxZcDIrOIqDXbQXhlAQ8vlT-9nSpSw0TT3BWOHArwQueS_om_GR6Wa0sm3fUc7j5cHJKWN99tqslkFNildH-faMONsPOdhNvDc2X1P5kECZmUrnLjk5IpRfjESOpd-fapoXkY29-l-N4_-ibSphMYpzX1P9cNV4MxB8nV3EU-k_?purpose=fullsize)

![Image](https://www.researchgate.net/profile/Peter_Beling/publication/297725489/figure/download/fig1/AS%3A338431147298818%401457699681518/Limit-order-book-diagram-A-new-buy-limit-order-arrives-at-price-bt-increasing-the.png)

For queue-sensitive HFT strategies, this difference between aggregate depth and individual FIFO structure matters enormously.

---

# 61. How do we organize price levels?

You need fast operations:

```text
find price level
insert level
remove level
best bid
best ask.
```

There is no universally best representation.

Useful options include:

```text
sorted contiguous vector
dense tick array
paged sparse array
flat tree/B-tree
specialized hash + best-price tracking.
```

---

# 62. Option A — sorted vector

For bids:

```text
worst ─────────────────► best

[99.90][99.91][99.95][100.00]
                          ↑
                         best
```

Best price is:

```cpp
levels.back();
```

Updates are strongly concentrated near the best levels.

This is exactly why Gross's reversed-vector order book performed well.

Advantages:

```text
compact
cache-friendly
excellent iteration
easy best-price lookup.
```

Disadvantages:

```text
insert/remove shifts elements.
```

For ~hundreds or ~thousands of levels, this may still beat theoretically superior trees.

Measure your real feed.

---

# 63. Option B — dense price ladder

Suppose price is represented in ticks:

```text
Price = 10,000 ticks.
```

Then:

```cpp
levels[10000]
```

directly accesses the level.

Advantages:

```text
O(1)
no search
predictable.
```

Problem:

```text
price universe may be huge.
```

Imagine:

```text
0 to $1,000,000
```

at:

```text
$0.0001 resolution.
```

A direct global array becomes enormous.

---

# 64. Option C — paged price ladder

A production compromise:

```text
price
 ↓
page number + offset
```

Only allocate/pre-create pages used in the permitted price region.

Conceptually:

```text
Page 0:
[levels...]

Page 1:
[levels...]

Page 400:
[levels...]
```

Benefits:

```text
near O(1) lookup
less wasted memory
stable levels.
```

More implementation complexity.

---

# 65. What I would build first

For your initial HFT project:

> **use contiguous sorted price-level vectors.**

Why?

Because:

```text
easy to validate
easy to replay
easy to benchmark
small code footprint
cache friendly.
```

Only replace them if profiling proves level lookup/insertion is a bottleneck.

This follows Gross's philosophy rather than prematurely building an exotic price-ladder allocator.

---

# 66. Price representation

Never use:

```cpp
double
```

for order-book keys unless the protocol specifically forces a conversion.

ITCH prices are fixed-point integers.

Store:

```cpp
using Price = std::uint32_t;
```

with documented scale.

Example:

```text
wire price = 1,000,100

scale = 10,000

display = 100.0100.
```

Integer comparisons are:

```text
exact
deterministic
fast.
```

---

# 67. Add Order handler

Conceptually:

```text
ITCH Add

id = 10
stock = 400
side = BUY
price = 100.00
qty = 500
```

Steps:

```text
1. validate order ID not already live
2. allocate OrderNode
3. initialize node
4. insert OrderID → slot index
5. find/create price level
6. append order to level tail
7. increase aggregate quantity
8. update best price if necessary
9. emit book delta / feature update
```

For FIFO reconstruction:

```text
append to tail
```

because messages arrive in exchange event order.

---

# 68. Cancel handler

ITCH `X` represents partial cancellation. ([NASDAQ Trader][1])

Suppose:

```text
Order 10 remaining = 500
cancelled = 200
```

Then:

```text
order.qty = 300
level.total -= 200.
```

Do not remove order.

Unless:

```text
remaining becomes zero
```

according to your defensive semantics; normally full removal is represented separately by Delete.

---

# 69. Delete handler

`D` says the remaining order is no longer accessible and should be removed from the book. ([NASDAQ Trader][1])

Steps:

```text
1. lookup OrderID
2. retrieve its remaining qty
3. level.total -= remaining
4. unlink order from price-level queue
5. erase ID mapping
6. return node to pool
7. if level empty, remove price level
8. update best if required.
```

---

# 70. Execute handler

Suppose:

```text
Order 10 qty = 500

Executed shares = 100
```

New:

```text
qty = 400
level total -= 100.
```

If execution consumes the full remaining amount:

```text
remove order
remove empty level if needed.
```

Execution is semantically different from cancellation because strategy analytics may care:

```text
trade occurred
```

versus:

```text
liquidity disappeared.
```

So even if book mutation is similar:

```text
quantity decreases
```

the event meaning is different.

---

# 71. Order Executed With Price

ITCH also supports executions where the execution price differs from the displayed order price.

Important distinction:

```text
displayed book location
```

is determined by the resting order state.

The execution-price field is useful for:

```text
trade analytics / time-and-sales.
```

Your displayed level still needs to be reduced using the referenced order's stored book state.

This is why the order store must be authoritative for local display reconstruction.

---

# 72. Replace handler

ITCH Replace supplies:

```text
old order ID
new order ID
new displayed quantity
new price.
```

Nasdaq says the new reference number is used for subsequent updates, while side, stock symbol and attribution are retained from the original Add. ([NASDAQ Trader][9])

Conceptually:

```text
remove old order
       ↓
create new order
       ↓
new ID
new qty
new price.
```

Do not merely mutate:

```cpp
old.price = new_price;
```

and leave the old identity/queue state intact.

---

# 73. Non-displayed Trade message

This is an easy implementation bug.

ITCH `P` Trade messages describe executions involving non-displayable order types.

Nasdaq explicitly says these messages **do not affect the displayed book** and may be ignored by a system whose sole purpose is maintaining the Nasdaq display book. ([NASDAQ Trader][9])

So:

```text
P Trade
```

should feed:

```text
time-and-sales
trade statistics
signals
```

but not:

```text
displayed order-book quantity mutation.
```

---

# 74. Broken trades

Similarly, Nasdaq says Broken Trade messages are relevant to time-and-sales/statistics but do not affect the current displayed book. ([NASDAQ Trader][10])

This shows why your decoder should separate:

```text
BookEvent
TradeEvent
ReferenceEvent
AuctionEvent
SystemEvent.
```

Do not send everything through:

```cpp
book.update();
```

---

# 75. ITCH event classification

A useful design is:

```text
                    ITCH
                      │
        ┌─────────────┼─────────────┐
        ▼             ▼             ▼
     Book          Trades        Control
     events        events         events
        │             │             │
        ▼             ▼             ▼
 Order Book      Signals/T&S   Feed state
```

Examples:

### Book

```text
A F E C X D U
```

### Trade/statistical

```text
P Q B
```

### Reference/control

```text
Stock Directory
Trading Action
System Event
NOII
```

Exact classifications should match the current venue specification.

---

# 76. Keep the order book internally consistent

Every handler should preserve invariants.

For example:

$$
LevelQty
=
\sum_{orders\ at\ level}OrderQty.
$$

And:

```text
every live OrderID exists exactly once.
```

And:

```text
every order belongs to exactly one price level.
```

And:

```text
level head.prev == invalid
level tail.next == invalid.
```

And:

```text
empty level has no head/tail.
```

And:

```text
best bid is maximum active bid.
```

These invariants are more important than optimization.

---

# 77. Assert them aggressively in debug/replay builds

Production hot build:

```text
minimal checks.
```

Replay/test build:

```text
validate entire book after every event.
```

For example:

```cpp
#ifdef BOOK_PARANOIA
book.validate_all();
#endif
```

Then replay millions of recorded messages.

This is how you gain confidence before turning checks off.

---

# 78. Debug correctness and hot-path correctness can use different builds

One codebase can support:

```text
DEBUG / VALIDATION

bounds checks
full invariants
extra metadata
logging
slow diagnostics
```

and:

```text
PRODUCTION

same business logic
minimal instrumentation
no formatting
cold error paths.
```

Never maintain two unrelated implementations if you can avoid it.

Otherwise:

```text
the code you tested
≠
the code you trade.
```

---

# 79. Derived market state

Strategies shouldn't recompute everything from scratch.

Maintain incrementally:

```text
best bid
best ask
bid qty
ask qty
mid
spread
imbalance
possibly microprice
trade flow.
```

If only one level changes:

```text
update only affected features.
```

---

# 80. Top-of-book structure

Something compact:

```cpp
struct TopOfBook {
    Price bid_price;
    Quantity bid_qty;

    Price ask_price;
    Quantity ask_qty;
};
```

If best level doesn't change, changing quantity requires only:

```text
one field update.
```

If best level disappears:

```text
move to previous level.
```

With best-at-end vector layout:

```cpp
best = levels.back();
```

very cheap.

---

# 81. Don't calculate `double mid` unless needed

Suppose:

```text
bid = 10000 ticks
ask = 10001 ticks.
```

You can represent doubled midpoint:

$$
Mid2=Bid+Ask=20001
$$

instead of:

```text
10000.5.
```

This avoids floating point and preserves exact half ticks.

Only convert when model logic actually needs a floating representation.

---

# 82. Update imbalance incrementally

Top-level imbalance:

$$
I=
\frac{Q_b-Q_a}
{Q_b+Q_a}.
$$

Do not necessarily evaluate a division after every update if strategy only needs a threshold.

For example compare:

$$
Q_b > 2Q_a
$$

using integer arithmetic.

This is the kind of tiny optimization to consider **after** architecture is correct.

---

# 83. Strategy-facing API

Don't force strategy to poll giant book objects.

Provide direct event callbacks.

Conceptually:

```cpp
strategy.on_book_change(
    stock,
    old_bbo,
    new_bbo,
    exchange_timestamp,
    rx_timestamp
);
```

Or if strategy lives in same loop:

```cpp
if (features_changed)
    strategy.evaluate(instrument);
```

That eliminates redundant work when an ITCH event doesn't affect the features the strategy cares about.

---

# 84. Event filtering is very valuable

Suppose strategy trades:

```text
100 instruments.
```

Feed contains:

```text
10,000 instruments.
```

Your feed handler may need to keep the full book for some purposes, but strategy doesn't need callbacks for all of them.

Use the small Stock Locate index:

```cpp
if (instrument[locate].strategy_mask != 0)
    notify_strategy(...);
```

This is vastly better than:

```text
string matching symbols.
```

---

# 85. Feature dependency masks

Even among subscribed instruments, strategy may care only about:

```text
BBO
```

not every depth update.

If an order changes level 300 and top five remain unchanged:

```text
do we really need strategy evaluation?
```

Perhaps no.

Maintain dependency flags such as:

```text
BBO changed?
Top5 changed?
trade occurred?
queue of my level changed?
```

Then call only relevant strategy logic.

This can remove enormous unnecessary computation.

---

# 86. Queue-position model hooks

Since Chapter 4 discussed queue position, an MBO book can maintain strategy-specific order-level data.

Suppose you place your own order at:

```text
100.00
```

and estimate:

```text
3000 shares ahead.
```

Every public event at that level can update:

```text
queue_ahead.
```

With order-level ITCH you can do better than aggregate heuristics because you know individual displayed order additions/cancels and their event order.

Hidden liquidity remains unobservable.

---

# 87. Keep public book and own orders logically separate

Remember:

```text
ITCH public book
```

and:

```text
OUCH private order session
```

are different feeds.

Do not inject your own order into the ITCH book merely because you sent it.

Wait for the correct public-book event if the strategy needs the public representation.

Maintain separate:

```text
ExchangePublicBook
OwnOrderState.
```

Then combine them in strategy logic.

---

# 88. Hardware timestamps

Packet metadata should ideally contain:

```text
exchange message timestamp
RX NIC hardware timestamp
CPU processing timestamp.
```

These are different.

Example:

```text
ITCH exchange timestamp:
   event produced at venue

NIC RX timestamp:
   packet reached your interface

TSC timestamp:
   parser began/ended.
```

With these, you can build latency decomposition.

---

# 89. Do not put metrics formatting on the hot path

Bad:

```cpp
printf("processed seq=%llu\n", seq);
```

Terrible for latency.

Instead increment per-core counters:

```cpp
++metrics.messages;
++metrics.adds;
```

or store small timing samples into a preallocated ring.

A background process can:

```text
read/aggregate/export.
```

---

# 90. Which metrics are essential?

At minimum track:

| Domain   | Metrics                                 |
| -------- | --------------------------------------- |
| Network  | packets, bytes, RX drops                |
| Feed     | sequence, gaps, duplicates              |
| A/B      | winner A/B counts, differential latency |
| Recovery | rerequests, recovered messages          |
| Parser   | malformed messages                      |
| Book     | live orders, levels, errors             |
| Latency  | RX→decode, RX→book, RX→strategy         |
| Capacity | RX ring/mempool pressure                |
| Health   | stale age, last packet time             |

The important ones aren't merely totals.

Also record:

```text
rates
maxima
histograms
p99/p99.9.
```

---

# 91. A/B differential latency is very useful

For sequences seen on both feeds:

```text
timestamp_A(seq)
timestamp_B(seq)
```

calculate:

$$
\Delta_{AB}
=
T_B-T_A.
$$

Then monitor distribution:

```text
which path is usually faster?
how often does leadership switch?
did one feed suddenly become 20 µs slower?
```

This is valuable operational intelligence.

---

# 92. Sequence lag

Suppose:

```text
last received sequence = 20,000
last processed sequence = 19,000.
```

Even if no packets were technically lost:

```text
you are 1000 messages behind.
```

That's effectively stale market data.

Track:

```text
received sequence
processed sequence
```

separately where architecture permits.

---

# 93. Queue backlog is often more dangerous than packet loss

A packet drop:

```text
visible
```

because sequence gap appears.

CPU overload can produce:

```text
perfectly complete data
```

that is:

```text
200 µs late.
```

No sequence gap.

Therefore measure:

```text
packet age
sequence processing delay
RX timestamp → book timestamp.
```

A feed can be:

```text
correct but uselessly stale.
```

---

# 94. Memory preallocation

At startup:

```text
allocate packet buffers
allocate order pool
allocate hash tables
reserve price-level vectors
allocate recovery buffers
allocate metric rings.
```

Then:

```text
touch memory.
```

This ensures pages become mapped before the market opens.

Steady state:

```text
no malloc
no new
no vector capacity growth.
```

---

# 95. `reserve()` is not enough if you don't know peak size

Suppose:

```cpp
orders.reserve(1'000'000);
```

but market reaches:

```text
1,000,001.
```

Suddenly:

```text
reallocation
copy
page faults
latency spike.
```

Capacity planning is part of feed-handler design.

You need real peak estimates plus safety margin.

---

# 96. What should happen when a pool is exhausted?

Not:

```text
fall back to malloc.
```

That hides the capacity failure.

Instead:

```text
mark book unsafe
stop trading
raise critical alarm
preserve evidence.
```

A bounded production system must fail:

# explicitly.

---

# 97. NUMA layout

Suppose:

```text
NIC PCIe root → NUMA 0
```

Then ideally:

```text
RX core      → NUMA 0
packet pool  → NUMA 0
order book   → NUMA 0
strategy     → NUMA 0
TX NIC       → NUMA 0.
```

DPDK specifically recommends local NUMA memory for mbufs associated with local PCIe interfaces and discourages sharing RX/TX queues across logical cores. ([DPDK][2])

Bad:

```text
NIC node 0
↓
packet memory node 1
↓
strategy node 0.
```

Now every update can cross the socket interconnect.

---

# 98. One RX queue → one owning core

Do not casually have:

```text
Core A ─┐
        ├── same RX queue
Core B ─┘
```

DPDK assumes one logical core polls a particular RX queue unless you explicitly build synchronization around it; sharing introduces locks/coordination and hurts performance. ([DPDK][2])

The same principle appears throughout HFT:

> make ownership explicit.

---

# 99. Cache-line alignment

Use cache-line alignment for metadata shared across cores:

```cpp
struct alignas(64) SharedCounters {
    std::atomic<uint64_t> ...
};
```

But do not do:

```cpp
struct alignas(64) OrderNode
```

for every 20-byte order unless there is a measured reason.

Otherwise:

```text
20 useful bytes
44 wasted bytes
```

and your order-store WSS explodes.

---

# 100. Prefetching

Suppose Mold packet contains several ITCH messages.

While processing message \(i\), you may know where message \(i+1\) begins.

Potentially:

```cpp
__builtin_prefetch(next_msg);
```

Likewise if order-ID lookup hashes to slot \(h\), you can potentially prefetch the order-table region before all decoding work finishes.

But:

> manual prefetching can make things worse.

Modern CPUs already prefetch sequential data well.

Use PMU measurements before adding it.

---

# 101. Software prefetch is most useful when future address is known early

For example:

```text
decode order ID
    ↓
calculate hash
    ↓
prefetch hash-table bucket
    ↓
decode timestamp/quantity/etc.
    ↓
lookup bucket
```

Now useful independent work occurs while memory loads.

That's a more promising use than prefetching a perfectly sequential message buffer that hardware already handles.

---

# 102. Branch prediction

Message-type distribution may be highly skewed:

```text
Add
Cancel
Execute
```

far more common than rare system messages.

Keep rare handlers out of line.

Something like:

```cpp
[[gnu::cold]]
void malformed_message(...);
```

may help keep I-cache hot.

Again, benchmark on actual compilers.

---

# 103. Avoid converting protocol values repeatedly

Bad:

```text
wire price
 ↓
double
 ↓
decimal class
 ↓
strategy ticks
```

Better:

```text
wire fixed integer
 ↓
internal integer ticks
```

once.

Similarly:

```text
Stock Locate
```

should stay a numeric index throughout the hot path.

---

# 104. AF_XDP alternative

If you aren't ready for DPDK/ef_vi, AF_XDP provides an interesting Linux-native option.

AF_XDP uses a registered memory region called UMEM and four relevant ring types:

```text
FILL
RX
TX
COMPLETION.
```

The kernel documentation describes the rings as SPSC structures and RX/TX descriptors as references into UMEM. ([Linux Kernel Documentation][11])

Conceptually:

```text
                    UMEM

           [packet][packet][packet]

               ▲           │
               │           ▼
             RX Ring     TX Ring
```

Notice the same recurring design:

```text
bounded memory
ownership transfer
rings
single producer/consumer.
```

---

# 105. ef_vi alternative

For even more direct low-latency NIC programming, AMD's current Onload documentation describes `ef_vi` as a Layer-2 API targeting low-latency packet processing. ([AMD Documentation][12])

Architecturally your upper layers should not care whether packet input came from:

```text
DPDK
AF_XDP
ef_vi
PCAP replay.
```

That is why the minimal `PacketView` boundary is useful.

---

# 106. Live and replay paths should converge immediately

Bad:

```text
LiveFeedHandler.cpp

ReplayFeedHandler.cpp

completely different logic.
```

Eventually they diverge.

Instead:

```text
       Live DPDK
           │
           ▼
       PacketView
           │
           │
Replay ────┘
           │
           ▼
      SAME DECODER
           │
           ▼
       SAME BOOK
```

This is one of the most valuable architecture decisions you can make for development.

---

# 107. Deterministic packet replay

Capture:

```text
packet bytes
RX timestamp
feed ID
```

Then replay exactly those packets.

Expected:

```text
same ordered input
       ↓
same book state.
```

Hash end-of-run:

```text
orders
levels
best bid/ask
sequence.
```

Repeated replay must produce identical hash.

---

# 108. Build golden test files

Create tiny synthetic feeds.

Example:

```text
seq 1 Add BID order A qty100 @100
seq 2 Add BID order B qty200 @100
seq 3 Cancel A qty40
seq 4 Execute B qty50
seq 5 Delete A
```

Expected:

```text
B remains
qty=150
price=100.
```

Then test:

```text
ordered packets
duplicate packets
A/B duplicates
missing packet
out-of-order recovery
partial packet overlap.
```

You want tests for transport correctness separately from book correctness.

---

# 109. Fuzz the network parser

Your protocol parser operates on untrusted bytes.

Fuzz:

```text
truncated Ethernet
invalid IPv4 lengths
invalid UDP lengths
Mold count inconsistent with bytes
message length too long
unknown ITCH type
```

Requirements:

```text
never over-read
never corrupt book
never hang.
```

Performance code still needs memory safety.

---

# 110. Property-based tests for book state

Generate random sequences of:

```text
Add
Cancel
Execute
Delete
Replace
```

while ensuring valid semantics.

After each event assert invariants.

Compare optimized implementation with a slow reference book such as:

```text
std::map
std::unordered_map
```

used only in test code.

If both receive identical event streams:

```text
state must match.
```

This is extremely powerful.

---

# 111. Reference implementation + optimized implementation

Build:

```text
SimpleBook
```

first.

Use:

```text
std::map
std::unordered_map
clear readable code.
```

Then build:

```text
FastBook.
```

Replay same dataset.

Compare state after every event.

This gives you confidence while experimenting with exotic memory layouts.

---

# 112. Performance benchmark must use real distributions

Do not benchmark:

```text
50% Add
50% Delete
uniform random prices
```

if real market traffic looks nothing like that.

Record distributions such as:

```text
message type frequency
levels per instrument
distance from best
live-order count
instrument skew
burst sizes.
```

Then replay real/synthetic representative traffic.

This is precisely the lesson from Gross's order-book experiment.

---

# 113. Benchmark end-to-end, not just `on_add()`

Useful microbenchmark:

```text
Book::on_add = 40 ns.
```

But actual hot path is:

```text
RX
↓
Mold
↓
sequence
↓
ITCH
↓
hash lookup
↓
book
↓
features.
```

You need both:

```text
microbenchmarks
```

and:

```text
whole-path benchmarks.
```

---

# 114. Latency histogram points

Timestamp these boundaries:

```text
T0 NIC RX

T1 packet available to poll loop

T2 sequence validation complete

T3 ITCH decoded

T4 book mutation complete

T5 features complete

T6 strategy callback complete.
```

Then calculate:

$$
T_{book}=T_4-T_3
$$

and:

$$
T_{market\ path}=T_5-T_0.
$$

Don't instrument every message permanently with six expensive timestamps unless you have measured the overhead.

Sampling can help.

---

# 115. Performance regression testing

Record baseline:

```text
RX→Book
p50 = X
p99 = Y
p99.9 = Z.
```

CI performance test detects:

```text
p99 regression > threshold.
```

Not just:

```text
all correctness tests passed.
```

This is exactly Gross's:

> staying fast is harder than getting fast.

---

# 116. Feed stale timer

Track:

```text
last RX hardware timestamp
last exchange event timestamp
last processed sequence.
```

If no traffic appears for too long when traffic is expected:

```text
feed → SUSPECT/STALE.
```

Do not let your strategy keep quoting using an old book indefinitely.

The threshold must be market/session-aware because quiet markets can legitimately produce no updates.

---

# 117. Heartbeats matter

MoldUDP64 uses:

```text
message count = 0
```

for heartbeat packets. ([NASDAQ Trader][4])

A heartbeat:

```text
does not mutate book
```

but proves:

```text
channel transport is alive.
```

It can also help identify sequence state without waiting for a market event.

---

# 118. Session changes

MoldUDP64 has an explicit session identifier. ([NASDAQ Trader][4])

Never continue:

```text
expected_sequence from previous session
```

into a new session blindly.

On unexpected session change:

```text
stop mutation
validate state
resynchronize
```

because sequence numbers only make sense within the correct session context.

---

# 119. Startup sequence

A robust startup might be:

```text
1. initialize CPU/NIC
2. allocate/pre-touch memory
3. initialize packet pools
4. join multicast A/B
5. begin buffering real-time packets
6. obtain reference data/snapshot if required
7. build book
8. replay buffered delta after snapshot sequence
9. validate book
10. mark LIVE
11. enable strategy.
```

Note:

> strategy activation is the final step.

Not the first.

---

# 120. Shutdown sequence

Similarly:

```text
1. disable new trading decisions
2. cancel/hand off own orders
3. drain necessary state
4. capture final sequence
5. persist diagnostics
6. leave feed
7. release resources.
```

Do not simply:

```text
kill -9.
```

A production system needs lifecycle discipline.

---

# 121. Recovery after application restart

Imagine feed handler restarts at noon.

Live multicast is already sequence:

```text
80,000,000.
```

You cannot build a correct MBO book from only future deltas.

You need:

```text
snapshot
+
sequence point
+
live delta catch-up.
```

Nasdaq GLIMPSE is designed for this sort of order-book snapshot/recovery use case. ([NASDAQ Trader][13])

---

# 122. Raw capture

Run a side path:

```text
RX packet
  │
  ├────► hot parser
  │
  └────► packet recorder
```

But don't copy synchronously to disk in the feed thread.

Use:

```text
shared memory
secondary NIC feature
capture appliance
bounded async queue
```

depending on requirements.

The recorder should never make the trading path wait.

---

# 123. What should happen on corrupt book detection?

Suppose:

```text
Cancel refers to unknown OrderID.
```

Possible causes:

```text
previous gap
parser bug
logic bug
bad snapshot
unexpected protocol state.
```

Do not simply:

```cpp
return;
```

and continue trading.

That means your local book has diverged from reality.

Correct response is usually something like:

```text
increment critical error
mark channel/book invalid
disable affected strategy
capture context
begin resynchronization.
```

Fail closed.

---

# 124. Unknown Order ID is a very valuable invariant alarm

Because in a correct full-order feed:

```text
Cancel/Delete/Execute
```

should normally reference an order you previously created from Add/Replace.

So:

```text
unknown ID
```

often signals a much deeper synchronization problem.

Treat it as a health event.

---

# 125. Duplicate Add is similarly suspicious

If:

```text
order ID already exists
```

and you receive another Add with the same ID within the same logical session/state:

```text
something is wrong.
```

Possible:

```text
duplicate message escaped arbitration
book wasn't cleared correctly
protocol misunderstanding.
```

Detect it.

---

# 126. Your state model should include generations

When pool slots are reused:

```text
slot 100
```

may first contain Order A, later Order B.

If you expose stable handles outside the book, use:

```text
index + generation.
```

Conceptually:

```cpp
struct Handle {
    uint32_t index;
    uint32_t generation;
};
```

Then stale references can be detected.

Inside a strictly single-owner hot path, raw indexes may be enough, but generations are extremely useful across component boundaries.

---

# 127. Avoid cross-thread pointers into mutable book memory

Suppose strategy thread receives:

```cpp
const OrderNode*
```

pointing into the feed thread's pool.

Meanwhile feed thread removes/reuses that node.

Now strategy can observe:

```text
stale/reused memory.
```

If components are separate threads, publish:

```text
immutable snapshots
compact events
handles with lifetime guarantees.
```

Don't casually share internal pointers.

---

# 128. Single-thread book ownership is ideal

The cleanest model:

```text
Feed/Book Core

owns:
order table
level arrays
reference mapping
book state.
```

No other core mutates these.

Other cores receive:

```text
events
snapshots
derived values.
```

This removes:

```text
book locks
atomic quantity updates
concurrent container mutation.
```

Again:

> ownership is performance.

---

# 129. If strategy lives on same core, it's even simpler

Then:

```text
book
+
strategy
```

can share normal non-atomic memory.

Sequence:

```text
apply market event
 ↓
state now internally coherent
 ↓
call strategy
```

No seqlock.

No atomic.

No queue.

This is why run-to-completion designs can be so powerful.

---

# 130. But keep strategy bounded

If strategy occasionally spends:

```text
100 µs
```

while feed continues to arrive, your RX backlog grows.

So same-core architecture requires:

```text
strategy execution time bounded tightly.
```

Otherwise separate cores may be safer despite the handoff cost.

This is architecture, not ideology.

---

# 131. Backpressure is dangerous

A market feed cannot normally tell the exchange:

```text
Please wait, my strategy is busy.
```

Multicast keeps coming.

Therefore:

```text
producer rate > consumer rate
```

eventually means:

```text
backlog
drop
stale data.
```

Your market-data path must be provisioned for:

# bursts, not averages.

---

# 132. Capacity budgeting

Suppose worst observed burst:

```text
5 million messages/sec
```

for:

```text
10 ms.
```

That's:

```text
50,000 messages
```

in a short burst.

If your processing capacity is:

```text
6 million/sec
```

you can catch up.

If capacity:

```text
4 million/sec
```

backlog can grow indefinitely during sustained bursts.

The book component therefore needs:

```text
headroom.
```

---

# 133. Production objective is not maximum throughput

A parser capable of:

```text
100 million msgs/sec
```

but with:

```text
p99 = 50 µs
```

may be worse for HFT than one capable of:

```text
20 million msgs/sec
```

with:

```text
p99 = 1 µs
```

assuming the feed never exceeds its capacity.

The objective is:

```text
adequate throughput
+
minimum tail latency
+
predictability.
```

---

# 134. Don't benchmark in a vacuum

Hardware environment matters:

```text
CPU frequency
C-states
SMT
NUMA
NIC firmware
PCIe topology
memory channels
compiler
kernel
background processes.
```

Record benchmark configuration.

Otherwise:

```text
version B is 10% faster
```

may actually mean:

```text
CPU turbo happened to be different.
```

---

# 135. Compiler choices

Typical performance build:

```text
-O3
-march=<appropriate target>
LTO possibly
PGO possibly
```

But don't assume:

```text
more compiler aggressiveness = faster.
```

Check:

```text
binary size
instruction cache
assembly
latency.
```

Profile-guided optimization can be especially useful when message distributions are stable enough for the compiler to benefit.

---

# 136. Sanitizers are still essential

Production build:

```text
fast.
```

Testing builds should use:

```text
ASan
UBSan
TSan where appropriate
```

A feed handler performing pointer arithmetic against raw network bytes is exactly the kind of code where sanitizers provide tremendous value.

You don't run them in the final latency benchmark.

You absolutely run them in testing.

---

# 137. A good C++ project layout

For your system, I would use something roughly like:

```text
marketdata/

  net/
    packet_view.hpp
    ethernet_parser.hpp
    dpdk_source.cpp
    replay_source.cpp

  transport/
    moldudp64.hpp
    sequence_tracker.hpp
    feed_arbitrator.hpp
    recovery.hpp

  protocol/
    itch_types.hpp
    itch_decoder.hpp
    endian.hpp

  book/
    order_pool.hpp
    order_index.hpp
    price_level.hpp
    instrument_book.hpp
    book_builder.hpp

  features/
    bbo.hpp
    imbalance.hpp
    queue_model.hpp

  metrics/
    latency.hpp
    counters.hpp

  tests/
    parser_tests.cpp
    sequence_tests.cpp
    book_tests.cpp
    replay_tests.cpp

  benchmark/
    decoder_bench.cpp
    book_bench.cpp
    end_to_end_bench.cpp
```

Notice architecture follows the actual packet path.

---

# 138. The main hot loop

Conceptually, your production loop eventually becomes remarkably small:

```cpp
while (running) {

    auto packets = rx.poll();

    for (auto packet : packets) {

        auto udp =
            network.parse(packet);

        auto mold =
            mold_decoder.parse(udp);

        arbitrator.accept(
            packet.feed_id,
            mold,
            [&](std::uint64_t seq,
                MessageView msg)
            {
                itch.decode(
                    seq,
                    msg,
                    book);
            }
        );
    }
}
```

Inside:

```text
book
```

you may immediately update:

```text
features
strategy.
```

The complicated engineering lives behind this tiny loop.

That's usually a sign of a good architecture.

---

# 139. Compile-time composition

You can go further:

```cpp
using Engine =
    MarketDataEngine<
        DpdkPacketSource,
        MoldUdp64,
        NasdaqItch,
        FastOrderBook,
        MyStrategy
    >;
```

Then the compiler sees the entire composition.

No virtual calls are necessary on the hot path.

For testing:

```cpp
using TestEngine =
    MarketDataEngine<
        ReplaySource,
        MoldUdp64,
        NasdaqItch,
        ReferenceBook,
        TestSink
    >;
```

This is a good use case for C++ templates: **zero-cost component composition**.

---

# 140. Don't template absolutely everything

Excessive templates can produce:

```text
huge compile times
code bloat
instruction-cache pressure
difficult debugging.
```

Use compile-time polymorphism where:

```text
the call occurs millions of times on hot path.
```

Cold management components can happily use:

```text
interfaces
virtual functions
std::string
smart pointers.
```

You don't need HFT-style code everywhere.

---

# 141. Production error policy

Every error needs classification.

### Recoverable

```text
Feed A packet loss
Feed B available
```

Continue.

### Recoverable but temporarily unsafe

```text
both feeds lost sequence
```

Pause affected book/trading, recover.

### Fatal book corruption

```text
internal invariant failure
```

Disable trading and rebuild.

### Infrastructure fatal

```text
NIC unavailable
order pool exhausted
```

Fail trading safely.

Make the policy explicit.

---

# 142. Per-instrument versus channel-level invalidation

Suppose sequence gap occurs on a feed channel containing:

```text
1000 instruments.
```

Until recovery, you generally do not know which missing message's instrument was affected.

Therefore conservative state may require:

```text
entire channel invalid.
```

Not merely the instrument whose next visible message you saw.

Once protocol-specific knowledge allows finer handling, you can refine this, but correctness comes first.

---

# 143. Trading-state messages

Even with a perfectly synchronized book:

```text
instrument may be halted.
```

So:

```text
book valid
```

does not imply:

```text
instrument tradable.
```

Maintain separate:

```text
book validity
market trading status
session phase.
```

Strategy checks all three.

---

# 144. Auction data

NOII / auction imbalance is not the same as continuous-order-book state.

If your strategy trades opening/closing auctions, build a separate:

```text
AuctionState.
```

Don't force auction semantics into:

```text
continuous Level2Book.
```

Different market regimes deserve different state objects.

---

# 145. A production instrument state

Conceptually:

```cpp
struct InstrumentState {
    InstrumentBook book;
    TopOfBook top;

    TradingState trading_state;
    BookState book_state;

    FeatureState features;

    // cold/reference information elsewhere
};
```

Stock Locate indexes this directly:

```cpp
auto& s =
    instruments[locate];
```

---

# 146. Message sequencing and book sequencing must remain coupled

For every applied message, retain:

```text
last_sequence_applied.
```

That means a captured book state can be described as:

```text
Book is valid through sequence N.
```

This is invaluable for:

```text
snapshots
replay
debugging
recovery
comparisons.
```

---

# 147. Snapshot metadata

A local state checkpoint should include:

```text
feed session
sequence
date/session identifier
protocol version
book data
reference-data generation.
```

Otherwise you cannot safely determine whether a snapshot is compatible with a later delta stream.

---

# 148. Don't persist raw pointers

If you serialize the order book:

```text
OrderNode* next
```

is useless after process restart.

Another advantage of index-based pools:

```text
next = 912
```

can potentially survive serialization.

This isn't necessarily why you should choose indexes, but it is a nice architectural property.

---

# 149. Production book validation through independent reconstruction

An excellent operational design is:

```text
FAST BOOK
   │
   │ events
   ▼
strategy

and separately

SLOW CHECKER BOOK
```

The checker can:

```text
replay same feed
use simple containers
compare hashes/BBO/level quantities.
```

It doesn't run in the hot path.

If states diverge:

```text
critical alarm.
```

This gives you independent correctness assurance.

---

# 150. Use redundancy at software level too

You already have:

```text
Feed A
Feed B.
```

You can also have:

```text
Primary market-data engine
Shadow market-data engine.
```

Both reconstruct independently.

Compare:

```text
sequence
BBO
book hash.
```

The shadow engine can use different server/hardware.

This is expensive but very valuable in serious systems.

---

# 151. The book hash

Computing a full cryptographic hash after every event would be too expensive.

But periodically the shadow path can compute:

```text
hash(order IDs, price, qty, etc.)
```

and compare.

Or maintain incremental checksums off-path.

This helps detect silent state divergence.

---

# 152. Important hot-path rule: do not log strings

Bad:

```cpp
logger.info(
   "Added {} shares of {} at {}",
   qty,
   symbol,
   price
);
```

even asynchronously if argument formatting occurs before enqueue.

Instead hot path can record:

```cpp
struct DiagnosticEvent {
    uint64_t tsc;
    uint64_t sequence;
    uint32_t code;
    uint32_t arg1;
    uint64_t arg2;
};
```

Background thread translates:

```text
code 17
```

into:

```text
"unknown cancel ID".
```

---

# 153. Flight recorder

A very useful production concept:

# in-memory flight recorder.

Maintain a circular buffer of the last:

```text
N packets/events
```

in compact binary form.

Normally:

```text
overwrite old entries.
```

On fatal error:

```text
freeze buffer
persist it.
```

Then you can inspect exactly what occurred before the failure.

This is far better than trying to log every message to text.

---

# 154. What not to do

Avoid an architecture like:

```text
recvfrom()
 ↓
JSON deserialize
 ↓
std::queue
 ↓
thread
 ↓
std::unordered_map<string,...>
 ↓
mutex
 ↓
new PriceLevel
 ↓
logger
 ↓
database
 ↓
strategy.
```

It might be fine for a normal trading application.

It is fundamentally the wrong architecture for HFT-like latency.

---

# 155. But don't cargo-cult HFT techniques

Also don't build:

```text
custom allocator
custom hash map
custom memcpy
custom scheduler
custom vector
custom kernel
```

before measuring.

The real philosophy is:

```text
specialize where evidence says specialization matters.
```

Not:

```text
reinvent the C++ standard library for fun.
```

---

# 156. Development roadmap for your project

I would build this system in **seven stages**.

### Stage 1 — Offline ITCH decoder

Input:

```text
binary ITCH messages.
```

Output:

```text
human-readable events.
```

No network.

No optimization.

Goal:

```text
protocol correctness.
```

---

### Stage 2 — Reference order book

Use:

```text
std::unordered_map order IDs
std::map price levels
```

Build perfect semantics.

Add:

```text
A
E
C
X
D
U
```

plus reference/system messages.

This becomes your oracle implementation.

---

### Stage 3 — MoldUDP64 + sequencing

Input:

```text
raw UDP payload.
```

Implement:

```text
session
sequence
message extraction
duplicate detection
gap detection
overlap handling.
```

Unit test brutally.

---

### Stage 4 — A/B arbitration + recovery simulator

Generate:

```text
Feed A
Feed B
```

with deliberate:

```text
loss
duplicates
reordering
delays.
```

Your output must always equal the original logical sequence.

Only after this works should you touch live multicast.

---

### Stage 5 — Fast book

Replace the reference book with:

```text
preallocated order pool
flat order index
contiguous price levels
integer IDs
zero allocation.
```

Run both books side-by-side during replay.

Require identical state.

---

### Stage 6 — Live network path

First:

```text
normal UDP multicast sockets
```

if useful for correctness.

Then move to:

```text
AF_XDP
DPDK
or ef_vi
```

without changing upper layers.

---

### Stage 7 — strategy integration

Finally:

```text
book
 ↓
derived features
 ↓
queue model
 ↓
strategy
 ↓
risk
 ↓
order gateway.
```

Now begin actual:

```text
tick-to-trade benchmarking.
```

---

# 157. The most important thing to understand from Chapter 5

A novice sees the feed handler as:

```text
UDP
 ↓
parse packet
 ↓
update map.
```

A production engineer sees:

```text
                       NETWORK
                          │
                    redundant paths
                          │
                     packet ownership
                          │
                     deterministic
                       sequencing
                          │
               bounded loss/recovery state
                          │
                     binary decoding
                          │
                 reference-data mapping
                          │
                   stable order storage
                          │
                 cache-aware price state
                          │
                   invariant-preserving
                       mutation
                          │
               incremental derived state
                          │
                  strategy publication
                          │
                   latency telemetry
                          │
                replay / verification
```

The hard part is not decoding four integers from an ITCH message.

The hard part is guaranteeing:

> **Every valid market event affects your state exactly once, in exactly the correct order, with bounded latency, and when that cannot be guaranteed your trading system knows immediately that its state is unsafe.**

That is what a production feed handler does.

---

# 158. Final end-to-end hot path

After all the engineering, the runtime critical path should look almost embarrassingly simple:

```text
                    MARKET PACKET

                         │
                         ▼
                    NIC RX queue
                         │
                         ▼
                     PacketView
                         │
                         ▼
             Ethernet/IP/UDP strip
                         │
                         ▼
                    MoldUDP64
                         │
                         ▼
           Sequence + A/B arbitration
                         │
                         ▼
                  ITCH message
                         │
                         ▼
              Stock Locate array
                         │
                         ▼
                 Order-ID lookup
                         │
                         ▼
               Price-level mutation
                         │
                         ▼
                 BBO / features
                         │
                         ▼
                     Strategy
```

And the normal-path properties are:

```text
No heap allocation
No locks
No blocking syscall
No strings
No database
No generic serialization
No unnecessary copy
No cross-core handoff if avoidable
No state mutation before sequencing
No strategy call while book invalid
```

That is the architecture I would aim for.

---

# 159. How Chapter 5 fits into your full HFT architecture

You now have:

```text
CHAPTER 1
Low-latency systems philosophy

        ↓

CHAPTER 2
CPU/data/concurrency design

        ↓

CHAPTER 3
NIC/network/exchange path

        ↓

CHAPTER 4
Microstructure + matching engine

        ↓

CHAPTER 5
PRODUCTION MARKET DATA ENGINE
        │
        ├─ packet ownership
        ├─ multicast
        ├─ sequencing
        ├─ A/B arbitration
        ├─ recovery
        ├─ ITCH decode
        ├─ order store
        ├─ price levels
        ├─ derived state
        └─ strategy interface

        ↓

NEXT:
STRATEGY / QUEUE MODEL
```

The next chapter should now be **Chapter 6 — Building the HFT Strategy Engine + Queue-Position / Fill-Probability Model**.

That is where we should take the book we just built and turn:

```text
Book Event
   ↓
BBO / depth / order flow
   ↓
fair-value calculation
   ↓
queue-position estimate
   ↓
fill probability
   ↓
expected markout
   ↓
inventory/risk
   ↓
JOIN / IMPROVE / CROSS / CANCEL
```

into an actual low-latency decision engine.

After that, Chapter 7 can build the other half of the production system: **OUCH/iLink-style order gateway, order state machine, pre-trade risk, throttling, TX path, acknowledgements, cancel races and recovery**.

Those two chapters will connect this feed handler into a complete trading loop.

[1]: https://classic.nasdaqtrader.com/content/technicalsupport/specifications/dataproducts/NQTVITCHSpecification.pdf?utm_source=chatgpt.com "Nasdaq TotalView-ITCH 5.0"
[2]: https://doc.dpdk.org/guides/prog_guide/ethdev/ethdev.html?utm_source=chatgpt.com "3.1. Poll Mode Driver — Data Plane Development Kit 26.07.0 documentation"
[3]: https://doc.dpdk.org/api/rte__ethdev_8h.html?utm_source=chatgpt.com "DPDK: lib/ethdev/rte_ethdev.h File Reference"
[4]: https://www.nasdaqtrader.com/content/technicalsupport/specifications/dataproducts/moldudp64.pdf?utm_source=chatgpt.com "Session

A Session is a sequence of one or more me"
[5]: https://www.nasdaqtrader.com/content/technicalsupport/specifications/dataproducts/moldudp.pdf?utm_source=chatgpt.com "MoldUDP"
[6]: https://nasdaqtrader.com/content/technicalsupport/specifications/dataproducts/moldudp64.pdf?utm_source=chatgpt.com "Investment Intelligence"
[7]: https://www.nasdaqtrader.com/content/technicalsupport/specifications/dataproducts/NQGlimpseSpecification_5.0.pdf?utm_source=chatgpt.com "ITCH"
[8]: https://classic.nasdaqtrader.com/content/technicalsupport/specifications/dataproducts/PSXTVITCHSpecification_5.0.pdf?utm_source=chatgpt.com "NASDAQ OMX BX TotalView-ITCH 4.0"
[9]: https://www.nasdaqtrader.com/content/technicalsupport/specifications/dataproducts/NQTVITCHspecification.pdf?utm_source=chatgpt.com "message. Firms should retain the side, stock symbol and MPID from the original Add Order message."
[10]: https://www.nasdaqtrader.com/content/technicalsupport/specifications/dataproducts/NQTVITCHSpecification.pdf?utm_source=chatgpt.com "1.5.3 Broken Trade / Order Execution Message"
[11]: https://docs.kernel.org/networking/af_xdp.html?utm_source=chatgpt.com "AF_XDP — The Linux Kernel documentation"
[12]: https://docs.amd.com/r/en-US/ug1586-onload-user/Layer-2-ef_vi-Latency?utm_source=chatgpt.com "Layer 2 ef_vi Latency - Layer 2 ef_vi Latency - UG1586"
[13]: https://www.nasdaqtrader.com/TraderNews.aspx?id=dtn2009-027&utm_source=chatgpt.com "Data Technical News #2009 - 27 REVISED: NASDAQ GLIMPSE Now Available for TotalView-ITCH 4.0f and TotalView-ITCH 3.1f"
