# Scull character driver sandbox

This note describes how to spin up a quick sandbox for the Scull driver from *Linux Device Drivers* and what the core code paths look like.

## Environment prerequisites
- Kernel headers or a kernel source tree available locally (set `KERNELDIR` if it is **not** `/lib/modules/$(uname -r)/build`).
- Build tooling for kernel modules (`make`, `gcc`, etc.).
- Root privileges to insert the module and create device nodes.

## Quickstart build and load
1. Build only the Scull module from this repository:
   ```bash
   cd scull
   make            # optionally override KERNELDIR=/path/to/linux/tree
   ```
2. Load the module and create device nodes using the provided helper (runs `insmod` and `mknod` under sudo/root):
   ```bash
   sudo ./scull_load
   ```
   By default the driver allocates four devices starting at major `0` (dynamic) and minor `0`; adjust with module parameters such as `scull_major`, `scull_minor`, and `scull_nr_devs` when loading the module if needed.
3. Interact with the character devices (e.g., `/dev/scull0`) using standard tools:
   ```bash
   echo "hello" | sudo tee /dev/scull0
   sudo head -c 5 /dev/scull0
   ```
4. Unload and clean up when finished:
   ```bash
   sudo ./scull_unload
   make clean
   ```

## What the driver implements
- **Configurable memory layout** – The driver stores data in linked lists of "quantum" buffers grouped into quantum sets (`scull_qset`). Defaults include a 4,000-byte quantum and 1,000-entry quantum set, and they can be changed at module load time via parameters like `scull_quantum` and `scull_qset`. Device state is tracked in `struct scull_dev`, which embeds a `struct cdev` for character-device registration and a mutex for serialization.【F:scull/scull.h†L55-L131】【F:scull/main.c†L41-L83】
- **Open/close and state reset** – `scull_open` wires each `struct file` to the appropriate device structure and trims existing data when a writer opens the device, ensuring a clean buffer for write-only sessions. `scull_release` is a no-op placeholder.【F:scull/main.c†L237-L261】
- **Read/write paths** – Reads and writes walk the quantum-set list with `scull_follow`, lazily allocating buffers on write and refusing to read uninitialized holes. Both paths are serialized with a device mutex and update the file position; writes extend the tracked size so subsequent reads see the new data.【F:scull/main.c†L263-L390】
- **IOCTL controls** – `scull_ioctl` exposes a small control surface to tweak runtime parameters (quantum and qset sizes) and report the current values, with admin-only semantics where appropriate. It also proxies buffer-size tweaks for the related `scullpipe` device to reuse the same handler.【F:scull/main.c†L392-L517】
- **File operations registration** – The `scull_fops` table wires these handlers into the VFS, and `scull_setup_cdev` registers each device number with the kernel using `cdev_add`, enabling multiple device instances based on the configured count.【F:scull/main.c†L521-L671】

## Notes for experimentation
- The top-level `Makefile` also builds other example modules; the commands above limit the build to the Scull driver to keep the sandbox minimal.
- Debugging helpers under `SCULL_DEBUG` add `/proc/scullmem` and `/proc/scullseq` entries that dump allocation metadata if you need visibility into how buffers are laid out.【F:scull/main.c†L85-L231】
- Because the driver uses dynamic allocation, tools like `dmesg` can help catch allocation failures or permission issues during load/unload.
