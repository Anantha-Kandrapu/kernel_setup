# 🐧 Linux Kernel Development & Debugging (arm64)

This guide provides a structured workflow for building, running, and debugging the Linux kernel using **QEMU** and **VS Code**.

---

## 1. Development Environment

Use a Docker-based devcontainer to ensure all dependencies are met.

### `Dockerfile`
```dockerfile
FROM ubuntu:24.04
ARG DEBIAN_FRONTEND=noninteractive

RUN apt-get update && apt-get install -y \
    wget curl ca-certificates git build-essential \
    flex bison bc libncurses-dev libssl-dev libelf-dev \
    dwarves kmod cpio python3 pkg-config ccache \
    gdb-multiarch qemu-system-arm busybox-static \
    && rm -rf /var/lib/apt/lists/*

WORKDIR /linux
```

### `.devcontainer/devcontainer.json`
```json
{
  "name": "Linux Kernel Dev",
  "build": { "dockerfile": "../Dockerfile" },
  "mounts": [
    "source=linux-build-cache,target=/build,type=volume"
  ],
  "workspaceMount": "source=${localWorkspaceFolder},target=/linux,type=bind",
  "workspaceFolder": "/linux",
  "forwardPorts": [1234],
  "customizations": {
    "vscode": {
      "extensions": [
        "ms-vscode.cpptools"
      ]
    }
  }
}
```

---

## 2. Cleaning & Configuration

Before building, ensure the source tree is clean and configured correctly.

### Clean Source Tree
```bash
git clean -fdx --exclude=.vscode/ --exclude=.devcontainer/ --exclude=Dockerfile
```

### Initial Configuration
```bash
make O=/build defconfig
```

### Enable Debugging Symbols
To get a good debugging experience, enable these options in your config:
```bash
./scripts/config --objtree /build \
    --enable DEBUG_INFO \
    --enable GDB_SCRIPTS \
    --enable DEBUG_KERNEL \
    --enable KGDB \
    --disable RANDOMIZE_BASE
```
*Note: Disabling `RANDOMIZE_BASE` (KASLR) is critical for breakpoints to work.*

---

## 3. Building the Kernel

Build the kernel and modules using the `/build` volume for speed and separation.

```bash
# Build the core image
make O=/build CC="ccache gcc" -j$(nproc) Image

# Build specific modules (e.g., networking)
make M=net/core -j$(nproc)

# Install modules
make modules_install
```

---

## 4. Creating the Initramfs

The `initramfs` provides the minimal userspace environment.

### The `init` Script
Create a file at `ir/init` (or `initramfs/init`):
```bash
#!/bin/sh
mount -t proc proc /proc
mount -t sysfs sysfs /sys
mount -t devtmpfs devtmpfs /dev

ip link set lo up
ip link set eth0 up
ip addr add 10.0.2.15/24 dev eth0
ip route add default via 10.0.2.2
echo "nameserver 10.0.2.3" > /etc/resolv.conf

echo "Network up — Try: ping 10.0.2.2"

# cttyhack provides a working TTY for the shell
setsid cttyhack /bin/sh
exec /bin/sh
```
*Make sure to `chmod +x ir/init`.*

### Pack the Initramfs
```bash
cd ir && find . | cpio -o -H newc | gzip > /build/initramfs.cpio.gz && cd ..
```

---

## 5. Launching with QEMU

Run the kernel in QEMU with the GDB stub enabled (`-s -S`).

```bash
qemu-system-aarch64 \
  -machine virt \
  -cpu cortex-a72 \
  -m 1024 \
  -kernel /build/arch/arm64/boot/Image \
  -initrd /build/initramfs.cpio.gz \
  -append "console=ttyAMA0 nokaslr" \
  -nographic \
  -netdev user,id=net0 \
  -device virtio-net-pci,netdev=net0,romfile="" \
  -s -S
```
- `-s`: Opens GDB server on port 1234.
- `-S`: Freezes the CPU at startup (wait for debugger).

---

## 6. Debugging with VS Code

Configure `launch.json` to connect to the QEMU GDB stub.

### `.vscode/launch.json`
```json
{
  "version": "0.2.0",
  "configurations": [
    {
      "name": "Kernel Debug",
      "type": "cppdbg",
      "request": "launch",
      "program": "/build/vmlinux.unstripped",
      "miDebuggerServerAddress": "127.0.0.1:1234",
      "miDebuggerPath": "gdb-multiarch",
      "cwd": "/linux",
      "stopAtEntry": false,
      "setupCommands": [
        { "text": "set architecture aarch64" }
      ]
    }
  ]
}
```

> [!TIP]
> Use `/build/vmlinux` for symbols. It matches the code running in QEMU's `Image`.
