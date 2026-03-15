Linux Kernel Debugging with QEMU + GDB
The Stack
QEMU (emulates arm64 hardware: CPU, RAM, NIC, serial)
  └── Kernel (your compiled Image — raw aarch64 opcodes)
       └── initramfs (unpacked into RAM at boot)
            └── /init script runs
                 └── busybox provides shell + commands
Key Files from Kernel Build
vmlinux.unstripped — ELF binary with full debug symbols. GDB uses this. Biggest file.
vmlinux — ELF with partial symbols. Smaller.
Image (arch/arm64/boot/) — raw opcodes, no ELF header, no symbols. QEMU executes this.
All three contain the same machine code. Only difference is how much metadata is attached.

Build chain: C source → Clang/GCC → assembly → object files → linker → vmlinux (ELF) → objcopy → Image (raw binary)

ELF (Executable and Linkable Format)
Standard binary format on Linux. Container that holds: machine code, symbol table (function names → addresses), debug info (DWARF — source files, line numbers, variable types), section headers, metadata (architecture, endianness).

First 4 bytes of any ELF: 7f 45 4c 46 (\x7fELF).

Inspect with: readelf -h vmlinux, readelf -S vmlinux, readelf -s vmlinux | grep netif_receive_skb

QEMU
Command-line hardware emulator. Like VMware but exposes a GDB stub so you can pause the kernel mid-execution and step through source.

Emulates: CPU, RAM, NIC, serial console, etc. The kernel thinks it's on real hardware.

```
qemu-system-aarch64 \
  -kernel Image \
  -initrd initramfs.cpio.gz \
  -append "console=ttyAMA0" \
  -machine virt \
  -cpu cortex-a57 \
  -m 1024 \
  -nographic \
  -netdev user,id=net0 \
  -device virtio-net-device,netdev=net0 \
  -s -S
```

-s = start GDB server on port 1234
-S = freeze CPU at boot, wait for GDB to connect
-gdb tcp::PORT = long form of -s, use for multiple instances on different ports
GDB + QEMU Relationship

GDB is NOT running on QEMU. Two separate processes on your host connected via TCP.

QEMU has a built-in GDB server (GDB stub). It opens a TCP socket on port 1234. GDB connects to it. They communicate using the GDB Remote Serial Protocol — a simple text-based protocol over raw TCP (not HTTP).

GDB sends:    $g#67              (read registers)
QEMU replies: $aabbccdd...#xx    (register values)

GDB sends:    $m0xffff...,4#xx   (read memory at address)
QEMU replies: $deadbeef#xx       (memory contents)
How it works:

QEMU runs Image (opcodes) on emulated CPU
GDB loads vmlinux.unstripped (symbol table only, never executes it)
GDB connects to QEMU on port 1234
CPU hits address 0xffff... → GDB looks it up in vmlinux → "that's netif_receive_skb at net/core/dev.c:5765"
VSCode shows the source line
VSCode launch.json
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

Set breakpoints by clicking the gutter in VSCode. No manual GDB commands needed — launch.json handles target remote via miDebuggerServerAddress.

Multiple kernels: run each QEMU on a different port (-gdb tcp::1235, etc.), create a separate VSCode debug config per kernel.

initramfs
The kernel has no shell, no commands — just kernel code. It needs userspace. initramfs is a tiny filesystem packed into a .cpio.gz archive that gets loaded into RAM at boot.

The kernel decompresses it, unpacks it, and runs /init from it. That's your userspace entry point.

Structure
initramfs/
├── init              (shell script — first thing kernel executes)
├── bin/
│   ├── busybox       (single static binary)
│   ├── sh -> busybox (symlink)
│   ├── ping -> busybox
│   ├── mount -> busybox
│   └── ...
├── dev/
├── proc/
├── sys/
└── tmp/

## The init Script:

