# tt-rs Bring-up Plan: `tt-atlantis` + BH `ttsim` + KMD

## Step 1 — Minimal boot on `-M tt-atlantis` (done)
- Minimal `qemu-system-riscv64` (`--target-list=riscv64-softmmu`).
- `defconfig` + `tt-atlantis-minimal.fragment` kernel (`RUST`, `KUNIT`,
  `RUST_KERNEL_DOCTESTS`; modules off except core `MODULES=y` for KMD).
- GPEX host bridge added to `tt-atlantis`
  (ECAM@0x84000000, MEM@0x90000000, PIO@0xd4000000, HIGH@0x2000000000,
  INTx 40-43, windows mapped into soc memory).
- Boot: `./build/qemu-system-riscv64 -M tt-atlantis -kernel <Image>
  -initrd <initramfs> -nographic` (OpenSBI found automatically).
- Verified: kernel boots, PCI scanned, Rust doctest-kunit `ok`,
  BusyBox shell prompt.

## Step 2 — C KMD (`tt-kmd`) works on BH `ttsim`
- Goal: `tenstorrent.ko` loads in the guest, binds `1e52:b140`,
  `/dev/tenstorrent/0` appears, KMD tests pass.
- Tasks:
  1. Build `tt-kmd` against the guest kernel
     (`ARCH=riscv`, `KDIR=linux-build-rv64`).
  2. Pack `tenstorrent.ko` into initramfs, `insmod` at boot.
  3. Verify bind via dmesg + `/dev/tenstorrent/*` + `lspci -k`.
  4. Run kernel KUnit suite + KMD hardware tests
     (`tt-kmd/test/run-hardware-tests.sh`, starting with
     enumeration/config-space).
- Success: KMD probes BH sim with no errors; enumeration test passes.

## Step 3 — Rust KMD + Rust KUnit
- Goal: a Rust in-tree PCI driver for `1e52:b140` with KUnit coverage,
  booted and bound on `tt-atlantis`.
- Tasks:
  1. Scaffold minimal Rust PCI driver (probe/BAR map/basic chardev),
     following `samples/rust` patterns.
  2. Add KUnit tests (doctests + `kunit` test cases) for probe/bind paths.
  3. Enable via fragment, build, boot with BH `ttsim`, confirm bind
     and KUnit `ok`.
- Success: Rust driver binds the sim device; all Rust KUnit tests pass.

## Verified log
- 2026-10-09: baseline `defconfig` boot → `ttyS0` up, VFS panic w/o initrd
  (expected).
- 2026-10-09: `RUST=y` rebuild boots identically.
- 2026-10-09: fragment step 1 (`=y` 1494→1208, `=m` 223→0,
  Image 27.5MB→22.8MB, rebuild 6m47s) → `userspace alive!`.
- 2026-10-09: fragment step 2 (HWMON/WATCHDOG/I2C/SPI off, `KUNIT=y`)
  → `=y` 1163, rebuild 6m42s → `userspace alive!`.
- 2026-10-09: GPEX probe + bus scan OK, no Oops; BusyBox shell works,
  `lspci` shows bridge.
- 2026-10-09: fragment adds `RUST` + `RUST_KERNEL_DOCTESTS` (pushed).
- 2026-10-09: `-device ttsim,lib=libttsim_bh.so` → guest sees
  `pci 0000:00:01.0: [1e52:b140]`; `lspci` confirms.
- 2026-10-09 (Step 4): Debian trixie `nocloud-riscv64` rootfs as initramfs
  (`-initrd`, `-m 8G`) boots to dash shell; `lspci` names the BH device;
  fresh `tenstorrent.ko` insmods clean (exit 0).
- 2026-10-09 (Step 4): full Debian boot — Debian `update-initramfs`
  initrd for our kernel + root on SD (`mmcblk0`, LABEL=debian-root) +
  systemd → `tt-atlantis login:` prompt. Needed `UNIX`/`INET` in fragment,
  fstab LABEL fix, root pw set, firstboot masked.
- 2026-10-10: fragment cleanup (`=y` 1320, `=m` 0, NET tunnel/PHY
  modules off); final cmdline boots w/o initrd straight to SD root +
  `virtio-net-pci` enumerates (`1af4:1000`, BARs assigned), login prompt OK.
- 2026-10-09: Rust doctest-kunit `ok 158` in guest dmesg.
- 2026-10-09 (Step 3): in-tree Rust driver `drivers/misc/tt_bh/`
  (`TENSTORRENT_BH_RUST`, probe `1e52:b140`, BAR0 map+read) binds BH sim
  (`BAR0[0] = 0x0`); KUnit `tt_bh_tests: pass:2 fail:0`. (Uncommitted.)
