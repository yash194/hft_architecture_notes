Yes. This is the right next chapter.

The two David Gross talks taught you **how to design the software inside the trading machine**. What is still missing is how information physically gets **from the exchange into that software and then back out as an order**.

This chapter connects all of it:

```text
EXCHANGE MATCHING ENGINE
        │
        │ market event
        ▼
market-data encoder
        │
        ▼
Ethernet / multicast network
        │
        ▼
trader NIC
        │
        ▼
DMA + PCIe + RX ring
        │
        ▼
feed arbitration / sequencing
        │
        ▼
ITCH / SBE decoder
        │
        ▼
ORDER BOOK
        │
        ▼
STRATEGY
        │
        ▼
PRE-TRADE RISK
        │
        ▼
ORDER MANAGER / GATEWAY
        │
        ▼
OUCH / iLink / SBE
        │
        ▼
TX ring + NIC
        │
        ▼
EXCHANGE GATEWAY
        │
        ▼
MATCHING ENGINE
```

Once you understand this properly, you have the core mental model for a real electronic/HFT trading stack.

---

# Chapter 3 — From Ethernet Packet to Exchange Order

# 1. Start with the complete HFT architecture

There are actually **several different flows** inside a trading system. Beginners often combine all of them into one.

The most important distinction is between the **market-data path** and the **order-entry path**.

```text
                            EXCHANGE

                 ┌─────────────────────────┐
                 │     MATCHING ENGINE     │
                 └────────────┬────────────┘
                              │
                ┌─────────────┴─────────────┐
                │                           │
                │                           │
         MARKET DATA                    ORDER ENTRY
         one → many                    one ↔ one
                │                           │
                ▼                           ▲
         multicast feeds                    │
                │                           │
                ▼                           │
        ┌───────────────┐                   │
        │ Trading Firm  │                   │
        │      NIC      │                   │
        └───────┬───────┘                   │
                │                           │
                ▼                           │
         Feed Handler                       │
                │                           │
                ▼                           │
          Order Books                       │
                │                           │
                ▼                           │
           Strategy                         │
                │                           │
                ▼                           │
             Risk                           │
                │                           │
                ▼                           │
        Order Manager                       │
                │                           │
                ▼                           │
         Order Gateway ─────────────────────┘
```

Market data tells you:

> What is happening in the market?

Order entry tells the exchange:

> What do I want to do?

Nasdaq illustrates the distinction particularly clearly. TotalView-ITCH is an **outbound full-depth market-data feed**, while OUCH is the low-level native protocol for entering, replacing and cancelling your own orders and receiving updates about them. 

---

# 2. There are actually four planes

A useful professional mental model is:

| Plane                           | Purpose                                                    |
| ------------------------------- | ---------------------------------------------------------- |
| **Market-data plane**           | Exchange → trading firms                                   |
| **Order-entry plane**           | Trading firm ↔ exchange                                    |
| **Recovery/control plane**      | snapshots, retransmissions, session recovery               |
| **Post-trade/monitoring plane** | drop copies, positions, logs, surveillance, reconciliation |

Only the first two normally dominate the **critical tick-to-trade hot path**.

But the other two are necessary to keep the trading system correct.

---

# 3. Before packets: why HFT firms colocate

Imagine your server is:

```text
Mumbai
```

while the exchange matching engine is:

```text
New Jersey
```

No amount of:

```cpp
inline
constexpr
SIMD
```

can compensate for thousands of kilometres of network propagation.

Therefore competitive firms place machines physically near the exchange.

This is **colocation**.

Conceptually:

```text
Exchange Data Center

┌──────────────────────────────────────────────┐
│                                              │
│ Exchange matching engine                    │
│          │                                   │
│          │                                   │
│ Exchange network                             │
│          │                                   │
│         fiber                                │
│          │                                   │
│ Trader A server                              │
│ Trader B server                              │
│ Trader C server                              │
│                                              │
└──────────────────────────────────────────────┘
```

Nasdaq explicitly offers colocation for reduced market-data and order-entry latency at its primary infrastructure, and also provides direct-connect and POP services for firms located elsewhere. ([NASDAQ Trader][1])

CME likewise provides low-latency connectivity/colocation options; its current Aurora GLink service is described as its most performant physical access during its ongoing infrastructure transition. ([CME Group][2])

So the optimization hierarchy begins even before software:

```text
geography
   ↓
fiber path
   ↓
switches
   ↓
NIC
   ↓
OS/networking
   ↓
software
```

---

# 4. Cross-connect

Inside a data centre, your server isn't literally plugged directly into the matching-engine CPU.

You may have something like:

```text
Trader rack
   │
   │ fiber cross-connect
   ▼
Exchange access switch
   │
   ▼
Exchange network
   │
   ▼
gateway / feed infrastructure
```

A **cross-connect** is essentially a dedicated physical connection between infrastructure inside the facility.

Compared with Internet routing:

```text
your server
 ↓
ISP
 ↓
routers
 ↓
WAN
 ↓
exchange
```

you have a much shorter and more controlled path.

---

# 5. Now an event happens inside the exchange

Suppose someone sends:

```text
BUY 100 AAPL @ $200.00
```

The exchange matching engine processes it.

Perhaps the previous book was:

```text
BID               ASK

199.99 × 100       200.01 × 200
199.98 × 300       200.02 × 150
```

and the new order becomes the best bid:

```text
BID               ASK

200.00 × 100       200.01 × 200
199.99 × 100       200.02 × 150
```

The exchange now needs to tell market-data subscribers:

```text
new displayed order
price = 200.00
quantity = 100
side = buy
order id = X
```

That event becomes a market-data message.

---

# 6. The protocol stack: one of the most important concepts

Do not think:

```text
ITCH packet
```

as if ITCH were everything.

There are several layers.

A Nasdaq multicast packet can conceptually be:

```text
┌────────────────────────────────┐
│ Ethernet                        │  Layer 2
├────────────────────────────────┤
│ IPv4                            │  Layer 3
├────────────────────────────────┤
│ UDP                             │  Layer 4
├────────────────────────────────┤
│ MoldUDP64                       │  feed/session framing
├────────────────────────────────┤
│ ITCH message                    │  market semantics
├────────────────────────────────┤
│ ITCH message                    │
├────────────────────────────────┤
│ ...                             │
└────────────────────────────────┘
```

That distinction is crucial.

For Nasdaq:

```text
ITCH
```

answers:

> What does an Add Order / Cancel / Trade message look like?

Whereas:

```text
MoldUDP64
```

answers:

> How are those messages sequenced, packetized and recovered over UDP?

And:

```text
UDP/IP
```

answers:

> How is the datagram transported through the network?

Nasdaq currently supports TotalView-ITCH with MoldUDP64 among its transport options, and its live feed information publishes separate multicast and rerequest endpoints. 

---

# 7. Ethernet

At the lowest relevant networking layer we have an Ethernet frame.

Conceptually:

```text
┌───────────────┐
│ Destination   │
│ MAC address   │
├───────────────┤
│ Source MAC    │
├───────────────┤
│ EtherType     │
├───────────────┤
│ Payload       │
├───────────────┤
│ FCS           │
└───────────────┘
```

On the physical wire there is additional framing overhead such as the preamble and inter-frame gap.

The important concept is:

> The network transmits bits serially.

Therefore even the act of putting a packet onto a link takes time.

If a packet contains \(B\) bits on a link of rate \(R\):

$$
T_{serialization} = \frac{B}{R}
$$

Therefore:

```text
smaller packet
    ↓
less serialization time
```

and:

```text
faster link
    ↓
less serialization time
```

This is one reason binary financial protocols intentionally avoid huge verbose representations.

---

# 8. Why you do not send JSON market data

Imagine an order update as JSON:

```json
{
  "instrument": "AAPL",
  "side": "BUY",
  "price": 200.01,
  "quantity": 100
}
```

You have:

```text
field names
ASCII numbers
quotes
commas
braces
variable-length parsing
```

Compare with a binary record:

```text
instrument_id : uint16
side          : uint8
price         : uint32
qty           : uint32
```

Now the decoder can perform something much closer to:

```text
load bytes
interpret fields
continue
```

rather than expensive text parsing.

This is why ITCH, SBE and other exchange-native protocols are binary.

---

# 9. IPv4

Above Ethernet you typically encounter IP.

The packet now effectively identifies:

```text
source IP
destination IP
```

Market-data destination IPs are often **multicast addresses** rather than the address of one particular trading firm.

---

# 10. UDP

Above IP:

```text
UDP
```

UDP provides a lightweight datagram abstraction.

Conceptually:

```text
source port
destination port
length
checksum
payload
```

What UDP does **not** provide is the critical point:

```text
no guaranteed delivery
no built-in retransmission
no guaranteed ordering
no TCP-like connection
```

You might now ask:

> Why would an exchange use unreliable UDP for mission-critical market data?

Because the alternative guarantee machinery itself costs latency and creates undesirable one-sender/one-receiver semantics.

---

# 11. Why multicast is perfect for market data

Imagine an exchange has:

```text
10,000 subscribers
```

Without multicast the exchange could conceptually need:

```text
copy 1 → firm A
copy 2 → firm B
copy 3 → firm C
...
```

Instead, UDP multicast lets the exchange publish to a group:

```text
                    EXCHANGE
                       │
                       │ one logical feed
                       ▼
              MULTICAST NETWORK
                /      |      \
               /       |       \
              ▼        ▼        ▼
          Firm A    Firm B    Firm C
```

The network performs replication.

FIXP's documentation describes this same general communication pattern: multicast is a one-way publisher-to-many-listeners flow commonly used for market data. ([FIX Trading Community][3])

CME explicitly describes its Market Data Platform as using a **dual-feed UDP multicast architecture**. ([CME Group][4])

---

# 12. Multicast groups

A trading application subscribes to a multicast group.

Conceptually:

```text
233.x.x.x : port Y
```

The network is told:

> I want traffic sent to this multicast group.

One exchange may split instruments across multiple channels:

```text
Channel 1 → group of products
Channel 2 → another group
Channel 3 → ...
```

This distributes load and allows clients to subscribe only to relevant channels.

---

# 13. But UDP can lose packets

Suppose the expected market sequence is:

```text
1050
1051
1052
1053
1054
```

but your machine receives:

```text
1050
1051
1053
1054
```

Then:

```text
1052
```

has disappeared.

Perhaps 1052 was:

```text
DELETE order 8829
```

If you ignore that gap, your local book contains an order that no longer exists.

Now your strategy's market state is wrong.

Therefore:

> **Sequence integrity is fundamental to feed-handler correctness.**

---

# 14. Sequence numbers

Exchange transport protocols therefore associate messages or packets with sequence numbers.

Conceptually:

```text
packet

session = ABC123
first_sequence = 1050
message_count = 3

messages:
    seq 1050
    seq 1051
    seq 1052
```

The receiver maintains:

```cpp
expected_sequence
```

If:

```text
received == expected
```

process normally.

If:

```text
received < expected
```

it is usually a duplicate/old message.

If:

```text
received > expected
```

you have a gap.

---

# 15. Nasdaq MoldUDP64

MoldUDP64 is a particularly good concrete example.

A downstream packet conceptually provides:

```text
session
sequence number
message count
messages...
```

The receiver can therefore determine exactly where it is in the feed.

Nasdaq's receiver procedure explicitly says the client should join the multicast group, examine the session and sequence number, compare it with the next expected sequence, and issue a request when the sequence is not what is expected. 

---

# 16. First line of defence: redundant A/B feeds

Professional market-data feeds commonly arrive through multiple redundant paths.

Think:

```text
                    EXCHANGE

                 ┌──────┴──────┐
                 │             │
              FEED A        FEED B
                 │             │
          network path A  network path B
                 │             │
                 └──────┬──────┘
                        ▼
                  FEED ARBITRATOR
```

Both feeds carry essentially the same logical sequence.

But their network paths differ.

Suppose:

```text
Feed A: 100 101 --- 103 104

Feed B: 100 101 102 103 104
```

Then your receiver can obtain 102 from Feed B.

Databento describes exactly this production pattern: receive two multicast streams, compare packet sequence numbers, and use one feed to fill gaps in the other; it calls this A/B arbitration. ([OWU][5])

---

# 17. Feed arbitration

The feed arbitrator receives both.

Suppose:

```text
Feed A: seq 200 arrives at 12:00:00.000001
Feed B: seq 200 arrives at 12:00:00.000030
```

You normally want:

```text
the first valid copy
```

So:

```text
A 200 arrives
      ↓
forward 200
      ↓
B 200 arrives
      ↓
duplicate → drop
```

Then perhaps:

```text
A 201 missing
B 201 arrives
```

Use B.

Conceptually:

```cpp
if (seq == expected) {
    forward(message);
    ++expected;
}

else if (seq < expected) {
    discard_duplicate();
}

else {
    detect_gap();
}
```

Real arbitration becomes more complicated because packets can contain multiple messages and arrive in bursts or slightly out of order.

---

# 18. Why A/B feeds are not enough

Imagine both feeds lose:

```text
sequence 9281
```

or your server pauses for long enough that it misses many packets.

Now you need a dedicated recovery mechanism.

There are typically two broad strategies:

```text
small gap
    ↓
retransmit missing messages

large/desynchronized state
    ↓
snapshot + resume/replay
```

---

# 19. Retransmission

Nasdaq's MoldUDP64 supports a rerequest mechanism.

The client effectively says:

```text
Session: X
Starting sequence: 9281
Count: 1
```

The retransmission server returns the requested data.

Nasdaq's specification explicitly defines retransmission requests for sequence gaps, and its current feed directory publishes separate rerequest endpoints for TotalView-ITCH channels. ([NASDAQ Trader][6])

But notice something very important.

You should generally **continue receiving live packets while recovery is occurring**.

Otherwise:

```text
gap 9281 detected
       ↓
stop receiving
       ↓
request 9281
       ↓
while waiting you miss
9282–9500
```

which creates a bigger problem.

---

# 20. Reorder/recovery buffer

Instead:

```text
missing 9281

live:
9282 → buffer
9283 → buffer
9284 → buffer
9285 → buffer

recovery:
9281 arrives

then apply:

9281
9282
9283
9284
9285
```

Conceptually:

```text
            LIVE FEED
                │
                ▼
           gap detector
                │
        ┌───────┴────────┐
        │                │
        │              gap?
        │                │
        ▼                ▼
   normal flow       recovery request
        │                │
        │             missing data
        │                │
        └───────┬────────┘
                ▼
          reorder buffer
                │
                ▼
        ordered event stream
```

---

# 21. What if the gap is enormous?

Suppose your process crashed for:

```text
30 seconds
```

and millions of messages passed.

Requesting every missing message individually may be unreasonable.

Now you want:

# Snapshot recovery.

---

# 22. Snapshot + incremental updates

The classic recovery model is:

```text
               historical state
                      │
                      ▼
                  SNAPSHOT
                      │
                      │ at sequence 10,000
                      ▼
                current book
                      │
           apply messages 10,001+
                      │
                      ▼
                 live state
```

Nasdaq's GLIMPSE service provides book state and a corresponding TotalView-ITCH sequence point from which real-time processing can continue. ([NASDAQ Trader][7])

CME similarly provides incremental market-data updates and Market Data Snapshot Full Refresh messages through its recovery facilities. ([CME Group][8])

---

# 23. Snapshot correctness is subtle

Suppose:

```text
live feed already at 10,500
```

but the snapshot represents state at:

```text
10,000
```

You cannot simply install the snapshot and start from 10,500.

You would miss:

```text
10,001 ... 10,499
```

Therefore recovery logic typically looks like:

```text
begin buffering live updates

request/load snapshot

snapshot says:
state valid through seq 10,000

apply buffered:
10,001
10,002
...
10,500

switch to normal live processing
```

This is why feed recovery is a serious engineering subsystem, not just a UDP socket.

---

# 24. The NIC receives the packet

Assume Feed A produces the first copy of sequence 100.

The Ethernet signal reaches the trading machine's network interface card.

The NIC contains components conceptually like:

```text
fiber
  │
  ▼
PHY
  │
  ▼
MAC
  │
  ▼
packet processing / filters
  │
  ▼
RX queues
  │
  ▼
PCIe
  │
  ▼
host memory
```

---

# 25. PHY

The **PHY** deals with the physical signalling layer.

It transforms optical/electrical signals into digital data and vice versa.

Conceptually:

```text
photons/electrical signal
        ↓
       PHY
        ↓
digital bits
```

For ultra-low latency, even PHY/transceiver and switch design can matter.

---

# 26. MAC

The Ethernet MAC deals with frames.

It handles things such as:

```text
destination MAC
source MAC
frame parsing
FCS validation
```

The NIC can also perform hardware filtering and steering.

---

# 27. Receive queues

Modern NICs don't generally expose one giant global receive queue.

They often support:

```text
RX queue 0
RX queue 1
RX queue 2
...
```

Packets can be routed to particular queues through mechanisms such as:

```text
RSS
flow steering
filters
```

In low-latency systems you may deliberately steer a feed to a queue owned by a specific CPU core:

```text
ITCH feed
   ↓
NIC RX queue 3
   ↓
CPU core 7
```

rather than randomly distributing the work.

---

# 28. What exactly is an RX ring?

This is one of the important pieces Gross assumes you understand.

The RX ring isn't normally:

```text
a queue containing full packets
```

It is more like a circular array of **descriptors**.

A descriptor tells the NIC:

> Here is a memory buffer you may use for an incoming packet.

Conceptually:

```text
RX descriptor ring

┌───────────────┐
│ descriptor 0  │ ──► buffer A
├───────────────┤
│ descriptor 1  │ ──► buffer B
├───────────────┤
│ descriptor 2  │ ──► buffer C
├───────────────┤
│ descriptor 3  │ ──► buffer D
└───────────────┘
        ▲
        │
     circular
```

Intel's documented receive path follows this model: software supplies descriptors referring to empty buffers, the NIC obtains a buffer address, DMA-writes the incoming packet, writes completion information into the descriptor, and software then consumes it. ([Intel Caching Service][9])

---

# 29. DMA — Direct Memory Access

The CPU does not normally execute:

```text
copy byte 1 from NIC
copy byte 2
copy byte 3
...
```

Instead the NIC is a DMA-capable device.

DMA means:

> The device can transfer data directly into system memory without having the CPU copy each byte.

So:

```text
packet reaches NIC
       ↓
NIC finds RX descriptor
       ↓
descriptor points to buffer
       ↓
NIC DMA writes packet
       ↓
RAM
```

Intel describes the receive sequence explicitly as the NIC reading the RX descriptor and then using DMA to place the packet in the provided host buffer. ([Intel Caching Service][9])

---

# 30. Where does PCIe fit?

The NIC is typically connected to the CPU/platform through:

# PCI Express.

Conceptually:

```text
NIC
 │
 │ PCIe
 ▼
CPU / memory subsystem
```

The DMA operations traverse this interconnect.

Therefore:

```text
NIC DMA reads
NIC DMA writes
descriptor reads
descriptor writebacks
doorbells
```

all contribute PCIe traffic.

That means a low-latency networking design must think about:

```text
NIC
+
PCIe
+
cache
+
memory
+
CPU
```

as one system.

---

# 31. NIC → RAM does not necessarily mean slow DRAM every time

Modern coherent/platform behaviour is more complicated than the cartoon:

```text
NIC → DRAM → CPU
```

Packets, descriptors and metadata interact with the cache/memory hierarchy depending on platform and hardware.

For your architectural model, the important concept is:

> The NIC and CPU communicate through descriptor structures and packet buffers in host-accessible memory.

---

# 32. Descriptor ownership

An RX ring has a producer/consumer-style ownership protocol.

Conceptually:

```text
NIC owns:
descriptors ready to fill

CPU owns:
descriptors containing completed packets
```

Then:

```text
NIC fills descriptor
      ↓
marks complete
      ↓
CPU processes packet
      ↓
returns buffer/descriptor
      ↓
NIC can reuse it
```

This is remarkably similar to all the ring-buffer designs we discussed in Gross's talks.

The same pattern appears everywhere:

```text
ownership
head/tail
bounded memory
single producer/consumer patterns
```

---

# 33. The conventional Linux receive path

Normal application networking might resemble:

```text
NIC
 ↓
DMA
 ↓
kernel driver
 ↓
interrupt/NAPI
 ↓
kernel networking stack
 ↓
socket buffer
 ↓
UDP/IP processing
 ↓
socket queue
 ↓
recv()
 ↓
userspace
```

This architecture is excellent for general-purpose networking.

It supports:

```text
many processes
security
routing
firewalls
TCP
namespaces
traffic control
portable sockets
```

But an HFT feed handler asks:

> How much of this do I actually need?

---

# 34. Interrupt-driven networking

A conventional model can look like:

```text
CPU doing other work
      ↓
packet arrives
      ↓
interrupt
      ↓
CPU switches context
      ↓
network processing
```

That is energy-efficient.

But introduces:

```text
interrupt latency
scheduler effects
jitter
cache pollution
```

HFT often prefers a core that is already waiting for the packet.

---

# 35. Busy polling

Instead:

```cpp
while (running) {
    packets = poll_rx_queue();

    for (packet : packets)
        process(packet);
}
```

No packet?

```text
poll again
```

No packet?

```text
poll again
```

CPU utilization:

```text
~100%
```

Useful computation:

```text
possibly almost zero
```

But response latency can be much more predictable.

---

# 36. Kernel bypass

Now we arrive at the concept discussed in the 2024 Gross talk.

Instead of:

```text
NIC
 ↓
kernel networking stack
 ↓
userspace
```

we want approximately:

```text
NIC
 ↓
userspace packet buffers/rings
 ↓
application
```

Different technologies provide different degrees of this.

---

# 37. DPDK

DPDK provides poll-mode drivers that can access NIC RX/TX descriptors directly from userspace and avoid the normal interrupt-driven kernel networking path.

The current DPDK documentation describes its PMDs as polling RX/TX descriptors directly rather than relying on normal interrupts, specifically to reduce latency and improve packet-processing performance. ([DPDK][10])

A run-to-completion architecture might be:

```text
while (true) {

    packets = NIC_RX();

    for packet:
        decode
        update_book
        strategy
        maybe_make_order

    NIC_TX();
}
```

Notice what is interesting:

# No inter-thread queue is necessary.

One core can:

```text
receive
  ↓
decode
  ↓
calculate
  ↓
transmit
```

This eliminates communication latency.

DPDK explicitly supports this run-to-completion model. ([DPDK][11])

---

# 38. `ef_vi`

AMD's `ef_vi`, originating from the Solarflare low-latency NIC ecosystem, provides a very low-level Layer-2 API for direct access to network-adapter datapaths.

It is much closer to:

```text
NIC queues
+
packet buffers
+
events
```

than traditional sockets.

AMD documents `ef_vi` as a Layer-2 interface intended for extremely low send/receive latency. ([AMD Documentation][12])

---

# 39. AF_XDP

Linux also offers AF_XDP.

It provides high-performance packet handling using shared memory called:

```text
UMEM
```

and rings such as:

```text
FILL
RX
TX
COMPLETION
```

Conceptually:

```text
                      UMEM

             packet buffers live here

                       ▲
                       │
            ┌──────────┴──────────┐
            │                     │
          RX ring              TX ring
            │                     │
            ▼                     ▼
        application             NIC path
```

Linux documents AF_XDP RX/TX descriptor rings referencing buffers inside UMEM. ([Kernel][13])

So notice again:

```text
preallocated memory
+
bounded rings
+
ownership transfer
```

Exactly the architectural ideas Gross emphasized.

---

# 40. Kernel bypass is not magic

Do not conclude:

```text
kernel bypass = automatically fast.
```

You can still ruin performance with:

```text
bad NUMA placement
cache misses
too many threads
bad batching
shared RX queues
allocation
poor packet parsing
branch-heavy code
```

Kernel bypass merely removes one class of overhead.

---

# 41. Feed handler

Now the packet is accessible to userspace.

We reach perhaps the most important software component of the market-data path:

# Feed Handler

Its job is to transform:

```text
raw packet bytes
```

into:

```text
correct ordered market events
```

Conceptually:

```text
Ethernet packet
    ↓
IP/UDP framing
    ↓
MoldUDP64 framing
    ↓
sequence validation
    ↓
A/B arbitration
    ↓
ITCH decoding
    ↓
market event
```

---

# 42. Packet ≠ market message

One UDP packet may contain multiple ITCH messages.

For example:

```text
UDP datagram

MoldUDP header

Message Count = 3

├── ITCH Add Order
├── ITCH Cancel
└── ITCH Trade
```

So the feed handler needs two different parsing stages:

```text
transport framing parser
```

and:

```text
application message decoder.
```

---

# 43. ITCH

ITCH is designed to describe changes in the exchange's market state.

Nasdaq TotalView-ITCH provides full-depth order-level data and uses a series of messages to describe the lifecycle of orders. 

Typical lifecycle messages include:

```text
Add Order
Order Executed
Order Cancel
Order Delete
Order Replace
Trade
```

The current TotalView-ITCH specification explicitly contains these message families. 

---

# 44. An Add Order example

Conceptually:

```text
ADD ORDER

order_id = 7123421
instrument = AAPL
side = BUY
price = 200.00
quantity = 100
```

Your book builder processes:

```text
find AAPL book
     ↓
insert order 7123421
     ↓
price 200.00
     ↓
quantity 100
```

---

# 45. Cancel vs delete

Suppose:

```text
order 7123421
quantity = 100
```

Cancel message:

```text
cancel 40
```

Now:

```text
quantity = 60
```

A delete means:

```text
remove order entirely.
```

Correct semantics matter.

If you misunderstand exchange-specific message semantics, your local order book slowly diverges from reality.

---

# 46. Replace

Replace may conceptually mean:

```text
old order ID removed
new order ID introduced
```

depending on venue semantics.

Again, order identifiers and queue priority are important because a modification can change queue position.

---

# 47. Why Nasdaq has Stock Locate

An interesting mechanically sympathetic feature in ITCH is its Stock Locate value.

Nasdaq describes it as a low integer specifically intended to be usable as an array index for rapidly looking up instrument information. 

That is exactly the Gross philosophy.

Instead of:

```cpp
unordered_map<string, Instrument>
```

you can potentially do something more like:

```cpp
instrument = instruments[stock_locate];
```

So:

```text
protocol design
     ↓
enables
     ↓
fast memory access
```

This is an excellent example of performance being encoded in an API/protocol.

---

# 48. Endianness

Network binary protocols also specify byte order.

Nasdaq ITCH integers are big-endian/network byte order. 

Your x86 CPU is normally little-endian.

Therefore decoding may require conversion:

```text
network bytes

12 34 56 78

      ↓

host integer
```

This sounds tiny.

But if millions of fields are decoded per second, tiny operations matter.

---

# 49. SBE

Now let's contrast ITCH with SBE.

SBE stands for:

# Simple Binary Encoding.

FIX Trading Community defines SBE as a high-performance binary encoding designed for efficient financial messaging; it has also become an ISO/IEC standard. ([FIX Trading Community][14])

SBE is not one particular market feed.

Rather:

```text
SBE
=
encoding scheme
```

Exchange schemas define messages encoded using SBE.

---

# 50. Why SBE is fast

SBE favors:

```text
fixed positions
fixed-width numeric fields
known schemas
binary values
minimal parsing
```

instead of:

```text
key=value text
variable strings
reflection
dynamic field lookup
```

So decoding can often approach:

```text
buffer + known offset
```

rather than parsing general data structures.

---

# 51. CME MDP 3.0

CME's Market Data Platform 3.0 is a good example.

Conceptually:

```text
Ethernet
 ↓
IP
 ↓
UDP multicast
 ↓
CME packet/session framing
 ↓
SBE-encoded MDP messages
 ↓
book events
```

CME states that MDP uses a dual-feed UDP multicast architecture and that MDP 3.0 uses SBE/FIX Binary encoding. ([CME Group][4])

---

# 52. ITCH vs SBE

These aren't quite equivalent categories.

A simplified way to think:

| ITCH                                                           | SBE                                                 |
| -------------------------------------------------------------- | --------------------------------------------------- |
| Specific exchange-oriented market-data message protocol/family | Generic high-performance binary encoding            |
| Defines Add/Cancel/Trade etc.                                  | Defines how fields/messages are encoded efficiently |
| Nasdaq prominent example                                       | CME MDP/iLink prominent example                     |

So don't say:

```text
ITCH vs UDP
```

or:

```text
SBE vs multicast
```

They live at different layers.

---

# 53. A useful protocol-layer table

| Layer                | Nasdaq market data example | CME market data example |
| -------------------- | -------------------------- | ----------------------- |
| Physical             | Ethernet/fiber             | Ethernet/fiber          |
| Network              | IP                         | IP                      |
| Transport            | UDP multicast              | UDP multicast           |
| Session/feed framing | MoldUDP64                  | CME MDP framing         |
| Application format   | TotalView-ITCH             | MDP 3.0 SBE messages    |
| Semantic state       | Orders/trades/book         | Orders/trades/book      |

For order entry:

| Layer         | Nasdaq                           | CME                        |
| ------------- | -------------------------------- | -------------------------- |
| Application   | OUCH                             | iLink 3                    |
| Binary format | OUCH fixed messages              | SBE                        |
| Session       | commonly SoupBinTCP beneath OUCH | FIXP                       |
| Purpose       | send/cancel/replace orders       | send/cancel/replace orders |

Nasdaq explicitly describes SoupBinTCP as the typical lower-level protocol used to guarantee delivery/sequencing of host-to-client OUCH messages. 

CME describes iLink 3 as using SBE with FIXP, fixed positions and fixed-length fields for efficient order entry. ([CME Group][15])

---

# 54. From decoder to order book

Suppose ITCH says:

```text
AddOrder:
AAPL
BUY
200.00
100 shares
order_id 42
```

The feed handler now updates an internal:

# Limit Order Book.

---

# 55. Market-by-order vs market-by-price

You need to distinguish:

## Market by Order — MBO / L3

You know individual orders:

```text
200.00:

Order 1001 → 50
Order 1002 → 25
Order 1003 → 100
```

Total:

```text
175
```

MBO data tracks the lifecycle of individual order IDs. Databento, for example, describes MBO as every order-book event keyed by order ID, including add/cancel/modify/trade/fill events. ([Databento][16])

---

## Market by Price — MBP / L2

You only know aggregate level:

```text
200.00 → 175 shares
```

