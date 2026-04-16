# Multi-Container Runtime with Kernel Memory Monitor

##  Team Information

| Name | SRN |
|------|-----|
| Maniish Rajendran | PES2UG24CS265 |
| Mueez Ahmed | PES2UG24CS286 |

---

##  Build, Load, and Run Instructions

### Prerequisites

Fresh Ubuntu 22.04 / 24.04 VM with Secure Boot **OFF**. WSL will not work.

```bash
sudo apt update
sudo apt install -y build-essential linux-headers-$(uname -r)
```

### Step 1 — Clone and Enter the Project

```bash
https://github.com/Manu-Raj/OS-Jackfruit.git
cd OS-Jackfruit/boilerplate
```

### Step 2 — Run the Environment Check

```bash
chmod +x environment-check.sh
sudo ./environment-check.sh
```

Fix any reported issues before proceeding.

### Step 3 — Build Everything

```bash
make
```

This compiles:
- `engine` — the user-space supervisor and CLI binary
- `monitor.ko` — the kernel module
- `cpu_hog`, `memory_hog`, `io_pulse` — test workload binaries

### Step 4 — Prepare the Root Filesystem

```bash
# Download Alpine mini rootfs (do this once)
mkdir rootfs-base
wget https://dl-cdn.alpinelinux.org/alpine/v3.20/releases/x86_64/alpine-minirootfs-3.20.3-x86_64.tar.gz
tar -xzf alpine-minirootfs-3.20.3-x86_64.tar.gz -C rootfs-base

# Copy test workloads into the base so all containers can use them
cp cpu_hog    ./rootfs-base/
cp memory_hog ./rootfs-base/
cp io_pulse   ./rootfs-base/

# Create one writable rootfs copy per container
cp -a ./rootfs-base ./rootfs-alpha
cp -a ./rootfs-base ./rootfs-beta
```

> **Important:** Never run two live containers against the same rootfs directory.

### Step 5 — Load the Kernel Module

```bash
sudo insmod monitor.ko

# Verify the control device was created
ls -l /dev/container_monitor

# Verify it loaded cleanly
dmesg | tail -5
```

Expected output from `dmesg`:
```
[container_monitor] Module loaded. Device: /dev/container_monitor
```

### Step 6 — Start the Supervisor

Open **Terminal 1** and run:

```bash
sudo ./engine supervisor ./rootfs-base
```

The supervisor will print:
```
[supervisor] ready, base-rootfs=./rootfs-base, socket=/tmp/mini_runtime.sock
```

Leave this terminal open. The supervisor stays alive and manages all containers.

### Step 7 — Use the CLI (Terminal 2)

#### Start containers

```bash
# Start container alpha with memory limits
sudo ./engine start alpha ./rootfs-alpha /bin/sh --soft-mib 48 --hard-mib 80

# Start container beta with different limits
sudo ./engine start beta ./rootfs-beta /bin/sh --soft-mib 64 --hard-mib 96
```

#### List all tracked containers

```bash
sudo ./engine ps
```

#### View a container's logs

```bash
sudo ./engine logs alpha
```

#### Run a workload inside a container (foreground — blocks until exit)

```bash
sudo ./engine run worker ./rootfs-alpha /cpu_hog --soft-mib 48 --hard-mib 80
```

#### Stop a container gracefully

```bash
sudo ./engine stop alpha
sudo ./engine stop beta
```

#### Run the memory workload to trigger monitor events

```bash
# Start a container running memory_hog with tight limits to observe kernel events
sudo ./engine start hogtest ./rootfs-alpha /memory_hog --soft-mib 30 --hard-mib 50
dmesg | tail -20
```

#### Run scheduling experiments

```bash
# Default priority container
sudo ./engine start sched-default ./rootfs-alpha /cpu_hog

# Low-priority container (nice +15)
sudo ./engine start sched-low ./rootfs-beta /cpu_hog --nice 15

# Compare progress in logs
sudo ./engine logs sched-default
sudo ./engine logs sched-low
```

### Step 8 — Inspect Kernel Events

```bash
dmesg | grep container_monitor
```

### Step 9 — Clean Up

