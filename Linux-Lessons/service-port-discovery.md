# Linux Service Port Discovery

## Challenge Goal

Find which TCP port a Linux service named `app` is listening on.

Answer:

```text
26792
```

## Main Idea

A service is usually one or more Linux processes managed by something like `systemd`.

A listening port belongs to a process that has opened a network socket and is waiting for incoming connections. To identify the port, map the service to its process, then map that process to its listening socket.

## How Apps Commonly Run on Linux

Not every running app is managed the same way. `systemctl` works when the app is a `systemd` service, but `ps` works more generally because it lists processes directly.

Main families:

- **systemd services**: Long-running services managed by `systemd`, such as `nginx`, `ssh`, `postgres`, or this challenge's `app.service`. Use `systemctl status <service>` to see service state, logs, PID, and cgroup.
- **Manual shell processes**: Programs started directly from a terminal, such as `python app.py`, `npm start`, or `./server`. Use `ps`, `pgrep`, `jobs`, or your shell history to find them.
- **Containerized processes**: Apps started by Docker, containerd, or Kubernetes. Use `docker ps`, `docker inspect`, `kubectl get pods`, or `crictl`, then map back to host processes if needed.
- **Scheduled jobs**: Short-lived or periodic commands started by `cron`, `systemd timers`, or another scheduler. Use `crontab`, `/etc/cron*`, or `systemctl list-timers`.
- **Supervised app processes**: Apps run by a process manager such as `supervisord`, `pm2`, or a language/runtime-specific launcher. Use that tool first, then confirm with `ps`.

Useful rule:

- Use `systemctl` when you know or suspect the app is a service.
- Use `ps -ef`, `pgrep`, or `ss/lsof` when you only know there is a process somewhere.
- Use runtime-specific tools when the app is inside Docker, Kubernetes, Node `pm2`, etc.

In this challenge, `systemctl status app` worked because the app was registered as `app.service`.

## Commands Run

### 1. Check the Service

```bash
systemctl status app --no-pager --lines=5
```

Why:

This confirmed that `app.service` was running and showed its main process:

```text
● app.service - /usr/local/bin/app
     Loaded: loaded (/run/systemd/transient/app.service; transient)
  Transient: yes
     Active: active (running) since Wed 2026-04-29 17:07:12 UTC; 2min 37s ago
   Main PID: 1109 (app)
      Tasks: 7 (limit: 12019)
     Memory: 1.1M (peak: 1.5M)
        CPU: 4ms
     CGroup: /system.slice/app.service
             └─1109 /usr/local/bin/app
```

The key line is:

```text
Main PID: 1109 (app)
```

### 2. Get Just the Main PID

```bash
systemctl show -p MainPID --value app
```

Output:

```text
1109
```

Why:

This gives only the process ID, which is easier to reuse in other commands.

### 3. List Listening TCP Sockets

```bash
ss -ltnp
```

Why:

- `ss` lists sockets.
- `-l` shows listening sockets.
- `-t` shows TCP sockets.
- `-n` shows numeric ports instead of service names.
- `-p` shows the owning process where permissions allow.

Partial output:

```text
State  Recv-Q Send-Q Local Address:Port  Peer Address:PortProcess
LISTEN 0      4096   127.0.0.53%lo:53         0.0.0.0:*
LISTEN 0      4096      127.0.0.54:53         0.0.0.0:*
LISTEN 0      4096         0.0.0.0:50061      0.0.0.0:*
LISTEN 0      4096         0.0.0.0:22         0.0.0.0:*
LISTEN 0      4096               *:16279            *:*
LISTEN 0      4096               *:24525            *:*
LISTEN 0      4096               *:24391            *:*
LISTEN 0      4096               *:15732            *:*
LISTEN 0      4096               *:15561            *:*
LISTEN 0      4096               *:40059            *:*
LISTEN 0      4096               *:40060            *:*
LISTEN 0      4096               *:15479            *:*
LISTEN 0      4096               *:23334            *:*
LISTEN 0      4096               *:15219            *:*
LISTEN 0      4096               *:14991            *:*
LISTEN 0      4096               *:23221            *:*
LISTEN 0      4096               *:14873            *:*
LISTEN 0      4096               *:14865            *:*
LISTEN 0      4096               *:23158            *:*
```

Without elevated permissions, process names may be hidden. The machine also had many listening ports, so the raw list was too noisy to identify `app` confidently.

### 4. Re-run with Process Visibility and Filter by PID

```bash
sudo ss -ltnp | grep "pid=1109"
```

Output:

```text
LISTEN 0 4096 *:26792 *:* users:(("app",pid=1109,fd=3))
```

Why:

This directly mapped the listening TCP port to the `app` process.

Filtering by PID is safer than filtering by the word `app`, because unrelated process names can contain those letters too.

### 5. Confirm with lsof

```bash
sudo lsof -Pan -p 1109 -iTCP -sTCP:LISTEN
```

Output:

```text
COMMAND  PID USER   FD   TYPE DEVICE SIZE/OFF NODE NAME
app     1109 root    3u  IPv6   7728      0t0  TCP *:26792 (LISTEN)
```

Why:

`lsof` lists open files, and on Linux sockets are represented as file descriptors. This confirmed that PID `1109` had a listening TCP socket on port `26792`.

## Useful Commands

```bash
systemctl status <service>
systemctl show -p MainPID --value <service>
ss -ltnp
sudo ss -ltnp
sudo ss -ltnp | grep "pid=<pid>"
sudo lsof -Pan -p <pid> -iTCP -sTCP:LISTEN
```

## Conclusion

To find the port for a service, identify the service process, then inspect listening sockets with process ownership.

For this challenge, `app.service` had PID `1109`, and that process was listening on:

```text
26792
```