not:

```text
which individual orders.
```

Different strategies may require different levels of detail.

---

# 56. Why MBO is valuable

With MBO you can infer information such as:

```text
queue structure
queue position estimates
order arrivals
cancellation behaviour
order age
```

which can be valuable for market-microstructure strategies.

But it requires much more state.

---

# 57. Internal order-book architecture

A simplified book might maintain:

```text
Order ID Map

order_id
   ↓
price
quantity
side
location
```

plus:

```text
Bid price levels
Ask price levels
```

Conceptually:

```text
Order ID 987
     │
     ▼
price level 200.00
     │
     ▼
BUY book
```

This is exactly the structure Gross optimizes in his 2024 talk.

---

# 58. The book lives in the hot path

Market event:

```text
packet
 ↓
decode
 ↓
book update
```

is directly upstream of the trading decision.

Therefore:

```text
std::map pointer chasing
allocation
cache misses
branches
```

immediately increase tick-to-trade latency.

This is why his:

```text
map
→ vector
→ reversed vector
→ branchless search
→ linear search
```

experiment matters.

It wasn't an abstract container benchmark.

It was optimizing **this exact stage**.

---

# 59. Derived market state

The strategy may not want the entire raw book every time.

The feed handler/book component may maintain values such as:

```text
best bid
best ask
mid price
spread
microprice
imbalance
recent trade direction
queue sizes
volatility inputs
```

Example:

$$
Mid = \frac{BestBid + BestAsk}{2}
$$

A simple imbalance measure:

$$
I =
\frac{BidVolume - AskVolume}
{BidVolume + AskVolume}
$$

The critical point is to update these **incrementally**.

Do not recompute a 1,000-level book statistic from scratch every tick if only one level changed.

---

# 60. Feed handler and strategy: one thread or separate threads?

This is a major architecture decision.

### Pipeline design

```text
Feed thread
    ↓
SPSC queue
    ↓
Strategy thread
```

Advantages:

```text
modularity
core parallelism
failure isolation
```

Cost:

```text
queue
cache transfer
synchronization
extra latency
```

---

# 61. Run-to-completion design

For the fastest path:

```text
NIC RX
 ↓
decode
 ↓
book
 ↓
strategy
 ↓
risk
 ↓
NIC TX
```

all on one pinned CPU core.

No queue.

No cross-core cache transfer.

DPDK explicitly describes this as a run-to-completion processing model. ([DPDK][10])

This can be extremely powerful when the strategy's computation is small enough.

---

# 62. When multiple cores make sense

Suppose your strategy performs:

```text
complex option model
500 related instruments
risk calculations
machine-learning inference
```

One core may no longer be sufficient.

Then:

```text
Feed Core
   │
   ▼
shared state / queue
   │
   ├──── Strategy Core A
   ├──── Strategy Core B
   └──── Strategy Core C
```

Now the queue/seqlock/working-set lessons from the 2022 talk become relevant.

---

# 63. Strategy

The strategy receives the current market state and asks:

```text
Should I trade?

At what price?

How much?

Should I cancel an existing order?

Should I replace it?
```

For market making, one simplified model might be:

```text
fair value = 200.02
```

current market:

```text
200.00 bid
200.04 ask
```

strategy decides:

```text
buy quote = 200.01
sell quote = 200.03
```

But real strategies may incorporate:

```text
inventory
correlated products
volatility
market pressure
queue position
order-flow signals
risk limits
hedge prices
```

---

# 64. Software vs FPGA fast path

Now connect this with Gross's talks.

A software strategy might calculate:

```text
new fair value
risk state
volatility
desired quotes
```

and configure fast hardware logic.

Then a very simple FPGA trigger can react:

```text
if market condition X:
    cancel quote
```

without waiting for the full C++ model.

So:

```text
                      C++ STRATEGY
                            │
                       model state
                            │
                            ▼
                         FPGA
                            │
Market packet ──────────────┼────► immediate reaction
```

This is how sophisticated software and nanosecond hardware paths can coexist.

---

# 65. Pre-trade risk

The strategy cannot simply send arbitrary orders.

Before an order leaves the firm, controls may check:

```text
order size
position limit
price collars
credit exposure
duplicate/order state
maximum message rate
self-match protection
kill-switch status
```

But traditional enterprise risk architecture:

```text
strategy
 ↓
REST call
 ↓
risk microservice
 ↓
database
```

is impossible in a microsecond/nanosecond-sensitive path.

---

# 66. HFT risk should be local

A low-latency risk check is more like:

```text
position = 1200
new qty = 100
limit = 5000

1200 + 100 < 5000
        ↓
       PASS
```

State is:

```text
local
precomputed
cache resident
incrementally maintained
```

No database lookup.

No network RPC.

---

# 67. Risk hierarchy

A sophisticated firm may have both:

```text
ULTRAFAST INLINE CHECKS
```

and:

```text
SLOWER SUPERVISORY RISK
```

Conceptually:

```text
Strategy
   ↓
fast hard limits
   ↓
Order Gateway
   ↓
Exchange

Meanwhile:

Risk supervisor
   ↓
aggregate positions
firm-wide exposure
kill switch
limits
```

Not every risk calculation must be on the nanosecond path.

Only the mandatory ones.

---

# 68. Order Manager

Now the strategy decides:

```text
BUY 100 @ 200.01
```

But a production system needs to track its **own order state**.

Example:

```text
Order 7831

NEW_SENT
   ↓
ACKNOWLEDGED
   ↓
PARTIALLY_FILLED
   ↓
FILLED
```

or:

```text
NEW_SENT
   ↓
ACK
   ↓
CANCEL_SENT
   ↓
CANCELLED
```

This state machine is commonly part of an OMS/order manager/order gateway subsystem.

---

# 69. Why own order state is separate from the public book

Suppose you send:

```text
BUY 100
```

The exchange privately tells you:

```text
ACK
```

Before you necessarily see the corresponding change reflected through public market data.

Your private order-entry channel and public market-data feed are **different information paths**.

Therefore:

```text
my order state
```

must not be reconstructed solely from:

```text
public ITCH.
```

You track acknowledgements/executions from your order session.

---

# 70. Order gateway

The order gateway converts an internal order representation:

```cpp
Order {
    instrument,
    side,
    price,
    quantity
}
```

into the exchange-native wire format.

Nasdaq:

```text
internal order
      ↓
OUCH message
```

CME:

```text
internal order
      ↓
iLink 3 SBE message
```

---

# 71. OUCH

Nasdaq describes OUCH as its low-level native high-performance order-entry protocol.

It lets a participant:

```text
enter
replace
cancel
```

orders and receive executions/status updates. 

OUCH uses fixed-length logical messages and binary numeric fields. Nasdaq explicitly says it prioritizes maximum performance at the cost of flexibility/ease of use. 

That is an HFT protocol-design philosophy in one sentence:

```text
less generality
       ↓
less processing
       ↓
lower latency
```

---

# 72. Why fixed-length messages help

Suppose message layout is known:

```text
offset 0   message type
offset 1   side
offset 2   quantity
offset 6   instrument
offset 10  price
...
```

Encoder can effectively do:

```cpp
buffer[0] = NEW_ORDER;
store_qty(buffer + 2, qty);
store_price(buffer + 10, price);
```

Decoder does the inverse.

No:

```text
search for field name
parse delimiter
allocate string
dynamic schema lookup
```

---

# 73. CME iLink 3

CME iLink 3 provides the same broader goal through a different protocol design.

CME says iLink 3 uses:

```text
SBE
+
FIXP
+
fixed positions
+
fixed-length fields
```

to make order-entry messaging efficient. ([CME Group][15])

---

# 74. FIXP

FIXP is not the old human-readable FIX `tag=value` encoding.

FIXP means:

# FIX Performance Session Layer.

It manages things such as:

```text
session establishment
sequencing
recovery
message flow guarantees
```

while remaining lightweight and encoding-independent. FIX Trading Community describes it as a high-performance lightweight session protocol supporting point-to-point as well as multicast communication patterns. ([FIX Trading Community][3])

So CME might conceptually be:

```text
SBE
  ↓
defines efficient application message encoding

FIXP
  ↓
defines session behaviour
```

Different layers again.

---

# 75. Order-session sequencing

Private order sessions also require sequencing.

Suppose you sent:

```text
order 1
cancel 1
order 2
```

You need reliable understanding of:

```text
which messages exchange processed
which acknowledgements you received
what happened after reconnection
```

This is more stateful than market-data multicast.

That is one reason market data and order entry often use different transport/session mechanisms.

---

# 76. Constructing the outgoing packet

Order gateway produces something like:

```text
Ethernet
  ↓
IP
  ↓
transport/session
  ↓
OUCH/iLink payload
```

The bytes live in a preallocated transmit buffer.

Then software creates or updates:

# TX descriptor.

---

# 77. TX ring

TX is essentially the reverse of RX.

Conceptually:

```text
TX descriptor ring

descriptor
    │
    └────► packet buffer

CPU:
write packet
write descriptor
notify NIC

NIC:
read descriptor
DMA read packet
transmit
```

Intel's TX path documentation follows precisely this sequence: software writes packet bytes and a TX descriptor pointing to the buffer; the NIC reads the descriptor, fetches the packet and transmits it. ([Intel Caching Service][9])

---

# 78. The NIC “doorbell”

At some point software has to tell the NIC:

> New TX descriptors are available.

This often involves updating a queue tail/doorbell register.

Conceptually:

```text
CPU builds descriptors 100–104

then:

TX_TAIL = 105
```

The NIC now knows it may process those entries.

Hardware details differ, but this producer–consumer pattern is fundamental.

---

# 79. Why doorbell batching exists

Suppose you send:

```text
descriptor
doorbell
descriptor
doorbell
descriptor
doorbell
```

You repeatedly perform relatively expensive device-visible operations.

Instead:

```text
descriptor
descriptor
descriptor
descriptor

doorbell once
```

amortizes overhead.

But larger batches increase waiting latency.

So once again:

```text
throughput
↔
latency
```

tradeoff.

---

# 80. PCIe writebacks

The NIC may also need to tell the CPU:

```text
these TX descriptors are complete.
```

Doing a PCIe writeback for every single descriptor can be expensive.

