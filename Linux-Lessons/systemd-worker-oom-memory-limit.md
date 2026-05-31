# Troubleshooting a systemd Worker Killed by a Memory Limit

## Challenge Goal

Fix `worker.service`, a background batch-processing worker that kept disappearing shortly after it picked up its first batch of work.

The service appeared to start normally, printed heartbeat logs, began processing a batch, and then died. `systemd` restarted it, but every restart repeated the same pattern, so the worker never made progress beyond the first batch.

Root cause:

```text
worker.service had MemoryMax=200M, and the batch handler hit that cgroup memory limit.
```

Fix:

```ini
[Service]
MemoryMax=350M
```

## Main Idea

`systemd` is the service manager used by many Linux distributions. It is not the Linux kernel itself. It is a userspace init and supervision system that starts early in boot, launches services, tracks their processes, captures logs through the journal, and applies resource controls through Linux kernel features such as cgroups.

`systemctl` is the command-line client used to talk to `systemd`. You use it to inspect, start, stop, restart, enable, disable, and configure services.

`journalctl` is the command-line client used to read logs from the `systemd` journal. Service output written to stdout or stderr is usually captured there, and kernel messages can also be viewed there with `journalctl -k`.

In this challenge, the worker did not fail because the shell script could not start. It failed because `systemd` launched it inside a dedicated cgroup with a hard memory ceiling. When work processing exceeded that ceiling, the kernel's OOM handling killed a process in the service cgroup, and `systemd` reported the unit failure as `oom-kill`.

## systemd, systemctl, and journalctl

### What is systemd?

`systemd` is a userspace service manager and init system. On many Linux machines it runs as PID 1, which means it is the first userspace process started by the kernel.

It is responsible for tasks like:

- Starting system services during boot.
- Keeping long-running daemons alive.
- Restarting failed services when configured to do so.
- Tracking service processes and child processes.
- Grouping services into cgroups.
- Applying service-level resource limits such as `MemoryMax`, `MemoryHigh`, and `CPUQuota`.
- Capturing service logs through the journal.
- Managing timers, sockets, mounts, targets, and other unit types.

### What is systemctl?

`systemctl` is the main CLI for controlling and querying `systemd`.

Common examples:

```bash
systemctl status worker.service
sudo systemctl restart worker.service
sudo systemctl enable --now nginx.service
systemctl cat worker.service
systemctl show worker.service -p MemoryMax -p MainPID
sudo systemctl edit worker.service
```

High-level mental model:

```text
systemctl  --->  asks systemd to do something  --->  systemd manages the service
```

### What is journalctl?

`journalctl` reads logs from the `systemd` journal.

Common examples:

```bash
journalctl -u worker.service -n 100 --no-pager
journalctl -u worker.service -f
journalctl -k --since "15 minutes ago" --no-pager
```

Useful flags:

- `-u worker.service`: show logs for one unit.
- `-f`: follow logs live.
- `-k`: show kernel logs.
- `--since "15 minutes ago"`: limit the time range.
- `--no-pager`: print directly instead of opening a pager.

### Are these part of the Linux kernel?

No. The Linux kernel provides primitives such as processes, signals, cgroups, namespaces, filesystems, and sockets.

`systemd`, `systemctl`, and `journalctl` are userspace programs maintained as part of the open source `systemd` project. They use kernel features, but they are not themselves the kernel.

The split looks like this:

```text
Linux kernel
  - processes
  - signals
  - cgroups
  - memory accounting
  - OOM killing
  - namespaces

systemd userspace project
  - systemd service manager
  - systemctl CLI
  - journalctl CLI
  - journald logging service
  - unit files and drop-ins
```

## systemd Service vs Running a Script Manually

Running a script manually is simple:

```bash
/usr/local/bin/worker
```

or:

```bash
./start-worker.sh
```

That starts a process from the current shell. But the shell is not a full service supervisor.

If you run a service manually, you usually need to solve these questions yourself:

- What restarts it if it crashes?
- What starts it again after reboot?
- Where do logs go?
- Which user should it run as?
- What environment variables does it need?
- How do you prevent it from consuming too much CPU or memory?
- How do you reliably find its main PID later?
- How do you stop the whole process tree, not just the parent shell script?

A `systemd` service answers those through a unit file:

```ini
[Service]
ExecStart=/usr/local/bin/worker
Restart=on-failure
MemoryMax=350M
```

That means `systemd` starts the process, tracks it, captures logs, applies resource controls, and can restart it when it fails.

