# systemd Service for a Long-Running Process

## Challenge Goal

Turn `/usr/local/bin/simple-server` into a managed `systemd` service.

The service needed to:

- Live at `/etc/systemd/system/simple-server.service`.
- Run `/usr/local/bin/simple-server`.
- Start now.
- Start automatically after reboot.
- Restart automatically after abnormal termination.
- Serve HTTP on port `8080`.

## Main Idea

Running a long-lived process directly from a shell is fragile. If the terminal closes, the process can die. If the process crashes, nothing restarts it. If the machine reboots, it does not come back automatically.

`systemd` is the common Linux service manager. It can start, stop, restart, enable, monitor, and restart services.

## Unit File Created

```ini
[Unit]
Description=Simple HTTP Server
After=network.target

[Service]
ExecStart=/usr/local/bin/simple-server
Restart=on-failure
RestartSec=1s

[Install]
WantedBy=multi-user.target
```

Why:

- `[Unit]` describes the service and ordering.
- `After=network.target` starts it after basic networking is available.
- `[Service]` defines the process to run.
- `ExecStart=` is the main command.
- `Restart=on-failure` restarts after crashes, non-zero exits, and signals like `SIGKILL`.
- `RestartSec=1s` waits one second before restarting.
- `[Install]` defines how the service is enabled for boot.
- `WantedBy=multi-user.target` hooks it into the normal multi-user boot target.

## Commands Run

### 1. Confirm the Binary Exists

```bash
ls -l /usr/local/bin/simple-server
```

Output:

```text
-rwxr-xr-x 1 root root 8340346 Mar 13 17:49 /usr/local/bin/simple-server
```

### 2. Create the Unit File

```bash
sudo tee /etc/systemd/system/simple-server.service >/dev/null <<'EOF'
[Unit]
Description=Simple HTTP Server
After=network.target

[Service]
ExecStart=/usr/local/bin/simple-server
Restart=on-failure
RestartSec=1s

[Install]
WantedBy=multi-user.target
EOF
```

Why:

System-level unit files live under `/etc/systemd/system/`.

### 3. Reload systemd

```bash
sudo systemctl daemon-reload
```

Why:

After creating or editing a unit file, `systemd` needs to reload its configuration.

### 4. Enable and Start the Service

```bash
sudo systemctl enable --now simple-server.service
```

Output:

```text
Created symlink /etc/systemd/system/multi-user.target.wants/simple-server.service -> /etc/systemd/system/simple-server.service.
```

Why:

- `enable` makes it start on boot.
- `--now` starts it immediately.

### 5. Check Service Status

```bash
systemctl status simple-server.service --no-pager --lines=8
```

Output:

```text
● simple-server.service - Simple HTTP Server
     Loaded: loaded (/etc/systemd/system/simple-server.service; enabled; preset: enabled)
     Active: active (running) since Wed 2026-04-29 17:27:37 UTC; 6ms ago
   Main PID: 1104 ((e-server))
      Tasks: 1 (limit: 1160)
     Memory: 0B (peak: 0B)
        CPU: 0
     CGroup: /system.slice/simple-server.service
             └─1104 /usr/local/bin/simple-server
```

### 6. Confirm It Listens on Port 8080

```bash
sudo ss -ltnp "sport = :8080"
```

Output:

```text
State  Recv-Q Send-Q Local Address:Port Peer Address:PortProcess
LISTEN 0      4096               *:8080            *:*    users:(("simple-server",pid=1104,fd=3))
```

### 7. Confirm HTTP Response

```bash
curl -i --max-time 3 http://127.0.0.1:8080/
```

Output:

```text
HTTP/1.1 200 OK
Date: Wed, 29 Apr 2026 17:27:37 GMT
Content-Length: 24
Content-Type: text/plain; charset=utf-8

Hello from linux/amd64!
```

### 8. Confirm Enabled and Active

```bash
systemctl is-enabled simple-server.service
systemctl is-active simple-server.service
```

Output:

```text
enabled
active
```

### 9. Kill the Main Process

```bash
old=$(systemctl show -p MainPID --value simple-server.service)
sudo kill -9 "$old"
sleep 2
systemctl show -p MainPID --value simple-server.service
```

Output:

```text
1104
1156
```

Why:

The PID changed from `1104` to `1156`, proving `systemd` restarted the service after the process was killed.

### 10. Reboot and Verify Auto-Start

```bash
sudo reboot
```

After reconnecting:

```bash
systemctl is-enabled simple-server.service
systemctl is-active simple-server.service
systemctl status simple-server.service --no-pager --lines=8
sudo ss -ltnp "sport = :8080"
curl -sS --max-time 3 http://127.0.0.1:8080/
```

Output:

```text
enabled
active

● simple-server.service - Simple HTTP Server
     Loaded: loaded (/etc/systemd/system/simple-server.service; enabled; preset: enabled)
     Active: active (running) since Wed 2026-04-29 17:27:51 UTC; 7s ago
   Main PID: 809 (simple-server)
      Tasks: 3 (limit: 1160)
     Memory: 6.0M (peak: 6.2M)
        CPU: 9ms
     CGroup: /system.slice/simple-server.service
             └─809 /usr/local/bin/simple-server

State  Recv-Q Send-Q Local Address:Port Peer Address:PortProcess
LISTEN 0      4096               *:8080            *:*    users:(("simple-server",pid=809,fd=3))

Hello from linux/amd64!
```

## Useful Commands

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now <service>
systemctl status <service> --no-pager
systemctl is-enabled <service>
systemctl is-active <service>
systemctl show -p MainPID --value <service>
sudo ss -ltnp "sport = :<port>"
journalctl -u <service> --no-pager
```

## Conclusion

`systemd` turns a foreground command into a managed service. In this challenge, `simple-server` became a boot-enabled, self-restarting service listening on port `8080`.

The critical directives were:

```ini
ExecStart=/usr/local/bin/simple-server
Restart=on-failure
WantedBy=multi-user.target
```
