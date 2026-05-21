# Host-PCI DMA TX/RX Ring Design

This note describes a simple host/device DMA ring model built around four
pointers. The main idea is to use one unified ring structure for both TX and
RX, and to make four independent progress domains explicit instead of
implicitly collapsing them together.

The design is general. It is not tied to one operating system or one hardware
device. It is useful anywhere software and a PCI device exchange ownership of
buffer descriptors through shared memory.

## Notation

- `BD`: buffer descriptor
  - points to a DMA buffer
  - a packet may span multiple descriptors
- `sw`: host software
- `hw`: PCI device or hardware endpoint
- `sw_get`: next slot software will reclaim/process
- `sw_put`: next slot software will publish
- `hw_get`: next slot hardware will reclaim/process
- `hw_put`: next slot hardware will publish

## Core Design

Use one descriptor ring with four pointers:

```text
sw_get <- sw_put <- hw_get <- hw_put <- (sw_get)
```

Rules:

- software reads `hw_*` and advances `sw_*`
- hardware reads `sw_*` and advances `hw_*`
- the ring is divided into four regions by those four pointers
- TX and RX use the same ring mechanics
- only the meaning of the data processing differs

That is the key simplification: the queue-management logic stays the same for
TX and RX, while ownership and progress remain explicit.

## Why Four Pointers

The four-pointer model gives you:

- a single ring structure for both directions
- explicit separation of four independent timing domains
- explicit ownership handoff between software and hardware
- cheap state inspection
- straightforward diagnostics when progress stops

Because all pointers live in memory, the host CPU does not need expensive PCIe
reads just to inspect queue state.

The deeper reason this works well is that it separates four distinct kinds of
progress:

- software publishing new work
- hardware consuming published work
- hardware publishing completion or received data
- software reclaiming or acknowledging that completion

Those steps do not need to run at the same speed. By giving each one its own
pointer, the design makes uneven timing normal instead of treating it as a
special case.

## TX Flow

For TX, software fills host memory and hardware consumes it.

### Setup

1. Allocate a packet buffer with `N` equal-sized slots in contiguous host
   memory.
2. Allocate a descriptor ring of size `N`.
3. Associate each descriptor with one data-buffer slot.
4. Initialize:
   - `sw_put = 0`
   - `hw_get = 0`
   - `hw_put = 0`
   - `sw_get = N - 1`

That means software initially owns `N - 1` free slots to fill.

### TX Procedure

1. A software path calls `pkt_tx()`.
2. Software checks whether it can advance `sw_put` without colliding with
   `sw_get`.
3. If enough slots are available, software fills the packet data into host
   memory.
4. Software advances `sw_put`.
5. Hardware observes that `hw_get` is behind `sw_put`.
6. Hardware advances `hw_get`, consumes the published descriptors, DMA-fetches
   packet data, and advances `hw_put` when done.
7. A software reclaim path advances `sw_get` after completion and returns those
   slots to the free region.

## RX Flow

For RX, hardware fills host memory and software consumes it.

### Setup

1. Allocate the same packet buffer and descriptor ring model.
2. Initialize:
   - `hw_put = 0`
   - `hw_get = N - 1`
   - `sw_put = 0`
   - `sw_get = 0`

That means hardware initially owns `N - 1` free slots to fill with received
data.

### RX Procedure

1. Hardware receives a packet and checks whether it can advance `hw_put`
   without colliding with `hw_get`.
2. If enough slots are available, hardware DMA-writes packet data into host
   memory.
3. Hardware advances `hw_put`.
4. Software observes that `sw_get` is behind `hw_put`.
5. Software advances `sw_get`, processes the received data, then advances
   `sw_put` to return space.
6. Hardware observes the new `sw_put` and advances `hw_get` accordingly.

Again, the queue logic is symmetric. Only who fills the data and who consumes
it is different.

## Queue State Interpretation

The strength of the design is that pointer position directly exposes health.

### TX Interpretation

Ring regions represent:

- free buffers available for software fill
- filled descriptors waiting for hardware pickup
- hardware-owned descriptors in flight
- completed descriptors waiting for software reclaim

Healthy TX usually looks like:

- many free buffers
- small in-flight regions
- reclaim keeping up

Bad TX usually looks like one of:

- almost no free buffers: software is blocked
- many published descriptors waiting for hardware: hardware stalled
- many completed descriptors not reclaimed: software reclaim path stalled

### RX Interpretation

Ring regions represent:

- hardware-available receive buffers
- received data waiting for software consumption
- software-processed descriptors waiting for hardware to take back
- acknowledgment/reclaim progression

Healthy RX usually looks like:

- many buffers available to hardware
- only a small backlog waiting on software

Bad RX usually looks like:

- almost no buffers available to hardware: receive path starved
- many received descriptors waiting for software: software stalled
- large reclaim lag: ownership handoff is broken or delayed

## Why This Is Useful

This model has several practical benefits:

- it is non-blocking when software and hardware keep up
- it can reach full hardware rate, limited mainly by DMA and PCIe bandwidth
- TX and RX can share the same queue-management logic
- ownership and backpressure stay explicit
- uneven progress between the four stages is naturally represented
- queue state is easy to inspect during debugging
- "which side is stuck?" becomes much easier to answer

## Diagnostics Value

The four pointers are not just control variables. They are diagnostics.

By looking at the spacing and progression of:

- `sw_get`
- `sw_put`
- `hw_get`
- `hw_put`

you can quickly determine whether:

- software publish is stalled
- hardware consumption is stalled
- hardware completion is stalled
- software reclaim is stalled

That makes this design attractive not only for performance, but also for
observability.

More importantly, each pointer gap corresponds to a specific kind of lag:

- publish lag
- consume lag
- completion lag
- reclaim lag

So the same structure gives both the transport mechanics and a direct way to
reason about which stage is falling behind.

## Adapting To Real Hardware

Real devices may not expose exactly this model.

That is fine. A small adapter layer can translate between:

- the hardware's real descriptor semantics
- the software's four-pointer ring model

The adapter is responsible for:

- programming descriptors in the device-specific format
- translating hardware completion/progress into `hw_get` and `hw_put`
- preserving the software-visible ownership model

As long as the hardware behavior can be mapped into the same ownership and
progress semantics, the higher-level software logic can stay clean.

## Closing

The main point is simple:

use one explicit ownership ring model for both TX and RX, and let each
independent stage of progress advance on its own timeline.

That keeps the transport mechanics simple, the ownership boundaries explicit,
and the stalled stage easy to identify when something goes wrong.
