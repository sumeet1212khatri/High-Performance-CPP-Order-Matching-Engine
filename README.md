# ⚡ HFT Order Book Engine

A high-performance, C++20 limit order book engine capable of processing **20 million orders/sec** with sub-microsecond latency. Built from scratch with production-grade matching logic, lock-free concurrency primitives, and a live benchmark UI deployed on Hugging Face Spaces.

> **Live Demo:** [HuggingFace Space](https://huggingface.co/spaces/) — run real benchmarks from your browser, results served directly from the C++ binary.

---

## Performance

| Metric | Value |
|---|---|
| Throughput | ~20M orders/sec |
| P50 Latency | < 100 ns |
| P99 Latency | < 500 ns |
| Orders Benchmarked | 20,000,000 |
| Build Standard | C++20, `-O3 -march=native` |

---

## Architecture

```
┌─────────────────────────────────────────────────────────┐
│                    FastAPI Backend (Python)              │
│              app.py  ──►  /api/benchmark                │
└────────────────────────┬────────────────────────────────┘
                         │ subprocess
                         ▼
┌─────────────────────────────────────────────────────────┐
│                  C++20 Matching Engine                   │
│                                                         │
│  Benchmark  ──►  Orderbook  ──►  MatchOrders()         │
│      │               │                                  │
│  BenchmarkConfig  OrderEntry       Price-Time Priority  │
│  latency samples  bids_ / asks_    Stop-Limit injection │
└────────────────────────┬────────────────────────────────┘
                         │ JSON stdout
                         ▼
┌─────────────────────────────────────────────────────────┐
│              Frontend (Vanilla HTML/CSS/JS)              │
│         Live KPIs  ──  Latency histogram  ──  JSON out  │
└─────────────────────────────────────────────────────────┘
```

### Core Components

| File | Responsibility |
|---|---|
| `include/Orderbook.h / src/Orderbook.cpp` | Central matching engine — order lifecycle, matching loop, stop injection, GFD pruning thread |
| `include/Order.h` | Order model — GTC, FAK, FOK, IOC, Market, Iceberg, PostOnly, StopLimit |
| `include/Benchmark.h` | Configurable load generator with latency percentile collection |
| `include/LockFreeQueue.h` | SPSC / MPMC ring buffers with cache-line padding |
| `include/MemoryPool.h` | Lock-free object pool + arena allocator for zero-heap-alloc hot path |
| `include/Types.h` | Fixed-width primitives, enums, constants |
| `include/Trade.h` | Trade record, OrderModify, OrderbookSnapshot |
| `src/main.cpp` | CLI driver with `--orders`, `--json`, `--quiet` flags |
| `tests/test_orderbook.cpp` | 25-test suite covering all order types, edge cases, and stats |
| `app.py` | FastAPI server wrapping the C++ binary |
| `frontend/index.html` | Dark-theme benchmark dashboard |

---

## Order Types Supported

- **GoodTillCancel (GTC)** — rests in book until filled or cancelled
- **FillAndKill (FAK) / ImmediateOrCancel (IOC)** — fills what it can, cancels remainder instantly
- **FillOrKill (FOK)** — fully fills or rejects entirely; no partial fills
- **GoodForDay (GFD)** — auto-cancelled at 16:00 by background pruning thread
- **Market** — converted to GTC at worst available price for deterministic matching
- **Iceberg** — displays only peak quantity; replenishes automatically on each fill
- **PostOnly** — rejected if it would cross the spread (maker-only)
- **StopLimit** — parked until last trade price crosses stop trigger; injected as GTC

---

## Matching Engine Design

### Price-Time Priority
Bids are stored in a `std::map<Price, OrderPointers, std::greater<>>` (descending), asks in `std::map<Price, OrderPointers, std::less<>>` (ascending). FIFO within each price level via `std::list`.

### Stop Order Injection (Deferred)
Stop orders triggered mid-match are collected in `pendingStops_` and injected **after** the main matching loop exits. This breaks the recursive `MatchOrders → CheckAndTriggerStops → AddOrderInternal → MatchOrders` chain that would stack-overflow on stop chains.

### GFD Pruning Thread
A background `std::thread` wakes at 16:00 daily using `condition_variable::wait_for`, collects all GFD order IDs under lock, and cancels them via `CancelOrdersInternal`.

### Lock Strategy
Single `std::mutex` (`ordersMutex_`) guards `bids_`, `asks_`, `orders_`, and `levelData_`. Stats counters use `std::atomic` to avoid lock contention on the read path.

---

## Lock-Free Primitives

### SPSCQueue
Single-producer single-consumer ring buffer. Stores **raw monotonically increasing indices** (not masked) so `size()` is correct and the full-check `(tail - head) > Mask` works correctly across wraparound.

### MPMCQueue
Multi-producer multi-consumer queue using CAS on sequence numbers per slot — Dmitry Vyukov's design.

### ObjectPool
Fixed-size pool with a lock-free free list using `compare_exchange_weak`. Eliminates heap allocation latency on the order hot path.

---

## Benchmark

The benchmark runs configurable synthetic workloads:

```
./build/benchmark --orders 20000000 --cancel-ratio 0.20 --market-ratio 0.10 --json
```

**Workload mix (default):**
- 65% limit orders (GTC)
- 20% cancels (randomly selected from live orders)
- 10% market orders
- 5% modifies
- 2% iceberg orders

**Latency sampling:** 1M samples taken evenly across the run (`sampleEvery = totalOrders / 1M`). First iteration skipped to exclude cold-cache noise. Percentiles computed on sorted sample vector.

---

## Build

### Prerequisites
- `g++` ≥ 12 or `clang++` ≥ 15 (C++20 required)
- `cmake` ≥ 3.18
- `python3`, `pip` (for API server)

### Local Build

```bash
mkdir build && cd build
cmake .. -DCMAKE_BUILD_TYPE=Release -DCMAKE_CXX_STANDARD=20
make -j$(nproc)

# Run tests
./run_tests

# Run benchmark
./benchmark --orders 1000000

# Run full 20M benchmark with JSON output
./benchmark --orders 20000000 --json --quiet
```

### Docker (Hugging Face Spaces)

```bash
docker build -t hft-orderbook .
docker run -p 7860:7860 hft-orderbook
```

The Dockerfile builds the C++ binary at image build time, then starts the FastAPI server. Benchmark requests hit `/api/benchmark?orders=N&cancel=R&market=R` which shells out to the compiled binary with a 120-second timeout.

### Build Script

```bash
./scripts/build.sh          # build + test
./scripts/build.sh --quick  # + 1M order benchmark
./scripts/build.sh --full   # + 20M order benchmark
./scripts/build.sh --json   # + JSON output
```

---

## API

```
GET /api/benchmark?orders=20000000&cancel=0.20&market=0.10
```

**Response:**
```json
{
  "totalOrders": 20000000,
  "totalTrades": 3142857,
  "totalCancels": 4000000,
  "ordersPerSecond": 19823411.23,
  "tradesPerSecond": 3112345.67,
  "wallTimeSeconds": 1.01,
  "latency": {
    "avgNs": 48.32,
    "minNs": 12.00,
    "p50Ns": 44.00,
    "p90Ns": 88.00,
    "p95Ns": 110.00,
    "p99Ns": 210.00,
    "p999Ns": 480.00,
    "p9999Ns": 890.00,
    "maxNs": 4200.00,
    "stdDevNs": 31.12
  },
  "finalBookDepth": 47382
}
```

---

## Test Suite

25 tests covering:

- Basic GTC match, partial fill
- FAK hit and miss
- FOK hit and miss
- Market order sweeping multiple levels
- Cancel (valid and no-op on non-existent ID)
- Modify triggering a match
- Modify side change
- Iceberg resting and replenishment
- PostOnly rejection on cross
- Price-time priority (FIFO within level)
- Spread and mid-price calculation
- Multi-level sweep (5 levels)
- Duplicate order ID rejection
- Stress test (100K orders)
- Event callbacks (trade, cancel)
- Snapshot accuracy
- `GetTotalTrades` and `GetTotalCancels` stat tracking

```bash
./build/run_tests
```

---

## Key Engineering Decisions

**Fixed-point pricing** — `Price` is `int32_t` (actual price × 100). Eliminates float comparison bugs and keeps price comparisons branch-predictable.

**`std::list` for price levels** — O(1) front removal during matching and O(1) iterator-based cancellation. `orders_` stores the iterator directly so cancel is O(1) lookup + O(1) erase.

**No exceptions in hot path** — `Fill()` throws only on programmer error (overfill). All normal paths return values or use sentinel constants.

**`shared_ptr<Order>`** — chosen over raw pointers for safety given the multiple ownership patterns (book levels + `orders_` map + pending stops). Overhead is acceptable at this benchmark scale.

**Deferred stop injection** — avoids recursive matching stack growth at the cost of one extra vector swap per match cycle.

---

## Project Structure

```
hft_orderbook/
├── include/
│   ├── Types.h
│   ├── Order.h
│   ├── Trade.h
│   ├── Orderbook.h
│   ├── Benchmark.h
│   ├── LockFreeQueue.h
│   └── MemoryPool.h
├── src/
│   ├── Orderbook.cpp
│   └── main.cpp
├── tests/
│   └── test_orderbook.cpp
├── frontend/
│   └── index.html
├── scripts/
│   └── build.sh
├── CMakeLists.txt
├── app.py
└── Dockerfile
```

---

## License

MIT
