# Limit Order Book Simulation

## 1. Goals

Build a simulated limit order book (LOB) that behaves like a real venue, generates realistic event streams from stochastic processes, and exposes its state through a market-data interface compatible with eventual real-data backtesting and live strategy code.

The end use case is testing market-making strategies against a book that **reacts in real time** to the strategy's own orders — something pure historical backtesting cannot do.

## 2. Non-Goals

- Modeling fees, rebates, or exchange-specific microstructure rules (initially).
- Multi-venue routing or fragmentation.
- Asset-class-specific quirks (auctions, halts, circuit breakers) — added later if needed.
- Production-grade performance. Correctness and clarity first.

## 3. Architecture Overview

```
                 ┌──────────────────────────────┐
                 │      Simulation Kernel       │
                 │  (clock, scheduler, RNG)     │
                 └──────┬────────────────┬──────┘
                        │                │
            polls next  │                │ submits events
            event from  ▼                ▼
                ┌──────────────┐   ┌──────────────────┐
                │ Sim Processes│   │ Matching Engine  │
                │ (Poisson,    │   │ (validates,      │
                │  Hawkes,     │   │  matches, mutates│
                │  state-dep.) │   │  the book)       │
                └──────────────┘   └────────┬─────────┘
                                            │
                                            ▼
                                   ┌──────────────────┐
                                   │      Tape        │
                                   │ (event log +     │
                                   │  periodic snaps) │
                                   └────────┬─────────┘
                                            │ subscribe
                                            ▼
                                   ┌──────────────────┐
                                   │   Market Data    │
                                   │   API (push)     │
                                   └────────┬─────────┘
                                            │
                          ┌─────────────────┴────────────────┐
                          ▼                                  ▼
                 ┌──────────────────┐              ┌──────────────────┐
                 │  Visualization   │              │  Strategy Layer  │
                 │   (ladder UI)    │              │    (future)      │
                 └──────────────────┘              └──────────────────┘
```

## 4. Components

### 4.1 Simulation Kernel

The kernel owns simulated time, the master RNG, and the event scheduler. It is the only thing that advances the clock.

Responsibilities:
- Maintain a priority queue of pending events keyed by timestamp.
- Ask each simulation process for its next event time and enqueue.
- Pop the next event, advance the clock, route to the matching engine.
- Forward the resulting event(s) to the tape.
- Re-poll any process whose state changed (for state-dependent processes).

The kernel knows nothing about prices, sides, or matching rules. It is a generic discrete-event scheduler.

### 4.2 Simulation Processes

Each process generates events of a particular type (e.g., buy limit arrivals, sell market arrivals, cancellations). They are **stateful objects**, not pure functions: their internal state holds whatever history is needed to compute the next event time.

Interface (sketch):

```python
class SimulationProcess:
    def next_event_time(self, now: float, book: OrderBookView) -> float:
        """Return the absolute time of the next event this process will emit."""

    def generate_event(self, now: float, book: OrderBookView) -> Event:
        """Construct the actual event (price, size, etc.) at firing time."""

    def observe(self, event: Event) -> None:
        """Update internal state in response to *any* tape event (for Hawkes,
        state-dependent processes that excite off other event types)."""
```

Process types planned:
- **Homogeneous Poisson** — constant rate. Stateless.
- **State-dependent Poisson** — rate is a function of the current book (e.g., spread, imbalance). Stateless across calls but reads book state at fire time.
- **Hawkes** — self-exciting; stores a list of past firing times to compute current intensity.

Each process gets its own seeded sub-RNG from the kernel.

### 4.3 Matching Engine

Pure function from `(book_state, event) → (new_book_state, resulting_events)`. No clock, no randomness, no I/O.

Validates the event, applies it, and emits any derived events (trades, fills, partial-fill remainders). The kernel feeds those derived events to the tape and to subscribers.

Order of operations on a market order:
1. Validate side, size, account (if applicable).
2. Walk opposite book from best price outward.
3. Generate a `trade` event per level consumed.
4. If unfilled remainder and the order type is `IOC`/market, drop. If it's a marketable limit, rest it.

### 4.4 Order Book Data Structures

Two indexes over the same set of orders:

- **Price-level index**: sorted structure keyed by price (e.g., a sorted dict or pair of red-black trees, one per side). Each price level holds a FIFO queue of orders. Used for top-of-book reads and matching walks.
- **Order ID index**: hash map `order_id → order_ref`. Used for O(1) cancel lookup. Each entry points to the order sitting inside its price-level queue.

Cancel flow: hash lookup → unlink from queue → remove level if empty.

### 4.5 Tape

The tape is the **canonical record** of the simulation. Two storage layers:

- **Event log**: append-only stream of every event with sequence number, timestamp, type, side, price, size, order ID. This is the source of truth.
- **Periodic snapshots**: full L2 book state every N events (or every T seconds of sim time). Snapshots are checkpoints — the book at any time can be reconstructed by loading the nearest preceding snapshot and replaying forward.

Snapshot frequency is a tunable parameter, traded off against memory.

### 4.6 Market Data API

Exposes the tape as a market data feed. Two channels:

- **Push channel (primary)**: subscribers register a callback; the API invokes it for every new tape event, in order. In-process for now (`subscriber.on_event(event)` called by the kernel after each tape append). Later, replaceable with WebSocket without changing subscriber code.
- **Snapshot endpoint**: request the current full L2 state. Used on subscriber startup to bootstrap, then push takes over.