## systemd and Docker

Docker did not exactly replace `systemd`; it solves a different operational problem.

`systemd` manages services on a Linux host. Docker packages and runs applications in containers. A container often has its own isolated filesystem, process tree, network namespace, and dependency set.

### Where systemd fits well

Use `systemd` when you are managing host-level services, for example:

- `sshd`
- `nginx` installed directly on the VM
- `postgresql` installed from OS packages
- backup agents
- monitoring agents
- one-off internal daemons
- host maintenance jobs
- `docker.service` itself

In fact, on many Linux servers Docker is started by `systemd`:

```bash
systemctl status docker.service
```

So Docker often runs under `systemd`, rather than replacing it entirely.

### Where Docker fits well

Use Docker when you want application packaging and runtime isolation, for example:

- The app needs a specific dependency stack.
- You want the same image to run on a laptop, CI, staging, and production.
- You want to avoid installing app dependencies directly on the host.
- You want immutable deploy artifacts.
- You want to run multiple versions of an app on the same host.
- You are deploying through Kubernetes or another container orchestrator.

### Why many Docker containers do not run systemd inside

Many containers, especially Alpine-based images, run a single application process directly:

```dockerfile
CMD ["/usr/local/bin/worker"]
```

That is common because the container runtime already provides some supervision and isolation from the outside:

- Docker starts and stops the container.
- Docker captures stdout and stderr logs.
- Docker can apply memory and CPU limits.
- Docker can restart containers using restart policies.
- Kubernetes can restart failed containers and reschedule pods.

So putting a full init system like `systemd` inside every application container is often unnecessary. Containers are usually designed around one main process, while a VM or bare-metal host usually needs a full service manager for many services.

### Simple rule of thumb

Use `systemd` for host service management.

Use Docker for packaging and isolating applications.

Use both when appropriate: `systemd` can manage Docker itself, while Docker manages application containers.

## Commands Run

### 1. Check Service Status

```bash
systemctl status worker.service
```

Why:

This shows whether the unit is active, failed, restarting, or recently killed. It also shows the main PID, recent logs, memory usage, cgroup path, and sometimes the failure result.

In this challenge, the service restarted repeatedly after processing began.

### 2. Follow Worker Logs

```bash
sudo journalctl -u worker.service -f
```

Why:

This follows only `worker.service` logs live. It showed the worker starting normally, printing heartbeat messages, picking up a batch, and then being terminated.

Observed logs:

```text
May 31 16:29:29 ubuntu-01 worker[3786]: [worker] Heartbeat - idle, waiting for work
May 31 16:29:29 ubuntu-01 worker[3786]: [worker] Picking up a batch for processing...
May 31 16:29:29 ubuntu-01 systemd[1]: worker.service: The kernel OOM killer killed some processes in this unit.
May 31 16:29:29 ubuntu-01 worker[3786]: /usr/local/bin/worker: line 21:  3839 Killed                     timeout --foreground 5s handler 300 2>&1 > /dev/null
May 31 16:29:29 ubuntu-01 worker[3786]: [worker] Received termination signal; exiting...
May 31 16:29:29 ubuntu-01 systemd[1]: worker.service: Failed with result 'oom-kill'.
May 31 16:29:29 ubuntu-01 systemd[1]: worker.service: Consumed 242ms CPU time over 10.604s wall clock time, 200M memory peak.
May 31 16:29:35 ubuntu-01 systemd[1]: worker.service: Scheduled restart job, restart counter is at 35.
May 31 16:29:35 ubuntu-01 systemd[1]: Started worker.service - Background batch processing worker.
```

Important clues:

```text
The kernel OOM killer killed some processes in this unit.
Failed with result 'oom-kill'.
200M memory peak.
```

### 3. Check Kernel Logs for OOM Events

```bash
sudo journalctl -k | grep -i "killed\|oom\|memory"
sudo journalctl -k | grep -i "worker.service"
```

Why:

When a process is killed due to memory pressure or a cgroup memory limit, the kernel records that event. `journalctl -k` reads kernel logs from the journal.

This is especially useful when an application appears to disappear without producing its own stack trace or error message. A `SIGKILL` cannot be caught by the application, so the useful evidence is often outside the app logs.

### 4. Inspect the Unit File

```bash
systemctl cat worker.service
```

Why:

This prints the full unit definition as seen by `systemd`, including the base unit file and any drop-in overrides.

The relevant configuration was:

```ini
MemoryAccounting=yes
MemoryMax=200M
```

