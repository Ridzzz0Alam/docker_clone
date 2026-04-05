# Linux Containers in 500 Lines of C

A minimal Linux container runtime built from scratch in C, based on the article ["Linux Containers in 500 Lines of Code"](https://blog.lizzie.io/linux-containers-in-500-loc.html) by Lizzie Dixon.

## Why I Built This

I already had a general idea of how Docker works, but I wanted to go deeper. As someone focusing on distributed systems, understanding how containers work at the
kernel level felt like an important gap to fill. This project gave me exactly that — a ground-up look at what actually happens when you run a container, without
any abstractions hiding the details.


## What It Does

Running this program creates an isolated environment where a process has its own:
- **Filesystem** — via mount namespaces and `pivot_root`
- **Hostname** — via UTS namespaces
- **Process list** — via PID namespaces
- **Network devices** — via network namespaces
- **User and group IDs** — via user namespaces
- **IPC objects** — via IPC namespaces

It also drops dangerous Linux capabilities and filters system calls using seccomp to prevent the contained process from escaping or harming the host system.

## How It Works

Modern Linux containers are built on four kernel mechanisms:

### 1. Namespaces
The `clone()` system call creates a new process with isolated views of system resources. This project uses six namespace types:
- `CLONE_NEWNS` — mount namespace (filesystem isolation)
- `CLONE_NEWPID` — PID namespace (process isolation)
- `CLONE_NEWNET` — network namespace (network isolation)
- `CLONE_NEWUTS` — UTS namespace (hostname isolation)
- `CLONE_NEWIPC` — IPC namespace (inter-process communication isolation)
- `CLONE_NEWCGROUP` — cgroup namespace

### 2. Capabilities
Linux capabilities subdivide the power of root into granular permissions. This project drops dangerous ones including:
- `CAP_SYS_ADMIN` — prevents mounting, vm86, and other privileged operations
- `CAP_SYS_BOOT` — prevents rebooting the host
- `CAP_MKNOD` — prevents creating device files (e.g. recreating `/dev/sda`)
- `CAP_DAC_READ_SEARCH` — prevents the "shocker" container escape exploit
- `CAP_SYS_MODULE` — prevents loading kernel modules
- And many more...

### 3. Seccomp
Specific system calls are blocked even after capability dropping:
- `chmod` with setuid/setgid bits - prevents privilege escalation
- `TIOCSTI` ioctl - prevents injecting input into the parent terminal
- `keyctl`, `add_key`, `request_key` - kernel keyring is not namespaced
- `ptrace` - could be used to escape seccomp filters
- `userfaultfd` - used in kernel exploits to pause execution
- `perf_event_open` - can leak kernel addresses

### 4. Filesystem Isolation (Mount Dance)
The container gets its own root filesystem via:
1. Bind mount the target directory to a temp location
2. Call `pivot_root()` to swap it with `/`
3. Unmount the old root entirely

This means the container cannot see or access the host filesystem at all.

## Prerequisites

- Linux (or WSL2 on Windows)
- GCC
- `libcap-dev`
- `libseccomp-dev`

Install dependencies on Ubuntu/Debian:
```bash
sudo apt install gcc libcap-dev libseccomp-dev
```

## Building

```bash
gcc -Wall -Werror contained.c -o contained -lcap -lseccomp
```

## Usage

First you need a root filesystem to run inside the container. The easiest way is to use BusyBox:

```bash
sudo docker export $(sudo docker create busybox) -o busybox.tar
mkdir busybox-img
sudo tar -xf busybox.tar -C busybox-img
```

Then run the container:
```bash
sudo ./contained -m ./busybox-img -u 0 -c /bin/sh
```

### Options
| Flag | Description |
|------|-------------|
| `-m` | Path to the root filesystem directory |
| `-u` | User ID to run as inside the container (0 = root) |
| `-c` | Command to run inside the container |

### Example Session
```
=> validating Linux version...6.6.87.2 on x86_64.
=> remounting everything with MS_PRIVATE...remounted.
=> making a temp directory and a bind mount there...done.
=> pivoting root...done.
=> unmounting /oldroot.ZIEYvE...done.
=> trying a user namespace...done.
=> switching to uid 0 / gid 0...done.
=> dropping capabilities...bounding...inheritable...done.
=> filtering syscalls...done.
/ # whoami
root
/ # hostname
00fc3c-ace-of-swords
/ # ps
PID   USER     TIME  COMMAND
    1 root      0:00 /bin/sh
    4 root      0:00 ps
/ # exit
```

## Notes

- This code was written for **Linux kernel 4.7/4.8** originally. If you're running a modern kernel (5.x or 6.x) you need to update the kernel version check in `main()`.
- If your system uses **cgroups v2** (most modern Linux systems and WSL2), the resource limiting section needs to be disabled as the code uses cgroups v1 paths.
- This is a **learning project** not intended for production use. For production containers use Docker, Podman, or containerd.

## What This Taught Me

By reading and running this code you learn:
- How Docker and other container runtimes actually work under the hood
- What Linux namespaces are and how they provide isolation
- How Linux capabilities subdivide root privileges
- How seccomp filters system calls
- How `pivot_root` changes the filesystem view of a process
- Why certain kernel features are dangerous in containers

## Relevance to Distributed Systems
Understanding containers at this level is directly relevant to distributed systems. Container orchestration platforms like Kubernetes rely entirely on these
kernel primitives. Knowing what namespaces, cgroups, and capabilities actually do makes it much easier to reason about things like container isolation, resource
limits, pod security policies, and why certain container escape vulnerabilities exist. This project gave me a much stronger foundation for understanding how
distributed workloads are actually isolated from each other at the OS level.

## License

GPLv3 — see [https://www.gnu.org/licenses/gpl-3.0.en.html](https://www.gnu.org/licenses/gpl-3.0.en.html)

## Credits

Based on the original work by [Lizzie Dixon](https://blog.lizzie.io/linux-containers-in-500-loc.html).


