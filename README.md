# tt-rs
Tenstorrent System Software in Rust

## Setup
~/.bashrc
export TT_RS_HOME=/your/path
export ARCH=riscv
export CROSS_COMPILE=riscv64-linux-gnu-

### git repos
git clone git@github.com:asdfkwang/linux.git $TT_RS_HOME
git clone git@github.com:asdfkwang/qemu.git $TT_RS_HOME
git clone git@github.com:asdfkwang/tt-kmd.git $TT_RS_HOME

### debian rootfs (riscv64, prebuilt)
https://cdimage.debian.org/cdimage/cloud/trixie/latest/debian-13-nocloud-riscv64.tar.xz

### run: kernel + busybox (driver test loop, no SD needed)
./qemu/build/qemu-system-riscv64 -M tt-atlantis \
  -kernel linux-build-rv64/arch/riscv/boot/Image \
  -initrd initramfs-mini/mini.cpio \
  -nographic -m 2G -no-reboot \
  -device ttsim,lib=sim/libttsim_bh.so \
  -append "console=ttyS0 earlycon root=/dev/ram rdinit=/init"

### run: full Debian (SD root, milestone only)
./qemu/build/qemu-system-riscv64 -M tt-atlantis \
  -kernel linux-build-rv64/arch/riscv/boot/Image \
  -nographic -m 8G \
  -device ttsim,lib=sim/libttsim_bh.so \
  -drive file=sdcard.img,if=sd,format=raw \
  -device virtio-net-pci,netdev=net0 -netdev user,id=net0 \
  -append "console=ttyS0 earlycon root=/dev/mmcblk0 rootwait rw"

