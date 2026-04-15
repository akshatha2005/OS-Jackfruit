# Multi-Container Runtime

A lightweight Linux container runtime in C with a supervisor process and a kernel-space memory monitor.

## Team

| Name | SRN |
|------|-----|
| Akshatha P | PES2UG24CS048 |
| Aditi Agarwal | PES2UG24CS029 |

---

## Setup & Execution

### Prerequisites
- Ubuntu 22.04/24.04 (VM, Secure Boot OFF, no WSL)

### Install dependencies
```bash
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

Copy workloads:

cp cpu_hog memory_hog io_pulse rootfs/

Terminal 1

sudo ./engine supervisor ./rootfs

Terminal 2

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
Demo Highlights
Multi-container execution under one supervisor
ps shows container metadata
Logging via bounded buffer
CLI via UNIX socket (/tmp/mini_runtime.sock)
Soft limit → warning
Hard limit → SIGKILL
Clean shutdown (no zombies)
Key Concepts
Isolation
clone() with namespaces (PID, UTS, mount)
chroot() + /proc
Shared kernel
Supervisor
Tracks containers
Handles SIGCHLD
CLI via socket
IPC & Threads
Pipe → logging
UNIX socket → control
Mutex + condition variables
Memory Control
RSS-based tracking
Soft limit → warning
Hard limit → kill
Scheduling
Linux CFS
Lower nice ⇒ more CPU
CPU-bound > I/O-bound
Design Tradeoffs
No network namespace
Single-threaded supervisor
Simple logging design
Kernel-based enforcement
Nice values instead of cgroups
Experiments
CPU vs CPU
nice -10 > nice -5
CPU vs I/O
CPU-bound ≈ high CPU
I/O-bound ≈ low CPU
