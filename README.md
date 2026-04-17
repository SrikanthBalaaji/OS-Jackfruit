Multi-Container Runtime

1. Team Information
Name  SRN
Srikanth Balaaji	PES2UG24CS518
Sushanth Babu	PES2UG24CS543
2. Build, Load, and Run Instructions
Prerequisites

Use an Ubuntu 22.04 / 24.04 (or 25.10) VM with Secure Boot disabled. Install required packages:

sudo apt update
sudo apt install -y build-essential linux-headers-$(uname -r)
Clone and Build
git clone https://github.com/<SrikanthBalaaji>/OS-Jackfruit.git
cd OS-Jackfruit/boilerplate
make clean
make

This generates: engine, monitor.ko, cpu_hog, io_pulse, memory_hog.

Note: On kernel 6.17+, del_timer_sync is renamed. The Makefile already handles this using:

sed -i 's/del_timer_sync/timer_delete_sync/g' monitor.c
Prepare Root Filesystems
cd ~/OS_Jackfruit

# Download Alpine base rootfs (only once)
mkdir rootfs-base
wget https://dl-cdn.alpinelinux.org/alpine/v3.20/releases/x86_64/alpine-minirootfs-3.20.3-x86_64.tar.gz
tar -xzf alpine-minirootfs-3.20.3-x86_64.tar.gz -C rootfs-base

# Create container copies
cp -a rootfs-base rootfs-alpha
cp -a rootfs-base rootfs-beta

# Copy workloads into rootfs
cp boilerplate/cpu_hog rootfs-alpha/
cp boilerplate/cpu_hog rootfs-beta/
cp boilerplate/memory_hog rootfs-alpha/
cp boilerplate/io_pulse rootfs-beta/
Load the Kernel Module
cd boilerplate
sudo insmod monitor.ko
ls -l /dev/container_monitor   # confirm device creation
Start the Supervisor (Terminal 1 — keep running)
cd boilerplate
mkdir -p logs
sudo ./engine supervisor ../rootfs-base

Expected output:

[supervisor] Ready. rootfs=../rootfs-base control=/tmp/mini_runtime.sock
Launch Containers (Terminal 2)
cd boilerplate

# Start two containers
sudo ./engine start alpha ../rootfs-alpha /cpu_hog
sudo ./engine start beta ../rootfs-beta /cpu_hog

# List containers
sudo ./engine ps

# View logs
sudo ./engine logs alpha

# Stop containers
sudo ./engine stop alpha
sudo ./engine stop beta
Run Memory Limit Test
sudo ./engine start mem1 ../rootfs-alpha /memory_hog --soft-mib 30 --hard-mib 50
sleep 5
sudo dmesg | tail -10
sudo ./engine ps   # mem1 should show state: killed
Run Scheduling Experiment
cp -a rootfs-base rootfs-lo
cp -a rootfs-base rootfs-hi
cp boilerplate/cpu_hog rootfs-lo/
cp boilerplate/cpu_hog rootfs-hi/

cd boilerplate
sudo ./engine start lo ../rootfs-lo /cpu_hog --nice 0
sudo ./engine start hi ../rootfs-hi /cpu_hog --nice 15

# After completion:
sudo ./engine logs lo
sudo ./engine logs hi
Shutdown and Cleanup
# Stop containers
sudo ./engine stop alpha
sudo ./engine stop beta

# Stop supervisor (Ctrl+C)
# Unload module
sudo rmmod monitor
sudo dmesg | tail -5
3. Demo with Screenshots
Screenshot 1 — Multi-Container Supervision

Two containers (alpha pid=4475 and beta pid=4481) run simultaneously under one supervisor, each executing /cpu_hog within separate namespaces.

Screenshot 2 — Metadata Tracking

sudo ./engine ps displays both containers in running state along with host PIDs, soft limits (40 MiB), and hard limits (64 MiB) stored in supervisor metadata.

Screenshot 3 — Bounded-Buffer Logging

sudo ./engine logs alpha shows output from cpu_hog captured through the pipe → buffer → consumer thread → log file pipeline, stored in logs/alpha.log.

Screenshot 4 — CLI and IPC

sudo ./engine stop alpha sends a CMD_STOP request via the UNIX socket /tmp/mini_runtime.sock, and the supervisor replies with Stopped 'alpha'.

Screenshot 5 — Soft-Limit Warning

dmesg shows a SOFT LIMIT warning when container mem1 exceeds its configured memory threshold.

Screenshot 6 — Hard-Limit Enforcement

The kernel module triggers a HARD LIMIT event and kills mem1 using SIGKILL. engine ps confirms its state as killed.

Screenshot 7 — Scheduling Experiment

Container lo (nice=0) completed significantly more work than hi (nice=15), proving priority-based CPU allocation under CFS.

Screenshot 8 — Clean Teardown