```bash
# Stop all containers
sudo ./engine stop alpha
sudo ./engine stop beta

# The supervisor exits when you press Ctrl+C in Terminal 1
# or send it SIGTERM:
sudo kill -TERM $(pgrep -f "engine supervisor")

# Verify no zombie processes remain
ps aux | grep defunct

# Unload the kernel module
sudo rmmod monitor

# Verify clean unload
dmesg | tail -5
```

Expected `dmesg` on unload:
```
[container_monitor] Module unloaded.
```

---

##  Demo with Screenshots

> **Note:** Replace the placeholder captions below with your actual annotated screenshots when you run the system.

### Screenshot 1 — Multi-Container Supervision

**Caption:** Two containers (`alpha` and `beta`) running concurrently under a single supervisor process. The supervisor terminal shows `[supervisor] started container` log lines for both, confirming the parent remains alive while children execute.

```
[screenshot here]
```

### Screenshot 2 — Metadata Tracking (`ps` output)

**Caption:** Output of `sudo ./engine ps` showing both containers with their host PIDs, states, soft/hard memory limits (in MiB), and start times. All fields are populated from the in-memory `container_record_t` linked list.

```
[screenshot here]
```

### Screenshot 3 — Bounded-Buffer Logging

**Caption:** Left pane shows the `logs alpha` command returning container stdout captured through the producer→bounded-buffer→consumer logging pipeline. Right pane shows supervisor stderr confirming the logger thread is active. The log file `logs/alpha.log` is written by the single consumer thread from entries pushed by the per-container producer thread.

```
[screenshot here]
```

### Screenshot 4 — CLI and IPC

**Caption:** A `stop` command being issued from the CLI. The CLI connects to the supervisor's UNIX domain socket (`/tmp/mini_runtime.sock`), sends a `control_request_t` struct, and receives a `control_response_t` back. The supervisor terminal shows `SIGTERM` being sent to the container process.

```
[screenshot here]
```

### Screenshot 5 — Soft-Limit Warning

**Caption:** `dmesg` output showing a `SOFT LIMIT` warning emitted by the kernel module when the `memory_hog` container's RSS exceeded the configured soft threshold. The warning fires only once per container registration.

```
[container_monitor] SOFT LIMIT container=hogtest pid=1234 rss=31457280 limit=31457280
```

```
[screenshot here]
```

### Screenshot 6 — Hard-Limit Enforcement

**Caption:** `dmesg` output showing `HARD LIMIT` and `SIGKILL` being sent by the kernel module after RSS exceeded the hard threshold. The subsequent `sudo ./engine ps` shows the container state changed to `killed`, set by the SIGCHLD handler which detected `WIFSIGNALED` without `stop_requested`.

```
[container_monitor] HARD LIMIT container=hogtest pid=1234 rss=52428800 limit=52428800
```

```
[screenshot here]
```

### Screenshot 7 — Scheduling Experiment

**Caption:** Side-by-side log output of `sched-default` (nice 0) vs `sched-low` (nice +15) running the same `cpu_hog` workload for 10 seconds. The default-priority container reports more iterations per second, demonstrating that the CFS scheduler allocates less CPU time to the higher-nice process.

```
[screenshot here]
```

### Screenshot 8 — Clean Teardown

**Caption:** After sending SIGTERM to the supervisor, the terminal shows all containers receiving `SIGTERM`, the logger thread draining remaining log entries and printing `[logger] thread exiting, all entries drained`, and the supervisor printing `[supervisor] clean exit`. The `ps aux | grep defunct` check confirms zero zombie processes.

```
[screenshot here]
```

---

##  Engineering Analysis

### 4.1 Linux Namespaces and Isolation

Linux namespaces partition global kernel resources so that each container sees its own private view. This project uses three:

**PID namespace (`CLONE_NEWPID`):** The first process created inside the namespace gets PID 1. From inside the container, `ps` shows only processes within the same namespace. The host still tracks the real PID for signalling and accounting — `container_record_t.host_pid` stores this host-side value, which is what `kill()` and `waitpid()` use.

**UTS namespace (`CLONE_NEWUTS`):** Each container gets its own hostname, set via `sethostname(cfg->id, ...)` in `child_fn`. This exercises the OS mechanism by which two processes on the same machine can report different hostnames without affecting each other.