`MemoryMax=200M` is a hard memory limit for the service cgroup. When processes in that service cgroup collectively exceed the limit, the kernel can kill processes in the cgroup.

### 5. Check Memory Limit Directly

```bash
systemctl cat worker.service | grep -i memory
```

Output:

```text
MemoryAccounting=yes
MemoryMax=200M
```

Why:

This quickly filtered the unit file to the memory-related directives.

For a fuller property view, this command is also useful:

```bash
systemctl show worker.service -p MemoryAccounting -p MemoryMax -p MemoryHigh -p MemoryCurrent -p MemoryPeak
```

## Root Cause

The worker was configured with:

```ini
MemoryMax=200M
```

When the batch handler started, memory usage reached the cgroup's 200 MB hard limit. The kernel OOM killer killed a process in `worker.service`, and `systemd` marked the unit as failed with:

```text
Failed with result 'oom-kill'.
```

Because the unit was configured to restart, `systemd` brought it back up. But every restart hit the same memory limit again, creating a loop:

```text
start -> heartbeat -> pick up first batch -> hit 200M -> OOM kill -> restart -> repeat
```

## Fix Applied

### 1. Create a systemd Drop-In Override

```bash
sudo systemctl edit worker.service
```

Add:

```ini
[Service]
MemoryMax=350M
```

Why:

`systemctl edit` creates a drop-in override under `/etc/systemd/system/worker.service.d/`. This is usually better than editing the original unit file directly because it keeps local changes separate from the packaged or generated unit.

A drop-in only needs to include the directives being changed. The rest of the unit remains inherited from the original service definition.

### 2. Reload systemd

```bash
sudo systemctl daemon-reload
```

Why:

After changing unit files or drop-ins, `systemd` needs to reload its configuration.

### 3. Restart the Worker

```bash
sudo systemctl restart worker.service
```

Why:

The new memory limit applies to newly started service processes. Restarting the unit starts the worker under the updated cgroup configuration.

### 4. Verify the New Limit

```bash
systemctl show worker.service | grep MemoryMax
```

Why:

This confirms that `systemd` now sees the updated effective memory limit.

A more targeted version is:

```bash
systemctl show worker.service -p MemoryMax
```

Expected result:

```text
MemoryMax=367001600
```

`350M` appears as bytes because `systemctl show` prints normalized property values. `367001600` bytes equals 350 MiB.

### 5. Verify the Service Stays Up

```bash
systemctl status worker.service
sudo journalctl -u worker.service -f
```

Why:

The service should keep running, and the logs should progress through multiple batches instead of dying at the first batch.

## Alternative Fixes

### Remove the Memory Limit Temporarily

For testing only:

```ini
[Service]
MemoryMax=infinity
```

This helps prove that the memory limit is the cause, but it removes a useful safety boundary.

### Set a Larger Bounded Limit

For a safer production-style fix:

```ini
[Service]
MemoryMax=512M
```

A bounded limit is usually better than no limit because it prevents one service from consuming all host memory.

### Tune the Worker Instead

Increasing the limit fixes the symptom if the original limit was too small, but the worker's memory profile still matters.

Other possible improvements:

- Process smaller batches.
- Stream work instead of loading everything into memory.
- Reduce concurrency.
- Fix memory leaks.
- Set `MemoryHigh` as a soft pressure threshold and `MemoryMax` as a hard ceiling.
- Monitor `MemoryPeak` and cgroup `memory.events` over time.

## Useful Commands

```bash
systemctl status worker.service
systemctl cat worker.service
systemctl show worker.service -p MemoryMax -p MemoryHigh -p MemoryCurrent -p MemoryPeak
sudo systemctl edit worker.service
sudo systemctl daemon-reload
sudo systemctl restart worker.service
journalctl -u worker.service -n 100 --no-pager
sudo journalctl -u worker.service -f
sudo journalctl -k | grep -i "killed\|oom\|memory"
systemctl show worker.service | grep MemoryMax
```

## Conclusion

The worker was not randomly disappearing. It was being killed by the kernel because `systemd` placed it in a cgroup with a hard memory limit of 200 MB.

The key evidence was in the journal:

```text
worker.service: The kernel OOM killer killed some processes in this unit.
worker.service: Failed with result 'oom-kill'.
worker.service: Consumed ... 200M memory peak.
```

The fix was to override the service memory limit with a systemd drop-in:

```ini
[Service]
MemoryMax=350M
```

After reloading `systemd` and restarting the service, the worker had enough memory to process batches continuously.
