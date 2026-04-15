Multi-Container Runtime
A lightweight Linux container runtime in C with a long-running supervisor and a kernel-space memory monitor.
1. Team Information
Akshatha P PES2UG24CS048   Aditi Agarwal PES2UG24CS029
2. Build, Load, and Run Instructions
Prerequisites
Ubuntu 22.04 or 24.04 in a VM (Secure Boot OFF, no WSL)
Dependencies:
bashsudo apt update
sudo apt install -y build-essential linux-headers-$(uname -r)
Clone the Repository
bashgit clone https://github.com/akshatha2005/OS-Jackfruit.git
cd OS-Jackfruit/boilerplate
Prepare the Root Filesystem
bashmkdir rootfs
wget https://dl-cdn.alpinelinux.org/alpine/v3.20/releases/x86_64/alpine-minirootfs-3.20.3-x86_64.tar.gz
tar -xzf alpine-minirootfs-3.20.3-x86_64.tar.gz -C rootfs
Build
bashmake
This builds engine, memory_hog, cpu_hog, io_pulse, and monitor.ko.
Load Kernel Module
bashsudo insmod monitor.ko
ls -l /dev/container_monitor   # verify device exists
sudo dmesg | tail -3           # verify module loaded
Run
bash# Copy workloads into rootfs
cp cpu_hog ./rootfs/
cp memory_hog ./rootfs/
cp io_pulse ./rootfs/

# Terminal 1: Start supervisor
sudo ./engine supervisor ./rootfs

# Terminal 2: Start containers
sudo ./engine start alpha ./rootfs /cpu_hog
sudo ./engine start beta ./rootfs /memory_hog --soft-mib 10 --hard-mib 20

# List tracked containers
sudo ./engine ps

# Inspect logs
sudo ./engine logs alpha

# Stop a container
sudo ./engine stop alpha
Unload and Clean Up
bash# Stop all containers
sudo ./engine stop alpha
sudo ./engine stop beta

# Ctrl+C the supervisor (it will print "Supervisor exited cleanly.")

# Unload kernel module
sudo rmmod monitor

# Verify no zombies
ps aux | grep defunct

# Clean build artifacts
make clean

4. Engineering Analysis
4.1 Isolation Mechanisms
Each container is created using clone() with CLONE_NEWPID, CLONE_NEWUTS, and CLONE_NEWNS flags. These create isolated namespaces so that each container has its own PID space (PID 1 inside the container), its own hostname (set via sethostname()), and its own mount namespace.
chroot() is used to restrict the container's filesystem view to the Alpine rootfs. /proc is then mounted inside the container so that process utilities like ps work correctly within the isolated environment.
The host kernel is still shared across all containers — they run on the same kernel, share kernel memory, and use the same scheduler. Isolation is logical, not physical. The kernel itself, system calls, hardware drivers, and physical memory management remain common to all containers and the host.
4.2 Supervisor and Process Lifecycle
A long-running parent supervisor is useful because it maintains state about all running containers, handles their lifecycle events, and provides a control interface. Without a persistent supervisor, there would be no entity to reap exited children, track metadata, or respond to CLI commands.
When a container is started, clone() creates a child process with new namespaces. The supervisor records the child's host PID in its metadata list. When the child exits, the kernel sends SIGCHLD to the supervisor. The SIGCHLD handler calls waitpid(-1, WNOHANG) in a loop to reap all exited children without blocking, preventing zombie processes. The handler also updates the container's state in the metadata list to exited, stopped, or killed depending on how the process terminated.
SIGINT and SIGTERM to the supervisor set a should_stop flag that causes the event loop to exit, triggering an orderly shutdown sequence.
4.3 IPC, Threads, and Synchronization
The project uses two IPC mechanisms:
Logging pipe: Each container's stdout and stderr are redirected into a pipe(). A dedicated producer thread reads from the pipe and inserts chunks into the bounded buffer. A consumer (logger) thread removes chunks and writes them to per-container log files. The bounded buffer is protected by a pthread_mutex_t with two pthread_cond_t variables (not_empty and not_full).
Without synchronization, producers and consumers could read/write the head and tail pointers simultaneously, causing lost data, duplicate reads, or buffer corruption. The mutex ensures only one thread modifies the buffer at a time. Condition variables allow threads to sleep efficiently instead of busy-waiting.
Control socket: The CLI client communicates with the supervisor via a UNIX domain socket at /tmp/mini_runtime.sock. This is a separate IPC mechanism from the logging pipe as required. The socket handles structured control_request_t and control_response_t messages.
Container metadata (the linked list of container_record_t) is protected by a separate pthread_mutex_t (metadata_lock) because it is accessed from both the event loop thread and the SIGCHLD signal handler.
4.4 Memory Management and Enforcement
RSS (Resident Set Size) measures the amount of physical RAM currently mapped and used by a process. It does not measure virtual memory, shared libraries counted multiple times, or memory that has been swapped out. RSS is a practical measure of a process's real memory pressure on the system.
Soft and hard limits represent different enforcement policies. A soft limit is a warning threshold — when RSS exceeds it, a warning is logged but the process continues running. This allows the operator to be notified of high memory usage without disrupting the workload. A hard limit is a termination threshold — when RSS exceeds it, the process is killed immediately with SIGKILL.
Enforcement belongs in kernel space rather than user space because a user-space monitor can be fooled, delayed, or killed by the very process it is monitoring. The kernel has reliable, privileged access to process memory maps via get_mm_rss() and can deliver signals atomically via send_sig(). A user-space monitor also introduces polling latency that could allow a process to greatly exceed its limit before being killed.
4.5 Scheduling Behavior
Linux uses the Completely Fair Scheduler (CFS), which assigns CPU time proportional to each process's weight. The weight is derived from the nice value — lower nice values result in higher weights and more CPU time.
In our experiment, two cpu_hog containers were started with nice values of -5 and -10. Both competed for CPU on a single-core VM. The nice -10 container received slightly more CPU time (93.3%) compared to the nice -5 container (91.1%), consistent with CFS weight calculations. The difference is modest because both processes are CPU-bound and the system has low background load, so the scheduler distributes time nearly equally with a small bias toward the higher-priority process.
In a CPU-bound vs I/O-bound scenario, the CPU-bound process would consume near 100% CPU while the I/O-bound process spends most of its time in the TASK_INTERRUPTIBLE sleep state waiting for I/O completion, voluntarily yielding the CPU. CFS gives the I/O-bound process a scheduling boost when it wakes up to improve responsiveness, but its overall CPU share remains low due to infrequent CPU use.