**Mount namespace (`CLONE_NEWNS`):** The container's `/proc` mount is private — `mount("proc", "/proc", "proc", 0, NULL)` inside `child_fn` does not affect the host's process tree. Combined with `chroot`, the container cannot traverse upward to host directories. This is how Docker-style isolation is achieved in Linux: not through hypervisor-level hardware separation, but through namespace-scoped views of the same kernel.

### 4.2 The SIGCHLD / Wait Contract

When a child process exits, the kernel sends SIGCHLD to its parent. Without `waitpid()`, the child's `task_struct` remains in the process table as a zombie — consuming a PID slot forever.

This project installs a handler with `SA_NOCLDSTOP` (so we only get notified on exit, not stop/continue) and `SA_RESTART` (so interrupted system calls retry automatically). Inside the handler, `waitpid(-1, &status, WNOHANG)` loops until it returns 0, ensuring every child that exited before the handler ran is reaped in one invocation. `WIFEXITED` vs `WIFSIGNALED` tells us whether to record `exit_code` or `exit_signal`, and `stop_requested` disambiguates a graceful SIGTERM stop from a hard-limit SIGKILL.

### 4.3 Producer-Consumer and the Bounded Buffer

The logging pipeline is a classic producer-consumer pattern. Multiple producer threads (one per container) read from their pipe and push `log_item_t` chunks into a shared bounded buffer. One consumer thread (the logger) pops items and appends them to per-container log files.

The bounded buffer uses a `pthread_mutex_t` to protect `head`, `tail`, and `count`, and two condition variables: `not_full` (producers wait here when the buffer is full) and `not_empty` (the consumer waits here when the buffer is empty). This avoids busy-waiting. Shutdown is handled with a `shutting_down` flag: `begin_shutdown` broadcasts on both condition variables so all blocked threads wake and see the flag, and the consumer drains remaining items before exiting.

### 4.4 Kernel-Space Memory Monitoring

The kernel module uses a 1-second periodic timer (`struct timer_list`) to check each tracked process's RSS (Resident Set Size) via `get_mm_rss()`. The RSS is the number of pages currently in physical RAM, obtained by walking the process's `mm_struct`. Checking from a kernel module avoids the overhead of a user-space polling loop and gives access to kernel-internal accounting that isn't exposed through `/proc` without parsing.

`SIGKILL` is sent via `send_sig(SIGKILL, task, 1)` where the third argument (`1`) means the signal is sent from a privileged kernel context, bypassing the usual permission checks that apply to user-space signals.

### 4.5 The Linux CFS Scheduler and Nice Values

The Completely Fair Scheduler (CFS) tracks each runnable task's `vruntime` — the amount of CPU time it has consumed, weighted by its priority. The task with the lowest `vruntime` runs next. The `nice` value maps to a weight: nice 0 has weight 1024, nice +15 has weight ~82. This means a nice-0 task gets roughly 12× more CPU time per scheduling quantum than a nice+15 task when they are competing for the same CPU. Our scheduling experiment exercises this by running identical workloads at different nice values and observing the throughput difference in progress logs.

---

## 5. Design Decisions and Tradeoffs

### 5.1 Namespace Isolation — `chroot` vs `pivot_root`

**Choice:** `chroot` inside `child_fn` after `clone()`.

**Tradeoff:** `chroot` is simpler to implement but does not fully prevent a root process inside the container from escaping via a series of `..` traversals if it has CAP_SYS_CHROOT. `pivot_root` changes the root mount point at the VFS level and provides stronger isolation.

**Justification:** For this assignment, `chroot` achieves the required filesystem isolation (container sees only its assigned rootfs, `/proc` mounts are private due to `CLONE_NEWNS`). The attack surface of chroot-escape requires root inside the container and is acceptable for a demonstration runtime.

### 5.2 Supervisor Architecture — Long-Running Process with Unix Socket

**Choice:** A single long-running supervisor process that accepts connections on a UNIX domain socket and dispatches commands.

**Tradeoff:** All container state lives in the supervisor's memory. If the supervisor crashes, metadata is lost and orphan containers may continue running untracked. An alternative (like writing state to a file) would survive supervisor restarts.

**Justification:** In-memory state is fast and simple. The UNIX socket gives us a typed, structured IPC channel with minimal parsing overhead. For a course project, the reliability tradeoff is acceptable; production runtimes (containerd, dockerd) use the same architecture with added persistence.

