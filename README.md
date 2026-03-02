# 💫 About Me:
🎓 Studying CS + ML at Princeton<br>🛠️ I've worked on everything from matching engines and mobile apps to race car sensors — the variety keeps it interesting<br>🤖 Particularly curious about robotics and CS research<br>💬 Most comfortable in Java, C++, and Python, pick up whatever the project needs<br>📬 av7847@princeton.edu


## 🌐 Socials:
[![LinkedIn](https://img.shields.io/badge/LinkedIn-%230077B5.svg?logo=linkedin&logoColor=white)](https://linkedin.com/in/https://www.linkedin.com/in/ahmed-virk-481446384/) [![email](https://img.shields.io/badge/Email-D14836?logo=gmail&logoColor=white)](mailto:av7847@princeton.edu) 

# 💻 Tech Stack:
![C](https://img.shields.io/badge/c-%2300599C.svg?style=for-the-badge&logo=c&logoColor=white) ![C++](https://img.shields.io/badge/c++-%2300599C.svg?style=for-the-badge&logo=c%2B%2B&logoColor=white) ![Python](https://img.shields.io/badge/python-3670A0?style=for-the-badge&logo=python&logoColor=ffdd54) ![R](https://img.shields.io/badge/r-%23276DC3.svg?style=for-the-badge&logo=r&logoColor=white) ![Java](https://img.shields.io/badge/java-%23ED8B00.svg?style=for-the-badge&logo=openjdk&logoColor=white) ![JavaScript](https://img.shields.io/badge/javascript-%23323330.svg?style=for-the-badge&logo=javascript&logoColor=%23F7DF1E) ![Next JS](https://img.shields.io/badge/Next-black?style=for-the-badge&logo=next.js&logoColor=white) ![NodeJS](https://img.shields.io/badge/node.js-6DA55F?style=for-the-badge&logo=node.js&logoColor=white) ![React Native](https://img.shields.io/badge/react_native-%2320232a.svg?style=for-the-badge&logo=react&logoColor=%2361DAFB) ![React Router](https://img.shields.io/badge/React_Router-CA4245?style=for-the-badge&logo=react-router&logoColor=white) ![SQLite](https://img.shields.io/badge/sqlite-%2307405e.svg?style=for-the-badge&logo=sqlite&logoColor=white) ![MySQL](https://img.shields.io/badge/mysql-4479A1.svg?style=for-the-badge&logo=mysql&logoColor=white) ![PyTorch](https://img.shields.io/badge/PyTorch-%23EE4C2C.svg?style=for-the-badge&logo=PyTorch&logoColor=white) ![scikit-learn](https://img.shields.io/badge/scikit--learn-%23F7931E.svg?style=for-the-badge&logo=scikit-learn&logoColor=white) ![TensorFlow](https://img.shields.io/badge/TensorFlow-%23FF6F00.svg?style=for-the-badge&logo=TensorFlow&logoColor=white) ![Pandas](https://img.shields.io/badge/pandas-%23150458.svg?style=for-the-badge&logo=pandas&logoColor=white) ![NumPy](https://img.shields.io/badge/numpy-%23013243.svg?style=for-the-badge&logo=numpy&logoColor=white) ![Matplotlib](https://img.shields.io/badge/Matplotlib-%23ffffff.svg?style=for-the-badge&logo=Matplotlib&logoColor=black) ![Git](https://img.shields.io/badge/git-%23F05033.svg?style=for-the-badge&logo=git&logoColor=white) ![Figma](https://img.shields.io/badge/figma-%23F24E1E.svg?style=for-the-badge&logo=figma&logoColor=white)
# 📊 GitHub Stats:
![](https://github-readme-stats.vercel.app/api?username=virkahmed&theme=shadow_blue&hide_border=false&include_all_commits=true&count_private=true)<br/>
![](https://nirzak-streak-stats.vercel.app/?user=virkahmed&theme=shadow_blue&hide_border=false)<br/>
![](https://github-readme-stats.vercel.app/api/top-langs/?username=virkahmed&theme=shadow_blue&hide_border=false&include_all_commits=true&count_private=true&layout=compact)

---
[![](https://visitcount.itsvg.in/api?id=virkahmed&icon=0&color=0)](https://visitcount.itsvg.in)

<!-- Proudly created with GPRM ( https://gprm.itsvg.in ) -->
# Limit Order Book & Matching Engine

A high-performance, price-time priority matching engine written in C++. Built from scratch with a focus on low-latency order processing and cache-efficient data structures.

```
2,080,000 orders/sec   ·   378ns median latency   ·   sub-microsecond P99
```

---

## Overview

A limit order book (LOB) is the core data structure behind every electronic exchange — NYSE, NASDAQ, CME, crypto platforms. It maintains all outstanding buy and sell orders, organized by price, and matches them when prices cross.

This implementation supports three operations:
- **Submit** — match an incoming order against the opposite side, rest any unfilled remainder
- **Cancel** — remove a resting order by ID in O(1)
- **Query** — retrieve best bid/ask or an N-level depth snapshot

---

## Performance

### Benchmark Results (Release build)

| Configuration | Throughput |
|---|---|
| Single-thread (raw book) | **2.08M orders/sec** |
| Single-thread (MatchingEngine) | **1.80M orders/sec** |
| 2 threads | 1.92M orders/sec |
| 4 threads | 1.62M orders/sec |

### Latency Distribution (500k orders, single-thread)

| Percentile | Latency |
|---|---|
| P50 (median) | **378 ns** |
| P90 | ~600 ns |
| P99 | **< 1 µs** |
| P99.9 | ~2 µs |
| Max | 317 µs *(one-time OS scheduler interrupt)* |

### Optimization History

| Version | 1 Thread | 4 Threads |
|---|---|---|
| Baseline (`std::map` + `std::unordered_map`) | 55k/sec | 58k/sec |
| + Pool allocator + Release build | 174k/sec | 342k/sec |
| + Tick arrays + custom hash map + block allocator | **2.08M/sec** | 1.62M/sec |

**33× improvement** over the original design.

---

## Architecture

### Price Levels — Flat Tick Arrays

Instead of a red-black tree (`std::map`), prices are converted to integer ticks:

```cpp
static constexpr double TICK_SIZE = 0.01;
static constexpr int64_t MAX_PRICE_TICKS = 1'000'000; // supports up to $10,000.00

static int64_t price_to_tick(Price p) {
    return std::llround(p / TICK_SIZE);
}
```

Two flat arrays of 1M levels — one per side. Accessing any price level is `array[tick]` — **O(1), no tree traversal, no pointer chasing.** At ~24 bytes per `Level` struct, this is ~24MB per side, comfortably fitting in RAM.

Best bid/ask are tracked as integers and updated with a short amortized O(1) scan when a level empties.

### Per-Level Queues — Intrusive Doubly Linked Lists

Each price level holds a doubly linked list of resting orders, enforcing **time priority** (FIFO). The `prev`/`next` pointers are embedded directly in the `Order` struct — no separate list node allocation, better cache locality.

- Append to tail: **O(1)**
- Match from head: **O(1)**
- Cancel from middle: **O(1)** — direct pointer from hash map, rewire neighbors

### Order Lookup — Custom Hash Map

`std::unordered_map` uses separate chaining with heap-allocated buckets — one pointer indirection per lookup. This implementation uses **open addressing with linear probing**: all entries live in a single contiguous array, cache-friendly by construction.

Hash function is splitmix64, which gives near-perfect distribution for sequential integer IDs:

```cpp
size_t hash(OrderId id) {
    id ^= id >> 33;
    id *= 0xff51afd7ed558ccdULL;
    id ^= id >> 33;
    id *= 0xc4ceb9fe1a85ec53ULL;
    id ^= id >> 33;
    return static_cast<size_t>(id);
}
```

### Memory — Block Allocator + Free List

Zero `malloc` on the hot path. Orders are pre-allocated in blocks of 1M. Cancelled or fully-filled orders are pushed onto a free list and recycled immediately — same pattern used by kernel slab allocators.

```cpp
Order* alloc_order(...) {
    if (free_list_) {           // O(1) — pop from free list
        o = free_list_;
        free_list_ = o->next;
    } else {                    // allocate from next pool block
        ...
    }
}
```

### Thread Safety

A single mutex guards the `MatchingEngine`. The critical section is fast enough (~500ns) that contention is minimal — 4 threads achieve **1.62M ops/sec**, only 12% below single-thread peak. For higher throughput, the natural next step is an SPSC queue with a dedicated matching thread.

### Trade Logging — Ring Buffer

A fixed-size `std::array<Trade, 1000>` used as a ring buffer. **O(1)** insert always, no shifting, no allocation. Old trades wrap around naturally.

---

## Complexity Summary

| Operation | Time | Notes |
|---|---|---|
| Best bid/ask | O(1) | Integer read |
| Insert (no match) | O(1) | Array index + list append + hash insert |
| Match per fill | O(1) | List head + hash erase |
| Cancel | O(1) | Hash lookup + list unlink |
| Level scan after empty | Amortized O(1) | Short linear scan over sparse ticks |

---

## Building & Running

```bash
# Build (Release)
cmake -B build -DCMAKE_BUILD_TYPE=Release
cmake --build build

# Run benchmarks
./build/Release/benchmark_orders

# Run tests
./build/Release/tests
```

---

## Order Types

Currently supports **limit orders**. Market orders can be simulated by submitting at an extreme price. IOC/FOK support would require a flag on the `Order` struct and post-match cancellation logic.

---

## What I'd Do Next

- SPSC queue model: dedicated matching thread, zero lock contention
- Symbol sharding: one book + mutex per instrument
- Persistence: append-only order journal with deterministic replay
- Market data feed: publish book updates via multicast after each match