5. Design Decisions and Tradeoffs
Namespace Isolation
Choice: CLONE_NEWPID | CLONE_NEWUTS | CLONE_NEWNS with chroot().
Tradeoff: Network namespace (CLONE_NEWNET) was not included, so containers share the host network stack.
Justification: The project scope focuses on process and filesystem isolation. Adding network namespaces would require additional setup (virtual ethernet pairs, bridge configuration) beyond the project requirements.
Supervisor Architecture
Choice: Single-threaded event loop accepting one connection at a time on a UNIX domain socket.
Tradeoff: CLI commands are serialized — a slow command blocks the next one.
Justification: Simplicity and correctness over concurrency. For a project runtime with a small number of containers, serialized CLI handling is safe and easy to reason about. A multi-threaded event loop would require careful locking of the supervisor context.
IPC and Logging
Choice: pipe() for log capture, UNIX domain socket for CLI control, bounded buffer with mutex and condition variables.
Tradeoff: Opening and closing the log file on every buffer pop adds I/O overhead.
Justification: Per-pop file opens keep the implementation simple and avoid holding open file descriptors indefinitely. The performance cost is acceptable for a logging system that is not on the critical path.
Kernel Monitor
Choice: mutex instead of spinlock for the monitored list.
Tradeoff: Mutexes can sleep, which is not allowed in hard interrupt context.
Justification: The timer callback runs in softirq context where sleeping is allowed with care, and del_timer_sync() is called before taking the lock in module exit, making the mutex safe. A spinlock would have been necessary if the lock were taken in a hardware interrupt handler.
Scheduling Experiments
Choice: nice values via setpriority() (called through nice()) rather than CPU affinity or cgroups.
Tradeoff: Nice values affect scheduling weight but do not hard-limit CPU usage the way cgroups can.
Justification: Nice values are the simplest and most portable way to influence CFS scheduling priority, directly demonstrating the relationship between nice values and CPU share without additional infrastructure.
6. Scheduler Experiment Results
Experiment 1 — Two CPU-bound containers with different priorities
Both containers ran /cpu_hog simultaneously on the same VM. Observed CPU usage after 10 seconds:
ContainerNice ValueObserved CPU%high-591.1%low-1093.3%
Analysis: The container with nice -10 received slightly more CPU time than nice -5, consistent with Linux CFS weight-based scheduling. The difference is small because the VM has low background load and a single CPU, so both CPU-bound processes get nearly equal time. On a more loaded system the difference would be more pronounced.
Experiment 2 — CPU-bound vs I/O-bound (observed behavior)
One container ran /cpu_hog (pure CPU computation) and one ran /io_pulse (write + fsync loop with sleep intervals).
ContainerWorkloadExpected CPU%Behaviorcpuworkcpu_hog~99%Continuously burns CPU, never sleepsioworkio_pulse~5-10%Spends most time in I/O wait, sleeps between iterations
Analysis: The CPU-bound process stays in the TASK_RUNNING state continuously and consumes its full time slice every scheduling period. The I/O-bound process frequently enters TASK_INTERRUPTIBLE while waiting for fsync() to complete, voluntarily giving up the CPU. CFS rewards the I/O-bound process with a scheduling boost on wakeup to improve responsiveness, but its total CPU share remains low due to its sleep-heavy nature. This demonstrates CFS's dual goals of fairness (equal weight processes get equal CPU) and responsiveness (I/O-bound processes wake up quickly).
