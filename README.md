# High-Performance-C++-Order-Matching-Engine
C++20 Limit Order Book Matching Engine

## 1 Project Overview

A deterministic, multi-threaded Limit Order Book (LOB) implemented in C++20. This project simulates the core logic of an exchange matching engine, focusing on correct implementation of complex order types, price-time priority, and concurrent housekeeping tasks.

Note to Reviewers: This implementation uses standard STL containers (std::map, std::shared_ptr) for clarity and correctness in demonstrating matching logic. In a production low-latency environment, these would be replaced by custom flat maps, memory arenas, and intrusive linked lists to minimize cache misses and heap allocations.

## 2 Core Functionality

The engine supports real-time order processing with full lifecycle management (Add, Cancel, Modify/Replace).

Supported Order Types

Limit Orders (GTC): Standard resting liquidity.

Market Orders: Immediate liquidity taking, sweeping multiple price levels until filled or book is empty.

Fill and Kill (FAK): Partial fills allowed; remainder is immediately cancelled.

Fill or Kill (FOK): Atomic execution; requires full quantity availability at requested price or better, otherwise entirely rejected (implemented via dry-run lookahead).

Good For Day (GFD): Orders automatically expire at end-of-session via an asynchronous pruning mechanism.

### 3 System Architecture & Key Decisions

Data Structure Hierarchy

The system uses a hybrid approach to balance ordered iteration with O(1) lookups.

Component

Structure

Justification & Trade-offs

Book State

std::map<Price, Level>

Keeps price levels sorted for FIFO matching. Trade-off: O(log N) insertions vs O(1) in a fixed-size flat array, accepted for dynamic price range flexibility.

Price Level

std::list<Order>

fast insertions/deletions at the end (for time priority). Crucial: Iterators remain valid during other insertions, unlike std::vector.

Order Lookup

std::unordered_map<ID, Iterator>

Provides O(1) access for cancellations/modifications without traversing price levels.

### 4 Concurrency Model

The engine uses a coarse-grained locking model to ensure state consistency between the main matching thread and maintenance threads.

Hot Path (Matching): Protected via mutex to ensure atomic processing of incoming orders.

Cold Path (Expiry): A dedicated background thread handles GFD (Good For Day) expiry, acquiring locks only when pruning is required to minimize contention on the matching hot path.

### 5 Testing Strategy

Implemented a file-based deterministic replay system using Google Test.

Input: Trace files containing sequences of mixed order types (ADD, CANCEL, MODIFY).

Assertion: Validates final book state (total orders, bid/ask counts) against expected output.

Coverage: extensive tests for edge cases, including FOK atomicity failures, self-matching prevention, and empty book market sweeps.

Build & Run

Requirements: C++20 compiler, CMake, Google Test.

mkdir build && cd build && cmake .. && make && ./orderbook_tests
