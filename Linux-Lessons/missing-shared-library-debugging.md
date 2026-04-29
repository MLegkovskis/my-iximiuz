# Missing Shared Library Debugging

## Challenge Goal

Fix `imgconvd`, an image conversion daemon that exited immediately at startup.

Root cause:

```text
libwebp.so.7 was missing
```

Fix:

```bash
sudo apt-get install -y libwebp7
```

## Main Idea

Some Linux binaries are dynamically linked. That means the executable does not contain all the code it needs; at startup, the dynamic linker loads shared libraries from the system.

If a required `.so` file is missing, the program can fail before its own code runs. In that case, app logs may not help because the application never reaches `main()`. Use `ldd` to inspect what the binary needs and which libraries cannot be found.

## Commands Run

### 1. Find the Binary

```bash
which imgconvd
```

Output:

```text
/usr/local/bin/imgconvd
```

### 2. Reproduce the Failure

```bash
timeout 3s imgconvd
```

Output:

```text
imgconvd: error while loading shared libraries: libwebp.so.7: cannot open shared object file: No such file or directory
```

Why:

This error came from the OS dynamic linker, not from the daemon itself.

### 3. Inspect the Binary Type

```bash
file /usr/local/bin/imgconvd
```

Output:

```text
/usr/local/bin/imgconvd: ELF 64-bit LSB pie executable, x86-64, version 1 (SYSV), dynamically linked, interpreter /lib64/ld-linux-x86-64.so.2, BuildID[sha1]=2d431222e50eb4eba24a2e43bc449ab3bfb4de69, for GNU/Linux 3.2.0, not stripped
```

Why:

`dynamically linked` means the binary needs shared libraries available on the system.

### 4. Check Shared Library Dependencies

```bash
ldd /usr/local/bin/imgconvd
```

Output:

```text
linux-vdso.so.1 (0x00007ffc78f96000)
libwebp.so.7 => not found
libc.so.6 => /lib/x86_64-linux-gnu/libc.so.6 (0x00007f0167600000)
/lib64/ld-linux-x86-64.so.2 (0x00007f0167855000)
```

Why:

`ldd` showed the missing dependency directly:

```text
libwebp.so.7 => not found
```

### 5. Find the Package

```bash
apt-cache search "libwebp" | head -n 20
apt-cache policy libwebp7
```

Output:

```text
libwebp-dev - Lossy compression of digital photographic images
libwebp7 - Lossy compression of digital photographic images
libwebpdecoder3 - Library for the WebP graphics format (decode only)
libwebpdemux2 - Lossy compression of digital photographic images.
libwebpmux3 - Lossy compression of digital photographic images
librust-image-dev - Imaging library - Rust source code
librust-libwebp-sys-dev - Bindings to libwebp (bindgen, static linking) - Rust source code

libwebp7:
  Installed: (none)
  Candidate: 1.3.2-0.4build3
  Version table:
     1.3.2-0.4build3 500
        500 http://archive.ubuntu.com/ubuntu noble/main amd64 Packages
```

Why:

On Debian/Ubuntu, a library named `libfoo.so.N` is often provided by a package named `libfooN`. Here, `libwebp.so.7` mapped to `libwebp7`.

### 6. Install the Missing Library

```bash
sudo apt-get update
sudo apt-get install -y libwebp7
```

Relevant output:

```text
The following NEW packages will be installed:
  libsharpyuv0 libwebp7

Setting up libsharpyuv0:amd64 (1.3.2-0.4build3) ...
Setting up libwebp7:amd64 (1.3.2-0.4build3) ...
Processing triggers for libc-bin (2.39-0ubuntu8.7) ...
```

### 7. Verify ldd Again

```bash
ldd /usr/local/bin/imgconvd
```

Output:

```text
linux-vdso.so.1 (0x00007ffd6e3d8000)
libwebp.so.7 => /lib/x86_64-linux-gnu/libwebp.so.7 (0x00007f340bcbb000)
libc.so.6 => /lib/x86_64-linux-gnu/libc.so.6 (0x00007f340ba00000)
libsharpyuv.so.0 => /lib/x86_64-linux-gnu/libsharpyuv.so.0 (0x00007f340bcb3000)
libm.so.6 => /lib/x86_64-linux-gnu/libm.so.6 (0x00007f340b917000)
/lib64/ld-linux-x86-64.so.2 (0x00007f340bd42000)
```

Why:

`libwebp.so.7` now resolves to a real path.

### 8. Verify the Daemon Runs

```bash
timeout 12s imgconvd
```

Output:

```text
imgconvd v3.1.0: uptime=10s converted=7
imgconvd v3.1.0: uptime=12s converted=14
```

Why:

The challenge required `imgconvd` to run for at least 10 seconds. Running it under `timeout 12s` proved it stayed alive long enough.

## Useful Commands

```bash
which <binary>
file <binary>
ldd <binary>
apt-cache search <library-name>
apt-cache policy <package>
sudo apt-get update
sudo apt-get install -y <package>
timeout 12s <binary>
```

## Conclusion

The daemon was not broken internally. The OS could not start it because a required shared library was missing.

`ldd` identified:

```text
libwebp.so.7 => not found
```

Installing `libwebp7` fixed the dynamic linker error, and `imgconvd` then ran successfully for more than 10 seconds.
