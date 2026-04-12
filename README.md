# fusion

A kernel-assisted lightweight block device for SPDK

`fusion` is a high-performance hybrid block device framework that bridges
SPDK user-space I/O with the Linux kernel block layer.

It enables submitting I/O from SPDK directly into kernel block devices
with minimal overhead, while preserving:

- lock-free per-core execution model
- blk-mq scalability
- zero-copy data path (via page pinning)
- optional T10 PI (DIF) integrity passthrough

---

## Motivation

SPDK provides excellent performance for NVMe, but integration with:

- legacy block devices
- SCSI devices with T10 PI
- kernel storage stacks

is either inefficient or unavailable.

`fusion` fills this gap by combining:

- SPDK reactor model (user space)
- Linux blk-mq subsystem (kernel space)

into a **single fast I/O pipeline**.

---

## Architecture

```

SPDK (bdev module)
|
| shared memory ring (SQ/CQ)
V
fusion-blk kernel driver
|
V
Linux block layer (blk-mq)
|
V
Physical device (/dev/sdX, /dev/vdX, etc.)

```

### Key ideas

- Each SPDK reactor owns a queue
- Lock-free SQ/CQ ring shared with kernel
- Kernel builds `bio` from user buffers
- Completion is routed back to the same reactor
- No syscalls in fast path except "kick" (RUN)

---

## Features

- Near-native performance for kernel block devices
- Per-core queue model (SPDK-aligned)
- Almost no locks, almost no atomics
- Zero-copy I/O via page pinning
- Pin cache to avoid repeated GUP overhead
- Optional T10 PI (DIF Type 1 / Type 3) passthrough
- Completion on submission CPU (blk-mq affinity)
- Works with:
  - virtio-blk
  - SCSI disks
  - dm/md stacks (with caveats)

---

## Components

### 1. Kernel driver

Located in: `kernel/`

Responsibilities:

- Accept SQEs from shared ring
- Pin user pages via pin-cache
- Construct and submit `bio`
- Handle completion (`bio_end_io`)
- Push CQEs back to user space

---

### 2. SPDK bdev module

Located in: `spdk/`

Responsibilities:

- Expose block device to SPDK
- Manage shared rings per reactor
- Submit SQEs
- Poll CQEs
- Integrate with SPDK I/O channels

---

## Build

### Kernel module

```bash
cd kernel
KERNEL=/path/to/kernel make
sudo insmod hablk.ko
```

### SPDK module

```bash
cd spdk
SPDK_ROOT=/path/to/your/spdk
rm -rf ./build/
cmake -B build -D SPDK_ROOT_DIR=$SPDK_ROOT .
make -C build
sudo ./build/spdk_app -m 0xf
```

> Requires SPDK to be built in `$SPDK_ROOT` with headers available.

---

## Usage

See spdk/README.md

---

## Limitations

* Requires root privileges
* Assumes SPDK-managed memory (hugepages)
* No request merging yet (bio splitting only)
* Limited inline page reference storage (WIP)
* Integrity support depends on underlying device

---

## Performance Notes

* Avoids syscall-per-IO overhead
* Avoids kernel thread wakeups
* Batch submission via `blk_plug`
* Pin cache significantly reduces GUP overhead

Not widely tested yet. `fio` tests at QEMU ram disks
(virtio-blk, virtio-scsi-pci) show performance is at
parity with `bdev_uring` or slightly above (1...2 %).

---

## Development Status

* [x] Basic READ/WRITE path
* [x] Multi-queue support
* [x] Completion affinity
* [x] Pin cache
* [x] T10 PI passthrough
* [ ] Full zero-allocation fast path
* [ ] Direct blk-mq request submission (no bio)
* [ ] uring_cmd integration (future)

---

## Contributing

Contributions are welcome!

Focus areas:

* performance tuning
* memory optimization
* integrity pipeline
* rq-based fast path

---

## License

Kernel part: GPL-2.0

SPDK module follows SPDK licensing: BSD-3-Clause
