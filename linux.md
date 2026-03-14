MacOs setup
After cloning the linux kernel.

open in a devcontainer in vscode. (make sure docker is running)

Create a named docker volume and mount it to /build

To build kernel , choose a config:

Most common is defconfig.

`make defconfig -O=/build` in /linux dir.

then `make O=/build -j$(nproc) Image` in /linux dir.

your image is 