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

### run (from $TT_RS_HOME, SD root, no initramfs)
./qemu/build/qemu-system-riscv64 -M tt-atlantis \
  -kernel linux-build-rv64/arch/riscv/boot/Image \
  -nographic -m 8G \
  -device ttsim,lib=sim/libttsim_bh.so \
  -drive file=sdcard.img,if=sd,format=raw \
  -device virtio-net-pci,netdev=net0 -netdev user,id=net0 \
  -append "console=ttyS0 earlycon root=/dev/mmcblk0 rootwait rw"