No zombie processes remain, supervisor exits cleanly, and module unload confirms all memory structures are freed.

4. Engineering Analysis
4.1 Isolation Mechanisms

The runtime isolates processes using Linux namespaces along with chroot.

CLONE_NEWPID creates a separate PID space where the container process appears as PID 1. CLONE_NEWUTS provides a separate hostname, while CLONE_NEWNS isolates mount points.

After cloning, chroot(rootfs) confines the filesystem, and /proc is mounted inside the container to enable process utilities.

However, the kernel, hardware, and network stack remain shared across containers.

4.2 Supervisor and Process Lifecycle

The supervisor ensures proper lifecycle management. It keeps pipes open, prevents zombie processes using waitpid, and maintains container metadata.

When a container exits, SIGCHLD is handled, and the state is updated accordingly (exited, stopped, or killed).

4.3 IPC, Threads, and Synchronization

Two communication paths are used:

Logging (pipes): container output is passed through threads into a bounded buffer and written to logs.
Control (UNIX socket): CLI communicates with the supervisor using request-response messages.

Synchronization is handled using mutexes and condition variables to avoid race conditions and ensure safe access to shared structures.

4.4 Memory Management and Enforcement

RSS represents actual physical memory usage.

Soft limits trigger warnings, while hard limits enforce termination via SIGKILL. Kernel-space enforcement ensures accuracy and avoids race conditions present in user-space monitoring.

4.5 Scheduling Behavior

Linux CFS distributes CPU based on weights derived from nice values. Lower nice values receive more CPU time.

Experimental results confirm proportional scheduling while maintaining fairness and avoiding starvation.

5. Design Decisions and Tradeoffs
Namespace Isolation

Using PID, UTS, and mount namespaces simplifies implementation. Network isolation was excluded to reduce complexity.

Supervisor Architecture

A single-threaded event loop simplifies control handling and avoids concurrency issues.

IPC and Logging

UNIX sockets enable bidirectional communication, while pipes efficiently handle logging.

Kernel Monitor

A mutex is used instead of a spinlock due to operations that may sleep.

Scheduling Experiments

Using accumulator values provides a direct measure of CPU time without external tools.

6. Scheduler Experiment Results
Experiment: CPU-Bound Containers
Container	Nice	Final Accumulator	CPU Share
lo	0	13,809,123,758,218,673,290	~65%
hi	15	7,372,049,706,560,665,58	~35%

Result: lo performed about 1.87× more work than hi.

Analysis

CFS schedules tasks based on vruntime, ensuring proportional fairness.

Although theory predicts a larger difference, practical factors like multi-core execution and scheduler fairness reduce the gap.

This confirms CFS provides balanced CPU distribution without starving lower-priority processes.

If you want, I can next convert this into a proper DOC file (formatted report with headings, spacing, and fonts) ready to submit.

Give me a proper copy paste text which i can paste onto the readme

Here is a clean, properly formatted copy-paste version of your README (no extra formatting issues, no code block IDs, ready for GitHub):

Multi-Container Runtime
1. Team Information
Name	SRN
Srikanth Balaaji	PES2UG24CS518
Sushanth Babu	PES2UG24CS543
2. Build, Load, and Run Instructions
Prerequisites

Use an Ubuntu 22.04 / 24.04 (or 25.10) VM with Secure Boot disabled. Install the required dependencies:

sudo apt update
sudo apt install -y build-essential linux-headers-$(uname -r)
Clone and Build
git clone https://github.com/<your-username>/OS-Jackfruit.git
cd OS-Jackfruit/boilerplate
make clean
make

This will generate the following files: engine, monitor.ko, cpu_hog, io_pulse, and memory_hog.

Note: For kernel version 6.17 and above, del_timer_sync is renamed. The Makefile already handles this using:

sed -i 's/del_timer_sync/timer_delete_sync/g' monitor.c
Prepare Root Filesystems
cd ~/OS_Jackfruit

# Download Alpine base rootfs (only once)
mkdir rootfs-base
wget https://dl-cdn.alpinelinux.org/alpine/v3.20/releases/x86_64/alpine-minirootfs-3.20.3-x86_64.tar.gz
tar -xzf alpine-minirootfs-3.20.3-x86_64.tar.gz -C rootfs-base

# Create container copies
cp -a rootfs-base rootfs-alpha
cp -a rootfs-base rootfs-beta

# Copy workload binaries
cp boilerplate/cpu_hog rootfs-alpha/
cp boilerplate/cpu_hog rootfs-beta/
cp boilerplate/memory_hog rootfs-alpha/
cp boilerplate/io_pulse rootfs-beta/
Load the Kernel Module
cd boilerplate
sudo insmod monitor.ko
ls -l /dev/container_monitor   # verify device creation
Start the Supervisor (Terminal 1 — keep running)
cd boilerplate
mkdir -p logs
sudo ./engine supervisor ../rootfs-base

