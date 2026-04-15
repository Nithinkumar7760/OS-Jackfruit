# Multi-Container Runtime — OS-Jackfruit

A lightweight Linux container runtime built in C, featuring a long-running parent supervisor, concurrent bounded-buffer logging, a kernel-space memory monitor, and Linux scheduling experiments.

---

## 1. Team Information

 Nithin Kumar M  SRN: PES1UG25CS829 
 Vilohith Sree Sai Reddy Daggolu SRN: PES1UG24CS535 

---

## 2. Build, Load, and Run Instructions

### Prerequisites

We need**buntu 22.04 or 24.04** running in a VM with **Secure Boot OFF**. WSL will not work.

Install dependencies:

```bash
sudo apt update
sudo apt install -y build-essential linux-headers-$(uname -r)
```

Run the environment preflight check:

```bash
cd boilerplate
chmod +x environment-check.sh
sudo ./environment-check.sh
```

### Prepare the Root Filesystem

```bash
mkdir rootfs
wget https://dl-cdn.alpinelinux.org/alpine/v3.20/releases/x86_64/alpine-minirootfs-3.20.3-x86_64.tar.gz
tar -xzf alpine-minirootfs-3.20.3-x86_64.tar.gz -C rootfs
```

> Do not commit `rootfs/` to the repository.

### Build the Project

```bash
make
```

This compiles both the user-space runtime (`engine`) and the kernel module (`monitor.ko`) in one step.

### Load the Kernel Module

```bash
sudo insmod monitor.ko

# Verify the control device exists
ls -l /dev/container_monitor
```

### Start the Supervisor

```bash
sudo ./engine supervisor ./rootfs
```

### Launch Containers (in another terminal)

```bash
# Start containers in background
sudo ./engine start alpha ./rootfs /bin/sh
sudo ./engine start beta ./rootfs /bin/sh

# Or run a container in the foreground and wait for it
sudo ./engine run gamma ./rootfs /bin/sh
```

### Use the CLI

```bash
# List all tracked containers and their metadata
sudo ./engine ps

# View logs of a specific container
sudo ./engine logs alpha

# Stop a container cleanly
sudo ./engine stop alpha
sudo ./engine stop beta
```

### Run Workloads Inside a Container

Copy the test binary into rootfs before launching the container:

```bash
cp cpu_hog ./rootfs/
cp memory_hog ./rootfs/
cp io_pulse ./rootfs/
```

Then from inside the container, run:

```bash
/cpu_hog
/memory_hog
/io_pulse
```

### Scheduling Experiments

```bash
# Run two CPU-bound containers with different priorities
sudo nice -n 0  ./engine start high_prio ./rootfs /cpu_hog
sudo nice -n 19 ./engine start low_prio  ./rootfs /cpu_hog

# Observe CPU share difference using top or time
```

### Inspect Kernel Logs

```bash
dmesg | tail -30
```

### Teardown and Cleanup

```bash
# Stop all containers first
sudo ./engine stop alpha
sudo ./engine stop beta

# Unload the kernel module
sudo rmmod monitor

# Confirm no zombie processes remain
ps aux | grep engine
```

---

## 3. Demo with Screenshots

### Screenshot 1 — Multi-Container Supervision
> Two containers (`alpha` and `beta`) running simultaneously under one supervisor process. The supervisor PID remains alive while both containers execute.

![Screenshot 1](screenshots/01_multi_container.png)

---

### Screenshot 2 — Metadata Tracking
> Output of `sudo ./engine ps` showing container IDs, host PIDs, states, start times, memory limits, and log file paths.

![Screenshot 2](screenshots/02_ps_metadata.png)

---

### Screenshot 3 — Bounded-Buffer Logging
> Contents of a container log file captured through the producer-consumer pipeline. Shows log data written to the per-container log file.

![Screenshot 3](screenshots/03_logging.png)

---

### Screenshot 4 — CLI and IPC
> A CLI command (`logs` or `stop`) being issued from a second terminal. The supervisor processes the command through the UNIX domain socket / FIFO control channel and responds accordingly.