Event schema (L2-style):
```
{
  "seq": 12947,
  "ts": 17283.441,            // simulated time, seconds
  "type": "limit" | "market" | "cancel" | "trade",
  "side": "buy" | "sell",
  "price": 100.02,
  "size": 5,
  "order_id": "abc-123"
}
```

The schema is intentionally close to common L2 feed formats (ITCH-like) so that strategy code can be reused against real data later.

### 4.7 Visualization

Subscribes to the API, maintains a local book replica, renders a price ladder (price levels, sizes per side, last trade marker). Out of scope for this document beyond noting that it is a pure consumer of the API — it has no privileged access to the kernel or matching engine.

### 4.8 Strategy Layer (Future)

A strategy is just another event source for the kernel: it observes tape events through the API and emits orders. It is **scheduled by the kernel like any other process**, with explicit latencies (see §6.2). This is what makes the simulator distinct from historical backtesting.

Manual order arrivals (a human typing in an order via UI) are a special case of the strategy layer with a human in the loop.

## 5. Event Lifecycle

1. Kernel pops next pending event from its queue.
2. Clock advances to that event's timestamp.
3. Event is dispatched to the matching engine.
4. Matching engine validates, mutates the book, returns a list of resulting events (the input event plus any trades).
5. Kernel appends each resulting event to the tape with a sequence number.
6. Kernel notifies all API subscribers of each new tape event, in order.
7. Kernel notifies each simulation process via `observe()` so state-dependent and Hawkes processes can update.
8. Kernel re-queries any process whose `next_event_time` may have changed and updates the scheduler.
9. Loop.

## 6. Key Design Decisions

### 6.1 Time Modes

- **Event-driven (simulated time)**: kernel jumps from event to event as fast as the CPU allows. Used for backtesting, parameter sweeps, calibration, and anywhere quantitative throughput matters.
- **Wall-clock**: kernel sleeps between events so simulated time matches real time. Used for visualization, demos, and any human-in-the-loop or external-system testing.

Wall-clock mode is implemented as a thin wrapper around event-driven mode: same scheduler, with `sleep(next_ts - now)` between pops.

### 6.2 Latency Model

Latency is a first-class concept, not an afterthought. Two distributions per strategy:

- `submit_latency`: time from strategy emitting an order to the matching engine receiving it.
- `market_data_latency`: time from a tape event being appended to the strategy observing it.

Each is sampled per-event from a configurable distribution (constant, exponential, log-normal, or empirical). Without this, strategies see fills and cancel before adverse selection — which makes any P&L number meaningless.

### 6.3 Stateful Simulation Processes

Processes are objects with internal state, not pure functions. The kernel does not pass past-event history as a parameter — each process owns whatever history it needs internally. The kernel only delivers new events through `observe()`.

Rationale: Hawkes processes need their own list of past firing times; state-dependent Poisson processes need to know book state at fire time; future processes will have their own state requirements. Putting state inside the object keeps the kernel interface uniform regardless of process complexity.

### 6.4 Determinism and Replay

- Every random draw goes through a seeded RNG owned by the kernel. The kernel hands seeded sub-RNGs to each simulation process at construction.
- Every event gets a monotonically increasing sequence number.
- Same seed + same configuration = bit-identical simulation, every time.
- The tape *is* the run record. A simulation can be replayed from the event log; bugs are reproducible by seed.

### 6.5 Tape Storage

Storing a full L2 snapshot per event scales poorly — a busy book produces thousands of events per second. Instead:

- Append every event to the log.
- Take a full snapshot every N events (default: 1000) or every T seconds of simulated time, whichever comes first.
- Reconstruct the book at any point by loading the nearest preceding snapshot and replaying forward.

This mirrors how real exchange feeds work (ITCH/FIX-style event streams with periodic full-book refreshes).

### 6.6 Push vs Pull API

Push (subscriber callback / WebSocket-style). Polling would miss events between polls — fatal for market making, where missing a single aggressive order means missing the chance to react before adverse selection. Push also matches the natural shape of the tape (an event stream) and aligns with how production exchange feeds are delivered.

A snapshot endpoint exists for bootstrapping subscribers; after bootstrap, push is the only path for ongoing updates.

## 7. Data Formats

### 7.1 Event Schema

```json
{
  "seq": 12947,
  "ts": 17283.441,
  "type": "limit",
  "side": "buy",
  "price": 100.02,
  "size": 5,
  "order_id": "abc-123",
  "parent_seq": null
}
```

`parent_seq` links derived events (trades) to the originating event (the aggressor that caused them).

### 7.2 L2 Snapshot Schema

```json
{
  "seq": 12947,
  "ts": 17283.441,
  "bids": [[100.01, 12], [100.00, 30], ...],   // price, total size at level
  "asks": [[100.02, 8],  [100.03, 15], ...]
}
```

## 8. Build Order

Suggested implementation sequence — each stage is independently testable:

1. Order book data structures + matching engine. Unit-test against hand-crafted event sequences.
2. Kernel + a single homogeneous Poisson process. Verify event rates statistically.
3. Tape with event log only.
4. Snapshots and reconstruction.
5. Push API with in-process subscribers.
6. State-dependent and Hawkes processes.
7. Visualization.
8. Wall-clock mode.
9. Latency model.
10. Strategy layer + manual order entry.

## 9. Future Work

- WebSocket transport for the push API (cross-process strategy testing).
- Strategy harness with plug-and-play interface compatible with real L2 data.
- Calibration tooling: fit process parameters to historical L2 data so simulations are realistic.
- Multi-asset / correlated books.
- Latency distributions calibrated to real venue measurements.