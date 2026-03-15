MacOs setup
After cloning the linux kernel.

open in a devcontainer in vscode. (make sure docker is running)

Dockerfile
```

FROM ubuntu:24.04
ARG DEBIAN_FRONTEND=noninteractive

RUN apt-get update && apt-get install -y --no-install-recommends \
    # Build essentials
    build-essential flex bison bc \
    # Kernel build deps
    libncurses-dev libssl-dev libelf-dev dwarves kmod cpio \
    # Debug & emulation
    gdb-multiarch qemu-system-arm busybox-static \
    # Utilities
    git \
    && rm -rf /var/lib/apt/lists/*

WORKDIR /linux
```
-------

.devcontainer/devcontainer.json
```
{
  "name": "Linux Kernel Dev",
  "build": {
    "dockerfile": "../Dockerfile"
  },
  "mounts": [
    "source=linux-build-cache,target=/build,type=volume"
  ],
  "workspaceMount": "source=${localWorkspaceFolder},target=/linux,type=bind",
  "workspaceFolder": "/linux",
  "forwardPorts": [1234],
  "customizations": {
    "vscode": {
      "extensions": [
        "ms-vscode.cpptools",
        "ms-vscode.cmake-tools"
      ]
    }
  }
}
```

Launch.json for GDB server connects to QEMU Stub

```
{
    "version": "0.2.0",
    "configurations": [
        {
            "name": "Kernel Debug",
            "type": "cppdbg",
            "request": "launch",
            "program": "${workspaceFolder}/vmlinux.unstripped",
            "cwd": "${workspaceFolder}",
            "miDebuggerServerAddress": "localhost:1234",
            "miDebuggerPath": "aarch64-linux-gnu-gdb",
            "setupCommands": [
                { "text": "set architecture aarch64" },
                { "text": "directory ${workspaceFolder}/net/core" },
                { "text": "directory ${workspaceFolder}/net/ipv4" }
            ],
            "stopAtEntry": false
        }
    ]
}

```



Create a named docker volume and mount it to /build

To build kernel , choose a config:

Most common is defconfig.

`make defconfig -O=/build` in /linux dir.


```
./scripts/config --objtree /build --enable DEBUG_INFO \
--enable GDB_SCRIPTS \
--enable DEBUG_KERNEL \
--enable KGDB
```
then `make O=/build -j$(nproc) Image` in /linux dir.

your image is 