Intel's documented vector TX path, for example, can group completion writebacks to reduce PCIe bandwidth. ([Intel Caching Service][9])

Same pattern from Gross's queue:

```text
don't update shared metadata unnecessarily.
```

---

# 81. Packet leaves the NIC

Now:

```text
TX buffer
   ↓
NIC
   ↓
PHY
   ↓
fiber
```

The packet crosses perhaps:

```text
top-of-rack switch
exchange access switch
gateway network
```

and reaches the exchange.

---

# 82. Exchange gateway

The exchange generally doesn't expose its matching-engine memory directly to your Ethernet cable.

There is an order-entry infrastructure layer.

Conceptually:

```text
participant
    ↓
order entry gateway
    ↓
validation/session handling
    ↓
matching engine
```

The gateway may check:

```text
session validity
protocol correctness
permissions
message sequencing
basic exchange controls
```

then route the order to the appropriate matching-engine partition.

---

# 83. Market partitioning

Large exchanges generally partition markets.

Conceptually:

```text
Gateway
   │
   ├── Partition 1 → symbols/products A
   ├── Partition 2 → symbols/products B
   ├── Partition 3 → symbols/products C
   └── ...
```

Why?

Because one matching-engine thread/machine handling every instrument globally would become a scalability bottleneck.

Partitioning also creates something very important:

```text
within one partition
```

ordering can remain strictly deterministic.

---

# 84. Matching engine

The matching engine owns the authoritative order book.

Your local order book is merely a reconstructed replica.

This distinction matters:

```text
EXCHANGE BOOK
=
truth

YOUR BOOK
=
local reconstruction of truth
```

The matching engine decides whether your order:

```text
executes immediately
rests
is rejected
partially fills
```

---

# 85. Price-time priority

Nasdaq uses a price/time priority model for its displayed limit book.

At the same price:

```text
first arriving order
        ↓
first execution priority
```

Nasdaq explicitly describes displayed limit orders at the same price as being executed in received-time order. ([NASDAQ Trader][17])

This is why HFT firms care about getting an order to the venue quickly.

Being:

```text
100 ns earlier
```

can translate into:

```text
better queue position.
```

---

# 86. But not every exchange uses pure FIFO

This is important.

CME uses several matching algorithms depending on product:

```text
FIFO
Pro-Rata
Split FIFO/Pro-Rata
LMM variants
etc.
```

CME explicitly documents multiple Globex allocation algorithms. ([CME Group][18])

So a strategy must understand the specific microstructure of the venue/product.

---

# 87. What happens when the order executes?

Suppose your:

```text
BUY 100 @ 200.01
```

hits a resting seller.

The matching engine changes authoritative state.

Two kinds of information may then leave the exchange.

### Private response

To you:

```text
execution report:
filled 100 @ 200.01
```

### Public response

To market-data subscribers:

```text
trade / execution / book update
```

Those flows are separate.

---

# 88. The feedback loop

Now the architecture becomes circular:

```text
                 MARKET DATA
                     │
                     ▼
                  STRATEGY
                     │
                     ▼
                   ORDER
                     │
                     ▼
                 EXCHANGE
                     │
                     ▼
               MARKET CHANGES
                     │
                     ▼
                 MARKET DATA
                     │
                    ...
```

This loop may run millions of times.

---

# 89. Tick-to-trade

A common latency measurement is conceptually:

```text
market event received
        ↓
order transmitted
```

or:

$$
T_{tick-to-trade}
=
T_{order\_out} - T_{market\_in}
$$

But you must define exactly what:

```text
market_in
```

and:

```text
order_out
```

mean.

CPU timestamp?

NIC timestamp?

First packet byte?

Last packet byte?

Without precise definitions two latency numbers cannot be compared meaningfully.

---

# 90. Hardware timestamping

Suppose software records:

```text
recv timestamp
```

after the packet has:

```text
entered NIC
DMA'd
been processed by driver
reached userspace
```

That timestamp includes unknown upstream software latency.

Instead, sophisticated NICs can timestamp packets close to the wire.

Conceptually:

```text
fiber
  │
  ▼
NIC receives packet
  │
  ├── HARDWARE TIMESTAMP T0
  │
  ▼
DMA
  ↓
CPU
```

Linux explicitly supports NIC hardware timestamps and PTP hardware clocks, and its timestamping documentation notes that timestamps taken closer to the wire can yield more stable and precise timing. ([Kernel][19])

---

# 91. TX hardware timestamp

Likewise:

```text
CPU says send at:
T1
```

is not the same as:

```text
packet physically left NIC at:
T2
```

TX hardware timestamps can help distinguish:

```text
software latency
queueing inside NIC
actual wire departure
```

---

# 92. PTP

If different machines use timestamps, their clocks must agree.

Enter:

# Precision Time Protocol — PTP / IEEE 1588.

Conceptually:

```text
reference clock
      │
      ├────► server A clock
      ├────► server B clock
      └────► NIC hardware clock
```

Linux exposes standardized PTP hardware-clock infrastructure so user-space PTP software can synchronize clocks. ([Linux Kernel Archives][20])

Without synchronization:

```text
exchange timestamp
```

and:

```text
your timestamp
```

may not be comparable.

---

# 93. TSC

Inside one machine, ultra-low-latency software may measure tiny regions using the CPU:

# Time Stamp Counter.

Conceptually:

```cpp
t0 = read_tsc();

process();

t1 = read_tsc();
```

This gives cycle-level measurement.

But using it correctly requires thinking about:

```text
instruction ordering
core migration
TSC synchronization
conversion cycles → time
measurement overhead
```

Gross's later talk discusses this measurement philosophy.

---

# 94. A useful timestamp chain

For one market event you ideally want something like:

```text
T_exchange_send
       │
       ▼
T_nic_rx
       │
       ▼
T_decode
       │
       ▼
T_book
       │
       ▼
T_strategy
       │
       ▼
T_risk
       │
       ▼
T_tx_submit
       │
       ▼
T_nic_tx
```

Then you can calculate:

$$
T_{network-in}
=
T_{nic-rx} - T_{exchange-send}
$$

$$
T_{software}
=
T_{tx-submit} - T_{nic-rx}
$$

$$
T_{NIC-TX}
=
T_{nic-tx} - T_{tx-submit}
$$

This is how you stop guessing where latency went.

---

# 95. NUMA placement

Now connect networking to the 2022 Gross talk.

Suppose NIC is physically attached closest to:

```text
NUMA node 0
```

but your feed thread runs on:

```text
NUMA node 1.
```

You have:

```text
NIC
 │
NUMA 0
 │
cross-socket interconnect
 │
NUMA 1 CPU
```

Bad.

Prefer:

```text
NIC
 │
NUMA 0
 │
local memory
 │
NUMA 0 strategy core
```

DPDK's own design guidance explicitly recommends keeping NIC-associated buffers and processing on the local NUMA node where possible. ([DPDK][11])

---

# 96. A realistic CPU layout

One machine may look like:

```text
CPU package

Core 0   OS / housekeeping
Core 1   interrupts / management

Core 2   Feed A receiver
Core 3   Feed B receiver / arbitration

Core 4   Strategy 1
Core 5   Strategy 2

Core 6   order gateway
Core 7   recovery / control

Core 8+  monitoring / logging
```

Or for absolute minimal latency:

```text
Core 4:

RX
→ decode
→ book
→ strategy
→ risk
→ TX
```

There is no universal topology.

The correct design depends on workload.

---

# 97. Shared LLC still matters

Even if:

```text
Core 4 = strategy
Core 5 = analytics
```

and they share no C++ variables, they might share:

```text
L3 / LLC
```

Analytics can evict strategy data.

Thus:

```text
logical isolation
≠
hardware isolation
```

This is exactly what Gross demonstrated.

---

# 98. IRQ placement

If you're doing kernel bypass polling, you try to prevent unrelated interrupts from hitting your critical core.

Bad:

```text
strategy calculating
       ↓
storage IRQ
       ↓
interrupt handler
       ↓
strategy resumes
```

That produces jitter.

---

# 99. Power settings

Likewise:

```text
deep C-state
       ↓
packet arrives
       ↓
CPU wake-up
```

adds latency.

Dynamic frequency transitions can also alter execution behaviour.

So machine configuration is part of your network path.

---

# 100. Memory allocation

Notice how disastrous this would be:

```text
packet arrives
 ↓
new Message()
 ↓
malloc
 ↓
decode
 ↓
new OrderBookNode()
```

Hot-path allocation can introduce:

```text
allocator work
contention
page faults
cache disruption
```

So production feed handlers commonly use:

```text
fixed buffers
preallocated ring memory
object pools
bounded structures
```

again matching Gross's philosophy.

---

# 101. Zero-copy parsing

Suppose packet bytes already exist in a NIC-provided buffer.

Bad:

```text
NIC buffer
   ↓ copy
temporary network buffer
   ↓ copy
ITCH structure
   ↓ copy
event object
```

Potentially better:

```text
NIC buffer
   ↓
parse fields directly
```

or:

```text
NIC buffer
 ↓
very compact normalized event
```

But remember the lesson from the previous talk:

> Zero-copy isn't automatically better.

A tiny contiguous copy can sometimes simplify ownership enough to be worthwhile.

---

# 102. Normalization

Imagine your firm trades:

```text
Nasdaq
NYSE
CME
Eurex
```

Every venue has different messages.

Your strategy doesn't ideally want:

```cpp
if (nasdaq)
    parse_itch();

if (cme)
    parse_mdp();

if (eurex)
    ...
```

So feed handlers often produce a normalized internal event:

```cpp
struct BookEvent {
    InstrumentId instrument;
    Price price;
    Quantity qty;
    Side side;
    Action action;
};
```

Then:

```text
Nasdaq ITCH ─┐
             │
CME MDP ─────┼──► Internal BookEvent
             │
NYSE ────────┘
```

---

# 103. But normalization has a cost

If you produce an enormously abstract object:

```text
virtual MarketEvent
std::string symbol
std::variant<...>
shared_ptr
```

you may introduce:

```text
allocations
copies
indirection
large working set
```

So HFT internal protocols tend toward:

```text
small POD-like structures
numeric instrument IDs
fixed layouts
```

---

# 104. The recovery path should not dominate the hot path

Another design principle:

Normal case:

```text
packet
 ↓
fast validation
 ↓
book
```