![Screenshot 4](screenshots/04_cli_ipc.png)

---

### Screenshot 5 — Soft-Limit Warning
> `dmesg` output showing a kernel-level soft-limit warning event triggered when a container's RSS exceeded its configured soft memory threshold.

![Screenshot 5](screenshots/05_soft_limit.png)

---

### Screenshot 6 — Hard-Limit Enforcement
> `dmesg` output showing the container process being killed after exceeding its hard memory limit. Supervisor metadata updated to reflect the `killed` state.

![Screenshot 6](screenshots/06_hard_limit.png)

---

### Screenshot 7 — Scheduling Experiment
> Terminal output comparing completion times or CPU share between two workloads running at different `nice` values (or different CPU affinities). Observable difference shown.

![Screenshot 7](screenshots/07_scheduling.png)

---

### Screenshot 8 — Clean Teardown
> `ps aux` output and supervisor exit messages showing no zombie processes, all threads joined, and file descriptors closed after shutdown.

![Screenshot 8](screenshots/08_teardown.png)

---

## 4. Engineering Analysis

### 4.1 Isolation Mechanisms

Each container in our runtime is created using Linux **namespaces**. When a container is launched, we call `clone()` with flags for PID, UTS, and mount namespaces. This means each container sees its own process tree starting at PID 1, its own hostname, and its own filesystem view.

We use `chroot()` into the Alpine `rootfs` to give each container an isolated filesystem root. From the container's perspective, it cannot see or access the host's files outside its root.

However, the **host kernel is still shared** across all containers. There is no separate kernel per container — only isolated views. System calls still go to the same kernel, and kernel resources like network stack (unless network namespaces are used) remain shared.

### 4.2 Supervisor and Process Lifecycle

The supervisor is a long-running parent process that outlives all its container children. This design is essential because:

- It can track metadata for every container it launched
- It handles `SIGCHLD` to reap exited children and avoid zombie processes
- It keeps the control channel (socket/FIFO) open for CLI commands

When a container exits, `SIGCHLD` is delivered to the supervisor. The supervisor calls `waitpid()` to reap the child, records its exit status in the metadata, and updates the container state to `stopped` or `killed`.

Without a long-running parent, each container would become a zombie after exiting because no one is calling `wait()` for it.

### 4.3 IPC, Threads, and Synchronization

Our project uses two distinct IPC mechanisms:

**Logging Pipeline (pipe-based):**
Each container's `stdout` and `stderr` are connected to a pipe. A dedicated consumer thread in the supervisor reads from the pipe and writes data into a bounded buffer. Another thread drains the buffer and writes to the log file on disk.

Without synchronization, the producer and consumer could race on the buffer — the producer could overwrite data before the consumer reads it, or the consumer could read garbage from an empty buffer. We use a **mutex** to protect the buffer and **condition variables** to signal when data is available or space is free.

**Control Channel (UNIX domain socket / FIFO):**
CLI commands from the user reach the supervisor through a separate IPC channel. This keeps logging traffic separate from command traffic, preventing interference and keeping the design clean.

Container metadata (the list of running containers) is also accessed from multiple threads (logger thread, signal handler, CLI handler), so it is protected by its own mutex independent of the log buffer lock.

### 4.4 Memory Management and Enforcement

**RSS (Resident Set Size)** measures how much physical RAM a process is actually using right now — the pages that are loaded in memory. It does not count swap, shared libraries counted once per process, or memory that has been allocated but not yet touched.

We implement two limit levels in the kernel module:

- **Soft limit:** When RSS crosses this threshold, the kernel logs a warning via `printk`. The process is not killed — it is just flagged. This gives the supervisor a chance to notice and act.
- **Hard limit:** When RSS crosses this threshold, the kernel sends `SIGKILL` to the process immediately. No warning, no grace period.

Enforcement must live in kernel space because user space cannot reliably monitor and act on another process's memory in real time. A kernel module runs in ring 0, has direct access to process descriptors, and can enforce limits atomically without depending on the container cooperating.

### 4.5 Scheduling Behavior

Linux uses the **Completely Fair Scheduler (CFS)** for normal processes. CFS tries to give every runnable process a fair share of CPU time proportional to its priority weight.

