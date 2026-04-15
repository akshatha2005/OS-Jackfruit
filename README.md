Multi-Container Runtime

A lightweight Linux container runtime in C with a supervisor process and a kernel-space memory monitor.

1. Team
Name	          SRN
Akshatha P	    PES2UG24CS048
Aditi Agarwal	  PES2UG24CS029
2. Setup & Execution
Prerequisites
Ubuntu 22.04/24.04 (VM, Secure Boot OFF, no WSL)
Install dependencies:
sudo apt update
sudo apt install -y build-essential linux-headers-$(uname -r)
Clone & Prepare
git clone https://github.com/akshatha2005/OS-Jackfruit.git
cd OS-Jackfruit/boilerplate

mkdir rootfs
wget https://dl-cdn.alpinelinux.org/alpine/v3.20/releases/x86_64/alpine-minirootfs-3.20.3-x86_64.tar.gz
tar -xzf alpine-minirootfs-3.20.3-x86_64.tar.gz -C rootfs
Build & Load
make
sudo insmod monitor.ko
ls -l /dev/container_monitor
sudo dmesg | tail -3
Run
cp cpu_hog memory_hog io_pulse rootfs/

# Terminal 1
sudo ./engine supervisor ./rootfs

# Terminal 2
sudo ./engine start alpha ./rootfs /cpu_hog
sudo ./engine start beta ./rootfs /memory_hog --soft-mib 10 --hard-mib 20

sudo ./engine ps
sudo ./engine logs alpha
sudo ./engine stop alpha
Cleanup
sudo ./engine stop alpha beta
sudo rmmod monitor
ps aux | grep defunct
make clean
3. Demo Highlights
Multi-container execution under one supervisor
ps shows container metadata (PID, state)
Logging via bounded buffer (pipe + threads)
CLI ↔ supervisor via UNIX socket (/tmp/mini_runtime.sock)
Kernel logs:
Soft limit → warning
Hard limit → SIGKILL
Scheduling: lower nice ⇒ more CPU
Clean shutdown → no zombie processes
4. Key Concepts
Isolation
clone() with PID, UTS, mount namespaces
chroot() + /proc mount
Shared host kernel (logical isolation)
Supervisor
Tracks containers & lifecycle
Handles SIGCHLD → reaps processes
Provides CLI control via socket
IPC & Threads
Pipe → logging (producer-conser buffer)
UNIX socket → CLI communication
Mutex + condition variables ensure safety
Memory Control
RSS = actual physical memory used
Soft limit → warning
Hard limit → process killed (kernel enforced)
Scheduling
Linux CFS uses nice values
Lower nice ⇒ higher CPU share
I/O-bound tasks yield CPU, CPU-bound dominate
5. Design Tradeoffs
No network namespace (simpler scope)
Single-threaded supervisor (simpler, serialized CLI)
Logging opens file per write (simpler, slight overhead)
Kernel enforcement (reliable vs user-space polling)
Nice values over cgroups (simpler demonstration)
6. Experiments
CPU vs CPU
nice -10 > nice -5 (slightly higher CPU share)
CPU vs I/O
CPU-bound ≈ full CPU
I/O-bound ≈ low CPU (waits on I/O)
