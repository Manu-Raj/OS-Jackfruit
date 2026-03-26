# Task 1: Multi-Container Runtime with Parent Supervisor

Implement a parent supervisor process that can manage multiple containers at the same time instead of launching only one shell and exiting.

Demonstrate:

- Supervisor process remains alive while containers run
- Multiple containers can be started and tracked concurrently
- Each container has isolated PID, UTS, and mount namespaces
- Each container uses its own rootfs copy derived from the provided base rootfs
- `/proc` works correctly inside each container
- Parent reaps exited children correctly with no zombies

**Filesystem isolation:** Each container needs its own root filesystem view. Treat `rootfs-base/` as the template and run each container with a separate writable copy (for example `rootfs-alpha/`, `rootfs-beta/`). Do not run multiple live containers against the same writable rootfs directory. Use `chroot` (simpler) or `pivot_root` (more thorough — prevents escape via `..` traversal) to make the container see only its assigned `container-rootfs` directory as `/`. Inside the container, mount `/proc` so that tools like `ps` work:

```c
mount("proc", "/proc", "proc", 0, NULL);
```

To run helper binaries (e.g., test workloads) inside a container, copy them into that container's rootfs before launch (or copy into `rootfs-base` before creating per-container copies):

```bash
cp workload_binary ./rootfs-alpha/
```

For each container, the supervisor must maintain metadata in user space. At minimum track:

- Container ID or name
- Host PID
- Start time
- Current state (`starting`, `running`, `stopped`, `killed`, etc.)
- Configured soft and hard memory limits
- Log file path
- Exit status or terminating signal after completion

The internal representation is up to you to design, but it must be safe under concurrent access.
