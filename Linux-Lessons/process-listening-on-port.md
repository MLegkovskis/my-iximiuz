# Linux Process Discovery from a Port

## Challenge Goal

Find the PID of the process listening on TCP port `12345`.

Answer:

```text
1240
```

## Main Idea

A network service is just a process with a socket open. If you know the port, you can ask the kernel which process owns the listening socket.

This is the inverse of finding which port a service uses. Here, we already know port `12345`; the task is to map that port back to the owning PID, then use `ps` to inspect the process.

## Commands Run

### 1. Query the Listening Socket with ss

```bash
sudo ss -ltnp "sport = :12345"
```

Why:

- `ss` lists sockets.
- `-l` shows listening sockets.
- `-t` shows TCP sockets.
- `-n` keeps ports numeric.
- `-p` shows the owning process.
- `"sport = :12345"` filters to sockets whose source/local port is `12345`.
- `sudo` is used because process ownership details may be hidden from normal users.

Output:

```text
State  Recv-Q Send-Q Local Address:Port  Peer Address:PortProcess
LISTEN 0      4096               *:12345            *:*    users:(("eager-turtle",pid=1240,fd=3))
```

The key part is:

```text
pid=1240
```

### 2. Confirm with lsof

```bash
sudo lsof -Pan -iTCP:12345 -sTCP:LISTEN
```

Why:

`lsof` lists open files. On Linux, sockets are file descriptors too, so `lsof` can show which process has a listening TCP socket open on port `12345`.

Output:

```text
COMMAND    PID USER   FD   TYPE DEVICE SIZE/OFF NODE NAME
eager-tur 1240 root    3u  IPv6  13938      0t0  TCP *:12345 (LISTEN)
```

Again, the PID is:

```text
1240
```

### 3. Inspect the Process

```bash
ps -fp 1240
```

Why:

`ps` confirms what process the PID belongs to and shows the command used to start it.

Output:

```text
UID          PID    PPID  C STIME TTY          TIME CMD
root        1240       1  0 17:12 ?        00:00:00 /usr/local/bin/eager-turtle
```

## Useful Commands

```bash
sudo ss -ltnp "sport = :<port>"
sudo lsof -Pan -iTCP:<port> -sTCP:LISTEN
ps -fp <pid>
```

## Conclusion

To find which process is listening on a known port, filter listening sockets by that port and read the owning PID.

For this challenge, TCP port `12345` was owned by:

```text
PID 1240
```