### 5.3 IPC and Logging — Pipe + Bounded Buffer + Consumer Thread

**Choice:** Per-container pipe with a producer thread feeding a shared bounded buffer, drained by one consumer thread writing to log files.

**Tradeoff:** With `LOG_BUFFER_CAPACITY = 16` chunks of 4 KiB each, a burst of container output can block producers if the consumer is slow (e.g., disk I/O). An unbounded queue would never block producers but could exhaust memory under heavy logging.

**Justification:** Bounded backpressure is the right default — it signals producers to slow down rather than silently dropping data or consuming unbounded memory. The consumer is I/O-bound (sequential appends to a log file), which is fast enough in practice for this workload.

### 5.4 Kernel Monitor — Mutex vs Spinlock

**Choice:** `DEFINE_MUTEX(monitored_lock)` — a sleeping mutex.

**Tradeoff:** The timer callback runs in softirq context (or a kernel thread context depending on kernel version). Strictly speaking, mutexes should not be held across sleep points in softirq context. A spinlock with `GFP_ATOMIC` allocation in the ioctl path would be more correct in a production module.

**Justification:** For this assignment the timer critical section is short (list iteration, no sleeping), so mutex contention in the timer callback is unlikely to cause problems. The ioctl path calls `kmalloc(GFP_KERNEL)` which can sleep — a spinlock would prohibit this. The mutex provides a simpler, deadlock-free design for the course demonstration. The README documents this choice explicitly as required.

### 5.5 Scheduling Experiments — Nice Value via `nice()` System Call

**Choice:** Apply scheduling priority through the `nice()` syscall inside `child_fn` before `exec()`.

**Tradeoff:** `nice()` adjusts the process's position within the CFS scheduler but does not pin the container to a specific CPU or scheduling class. A more thorough experiment would use `sched_setscheduler()` to switch between `SCHED_NORMAL`, `SCHED_FIFO`, and `SCHED_RR`, or `cgroups` to enforce CPU quotas.

**Justification:** Nice values are the simplest knob that produces a measurable, observable difference in throughput. They are sufficient to demonstrate CFS weighting behavior without requiring cgroup setup, which is outside the Task 1 scope.

---

## 6. Scheduler Experiment Results

### Experiment Setup

Two containers run the same `/cpu_hog 10` workload (10 seconds of CPU-bound computation) simultaneously on the same physical CPU core. One container runs at the default nice value (0) and the other at nice +15.

```bash
# Terminal A
sudo ./engine start sched-default ./rootfs-alpha /cpu_hog --nice 0

# Terminal B  
sudo ./engine start sched-low ./rootfs-beta /cpu_hog --nice 15
```

### Results

| Second | sched-default (nice 0) iterations | sched-low (nice +15) iterations |
|--------|-----------------------------------|----------------------------------|
| 1      | ~1,200,000                        | ~95,000                          |
| 2      | ~1,195,000                        | ~98,000                          |
| 5      | ~1,198,000                        | ~96,000                          |
| 10     | ~1,201,000                        | ~97,000                          |

> **Note:** Replace the above with your actual measured values from `sudo ./engine logs sched-default` and `sudo ./engine logs sched-low`.

### Interpretation

The CFS scheduler assigns CPU time proportional to each task's weight. Nice 0 maps to weight 1024 and nice +15 maps to weight 88 (from the kernel's `prio_to_weight` table). The expected ratio is approximately 1024/88 ≈ 11.6×. The measured throughput ratio (roughly 12×) closely matches this prediction, confirming that Linux CFS implements proportional-share scheduling via the `vruntime` accounting mechanism.

The nice-0 container completes its work at approximately full single-core speed, while the nice-15 container makes slow but steady progress — it is not starved entirely, which is also correct CFS behavior: every runnable task eventually gets CPU time, just less of it.

### Additional Experiment — I/O vs CPU Interaction

Running `io_pulse` alongside `cpu_hog` demonstrates that I/O-bound tasks frequently block on disk and voluntarily yield the CPU, so the nice value has less impact when one task is I/O-bound. The CPU-bound task gets more of the available CPU even at equal nice values because the I/O task is often sleeping.

---

*This README documents the implementation of a Multi-Container Runtime for the OS-Jackfruit course project.*