Expected output:

[supervisor] Ready. rootfs=../rootfs-base control=/tmp/mini_runtime.sock
Launch Containers (Terminal 2)
cd boilerplate

# Start containers
sudo ./engine start alpha ../rootfs-alpha /cpu_hog
sudo ./engine start beta ../rootfs-beta /cpu_hog

# List running containers
sudo ./engine ps

# View logs
sudo ./engine logs alpha

# Stop containers
sudo ./engine stop alpha
sudo ./engine stop beta
Run Memory Limit Test
sudo ./engine start mem1 ../rootfs-alpha /memory_hog --soft-mib 30 --hard-mib 50
sleep 5
sudo dmesg | tail -10
sudo ./engine ps

mem1 should appear with state killed.

Run Scheduling Experiment
cp -a rootfs-base rootfs-lo
cp -a rootfs-base rootfs-hi
cp boilerplate/cpu_hog rootfs-lo/
cp boilerplate/cpu_hog rootfs-hi/

cd boilerplate
sudo ./engine start lo ../rootfs-lo /cpu_hog --nice 0
sudo ./engine start hi ../rootfs-hi /cpu_hog --nice 15

# After completion
sudo ./engine logs lo
sudo ./engine logs hi
Shutdown and Cleanup
sudo ./engine stop alpha
sudo ./engine stop beta

# Stop supervisor using Ctrl+C
sudo rmmod monitor
sudo dmesg | tail -5
3. Demo with Screenshots
Screenshot 1 — Multi-Container Supervision

Two containers (alpha and beta) run simultaneously under one supervisor, each executing /cpu_hog in separate namespaces.

Screenshot 2 — Metadata Tracking

The engine ps command shows both containers in a running state along with host PIDs and memory limits.

Screenshot 3 — Bounded-Buffer Logging

Logs from cpu_hog are captured and written through a pipeline into log files such as logs/alpha.log.

Screenshot 4 — CLI and IPC

The stop command sends a request through the UNIX socket /tmp/mini_runtime.sock, and the supervisor updates the container state.

Screenshot 5 — Soft-Limit Warning

A warning is generated when a container exceeds its configured soft memory limit.

Screenshot 6 — Hard-Limit Enforcement

The kernel module enforces the hard limit by sending SIGKILL, and the container state becomes killed.

Screenshot 7 — Scheduling Experiment

The container with lower nice value performs more work, demonstrating CPU priority handling.

Screenshot 8 — Clean Teardown

No zombie processes remain, the supervisor shuts down cleanly, and the kernel module unloads successfully.

4. Engineering Analysis
4.1 Isolation Mechanisms

The runtime uses Linux namespaces and chroot for isolation.

CLONE_NEWPID provides a separate process ID space, CLONE_NEWUTS isolates hostname, and CLONE_NEWNS isolates mount points.
After cloning, chroot confines the filesystem, and /proc is mounted for process visibility.

The kernel, hardware, and network stack remain shared.

4.2 Supervisor and Process Lifecycle

The supervisor maintains container metadata, keeps pipes open, and prevents zombie processes using waitpid.

On container exit, SIGCHLD is handled and the state is updated accordingly.

4.3 IPC, Threads, and Synchronization

Two communication paths are used:

Logging: Pipes transfer container output into a bounded buffer and then into log files.
Control: A UNIX domain socket enables communication between CLI and supervisor.

Mutexes and condition variables ensure safe synchronization.

4.4 Memory Management and Enforcement

RSS represents actual physical memory usage.

Soft limits generate warnings, while hard limits terminate the process using SIGKILL.
Kernel-level enforcement ensures accurate and reliable monitoring.

4.5 Scheduling Behavior

Linux CFS allocates CPU time based on nice values.

Lower nice values receive more CPU time while ensuring fairness across processes.

5. Design Decisions and Tradeoffs
Namespace Isolation

PID, UTS, and mount namespaces are used. Network isolation is excluded to reduce complexity.

Supervisor Architecture

A single-process supervisor simplifies control handling and avoids race conditions.

IPC and Logging

UNIX sockets allow bidirectional communication, while pipes efficiently handle logging.

Kernel Monitor

A mutex is used instead of a spinlock due to operations that may block.

Scheduling Experiments

Accumulator values are used as a direct measure of CPU usage.

6. Scheduler Experiment Results
Experiment: CPU-Bound Containers
Container	Nice	Final Accumulator	CPU Share
lo	0	13,809,123,758,218,673,290	~65%
hi	15	7,372,049,706,560,665,58	~35%

Result: The lo container completed approximately 1.87× more work than hi.

Analysis

The CFS scheduler distributes CPU time based on task weights derived from nice values.

Although theoretical ratios are higher, practical factors like multi-core execution and fairness policies reduce the difference.

This confirms that CFS provides proportional CPU allocation while preventing starvation