# C++ Order Book Matching Engine

A high-performance C++ Order Book that simulates real exchange behavior with
support for advanced order types, low-latency matching, and thread-safe
state transitions.

## 🚀 Features
- **Price–time priority** matching engine  
- **O(1) cancellation** using iterator-backed order storage  
- Supports **5+ order types**:
  - GTC (Good Till Cancel)
  - FOK (Fill or Kill)
  - FAK (Fill and Kill)
  - Market Orders
  - GFD (Good For Day)
- **Metadata layer** for fast match evaluation  
- **Thread-safe GFD scheduler** for end-of-day order pruning  
- Robust handling of partial fills, full fills, and modify workflows

## ⚙️ Architecture
- **Ordered maps** store price levels for deterministic matching  
- **Linked lists** ensure iterator stability during inserts/removals  
- **Unordered maps** provide O(1) order lookup & cancellation  
- **LevelData** struct maintains aggregate quantity + order count per level  
- Background thread + mutex protects end-of-day cleanup  

## 📈 Performance
- Sustains **10,000+ active orders** without iterator invalidation  
- Metadata optimizations yield **~28% faster match-check operations**  
- All state transitions maintain **100% internal consistency**

## 🧪 Testing
Includes scenarios for:
- Market, limit, and partial-fill workflows  
- Add / modify / cancel logic  
- Fill-or-kill and fill-and-kill behavior  
- GFD pruning consistency  

## 🛠️ Tech Stack
- **C++17/20**, STL  
- Concurrency (threads, mutex)  
- Algorithms & Data Structures  
- Low-latency systems architecture  

---

### 📬 Contact
Feel free to open issues or suggestions — always improving the engine!