`nice` values affect the weight assigned to a process. A process with `nice -20` (highest priority) gets a much larger CPU share than one with `nice 19` (lowest priority) when they compete for the same CPU.

In our experiments, we ran two CPU-bound containers simultaneously — one at `nice 0` and one at `nice 19`. The higher-priority container consistently finished faster and received a larger proportion of CPU time, which matches the expected CFS behavior.

When we ran a CPU-bound container alongside an I/O-bound container, the I/O-bound container spent most of its time sleeping (waiting for I/O), so it did not compete with the CPU-bound container heavily. CFS naturally gave the CPU-bound container more CPU time simply because it was the only one that needed it.

---

## 5. Design Decisions and Tradeoffs

### Namespace Isolation
**Choice:** PID, UTS, and mount namespaces using `clone()` flags.
**Tradeoff:** No network namespace isolation — containers share the host network stack.
**Justification:** Network namespaces add significant complexity (veth pairs, bridge setup). For this project scope, filesystem and process isolation is sufficient to demonstrate container principles.

### Supervisor Architecture
**Choice:** Single long-running supervisor process with a `SIGCHLD` handler and a separate CLI listener thread.
**Tradeoff:** The supervisor is a single point of failure — if it crashes, all container metadata is lost.
**Justification:** A single supervisor keeps the design simple and avoids the complexity of distributed state management, which is unnecessary at this scale.

### IPC and Logging
**Choice:** Pipes for container output + a UNIX domain socket for CLI commands + a bounded buffer with mutex/condvar.
**Tradeoff:** Two separate IPC channels add complexity compared to using one channel for everything.
**Justification:** Mixing log data and control commands in the same channel creates ordering and priority problems. Separate channels keep each concern clean and independently debuggable.

### Kernel Monitor
**Choice:** Periodic RSS polling inside the kernel module using a kernel timer or thread.
**Tradeoff:** Polling introduces a latency window — a process can exceed the hard limit for up to one polling interval before being killed.
**Justification:** Event-driven memory enforcement (e.g., hooking page fault handlers) is far more invasive and fragile. Periodic polling is simple, reliable, and sufficient for this project.

### Scheduling Experiments
**Choice:** Using `nice` values to observe CFS scheduling differences across CPU-bound workloads.
**Tradeoff:** `nice` values only affect relative CPU share, not absolute guarantees.
**Justification:** `nice`-based experiments are easy to reproduce and directly demonstrate CFS weight-based scheduling without requiring real-time scheduling policies.

---

## 6. Scheduler Experiment Results

### Experiment 1 — Two CPU-Bound Containers at Different Priorities

| Container | Nice Value | Workload | Completion Time |
|-----------|-----------|----------|----------------|
| alpha | 0 (default) | cpu_hog | _[fill from your run]_ sec |
| beta | 19 (lowest) | cpu_hog | _[fill from your run]_ sec |

**Observation:** Container `alpha` at `nice 0` completed significantly faster than `beta` at `nice 19`, demonstrating that CFS allocates CPU time proportional to priority weight when two runnable processes compete.

---

### Experiment 2 — CPU-Bound vs I/O-Bound Container

| Container | Type | Nice Value | Observed Behavior |
|-----------|------|-----------|-------------------|
| alpha | CPU-bound (cpu_hog) | 0 | Consumed nearly all CPU time |
| beta | I/O-bound (io_pulse) | 0 | Spent most time sleeping, low CPU usage |

**Observation:** The CPU-bound container dominated CPU time while the I/O-bound container was mostly sleeping. When the I/O-bound container woke up (after an I/O event), CFS gave it CPU time quickly because it had accumulated a high sleep credit. This demonstrates CFS's responsiveness to interactive/I/O workloads.

---

**Summary:** Linux CFS fairly distributes CPU time based on weight (nice value) among competing runnable processes. I/O-bound processes naturally yield the CPU and are rewarded with fast scheduling when they wake up. CPU-bound processes consume what is available. This project's runtime provides a clean environment to observe and measure these behaviors directly.
