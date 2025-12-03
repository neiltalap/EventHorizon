# EventHorizon

> **"The Black Box for Linux Infrastructure."**

![Status](https://img.shields.io/badge/Status-Alpha-red) ![Language](https://img.shields.io/badge/Language-C%2B%2B20%20%7C%20eBPF-blue) ![Architecture](https://img.shields.io/badge/Architecture-Event--Driven%20Columnar-green)

**EventHorizon** is a high-performance, **Level 3 (L3)** observability agent designed to permanently decouple system telemetry from the host machine.

It operates on a radical philosophy: **You should never need to SSH into a production server.**

Instead of sampling metrics (CPU is 80%) or aggregating counters, EventHorizon records the **singular atomic events** of the Linux Kernel—process execution, file I/O, network handshakes, and memory allocations—compresses them using domain-specific columnar algorithms, and streams them instantly to object storage (S3).

---

## 🚀 The Core Philosophy: Level 3 Observability

Industry standard observability is "lossy." EventHorizon is **lossless**.

| Level | Type | Example | What you miss |
| :--- | :--- | :--- | :--- |
| **L1** | **Metrics** | `cpu_usage: 90%` | *Why* is it 90%? Which thread? |
| **L2** | **Logs/Tracing** | `nginx: 500 Error` | What system calls failed? Was it a permission error? |
| **L3** | **EventHorizon** | `PID 400 (nginx) calling openat() on /etc/ssl/certs failed (EACCES) at 12:00:01.004` | **Nothing.** |

---

## 🏗 Architecture

EventHorizon is not a monitoring agent; it is a **streaming database engine** that lives on your host.

```mermaid
graph LR
    subgraph Kernel Space
        A[eBPF Probes] -->|Raw Structs| B[Ring Buffer]
    end
    subgraph User Space (C++ Agent)
        B --> C[Ingestor Thread]
        C --> D{Columnar Compressor}
        D -->|Delta-of-Delta| E[Timestamp Stream]
        D -->|Global Dictionary| F[String Stream]
        D -->|RLE| G[PID Stream]
        E & F & G --> H[In-Memory Parquet Chunk]
        H -->|HTTP PUT| I[S3 / MinIO]
    end
    subgraph "The Analyst's Laptop"
        I -->|DuckDB / ClickHouse| J[Forensic Querying]
    end
```

---

## 🛡 Process Starvation: The "Immortal" Agent

When a Linux machine is under "massive load" (Load Average > CPU Count), the kernel scheduler (CFS) starts delaying processes to give everyone a "fair share."

If your observability agent is treated 'fairly', you lose. You will miss events because the kernel won't give your C++ agent CPU time to empty the Ring Buffer.

To fix this, we turn the agent into a **Real-Time Process**.

### 1. The Weapon: SCHED_FIFO
Linux has a "Real-Time" scheduling class that takes precedence over everything else.
* **Normal Processes:** Use `SCHED_OTHER`. They fight for CPU.
* **EventHorizon:** Uses `SCHED_FIFO` with Priority 99 (Max).

If a `SCHED_FIFO` process wants the CPU, the kernel immediately pauses whatever else is running (even if it's halfway through a calculation) and gives the core to you.

### 2. The Shield: mlockall
High load often means Memory Pressure. When RAM is full, Linux starts "Swapping" (moving idle program memory to disk).
If Linux swaps the agent's code to disk, and then we try to process an event, we die.

We use `mlockall` to force the kernel to keep every byte of the agent in physical RAM, forever.

### 3. The Implementation (main.cpp)
We add a `make_immortal()` function to the start of `main.cpp`:

```cpp
#include <sys/mman.h>   // For mlockall
#include <sched.h>      // For sched_setscheduler
#include <sys/resource.h>

void make_immortal() {
    // 1. THE SHIELD: Lock memory to prevent swapping
    // MCL_CURRENT: Lock current memory
    // MCL_FUTURE: Lock any memory we allocate in the future
    if (mlockall(MCL_CURRENT | MCL_FUTURE) == -1) {
        std::cerr << "[WARN] Failed to lock memory. Sudo required?" << std::endl;
    } else {
        std::cout << "[INFO] Memory locked. Swap immune." << std::endl;
    }

    // 2. THE WEAPON: Set Real-Time Scheduler
    struct sched_param param;
    param.sched_priority = 99; // Maximum priority (1-99)

    // SCHED_FIFO: We run until we block (sleep) or yield. 
    // No other standard process can interrupt us.
    if (sched_setscheduler(0, SCHED_FIFO, &param) == -1) {
        std::cerr << "[WARN] Failed to set Real-Time priority." << std::endl;
    } else {
        std::cout << "[INFO] Running as Real-Time Process (Priority 99)." << std::endl;
    }
}

int main() {
    make_immortal(); // <--- Call this first
    
    // ... load eBPF and start loop ...
}
```

### The "Nuclear Option": CPU Isolation
If `SCHED_FIFO` isn't enough (e.g., high-frequency trading), we use CPU Pinning (`isolcpus=3`) to remove a CPU core from the scheduler entirely, dedicating it solely to EventHorizon.
