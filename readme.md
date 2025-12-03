# EventHorizon

> **"The Black Box for Linux Infrastructure."**

![Status](https://img.shields.io/badge/Status-Alpha-red) ![Language](https://img.shields.io/badge/Language-C%2B%2B20%20%7C%20eBPF-blue)

**EventHorizon** is a high-performance, **Level 3 (L3)** observability agent designed to permanently decouple system telemetry from the host machine.

It operates on a radical philosophy: **You should never need to SSH into a production server.**

Instead of sampling metrics (CPU is 80%) or aggregating counters, EventHorizon records the **singular atomic events** of the Linux Kernel—process execution, file I/O, network handshakes, and memory allocations—compresses them using domain-specific columnar algorithms, and streams them instantly to object storage (S3).

---

## [1] The Core Philosophy: Level 3 Observability

Industry standard observability is "lossy." EventHorizon is **lossless**.

| Level | Type | Example | What you miss |
| :--- | :--- | :--- | :--- |
| **L1** | **Metrics** | `cpu_usage: 90%` | *Why* is it 90%? Which thread? |
| **L2** | **Logs/Tracing** | `nginx: 500 Error` | What system calls failed? Was it a permission error? |
| **L3** | **EventHorizon** | `PID 400 (nginx) calling openat() on /etc/ssl/certs failed (EACCES) at 12:00:01.004` | **Nothing.** |

---

## [2] The "Load Average" Deception

Why do we need L3 observability? Because standard Linux metrics are mathematically broken.

On Linux, **Load Average** is defined as:

```
Load = (CPU Running + CPU Waiting) + (Disk Waiting)
       \_________________________/   \___________/
                 R-State                D-State
```

This metric is a lie. It sums two completely unrelated physical states—CPU Cycle Demand and Disk I/O Latency—into a single number.

| Metric | The Tool (`top`) | The Reality | The Ambiguity |
| :--- | :--- | :--- | :--- |
| **Load Avg: 10** | "System is busy." | **Scenario A:** 10 threads crunching math (R-State). <br> **Scenario B:** 10 threads deadlocked waiting for NVMe (D-State). | You cannot tell the difference. You guess. |
| **CPU: 50%** | "We have capacity." | The CPU is idle only because it is blocked by a saturated disk. | **Utilization hides Saturation.** |

### Why `top` belongs in the trash

Using `top` to debug a production outage is like trying to fly a plane by looking at a "Happiness Meter."

If the meter says "50," are you halfway to your destination, or halfway to crashing into a mountain? You don't know. It just says "50."

When you use Load Average to auto-scale, you are often setting money on fire. If your Load is high because of **Disk I/O** (D-State), adding more CPUs doesn't fix it—it just adds more threads to the queue waiting for the same slow disk. You didn't solve the bottleneck; you just made the traffic jam more expensive.

**EventHorizon removes the guesswork.**
It does not aggregate "Load." It records the atomic cause of the delay.

> **EventHorizon Log:**
> `PID 400 blocked for 520ms in state TASK_UNINTERRUPTIBLE (D-State) waiting on blk_mq_get_tag (Disk I/O).`

We replace "Symptom Monitoring" with **Deterministic Causality.**

---

## [3] Resiliency Under Load

EventHorizon is designed for reliability even under extreme system load. During periods of massive CPU or memory pressure, standard observability agents often fail or drop data due to process starvation and memory swapping. EventHorizon mitigates these risks using:

1.  **SCHED_FIFO (Real-Time Priority):** The agent is configured as a `SCHED_FIFO` process with maximum priority (99). This ensures the Linux kernel prioritizes EventHorizon's access to CPU resources, guaranteeing timely processing of events even when other system processes are heavily contended.
2.  **mlockall (Memory Locking):** EventHorizon locks its entire memory footprint into physical RAM. This prevents the operating system from swapping the agent's memory to disk, which would introduce critical latency and compromise data integrity during memory-intensive conditions.

These mechanisms ensure EventHorizon continues to capture and stream critical telemetry without interruption, regardless of host machine stress.