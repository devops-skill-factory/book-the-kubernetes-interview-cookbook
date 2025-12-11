# Control Groups

[**Linux control groups (cgroups)**](https://man7.org/linux/man-pages/man7/cgroups.7.html) are a Linux kernel feature that limits, accounts for, and isolates the usage of system resources (CPU, memory, disk I/O, network, etc.) for a collection of processes. 

## **Key Points**

* **Function:** cgroups allow for the hierarchical organization of processes into groups, enabling resource management and control.
* **Purpose:** The primary goal is to **prevent resource contention** and ensure fair resource distribution, particularly crucial in containerization technologies like Docker and Kubernetes.
* **Structure:** They operate using a virtual file system (typically mounted at `/sys/fs/cgroup`), with each controlled resource (like memory or CPU) managed by a specific **subsystem** (or *controller*).
    * **Subsystems/Controllers:** Examples include `cpu`, `memory`, `blkio` (block I/O), and [`net_cls`](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/6/html/resource_management_guide/sec-net_cls) (network packet tagging).
* **Hierarchy:** Processes are assigned to nodes within a tree-like hierarchy, and child nodes (cgroups) can further subdivide the resource limits inherited from their parent cgroups.

This feature is fundamental to system stability and multitenancy environments by enforcing resource boundaries.

## Cgroup Versions

The Linux kernel has two main control group (cgroup) versions: **cgroup v1** (the original implementation) and **cgroup v2** (the modern, unified standard). The transition to v2 was driven by limitations and inconsistencies in v1.

### cgroup v1: The Original (Legacy)

[Cgroup v1](https://docs.kernel.org/admin-guide/cgroup-v1/index.html) introduced the basic concept of resource control but suffered from a complex and inconsistent design.

* **Multiple Hierarchies:** This is the defining feature of v1. Each resource controller (subsystem) like `cpu`, `memory`, or `blkio` could be mounted as its own separate hierarchy.
    * *Result:* A single process could belong to a different cgroup path for CPU limits than it did for memory limits. This made management and resource allocation tracing extremely difficult. 
* **Non-Unified Semantics:** The rules for distributing resources were inconsistent across different controllers, making it hard to predict overall system behavior.
* **Thread Granularity:** Cgroup v1 allowed threads within the same process to be moved into different cgroups, which caused issues for resources shared by the entire process (like memory address space).

### cgroup v2: The Modern (Unified) Standard

[Cgroup v2](https://docs.kernel.org/admin-guide/cgroup-v2.html), officially introduced in Linux kernel 4.5, is a complete redesign that fixes the core architectural issues of v1. It is the **default** on most modern Linux distributions (e.g., Ubuntu, Fedora, RHEL 8+).

| Feature | cgroup v1 (Legacy) | cgroup v2 (Unified) |
| :--- | :--- | :--- |
| **Hierarchy** | **Multiple, separate trees** (one for each mounted controller). | **Single, unified tree** for all controllers. |
| **Resource Rules** | Inconsistent logic across different controllers. | **Unified and standardized** resource distribution rules. |
| **Process/Thread** | Threads of a single process could belong to different cgroups. | **All threads of a process must be in the same cgroup.** |
| **Delegation** | Complicated and unsafe for unprivileged users. | **Safe and simpler** delegation of sub-trees to unprivileged users (crucial for container runtimes). |
| **Interface File** | Uses `tasks` and `cgroup.procs`. | Primarily uses **`cgroup.procs`** and **`cgroup.controllers`**. |
| **Mount Point** | Multiple mount points (e.g., `/sys/fs/cgroup/cpu`, `/sys/fs/cgroup/memory`). | Single mount point, typically at **`/sys/fs/cgroup`**. |

### Key Improvements of v2

1.  **Unified Hierarchy:** All controllers are attached to a single hierarchy. This ensures that a process is always in a single, well-defined control group, simplifying resource accountability and management.
2.  **Pressure Stall Information (PSI):** A new feature in v2 that provides detailed metrics on how much time tasks are waiting for contended resources (CPU, memory, or I/O). This is invaluable for troubleshooting performance bottlenecks.
3.  **Subtree Control:** Resource management is controlled by the parent cgroup. If a parent delegates CPU time to its children, the parent writes the list of active controllers (`+cpu`) into its `cgroup.subtree_control` file.

### Coexistence and Adoption

While v2 is the intended replacement, most systems run in a **Hybrid Mode** where v1 controllers that haven't been fully migrated to v2 (like the legacy `devices` controller) can coexist alongside the primary v2 hierarchy. However, a single controller can only be active in *one* hierarchy (either v1 or v2).

Modern containerization platforms like Kubernetes and Docker are actively adopting or already require cgroup v2 for better isolation and resource quality-of-service (QoS).

## Practice

1. The most reliable, low-level method to see all subsystems supported by the kernel, regardless of which cgroup version is mounted or what tools are installed, is to read the `/proc/cgroups` file:
```bash
cat /proc/cgroups 
```
Example output:
```bash
#subsys_name    hierarchy       num_cgroups     enabled
cpuset  0       275     1
cpu     0       275     1
cpuacct 0       275     1
blkio   0       275     1
memory  0       275     1
devices 0       275     1
freezer 0       275     1
net_cls 0       275     1
perf_event      0       275     1
net_prio        0       275     1
hugetlb 0       275     1
pids    0       275     1
rdma    0       275     1
misc    0       275     1
```
2. If system uses the unified **cgroups v2** hierarchy (common on modern Ubuntu), the active controllers (subsystems) are listed in the root directory of the cgroup filesystem:
```bash
cat /sys/fs/cgroup/cgroup.controllers
```

Output:
```bash
cpuset cpu io memory hugetlb pids rdma
```
3. If you have a PID and want to find which cgroup it belongs to, you can read a file in the `/proc` filesystem:
```bash
cat /proc/<PID>/cgroup
```

Output example:
```bash
0::/system.slice/ssh.service
```
The part after the final `::/` is the relative cgroup path.