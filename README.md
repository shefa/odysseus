# Project Odysseus: High-Frequency Systematic Trading Architecture
Project Odysseus is a production-grade, ultra-low-latency systematic trading infrastructure written in modern C++20 and Python/TypeScript. Named after the epic voyage, this system handles the complete lifecycle of automated trading, simulating institutional market-making and high-frequency trading (HFT) mechanics on Linux.

```
                      [ MODE 1: LIVE CONTROL PLANE ]
                       Alpaca / IBKR Stream Sockets
                                    │
                                    ▼
┌──────────────────┐    IPC: Folly Lock-Free Queue   ┌──────────────────┐
│ CHARYBDIS        ├────────────────────────────────►│ SCYLLA           │
│ Ingestion Engine │    (POSIX Shared Memory)        │ Strategy Engine  │
│                  ├────────────────────────────────►│ ┌──────────────┐ │
└──────────────────┘                                 │ │ CERBERUS     │ │
                                                     │ │ Inline Risk  │ │
                      [ MODE 2: HIGH-THROUGHPUT ]    │ └──────────────┘ │
                      Raw Nasdaq ITCH 5.0 Replay     └─────────┬────────┘
                                                               │
                                                       (Internal Orders)
                                                               │
┌──────────────────┐         IPC UNIX Sockets          ┌───────▼────────┐
│ ARGUS            │◄────────────────────────────────► │ HERMES         │
│ Trader Dashboard │                                   │ Execution Gate │
└──────────────────┘                                   └────────────────┘
```
------------------------------
## 📌 Core Architecture & Component Breakdown
The infrastructure isolates responsibilities across three compiled C++ binaries (Charybdis, Scylla, Telemachus), one statically linked validation safety layer (Cerberus), and an interactive control panel (Argus).
## 1. 🌀 Charybdis: Market Data Ingestion Engine (Binary A)

* Responsibility: Ingests external raw market data, normalizes packet structures into zero-copy, byte-aligned internal C++ structs, and pushes them down the data plane.
* Operational Modes:
* Mode 1 (Live Control Plane): Establishes live asynchronous connections to retail broker endpoints (Alpaca WebSockets / Interactive Brokers API) over public internet sockets.
   * Mode 2 (Data Plane / High-Throughput Backtesting): Employs memory-mapped files (mmap) to read and replay raw, nanosecond-stamped binary files from historical Nasdaq TotalView-ITCH 5.0 data dumps.

## 2. ⚡ Scylla: Core Strategy & Execution Engine (Binary B)

* Responsibility: The centralized, core computational loop of the trading system. It handles processing pipelines, updates localized Limit Order Books (LOB), generates high-speed alpha signals, and builds order instructions.
* Low-Level Details:
* Tied to Charybdis via POSIX Shared Memory (shm_open) utilizing a Folly-based lock-free Single Producer Single Consumer (SPSC) ring buffer to eliminate kernel overhead.
   * The main loop is pinned to a dedicated CPU core (pthread_setaffinity_np) to maximize L1/L2 cache locality.

## 3. 🛡️ Cerberus: Inline Pre-Trade Risk Manager (Statically Linked Module)

* Responsibility: Act as an un-bypassable gatekeeper protecting the fund from catastrophic trading errors.
* Low-Level Details: Rather than existing as a separate binary (which introduces latency-killing IPC hops), Cerberus is compiled as a header-only utility statically linked directly inside Scylla. It executes inline within Scylla's execution thread, evaluating order validation parameters (Max order sizing, price collars, fat-finger prevention, and aggregate exposure limits) in under 100 nanoseconds before allowing an order to route.

## 4. 🕊️ Hermes: Order Management System & Execution Gateway (Binary C)

* Responsibility: Handles outbound execution connectivity, protocol serialization, and network state tracking.
* Low-Level Details:
* Receives valid internal order tokens from Scylla across a lock-free queue.
   * Live Mode: Acts as the network messenger, translating clean internal structs into external API protocols (REST/WebSockets) to talk to brokers.
   * Simulation Mode: Drops the live connection and spins up a local, deterministic price-time priority matching engine to execute trades instantly against the replayed data stream.
   * Implements the structural State Design Pattern to maintain and log the exact lifecycle of every order (Pending New, New, Partially Filled, Filled, Canceled, Rejected) without blocking the strategy thread.

## 5. 👁️ Argus: Trader Dashboard & Hot-Swap Controller (Web Interface)

* Responsibility: The central monitoring plane for human traders. It connects to the backend environment via an API gateway.
* Features:
* Hot-Swap Management: Allows traders to live-inject, update parameters, toggle, or fully enable/disable active quantitative trading strategies running inside Scylla on-the-fly without killing or restarting the compiled C++ processes.
   * Analytics & Observation: Provides real-time PnL tracking, order logs, current inventory levels, and a visual representation of active limit order books.

------------------------------
## 🛠️ Engineering Division of Labor
The project is structured to create a perfect separation of low-level systems architecture and design patterns/visualization frameworks.
## 🏎️ Core Systems & High-Frequency Pipelines (Developer 1)

* Core Stack: Modern C++ (C++20), POSIX Systems Interfaces, Linux System Kernel.
* Scope:
* Implementation of Folly lock-free queues inside POSIX shared memory chunks.
   * Memory mapping logic (mmap) and zero-copy binary parsers for Nasdaq ITCH data dumps.
   * Local limit order book data structures optimized for $O(1)$ allocations.
   * Hardware level optimizations (CPU pinning, thread scheduling, data structure alignment).

## 📐 Software Design Patterns, Verification & Web Plane (Developer 2)

* Core Stack: Python (FastAPI/WebSockets), TypeScript/React, C++ OOP Frameworks.
* Scope:
* Development of the Argus Web Dashboard UI and real-time streaming components.
   * Implementation of structural software design patterns (Factory Pattern for switching data sources, Observer for pushing execution updates, State Pattern for order state transitions inside Telemachus).
   * Design of a strict Unit Testing & Regression framework to run mock execution profiles through the local order books to find edge-case synchronization errors.
   * Strategy parameter hot-swap endpoints that write directly to volatile memory registers watched by Scylla.

------------------------------
## 📂 Repository Layout
```
├── CMakeLists.txt                 # Highly optimized build configuration (-O3, native)
├── README.md                      # Architecture specifications
├── src/
│   ├── common/                    # Header-only structures, shared primitives
│   │   ├── types.hpp              # Byte-aligned, fixed-width protocol structs
│   │   └── lockfree_queue.hpp     # Folly SPSC queue wrapper
│   ├── charybdis/                 # Binary A: Live Streams & Mapped ITCH Replayer
│   │   ├── include/
│   │   ├── live_feed_handler.cpp
│   │   ├── itch_file_replayer.cpp
│   │   └── main.cpp
│   ├── scylla/                    # Binary B: Order Books, Signals, Risk
│   │   ├── include/
│   │   ├── cerberus_risk.hpp      # Inline static pre-trade risk layer
│   │   ├── order_book.cpp         # Price/Time priority queues
│   │   └── main.cpp
│   ├── hermes/                    # Binary C: External Broker API & Local Simulator
│   │   ├── exchange_stub.cpp      
│   │   └── main.cpp
│   └── argus_api/                 # Python API Gateway connecting C++ to Dashboard
│       └── server.py              # FastAPI controller for hot-swapping parameters
└── ui/                            # React / TypeScript Argus Trader Frontend
    ├── src/
    └── package.json
```
------------------------------