Exceptional recovery:

```text
gap
 ↓
recovery subsystem
```

You don't want:

```text
huge generic recovery machinery
```

executing on every packet.

This is **hot/cold path separation** at an architectural level.

---

# 105. Drop copy

A trading firm often receives another stream known conceptually as:

# Drop Copy.

It provides an independent record of order/execution activity.

Why?

Your main trading process might:

```text
crash
lose local state
have a bug
```

An independent stream can support:

```text
risk
reconciliation
surveillance
recovery
post-trade processing
```

It normally isn't what your hottest strategy waits on.

---

# 106. Kill switch

A serious trading system needs a way to stop trading rapidly.

Examples:

```text
position runaway
exchange rejects explode
strategy bug
market-data corrupt
risk breach
connectivity failure
```

Kill switch may trigger:

```text
block new orders
cancel existing orders
disable strategy
disconnect session
```

Correctness and safety outrank nanoseconds here.

---

# 107. Market-data stale detection

Suppose your feed silently stops.

Your last book looks perfectly valid:

```text
bid 200
ask 201
```

But is now:

```text
500 ms old.
```

A strategy cannot continue treating it as live.

Therefore monitor:

```text
last packet timestamp
last sequence
heartbeat
channel state
gap state
```

and mark books:

```text
VALID
STALE
RECOVERING
INVALID
```

---

# 108. Never trade on an invalid book accidentally

A good architecture makes state validity explicit.

Bad:

```cpp
Price best_bid;
```

Better conceptual state:

```text
Book {
    state = LIVE / RECOVERING / INVALID
    sequence
    best_bid
    best_ask
}
```

Then strategy can refuse to trade when:

```text
state != LIVE.
```

---

# 109. Private-session failure is different

Market-data failure:

```text
recover book state.
```

Order-session failure is more dangerous:

```text
Did exchange receive my order?

Did it reject it?

Did it fill?

Did my cancel arrive?
```

This is called the:

# uncertain-state problem.

You must reconcile with exchange session/order state before blindly resending operations.

OUCH, for example, deliberately assigns participant-generated transaction identifiers and specifies retry/recovery semantics for host-bound messages. 

---

# 110. Why idempotency matters

Suppose:

```text
BUY order
```

was sent.

Connection dies before acknowledgement.

If you reconnect and simply resend:

```text
BUY order
```

could you accidentally create two orders?

Protocol/session designs use identifiers and defined retransmission behaviour to prevent or manage this.

That's the concept of:

```text
idempotency
```

or deduplication.

---

# 111. Order-entry path can be hardware accelerated too

Nasdaq currently advertises FPGA-backed OUCH gateway technology for dedicated high-performance order-entry infrastructure. ([NASDAQ Trader][21])

And a firm can also put its own:

```text
parser
risk checks
reaction logic
packet construction
```

into FPGA/SmartNIC hardware.

Then the architecture can become:

```text
                  ┌──────────── SOFTWARE
                  │
market data ─► FPGA
                  │
                  └──────────── order
```

avoiding PCIe/CPU round trips for selected reactions.

---

# 112. The SmartNIC / FPGA variant

Software path:

```text
wire
 ↓
NIC
 ↓
PCIe
 ↓
RAM
 ↓
CPU
 ↓
RAM
 ↓
PCIe
 ↓
NIC
 ↓
wire
```

FPGA/SmartNIC reaction:

```text
wire
 ↓
FPGA
 ↓
logic
 ↓
wire
```

This can remove entire architectural layers.

That's why hardware trading can reach latency domains general-purpose CPUs cannot.

---

# 113. But FPGA logic is expensive in another dimension

You pay in:

```text
development difficulty
verification
limited flexibility
hardware resources
deployment complexity
debugging
```

Therefore:

```text
not all strategy
```

moves to FPGA.

Again:

```text
latency
↔
flexibility
```

---

# 114. What “wire-to-wire” means

An FPGA benchmark may report:

```text
wire-to-wire latency
```

Meaning conceptually:

```text
incoming packet appears at network interface
          ↓
hardware reacts
          ↓
outgoing packet leaves interface
```

This is different from:

```text
software function execution time.
```

Always examine latency boundaries.

---

# 115. Serialization vs propagation vs processing latency

Network latency has several components.

$$
T_{network}
=
T_{serialization}
+
T_{propagation}
+
T_{switching}
+
T_{queueing}
$$

### Serialization

Time to put bits onto the link.

### Propagation

Time for signal to travel through fiber.

### Switching

Time network equipment takes.

### Queueing

Time waiting behind other traffic.

In colocation, propagation becomes small enough that:

```text
switch choice
packet size
queueing
PHY
NIC
```

can actually matter.

---

# 116. Bursts matter more than average bandwidth

Suppose average feed rate:

```text
1 Gbps
```

but bursts hit:

```text
10+ Gbps
```

for short intervals.

A system designed only for:

```text
average
```

drops packets.

This is why exchange feed engineering considers **microbursts**.

Nasdaq has historically warned clients that one-second bandwidth recommendations may not capture sub-second peaks and that latency-sensitive clients may need more capacity. ([NASDAQ Trader][22])

---

# 117. What happens during a microburst

NIC RX ring:

```text
empty
 ↓
packet
packet
packet
packet
packet
packet
...
```

If CPU cannot consume fast enough:

```text
ring fills
 ↓
no available RX descriptors
 ↓
packet drop
```

Intel's DPDK analysis similarly notes that when the hardware catches the software's available-descriptor boundary, receive-buffer exhaustion can cause drops. ([Intel][23])

So ring sizing matters.

---

# 118. But enormous rings aren't automatically good

Why not:

```text
1,000,000 descriptors?
```

Because a giant queue can hide overload by turning:

```text
drop
```

into:

```text
huge queueing latency.
```

Your strategy may receive perfectly complete data—

```text
500 μs late.
```

For HFT that's equally disastrous.

You want:

```text
enough buffering for normal bursts
```

but not:

```text
unbounded stale data.
```

---

# 119. Queue depth itself is a latency metric

Suppose processing takes:

```text
100 ns/message
```

and 1,000 messages are ahead.

Even if your queue push/pop costs:

```text
5 ns
```

the newest event waits approximately:

```text
100 μs
```

before processing.

So monitor:

```text
queue occupancy
consumer lag
sequence lag
```

not merely container benchmark latency.

---

# 120. Book sharding

Suppose a feed carries:

```text
10,000 instruments.
```

One CPU cannot process every expensive strategy.

You might shard:

```text
instruments 0–999    → core 1
1000–1999            → core 2
...
```

But now cross-instrument strategies may need state across shards.

This creates a tradeoff:

```text
parallelism
vs
cross-core communication.
```

---

# 121. Cross-product strategies make architecture harder

Suppose:

```text
future changes
    ↓
option price should change
```

or:

```text
ETF changes
    ↓
constituent relationship
```

A strategy may need:

```text
many market feeds
many books
many instruments
```

Now:

```text
working-set size
```

becomes critical.

That's where the 2022 Gross talk fits directly.

---

# 122. The truly complete software box

A more realistic HFT host might therefore contain:

```text
                   ┌─────────────────────────────┐
                   │          HFT HOST           │
                   │                             │
Market Feed A ─────┼─► RX Queue A               │
Market Feed B ─────┼─► RX Queue B               │
                   │       │                     │
                   │       ▼                     │
                   │    Arbitrator               │
                   │       │                     │
                   │       ▼                     │
                   │     Decoder                 │
                   │       │                     │
                   │       ▼                     │
                   │   Order Books               │
                   │       │                     │
                   │       ▼                     │
                   │   Derived State             │
                   │       │                     │
                   │       ▼                     │
                   │    Strategy                 │
                   │       │                     │
                   │       ▼                     │
                   │      Risk                   │
                   │       │                     │
                   │       ▼                     │
                   │ Order Manager               │
                   │       │                     │
                   │       ▼                     │
                   │ Order Encoder               │
                   │       │                     │
                   │       ▼                     │
                   │   TX Queue                  │
                   │                             │
                   │ Recovery                    │
                   │ Logging                     │
                   │ Drop Copy                   │
                   │ Metrics                     │
                   │ Risk Supervisor             │
                   └─────────────────────────────┘
```

---

# 123. But a whole trading platform is larger still

Outside the low-latency host:

```text
                   TRADING SYSTEM

              ┌───────────────────┐
              │ Reference Data    │
              └───────────────────┘
                        │
                        ▼
Market ───► Low Latency Trading Engine ───► Exchange
                        │
                        ├────► Drop Copy
                        │
                        ├────► Position Service
                        │
                        ├────► Firm Risk
                        │
                        ├────► P&L
                        │
                        ├────► Surveillance
                        │
                        ├────► Persistence
                        │
                        ├────► Monitoring
                        │
                        └────► Post Trade
```

Only a tiny fraction of a trading firm's total code belongs on the extreme hot path.

---

# 124. Hot path vs cold path

You should now deliberately classify functionality.

### Hot

```text
RX
sequence validation
decode
book update
strategy
inline risk
order encode
TX
```

### Warm

```text
order-state reconciliation
shared positions
parameter changes
```

### Cold

```text
logging
reporting
database persistence
analytics
GUI
historical storage
configuration
```

The architecture should prevent cold-path work from polluting hot-path latency.

---

# 125. The packet-to-order sequence, step by step

Let's follow one real conceptual event from start to finish.

At the exchange:

```text
1. An incoming order modifies the exchange book.
```

Then:

```text
2. Matching engine produces market-data event.
```

Then:

```text
3. Event encoded as ITCH/SBE/etc.
```

Then:

```text
4. Feed/session protocol assigns sequencing.
```

Then:

```text
5. Messages packed into UDP datagram.
```

Then:

```text
6. IP header added.
```

Then:

```text
7. Ethernet frame transmitted.
```

Then:

```text
8. Exchange multicast network replicates feed.
```

Then:

```text
9. Fiber delivers frame to your NIC.
```

Then:

```text
10. NIC PHY/MAC receives frame.
```

Then:

```text
11. NIC chooses RX queue.
```

Then:

```text
12. NIC DMA-writes packet into host buffer.
```

Then:

```text
13. NIC marks RX descriptor complete.
```

Then:

```text
14. Pinned CPU polling queue sees descriptor.
```

Then:

```text
15. MoldUDP/CME framing decoded.
```

Then:

```text
16. A/B feed arbitration performed.
```

Then:

```text
17. Sequence integrity checked.
```

Then:

```text
18. ITCH/SBE market message decoded.
```

Then:

```text
19. Local order book updated.
```

Then:

```text
20. Derived market state updated.
```

Then:

```text
21. Strategy evaluates.
```

Then:

```text
22. Strategy decides to trade.
```

Then:

```text
23. Pre-trade risk check executes.
```

Then:

```text
24. Order manager creates order state.
```

Then:

```text
25. OUCH/iLink/etc. encoder creates bytes.
```

Then:

```text
26. TX descriptor points NIC at order packet.
```

Then:

```text
27. NIC DMA-reads outgoing data.
```

Then:

```text
28. NIC transmits Ethernet frame.
```

Then:

```text
29. Exchange order gateway receives it.
```

Then:

```text
30. Gateway validates/routes it.
```

Then:

```text
31. Matching engine processes it.
```

Then:

```text
32. Private acknowledgement/execution returns.
```

And:

```text
33. Resulting market change appears on public feed.
```

Then the entire loop begins again.

---

# 126. Now the latency equation becomes meaningful

Your actual tick-to-order latency might conceptually be:

$$
T =
T_{NIC-RX}
+
T_{feed}
+
T_{decode}
+
T_{book}
+
T_{strategy}
+
T_{risk}
+
T_{encode}
+
T_{NIC-TX}
$$

But external response also includes:

$$
T_{complete}
=
T_{network-in}
+
T_{software}
+
T_{network-out}
+
T_{exchange-gateway}
+
T_{matching-engine}
$$

And each term has:

```text
median
p99
p99.9
jitter
```

not one fixed number.

---

# 127. Why HFT performance is multiplicative in complexity

Every unnecessary boundary can add:

```text
queue
cache transfer
copy
serialization
lock
scheduler event
allocation
```

Consider:

```text
NIC
 ↓
Feed Process
 ↓ queue
Book Process
 ↓ queue
Strategy Process
 ↓ RPC
Risk Service
 ↓ queue
Order Gateway
```

This is architecturally clean for many enterprise applications.

For HFT it can be catastrophic.

---

# 128. The extreme design

The other end is:

```text
NIC
 ↓
Core 4
 ├ decode
 ├ book
 ├ strategy
 ├ risk
 └ encode
 ↓
NIC
```

Working set:

```text
small
```

Allocations:

```text
none
```

Locks:

```text
none
```

Syscalls:

```text
none on steady-state path
```

Cross-core transfers:

```text
none
```

This is the natural destination of all the Gross principles.

---

# 129. The even more extreme design

Then:

```text
network
 ↓
FPGA
 ├ decode
 ├ book subset
 ├ signal
 ├ risk
 └ order
 ↓
network
```

CPU not involved in the critical reaction at all.

---

# 130. Protocol confusion — clear this up permanently

You should now be able to classify these correctly:

| Technology             | What it is                                            |
| ---------------------- | ----------------------------------------------------- |
| **Ethernet**           | Link-layer network framing                            |
| **IP**                 | Network addressing/routing                            |
| **UDP**                | Lightweight datagram transport                        |
| **TCP**                | Reliable ordered byte stream                          |
| **Multicast**          | One-to-many IP distribution model                     |
| **MoldUDP64**          | Nasdaq UDP feed/session framing + sequencing/recovery |
| **ITCH**               | Nasdaq market-data message protocol                   |
| **GLIMPSE**            | Nasdaq snapshot/recovery service                      |
| **OUCH**               | Nasdaq low-latency order-entry protocol               |
| **SBE**                | High-performance binary encoding                      |
| **FIXP**               | High-performance financial session layer              |
| **MDP 3.0**            | CME market-data protocol/platform                     |
| **iLink 3**            | CME order-entry protocol                              |
| **DPDK**               | Userspace packet-processing framework                 |
| **AF_XDP**             | Linux high-performance packet userspace interface     |
| **ef_vi**              | Low-level userspace NIC API                           |
| **DMA**                | Device transferring data to/from host memory directly |
| **PCIe**               | Hardware interconnect between NIC and host            |
| **RX/TX ring**         | Descriptor queues coordinating CPU and NIC            |
| **PTP**                | Clock synchronization protocol                        |
| **Hardware timestamp** | NIC/PHY timestamp near packet arrival/departure       |
| **NUMA**               | Non-uniform CPU/memory topology                       |
| **FPGA**               | Programmable hardware acceleration                    |

This table alone resolves a lot of common HFT confusion.

---

# 131. Five different kinds of “sequence”

Another subtle distinction.

You can encounter:

```text
packet sequence
market-data message sequence
order-session sequence
order ID
transaction/client reference ID
```

They are not interchangeable.

### Packet sequence

Used to detect network-feed gaps.

### Market-data sequence

Orders market events.

### Order-session sequence

Tracks your private session.

### Order ID

Identifies one resting order.

### Client transaction ID

Identifies one request sent by your system.

Protocols define each differently.

---

# 132. Why TCP isn't used for all market data

You might think:

```text
TCP already handles loss.
Use TCP.
```

But suppose 10,000 firms consume one market-data stream.

TCP requires independent connection state and retransmission behaviour.

A slow receiver could experience different delivery behaviour.

Multicast lets the exchange produce:

```text
one dissemination stream
```

and clients independently solve packet loss/recovery.

For high fan-out market data this is an excellent architecture.

---

# 133. Why UDP isn't necessarily used for all order entry

Order entry has different semantics.

You need reliable private state about:

```text
my order
my ack
my fill
my cancel
```

Point-to-point session protocols are therefore common.

FIXP itself supports reliable high-performance point-to-point models independent of the underlying transport. ([FIX Trading Community][24])

---

# 134. Why market data can be lossy but trading cannot

This sounds paradoxical.

Market data transport:

```text
loss can happen
```

but application correctness:

```text
loss cannot be ignored.
```

The system gets reliability through:

```text
sequence numbers
+
redundant feeds
+
retransmission
+
snapshots
```

rather than demanding reliability from UDP itself.

This is a recurring high-performance architecture concept:

> Move expensive guarantees to the level where you can implement exactly the guarantees you need.

---

# 135. Feed arbitration versus recovery

These should be conceptually separate.

```text
Feed A missing seq 100
Feed B has seq 100
```

Use:

```text
arbitration.
```

Both missing 100:

```text
retransmission.
```

State badly behind:

```text
snapshot/replay.
```

So the recovery hierarchy becomes:

```text
A/B arbitration
       ↓
small-gap retransmission
       ↓
snapshot/replay
       ↓
full session restart if necessary
```

---

# 136. Packet capture and replay

Serious trading systems often record raw market-data packets.

Why?

If strategy behaved strangely at:

```text
09:31:22.381928
```

you want to replay exactly:

```text
the bytes received.
```

This helps with:

```text
debugging
backtesting
postmortems
protocol upgrades
book-validation
```

Raw packet capture is therefore an important side system.

But capturing must not disrupt the hot path.

---

# 137. Deterministic replay

A good feed handler should ideally satisfy:

```text
same ordered input bytes
        ↓
same market state
```

Determinism is enormously valuable.

You can replay a production packet stream through:

```text
decoder
book
strategy
```

and reproduce behaviour.

---

# 138. Reference data

Before the market opens you also need:

```text
instrument definitions
tick sizes
lot sizes
trading states
symbol mappings
price limits
matching algorithm info
```

These may come from reference-data feeds/messages.

Nasdaq's ITCH Stock Directory, for example, sends security metadata and its intraday Stock Locate mapping. 

CME distributes Security Definition information that includes product metadata including matching-algorithm identifiers. ([CME Group][25])

---

# 139. Tick size

If instrument tick size is:

```text
$0.01
```

you cannot usually send:

```text
$100.00543
```

So price handling often uses integer ticks:

```text
price_ticks = 10000
```

rather than floating-point dollars.

This is faster and avoids rounding ambiguity.

---

# 140. Why finance protocols love integers

Instead of:

```cpp
double price = 123.45;
```

wire representation might be:

```text
1234500
precision = 4
```

Advantages:

```text
deterministic
exact
portable
fast comparison
```

Nasdaq ITCH explicitly represents prices using integer fixed-point fields with implied precision. 

---

# 141. Why HFT code often avoids strings

Strategy does not want:

```cpp
if (symbol == "AAPL")
```

on every event.

Prefer:

```cpp
instrument_id = 271;
```

then:

```cpp
instrument[271]
```

Again, Nasdaq's low-valued Stock Locate identifiers are intentionally compatible with fast indexed access. 

---

# 142. A beautiful connection between protocol and cache

Notice:

```text
Protocol:
Stock Locate = small integer
```

enables:

```text
Application:
array[StockLocate]
```

which enables:

```text
CPU:
simple address calculation
```

which enables:

```text
cache-friendly lookup.
```

Protocol design has literally influenced CPU-level performance.

That is mechanical sympathy from exchange to silicon.

---

# 143. The architecture has two sources of truth

This is subtle.

### Market truth

Your local reconstructed exchange book.

From:

```text
public market data.
```

### Your private order truth

Your own order state.

From:

```text
order-entry acknowledgements/executions.
```

Strategies combine both:

```text
MarketBook
+
MyOrders
+
MyPositions
=
TradingState
```

---

# 144. Position state

Suppose:

```text
buy fill +100
```

You immediately update:

```text
position += 100.
```

Future strategy decisions depend on this.

You don't wait for:

```text
database
post-trade report
end-of-day file.
```

Critical position state must be in-memory and current.

---

# 145. Hedging

Now imagine you are making markets in an option.

Fill:

```text
BUY option
```

creates risk.

Strategy may react:

```text
sell future
```

So one market event can produce a chain:

```text
option fill
 ↓
position changes
 ↓
delta exposure changes
 ↓
hedge calculation
 ↓
futures order
```

That is why cross-market data and architecture become complex.

---

# 146. One HFT system may consume many exchanges simultaneously

Real architecture might resemble:

```text
Nasdaq ─────┐
NYSE ───────┤
CME ────────┤
Eurex ──────┼──► Internal Market State
Options ────┤
Futures ────┘
                  │
                  ▼
              Strategy
```

