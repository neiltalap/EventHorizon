# EventHorizon

> **"The Black Box for Linux Infrastructure."**

![Status](https://img.shields.io/badge/Status-Alpha-red) ![Language](https://img.shields.io/badge/Language-C%2B%2B20%20%7C%20eBPF-blue)

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

## 🛡 Resiliency Under Load

EventHorizon is designed for reliability even under extreme system load. During periods of massive CPU or memory pressure, standard observability agents often fail or drop data due to process starvation and memory swapping. EventHorizon mitigates these risks using:

1.  **SCHED_FIFO (Real-Time Priority):** The agent is configured as a `SCHED_FIFO` process with maximum priority (99). This ensures the Linux kernel prioritizes EventHorizon's access to CPU resources, guaranteeing timely processing of events even when other system processes are heavily contended.
2.  **mlockall (Memory Locking):** EventHorizon locks its entire memory footprint into physical RAM. This prevents the operating system from swapping the agent's memory to disk, which would introduce critical latency and compromise data integrity during memory-intensive conditions.

These mechanisms ensure EventHorizon continues to capture and stream critical telemetry without interruption, regardless of host machine stress.
