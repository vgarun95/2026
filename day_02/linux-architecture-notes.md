The embedded Mermaid diagrams will render natively as interactive visual diagrams on GitHub.Markdown# Day 02: Linux Architecture, Processes, and systemd

## 1. High-Level Linux Architecture

Linux operates on a strict division between application access and hardware control. This division guarantees stability and security—if a user application crashes, it cannot bring down the operating system kernel.

```mermaid
flowchart TD
    subgraph UserSpace ["User Space (Restricted Access)"]
        Apps["Applications (NGINX, Docker, Python)"]
        Shell["Shell (Bash, Zsh)"]
        Systemd["systemd (PID 1)"]
    end

    subgraph SystemCalls ["System Call Interface (syscalls)"]
        Syscall["open(), fork(), read(), write(), execve()"]
    end

    subgraph KernelSpace ["Kernel Space (Unrestricted Access)"]
        Kernel["Linux Kernel"]
        ProcessMgr["Process Scheduler"]
        MemMgr["Memory Management (RAM/VMM)"]
        VFS["Virtual File System (VFS)"]
        NetStack["Network Stack"]
        Drivers["Device Drivers"]
    end

    Hardware["Hardware (CPU, RAM, Disks, NIC)"]

    Apps --> Syscall
    Shell --> Syscall
    Systemd --> Syscall
    Syscall --> Kernel
    Kernel --> ProcessMgr
    Kernel --> MemMgr
    Kernel --> VFS
    Kernel --> NetStack
    Kernel --> Drivers
    Drivers --> Hardware
Core Components BreakdownKernel: The core of the OS. It manages hardware resources (CPU, RAM, Disks, Network interfaces) and provides abstraction layers so software doesn't need to interact directly with hardware.User Space: The memory area where all user applications, background services (daemons), and shells execute. User processes must request Kernel intervention via System Calls (syscalls) to perform privileged actions (like reading a file or opening a network port).Init System (systemd): The very first process initialized by the kernel during boot (PID 1). It acts as the parent of all other processes in user space.2. Process Lifecycle & ManagementA process is simply an executing instance of a program loaded into memory.How Processes Are Created: fork() and exec()Linux creates new processes using a two-step mechanism:fork(): A parent process creates an exact duplicate copy of itself (the child process), inheriting environment variables, file descriptors, and memory state.execve(): The child process replaces its memory space with a new executable binary (e.g., launching NGINX).Code snippetstateDiagram-v2
    [*] --> Created: Parent calls fork()
    Created --> Running: Executed via execve()
    
    state Running {
        [*] --> ExecutingOnCPU
        ExecutingOnCPU --> ReadyToRun: Preempted by Scheduler
        ReadyToRun --> ExecutingOnCPU: Scheduled on CPU
    }

    Running --> InterruptibleSleep: Waiting for Event/IO (State: S)
    InterruptibleSleep --> Running: Event Received / Signal Sent

    Running --> UninterruptibleSleep: Waiting for Hardware/Disk IO (State: D)
    UninterruptibleSleep --> Running: Hardware Operation Completes

    Running --> Stopped: Received SIGSTOP / SIGTSTP (State: T)
    Stopped --> Running: Received SIGCONT

    Running --> Zombie: Process Finishes execution (State: Z)
    Zombie --> [*]: Parent calls wait() & reads exit code
The 5 Essential Process StatesState CodeState NameDescriptionDevOps Context / ActionRRunning / RunnableProcess is either currently executing on the CPU or sitting in the run queue ready to execute.Normal operation under load.SInterruptible SleepProcess is waiting for an event or resource (e.g., waiting for user input or network data). Can be woken up by signals.Default state for idle daemons (e.g., NGINX waiting for HTTP traffic).DUninterruptible SleepProcess is waiting on synchronous hardware IO (usually disk or network storage). Signals like kill -9 will be ignored until IO completes.High numbers indicate severe disk bottlenecks, slow NFS mounts, or failing hardware.TStopped / TracedExecution suspended manually via signal (e.g., Ctrl+Z or SIGSTOP) or controlled by a debugger.Process paused; resume using fg command or sending SIGCONT.ZZombie / DefunctProcess has terminated, but its parent process has not yet read its exit code using the wait() system call. It consumes no memory or CPU, only a PID table entry.High number of zombies indicates buggy application code or poorly designed process managers.3. systemd: The Init System (PID 1)systemd is the standard init system and service manager for modern Linux distributions (Ubuntu, RHEL, Debian, CentOS).Why systemd Matters for DevOpsProcess ID 1 (PID 1): It is the initial process spawned by the kernel. If PID 1 dies, the kernel panics and the system crashes.Parallel Booting: Boots services simultaneously rather than sequentially, radically improving system startup times.Automatic Service Recovery: Configured to automatically restart failed containers or daemons when they crash.Dependency Management: Guarantees that networking, mounts, and databases start before applications depending on them are initialized.Centralized Logging (journalctl): Collects binary logs across all managed services and the kernel in a single indexed format.4. Top 5 Daily Essential Linux Commands for DevOps1. ps (Process Status)Purpose: Takes a static snapshot of current running processes.Common Syntax & Usage:Bash# Display all running processes with full details (user, PID, CPU%, MEM%, start time, command)
ps aux

# View processes as a hierarchical tree (see parent-child relationships)
ps -ef --forest
Real-World DevOps Scenario: Find the PID of a frozen application process or check if a background script is actively running.2. top / htop (Real-Time Resource Monitor)Purpose: Provides a dynamic, interactive real-time view of system resource consumption (CPU, Memory, Load Average, Process lists).Common Syntax & Usage:Bash# Launch default dynamic viewer (Press 'q' to quit, 'P' to sort by CPU, 'M' to sort by Memory)
top

# Launch enhanced colorized interactive viewer (requires installation: apt install htop)
htop
Real-World DevOps Scenario: Diagnose CPU spikes, identify memory leaks on web servers, and track resource hogging.3. systemctl (systemd Controller)Purpose: Inspect and control the state of the systemd system and service manager.Common Syntax & Usage:Bash# Check status of a service (active, running, failed, logs)
systemctl status nginx

# Start, stop, or restart a service
sudo systemctl restart docker

# Enable a service to auto-start on boot
sudo systemctl enable daemon-name
Real-World DevOps Scenario: Verifying if application services (e.g., Docker, NGINX, Kubelet) are active and configuring them to recover on reboot.4. journalctl (Systemd Log Viewer)Purpose: Query and view logs generated by systemd services, kernel events, and system units.Common Syntax & Usage:Bash# View live real-time tail of logs for a specific service
journalctl -u nginx -f

# Show logs from current boot cycle only, filtered by priority (errors only)
journalctl -b -p err

# View logs within a specific timeframe
journalctl -u docker --since "1 hour ago"
Real-World DevOps Scenario: Root-cause analysis during production downtime when an application fails to start or crashes silently.5. kill / pkill (Signal Transmission)Purpose: Send termination or custom operational signals to processes by PID or process name.Common Syntax & Usage:Bash# Gracefully terminate a process (SIGTERM - Signal 15: allows cleanup)
kill 1234

# Forcefully kill an unresponsive process (SIGKILL - Signal 9: immediate kernel termination)
kill -9 1234

# Kill processes matching a specific name pattern
pkill -f python3
Real-World DevOps Scenario: Safely shutting down processes during deployments, or forcefully killing stuck processes blocking system ports.5. Quick Incident Troubleshooting PlaybookWhen an application crashes or server performance degrades:Code snippetflowchart LR
    A[Service Down / Slow] --> B[Check Status: systemctl status service_name]
    B --> C{Is service active?}
    C -- No --> D[Inspect Logs: journalctl -u service_name -e]
    C -- Yes --> E[Check Resources: top / htop]
    E --> F[Check Processes: ps aux | grep service_name]
    F --> G[Terminate stuck workers: kill -15 PID]
Check Service Health: Run systemctl status <service_name> to verify operational state.Inspect Failure Logs: Run journalctl -u <service_name> -n 50 --no-pager to pinpoint crash stack traces.Analyze Resource Constraints: Check top or htop for high load averages or out-of-memory (OOM) conditions.Identify & Clear Stale Processes: Use ps aux | grep <service> to find orphan tasks and clear them via kill -15 <PID>