- 2026-10-10 (Step 3 P1a): `tt_bh_rust` miscdev + C-ABI-identical
  `GET_DEVICE_INFO`/`GET_DRIVER_INFO` ioctls; guest userspace test
  (`test_tt`) → vendor/device/subsys/maxdma + api 2.11.1 all ok,
  bad ioctl rejected → RESULT: PASS; KUnit `pass:3`.
- 2026-10-10 (Step 3 P1b): TLB alloc/configure/free, NOC read/write,
  reset, mappings, harvesting ioctls + `bh.rs` KUnit (pack/pool/validate);
  standard loop is kernel + busybox mini initramfs
  (`initramfs-mini/mini.cpio`, 2.4MB, no SD) → RESULT: PASS.
- 2026-10-10 (Step 3 P2): MSI-X bring-up best-effort (sim has no MSI-X/
  INTx, C also continues; new `pci::irq_vector/request_irq/free_irq`
  abstractions), NOC_CLEANUP + POWER_STATE ioctls → RESULT: PASS.
- 2026-10-10 (Step 3 P3a): DMA core — new `mm::pin` GUP abstraction,
  `AsPageIter` size/page_count extension, PIN/UNPIN + ALLOC/FREE_DMABUF
  + LOCK_CTL + mmap (BAR/TLB/dmabuf) ioctls, iATU + KUnit → RESULT: PASS.
- 2026-10-10 (Step 3 P3b): MAP_PEER_BAR validation paths → RESULT: PASS.
- 2026-10-10 (Step 3 P4a): ARC firmware msgqueue (ring math + locate/
  trigger/send_sync/pump, SMC POST/POLL/ABANDON) + KUnit → RESULT: PASS.
- 2026-10-10 (Step 3 P4b): fwlog framing parser port + KUnit (6 tests);
  forwarding disabled without IOMMU (mirrors C) → RESULT: PASS.
- 2026-10-09 (Step 2): `tenstorrent.ko` (tt-kmd, riscv64 cross-build) insmod
  in guest → `Found a Tenstorrent Blackhole device`, `/dev/tenstorrent/0`
  created. Needed `HWMON=y` back in fragment (KMD telemetry).
- 2026-10-10 (Step 3 P4c): telemetry cache + full reset paths
  (config-space marker, timer interrupt, ARC DMC, save/restore) → PASS.
- 2026-10-10 (Step 3 P5): Wormhole backend (TLB/NOC/ARC/iATU/DBI/
  telemetry/reset/init) + chip dispatch; WH sim boots, binds
  `1e52:401e`, full suite → RESULT: PASS (BH regression also PASS).
- 2026-10-10 (Step 3 P4d/e): power aggregation via ARC, reset
  generation (stale-fd mmap rejection proven live), NOC-cleanup on
  release → RESULT: PASS.
- 2026-10-10 (Step 3 P6): dma-buf export of TLB windows (new Rust
  abstraction, refcount model, revoke guard) → export fd OK → PASS.
- 2026-10-11 (Step 3 P7): blocking `LOCK_CTL` (interruptible wait),
  PCIe link hot-reset (new `pci_upstream_bridge` helper), telemetry
  surfaced through a miscdev `read` handler, chip-specific power ARC
  type (BH 0x21 / WH 0xc0) with raw settings payload → RESULT: PASS
  on both `libttsim_bh.so` and `libttsim_wh.so`.
- 2026-10-11 (Step 3 P8): dma-buf revoke on unbind (mark `revoked` +
  `dma_buf_invalidate_mappings`), EBUSY-on-reset guard proven live
  with an open export fd → RESULT: PASS on both sims.
- 2026-10-11 (Step 3 P9): Kernel config harden (devtmpfs mount, RAM disk,
  initrd, SYNC_FILE, 8250 DW UART), C init binary for test execution
  (avoids busybox segfault in kernel userspace), all KUnit suites green
  (370 doctests + 27 driver tests). BH sim full test pass via C init
  binary; console visibility limited by DT/8250 binding order.

## Remaining

- `tt-kmd/test` (C++/gtest ABI-parity suite) is not run yet: it needs
  `g++-riscv64-linux-gnu`, which is not installed on the host.
- `EXPORT_TLB_DMABUF` revoke-on-reset only tracks live exports (no
  importer invalidation); the C driver additionally marks entries
  revoked when the device is torn down mid-flight.
- Telemetry is exposed via `read()` on `/dev/tt_bh_rust`; the C driver
  uses hwmon, which has no Rust binding upstream yet.