```
#!/bin/sh
mount -t proc proc /proc
mount -t sysfs sys /sys
mount -t devtmpfs dev /dev
echo "ello initramfs"
ip link set eth0 up
ip addr add 10.0.2.15/24 dev eth0
ip route add default via 10.0.2.2

exec /bin/busybox sh
```

Mounts virtual filesystems (needed for commands like ping, ip link)
exec /bin/sh drops you into a shell
How kernel executes /init
If /init is a compiled binary — kernel runs it directly. If /init is a shell script with #!/bin/sh — kernel sees the shebang, calls execve("/bin/sh", "/init"). So /bin/sh must exist and resolve.

Creating initramfs
mkdir -p initramfs/{bin,dev,proc,sys,tmp}
# copy compiled busybox into initramfs/bin/
  eg: cp /usr/bin/busybox initramfs/bin/
# run `busybox --install -s` to create symlinks
# write the init script
 `chmod +x initramfs/init`

# use cpio(file attacher attaches files end to end)
cd initramfs && find . | cpio -o -H newc | gzip > ../initramfs.cpio.gz
 Eg: 
 ```root@1e42bb7151be:/build# cd initramfs && find . | cpio -o -H newc | gzip > ../initramfs.cpio.gz
 3742 blocks
 ````

Not compulsory to use initramfs. Alternatives: root filesystem on disk image (-hda disk.img), or embed it into the kernel via CONFIG_INITRAMFS_SOURCE.

:::Info
cpio
Copy In/Copy Out. Old Unix archive format. Flat list of files serialized: [header][filename][data][header][filename][data].... The kernel's built-in extractor only understands newc format (-H newc). Kernel doesn't understand tar — that's why cpio.

busybox
Single static binary (~300 commands). Checks argv[0] to decide behavior.

/bin/ping 10.0.2.2         → symlink to busybox, argv[0]="ping" → runs ping code
/bin/busybox ping 10.0.2.2 → direct invocation, argv[1]="ping" → same result
busybox --install -s creates symlinks for all applets. Not strictly required — you could always invoke busybox <command> directly. But /bin/sh symlink is needed for the shebang in /init.

Compiled statically (no libc dependency) and cross-compiled for arm64. Zero dependencies — runs in a bare initramfs with nothing else.
:::

The Symlink Bug
busybox --install -s created absolute symlinks: sh -> /initramfs/bin/busybox. Inside QEMU the root is /, not /initramfs/. So #!/bin/sh → follow symlink → /initramfs/bin/busybox → doesn't exist → error -2. Fix: make symlinks relative (sh -> busybox).

Kernel vs OS
Kernel = the core. Scheduling, memory, drivers, networking. What you're debugging.
OS = kernel + userspace. Shell, tools, package manager, init system, etc.
Linux is a kernel. Ubuntu/Debian/Arch are operating systems built around it. Your QEMU setup (kernel + busybox initramfs) is a minimal OS.

## Qemu Start with your kernel

```
qemu-system-aarch64   -kernel /build/arch/arm64/boot/Image   -initrd /build/initramfs.cpio.gz   -append "console=ttyAMA0"   -machine virt   -cpu cortex-a57   -m 1024   -nographic   -netdev user,id=net0   -device virtio-net-device,netdev=net0   -s
```

Triggering Code Paths
The debugger pauses and steps through code, but networking code only runs when packets are sent/received. ping is the trigger.

ping in QEMU shell
  → busybox calls sendto() syscall
    → kernel enters net/core/dev.c, ip_output, etc.
      → breakpoint hits → VSCode pauses → shows source line

# QEMU Shell Setup

Do: 
  ping -c 1 10.0.2.2 and in editor it should stop at debug points. 


Key Breakpoints for net/core
RX path: netif_receive_skb → __netif_receive_skb_core → ip_rcv → icmp_rcv

TX path: icmp_echo → ip_output → dev_queue_xmit

For TCP: tcp_v4_rcv, tcp_transmit_skb, tcp_v4_connect