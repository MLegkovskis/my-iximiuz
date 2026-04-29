# Linux Cgroups Tutorial: Limiting Process Resources

## Main Idea

A normal Linux process can consume as much CPU or memory as the system allows. That is risky for noisy, buggy, or intentionally resource-hungry workloads because one process can affect everything else on the machine.

Cgroups are the Linux kernel mechanism for putting processes into resource-controlled groups. They are a foundation underneath tools like `systemd`, Docker, and Kubernetes.

## What is a Cgroup?

A **cgroup** (control group) is a Linux kernel feature that allows you to limit, account for, and isolate resource usage (CPU, memory, I/O, etc.) of groups of processes. It's like a container for managing how much system resources a process or group of processes can use, preventing them from hogging all available resources.

Cgroups are hierarchical and can be managed via the filesystem under `/sys/fs/cgroup` (cgroup v2) or through tools like `systemd`, `cgcreate`, or direct manipulation.

## What is OmniHog?

**OmniHog** is a simple Linux binary program designed to consume all available CPU and memory resources when run. It's used in this tutorial as an example process to demonstrate resource limiting, simulating a resource-intensive application that could otherwise overwhelm the system.

## Tutorial Overview

In this basic challenge, we start the OmniHog process while limiting its CPU usage to 10% and memory usage to 500 MB using cgroups.

### Key Principles

- **Resource Limiting**: Use cgroups to enforce hard limits on CPU and memory for processes.
- **Cgroup v2**: Modern Linux uses cgroup v2, where limits are set via files like `cpu.max` and `memory.max`.
- **Systemd Integration**: `systemd-run` can create transient services with resource limits, placing processes in specific slices (sub-cgroups).

### Commands Run

1. **Locate the Binary**:
   ```
   which omnihog
   ```
   Output: `/usr/local/bin/omnihog`

2. **Start the Process with Limits (Initial Attempt)**:
   ```
   systemd-run --user --property=CPUQuota=10% --property=MemoryLimit=500M /usr/local/bin/omnihog
   ```
   This starts OmniHog as a user service, but the limits may not be applied correctly in the slice.

3. **Check Current Limits**:
   ```
   cat /sys/fs/cgroup/user.slice/user-1001.slice/user@1001.service/app.slice/cpu.max
   ```
   Output: `max 100000` (unlimited)

4. **Set CPU Limit Manually**:
   ```
   echo "10000 100000" > /sys/fs/cgroup/user.slice/user-1001.slice/user@1001.service/app.slice/cpu.max
   ```
   - `10000 100000` means 10% CPU (10ms quota per 100ms period).

5. **Set Memory Limit Manually**:
   ```
   echo "500M" > /sys/fs/cgroup/user.slice/user-1001.slice/user@1001.service/app.slice/memory.max
   ```
   - Limits memory to 500 MB.

6. **Verify Limits**:
   ```
   cat /sys/fs/cgroup/user.slice/user-1001.slice/user@1001.service/app.slice/cpu.max
   cat /sys/fs/cgroup/user.slice/user-1001.slice/user@1001.service/app.slice/memory.max
   ```
   Outputs: `10000 100000` and `524288000` (500 MB in bytes).

## Conclusion

By using cgroups, we successfully constrained the OmniHog process to use no more than 10% CPU and 500 MB memory, preventing it from consuming unlimited resources. This demonstrates basic resource management in Linux.

For more advanced usage, explore creating custom cgroups, enabling controllers, or using tools like `cgexec` for process execution within cgroups.

## Next Steps

- Experiment with different limit values.
- Monitor resource usage with tools like `top`, `htop`, or `systemd-cgtop`.
- Learn about other cgroup controllers (I/O, network, etc.).
- Integrate with container runtimes like Docker for advanced isolation.