Then network latency between different exchange data centres matters too.

This creates entire domains such as:

```text
cross-market latency arbitrage
microwave networks
optimized fiber routes
```

which are another advanced chapter.

---

# 147. What your final mental architecture should look like

Here is the complete version I'd want you to remember:

```text
                           EXCHANGE

                  ┌────────────────────┐
                  │  MATCHING ENGINE   │
                  └───────┬────────────┘
                          │
             ┌────────────┴───────────────┐
             │                            │
        PUBLIC DATA                  PRIVATE ORDERS
             │                            ▲
       ITCH / SBE                         │
             │                        OUCH / iLink
       sequencing                         │
             │                            │
      UDP multicast                       │
             │                            │
       Feed A + B                         │
             │                            │
      Exchange switches                   │
             │                            │
           fiber                          │
             │                            │
             ▼                            │
      ┌─────────────── TRADING HOST ───────────────┐
      │                                            │
      │              NIC                           │
      │               │                            │
      │              PHY/MAC                       │
      │               │                            │
      │         RX descriptor ring                 │
      │               │                            │
      │             DMA/PCIe                       │
      │               │                            │
      │        kernel bypass/poll                  │
      │               │                            │
      │         A/B arbitration                    │
      │               │                            │
      │        gap/sequence check ─────► recovery  │
      │               │                            │
      │             decoder                        │
      │               │                            │
      │          ORDER BOOK                        │
      │               │                            │
      │       derived market state                 │
      │               │                            │
      │           STRATEGY                         │
      │               │                            │
      │         PRE-TRADE RISK                     │
      │               │                            │
      │        ORDER MANAGER                       │
      │               │                            │
      │         ORDER ENCODER                      │
      │               │                            │
      │          TX descriptor                     │
      │               │                            │
      │            DMA/PCIe                        │
      │               │                            │
      │              NIC ──────────────────────────┘
      │
      │ cold/off-path:
      │
      ├── raw capture
      ├── metrics
      ├── logging
      ├── drop copy
      ├── reconciliation
      ├── supervisory risk
      └── persistence
      └────────────────────────────────────────────
```

Running under the whole machine:

```text
CPU pinning
NUMA affinity
cache locality
C-state tuning
frequency tuning
IRQ isolation
memory preallocation
huge pages where beneficial
PTP synchronization
hardware timestamping
continuous latency measurement
```

And possibly bypassing much of the CPU path:

```text
                     FPGA / SmartNIC
Market Data ─────────────┬────────────► Order
                         │
                    fast reaction
```

---

# 148. How this connects the three chapters you've now studied

You can now place every concept from the two Gross talks in its real location.

### Working-set size

Here:

```text
order books
strategy state
risk state
instrument state
```

### Seqlock

Here:

```text
sharing latest state between cores/processes.
```

### SPMC queue

Here:

```text
feed handler fan-out.
```

### Data-oriented order book

Here:

```text
ITCH/SBE decode
        ↓
ORDER BOOK
```

### Kernel bypass

Here:

```text
NIC
 ↓
feed handler
```

### NUMA

Here:

```text
NIC ↔ PCIe ↔ memory ↔ CPU.
```

### C/P states

Here:

```text
the CPU waiting for incoming market packets.
```

### Instruction/cache optimization

Here:

```text
decode → book → strategy → risk → encode.
```

### FPGA

Here:

```text
wire → fast reaction → wire.
```

### Production timing

Across:

```text
NIC_RX
→ decode
→ book
→ strategy
→ risk
→ NIC_TX.
```

Now all of those previously isolated ideas belong to one architecture.

---

# 149. What you approximately know after these three chapters

At this point you understand the **core architecture** behind a modern low-latency electronic trading system:

```text
✓ colocation
✓ cross-connects
✓ Ethernet
✓ IP
✓ UDP
✓ multicast
✓ redundant market-data feeds
✓ feed arbitration
✓ sequence numbers
✓ retransmission
✓ snapshots
✓ ITCH
✓ MoldUDP64
✓ SBE
✓ CME MDP
✓ NIC RX/TX queues
✓ descriptors
✓ DMA
✓ PCIe
✓ busy polling
✓ kernel bypass
✓ DPDK
✓ AF_XDP
✓ ef_vi
✓ feed handlers
✓ order-book reconstruction
✓ MBO vs MBP
✓ market state
✓ strategy
✓ pre-trade risk
✓ order management
✓ OUCH
✓ iLink
✓ FIXP
✓ exchange gateway
✓ matching engine
✓ matching priority
✓ order acknowledgements
✓ execution reports
✓ PTP
✓ hardware timestamping
✓ TSC
✓ NUMA
✓ CPU isolation
✓ cache locality
✓ FPGA/SmartNIC paths
✓ recovery
✓ drop copy
✓ stale-feed handling
✓ tick-to-trade measurement
```

That is already a **substantially more accurate picture of HFT architecture** than the common “C++ strategy + fast broker API” view.

What you do **not yet have** is full knowledge of the *strategy/microstructure side* and some deeper implementation details. The remaining major chapters would be: **(1) exchange matching engines and market microstructure in depth**—price/time priority, queue position, maker/taker, order types, hidden/iceberg/peg orders, auctions, self-match prevention, adverse selection; **(2) building an actual production-grade feed handler and order book in C++** from packet bytes upward; **(3) order gateway/session engineering and pre-trade risk**; and **(4) hardware HFT**—FPGA feed parsing, cut-through processing, SmartNICs and wire-to-wire architecture.

If your objective is to reach the point where you could actually **design your own HFT-like low-latency platform**, rather than merely understand diagrams, I would make **exchange microstructure + matching engine** the next chapter. The networking architecture only becomes fully meaningful once you understand *why being 100 ns earlier changes queue position, fills, adverse selection and P&L*.

[1]: https://www.nasdaqtrader.com/trader.aspx?id=connectivityoptions&utm_source=chatgpt.com "Connectivity"
[2]: https://www.cmegroup.com/notices/electronic-trading/2026/05/20260518.html?utm_source=chatgpt.com "CME Globex Notice: May 18, 2026 - CME Group"
[3]: https://www.fixtrading.org/standards/fixp-online/?utm_source=chatgpt.com "FIX Performance Session Layer (FIXP) Online – FIX Trading Community"
[4]: https://www.cmegroup.com/market-data/distributor/market-data-platform.html?utm_source=chatgpt.com "Data Directly from the Market Data Platform - CME Group"
[5]: https://owu.terracat.net/docs/architecture/architecture-diagram?__owu_origin_v1=aHR0cHM6Ly9kYXRhYmVudG8uY29t&utm_source=chatgpt.com "Databento architecture guide | Databento Docs"
[6]: https://nasdaqtrader.com/content/technicalsupport/specifications/dataproducts/moldudp64.pdf?utm_source=chatgpt.com "MoldUDP64 Protocol Specification"
[7]: https://www.nasdaqtrader.com/content/technicalsupport/specifications/dataproducts/PSXGLIMPSESpecification_5.0.pdf?utm_source=chatgpt.com "PSX GLIMPSE 5.0"
[8]: https://www.cmegroup.com/tools-information/lookups/advisories/electronic-trading/20141215.html?utm_source=chatgpt.com "20141215 - CME Group"
[9]: https://cdrdv2-public.intel.com/767813/Intel_Ethernet_800_Performance_Evolution_on_DPDK_TechnologyGuide_767813v1.pdf "Intel® Ethernet 800 Series Network Adapter - Performance Evolution on DPDK Technology Guide"
[10]: https://doc.dpdk.org/guides/prog_guide/ethdev/ethdev.html?utm_source=chatgpt.com "3.1. Poll Mode Driver — Data Plane Development Kit 26.07.0 documentation"
[11]: https://doc.dpdk.org/guides-22.07/prog_guide/poll_mode_drv.html?utm_source=chatgpt.com "12. Poll Mode Driver — Data Plane Development Kit 22.07.0 documentation"
[12]: https://docs.amd.com/v/u/en-US/SF-114063-CD-ef_vi_User_Guide?utm_source=chatgpt.com "ef_vi User Guide - SF-114063-CD"
[13]: https://www.kernel.org/doc/html/latest/networking/af_xdp.html?highlight=af_xdp&utm_source=chatgpt.com "AF_XDP — The Linux Kernel documentation"
[14]: https://fixtrading.org/standards/simple-binary-encoding-sbe/?utm_source=chatgpt.com "Simple Binary Encoding (SBE) – FIX Trading Community"
[15]: https://www.cmegroup.com/notices/electronic-trading/2023/05/20230508.html?utm_source=chatgpt.com "20230508 - CME Group"
[16]: https://databento.com/docs/schemas-and-data-formats/mbo?utm_source=chatgpt.com "Market by order (MBO) | Databento schemas & data formats"
[17]: https://www.nasdaqtrader.com/trader.aspx?id=tradingusequities&utm_source=chatgpt.com "The NASDAQ Stock Market"
[18]: https://www.cmegroup.com/education/matching-algorithm-overview?utm_source=chatgpt.com "Matching Algorithm Overview"
[19]: https://kernel.org/doc/html/latest/networking/timestamping.html?utm_source=chatgpt.com "Timestamping — The Linux Kernel documentation"
[20]: https://cdn.kernel.org/doc/html/latest/driver-api/ptp.html?utm_source=chatgpt.com "PTP hardware clock infrastructure for Linux — The Linux Kernel documentation"
[21]: https://nasdaqtrader.com/Trader.aspx?id=ouch&utm_source=chatgpt.com "OUCH"
[22]: https://www.nasdaqtrader.com/TraderNews.aspx?id=dtn2016-29&utm_source=chatgpt.com "Data Technical News #2016 - 29 Nasdaq Increases Bandwidth Recommendations For Select Proprietary Data Feeds"
[23]: https://www.intel.com/content/www/us/en/docs/vtune-profiler/cookbook/2023-2/core-utilization-in-dpdk-apps.html?utm_source=chatgpt.com "Core Utilization in DPDK Apps"
[24]: https://fixtrading.org/standards/fixp/?utm_source=chatgpt.com "FIXP – FIX Trading Community"
[25]: https://www.cmegroup.com/tools-information/lookups/advisories/market-data/Q2008-261.html?utm_source=chatgpt.com "Q2008-261 - CME Group"
