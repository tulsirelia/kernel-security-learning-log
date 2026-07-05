# Linux Kernel & OS Internals — A Working Notebook

---

## How to use this book

This is written to be read **in order**, chapter by chapter, like a textbook — because each chapter assumes you know the one before it. Every chapter has:
- **Concept** — the mental model, explained plainly
- **Mechanics** — how it actually works under the hood
- **Hands-on** — real commands/code you can run on your machine or a VM
- **Why it matters for your project** — tying it back to kernel security / eBPF / LKMs

Recommended setup before you start: a Linux VM (Ubuntu or Debian, NOT WSL for kernel-building chapters — WSL2's kernel is a custom Microsoft build and won't let you compile/load your own modules cleanly). VirtualBox or a cheap cloud VM (₹100-200 of DigitalOcean/AWS credit) works fine. Give it at least 4GB RAM and 25GB disk if you plan to compile a full kernel.

```bash
# sanity check once your VM is up
uname -a
cat /etc/os-release
gcc --version
make --version
```

---

# Chapter 1 — Linux Architecture

## Concept

An operating system's job is to sit between "dumb hardware" and "programs that want to do useful things" and provide two things:
1. **Abstraction** — programs shouldn't need to know if storage is an SSD or a network drive; they just call `read()`.
2. **Arbitration** — 200 processes want the CPU and 16GB of RAM; the OS decides who gets what, when.

Linux does this with a classic **monolithic kernel** design (unlike microkernels like QNX or seL4, where most services run as separate processes). "Monolithic" means the core services — process scheduling, memory management, filesystems, networking, device drivers — all run in **one address space, at the highest privilege level**. This is a deliberate performance trade-off: fewer context switches and IPC calls, at the cost of a bigger trusted-computing base (this fact alone is *the* reason kernel security is a real discipline — a bug in one driver can compromise the entire system, not just one subsystem).

## Mechanics: Privilege Rings & Kernel/User Space

x86_64 CPUs support 4 privilege rings (0=most privileged, 3=least). Linux only uses two:

```
Ring 0 (kernel mode)   →  Kernel: full hardware access, all instructions allowed
Ring 3 (user mode)     →  Everything else: your apps, shells, browsers
```

This split is enforced **by the CPU itself**, not just convention — user-mode code that tries to execute a privileged instruction (e.g., disabling interrupts, touching arbitrary physical memory) causes a hardware trap (`General Protection Fault`). This is why user processes can't just "reach in" and corrupt kernel memory directly — they must go through a controlled gate: the **system call**.

```
        USER SPACE (Ring 3)
   ┌───────────────────────────┐
   │  your app, libc, bash...  │
   └─────────────┬─────────────┘
                 │  syscall (trap into ring 0)
   ┌─────────────▼─────────────┐
   │        KERNEL SPACE (Ring 0)
   │  ┌───────────┬──────────┬───────────┬────────────┐
   │  │ Process   │ Memory   │ VFS /     │ Network    │
   │  │ Scheduler │ Manager  │ Filesystems│ Stack      │
   │  ├───────────┴──────────┴───────────┴────────────┤
   │  │        Device Drivers (incl. LKMs you load)     │
   │  └──────────────────────────────────────────────┘
   └─────────────┬─────────────────────────────────────┘
                 │
        ┌────────▼────────┐
        │    HARDWARE      │
        └──────────────────┘
```

Each process gets the *illusion* of its own private address space (via virtual memory / paging — hardware MMU + kernel-managed page tables). The kernel's portion of that address space is mapped identically into every process (though not always accessible from user mode — see KPTI/Meltdown mitigations below), which is *why* a syscall can be a fast mode-switch instead of a full context switch.

## Key subsystems (you'll meet all of these again)

| Subsystem | Job | Where it lives in source |
|---|---|---|
| Process scheduler | Decides which task runs on which CPU, when | `kernel/sched/` |
| Memory manager (mm) | Virtual memory, paging, page cache, `slab`/`slub` allocators | `mm/` |
| VFS (Virtual Filesystem) | Uniform interface over ext4, xfs, procfs, etc. | `fs/` |
| Network stack | Sockets, TCP/IP, netfilter | `net/` |
| Device drivers | Talk to actual hardware (and where LKMs plug in) | `drivers/` |
| Security modules (LSM) | SELinux, AppArmor, seccomp hooks | `security/` |

## A security-relevant nuance: Meltdown/KPTI

Historically the *entire* kernel address space was mapped into every user process (just marked non-executable/non-readable from ring 3) purely for speed. The 2018 **Meltdown** vulnerability exploited speculative execution to read that "hidden" kernel memory from user mode anyway. The fix, **KPTI (Kernel Page Table Isolation)**, now maintains *separate* page tables for user vs. kernel mode, at some performance cost on syscalls. This is a good real example of "architecture decision → decades later, becomes a CVE" — exactly the kind of thing to have in your back pocket when your prof asks about kernel attack surface.

## Why it matters for your project

Every LKM you write executes in Ring 0, with **zero memory protection from the rest of the kernel**. A bug in your module isn't a segfault-and-restart like a Ring-3 app — it can panic the whole box, corrupt unrelated kernel data structures, or (worse, for a security project) open a genuine privilege-escalation hole. Kernel security work is fundamentally about minimizing and auditing this Ring-0 attack surface — which is exactly why LSM hooks, eBPF (sandboxed, verified code instead of raw modules), and seccomp exist: they're ways to get kernel-level enforcement *without* giving arbitrary code full Ring-0 power.

---

# Chapter 2 — Kernel Source Tree

## Concept

The Linux kernel source is one giant, self-contained repo (no external build deps beyond a compiler + a few tools). Get familiar with the top-level layout — you'll navigate it constantly once you start writing LKMs or tracing syscalls.

## Hands-on: get the source

```bash
# Option A: matches your distro's running kernel (recommended for module work)
apt-get source linux-image-$(uname -r)   # Debian/Ubuntu
# Option B: mainline, for reading/learning
git clone --depth 1 https://github.com/torvalds/linux.git
cd linux
```

## Mechanics: top-level directories

```
linux/
├── arch/          # architecture-specific code: arch/x86, arch/arm64, arch/riscv...
│                  #   ⚠ most of the kernel is architecture-INDEPENDENT; only low-level
│                  #   bits (boot, page tables, syscall entry) live per-arch
├── kernel/        # core kernel: scheduler, signals, fork/exec, cgroups, time
├── mm/            # memory management
├── fs/            # filesystems + the VFS layer
├── net/           # networking stack (net/core, net/ipv4, net/netfilter...)
├── drivers/        # device drivers — BY FAR the largest directory (>60% of LOC)
├── include/       # public headers
│   ├── linux/     #   generic kernel headers (linux/module.h, linux/syscalls.h...)
│   ├── uapi/      #   headers shared with USER SPACE (this boundary matters!)
│   └── asm-generic/
├── security/      # LSM framework, SELinux, AppArmor, Yama, seccomp
├── crypto/        # kernel crypto API (relevant if your UAV work touches in-kernel crypto)
├── init/          # kernel startup, main.c (start_kernel())
├── scripts/       # build helper scripts, Kconfig tooling, checkpatch.pl
├── tools/         # userspace tools shipped WITH the kernel: perf, bpf tools, etc.
│   └── testing/selftests/bpf/   # ← you'll live here once you start eBPF work
├── Documentation/ # increasingly good, especially Documentation/admin-guide, /bpf
├── Makefile       # the top-level Kbuild entrypoint (Chapter 4)
└── Kconfig        # root of the config option tree
```

## The `include/uapi` boundary — important for syscall work

Anything in `include/uapi/` is a **contract with user space** — struct layouts, constants, ioctl numbers that user programs `#include` directly (via glibc's copies). Change something in `include/linux/` freely; change something in `include/uapi/` and you can break every compiled binary that depends on it. When you implement a custom syscall (Chapter 7), your struct definitions for user-facing data need to go here, not in `include/linux/`.

## Hands-on: finding your way around

```bash
# find where a symbol/function is defined
grep -rn "asmlinkage long sys_open" fs/

# use cscope / ctags for real navigation (worth 10 minutes of setup)
sudo apt install cscope exuberant-ctags
cscope -R -b     # build the cross-reference database
cscope -d        # interactive lookup

# or, cleanest for this scale of codebase:
sudo apt install universal-ctags
ctags -R .
```

## Why it matters for your project

For an LKM/syscall/eBPF project you will mostly touch: `include/uapi/`, `kernel/`, `arch/x86/entry/syscalls/`, `security/`, and `tools/testing/selftests/bpf/`. You almost never need to touch `drivers/` or `fs/` unless your project specifically targets a driver or filesystem attack surface. Knowing the map means you stop grepping blindly.

---

# Chapter 3 — GNU Make

## Concept

Before Kbuild makes sense, you need Make itself. Make's whole job: given a set of **targets**, their **dependencies**, and **rules** to turn dependencies into targets, only rebuild what's actually stale (based on file modification timestamps). This is *the* reason kernel rebuilds after a one-line change take seconds, not hours.

## Mechanics: the anatomy of a rule

```makefile
target: prerequisites
	recipe   # <-- MUST be a TAB, not spaces. This trips up everyone once.
```

```makefile
# a tiny real example
hello: hello.o
	gcc -o hello hello.o

hello.o: hello.c
	gcc -c hello.c

clean:
	rm -f hello hello.o
```

Run `make` → it walks the dependency graph, checks timestamps, rebuilds only what changed. Run `make clean` → runs that rule unconditionally (it's a "phony" target — doesn't correspond to a real file).

## Variables & pattern rules (what Kbuild is built from)

```makefile
CC = gcc
CFLAGS = -Wall -O2

%.o: %.c        # pattern rule: "any .o comes from a .c of the same name"
	$(CC) $(CFLAGS) -c $< -o $@
```

Automatic variables you'll see constantly in kernel Makefiles:
- `$@` — the target
- `$<` — the first prerequisite
- `$^` — all prerequisites
- `$(CC)`, `$(LD)`, `$(AR)` — toolchain variables, overridable from the command line (`make CC=clang`)

## Conditionals & includes

```makefile
ifeq ($(DEBUG),1)
    CFLAGS += -g -DDEBUG
endif

include config.mk    # pull in another makefile — Kbuild uses this HEAVILY
```

## Hands-on

```bash
mkdir make_practice && cd make_practice
cat > hello.c << 'EOF'
#include <stdio.h>
int main(void) { printf("hello, make\n"); return 0; }
EOF
cat > Makefile << 'EOF'
CC = gcc
CFLAGS = -Wall

hello: hello.o
	$(CC) -o $@ $^

%.o: %.c
	$(CC) $(CFLAGS) -c $< -o $@

.PHONY: clean
clean:
	rm -f hello *.o
EOF
make        # builds
make clean  # tears down
```

## Why it matters for your project

Kernel Makefiles (next chapter) are just GNU Make with a *large* pile of conventions layered on top (the Kbuild "language"). If `$(obj)`, `$(src)`, `obj-y`, `obj-m` look like magic later, remember: they're just Make variables set by a parent Makefile, expanded through pattern rules exactly like above. Nothing about it is a different tool — it's the same Make, disciplined.
## In simple terms
The kernel's build system (Kbuild, which you'll meet next) is not a different tool — it's regular Make, just with a big pile of conventions and pre-written rules layered on top, written by kernel developers over 30 years. So later when you see kernel-specific-looking things like obj-y or $(obj), don't panic thinking it's some special new language — it's just more variables and pattern rules, working exactly the way you just saw above, just applied at a much bigger scale.

---

# Chapter 4 — Kernel Makefiles (Kbuild)

## Concept

Kbuild is the kernel's own build system, layered on top of plain GNU Make. It solves a problem plain Make handles badly at scale: the kernel has tens of thousands of source files, most of which shouldn't even be *compiled* depending on which config options (`CONFIG_*`) are enabled. Kbuild = Make + Kconfig (the config-option system) working together.

## Mechanics: the three-Makefile hierarchy

```
Linux/Makefile                 ← top-level, arch-independent, orchestrates everything
arch/x86/Makefile               ← arch-specific flags, boot rules
somedir/Makefile                ← every subdirectory has one of these, tiny and declarative
```

A typical subdirectory Makefile is almost nothing but variable assignments:

```makefile
# drivers/example/Makefile
obj-y   += always_built.o
obj-m   += module_if_enabled.o
obj-$(CONFIG_MY_FEATURE) += feature.o
feature-y := feature_core.o feature_helper.o   # multi-file "composite" object
```

Read this as:
- `obj-y` → compiled **into the kernel image** (built-in, not a module)
- `obj-m` → compiled as a **loadable module** (`.ko` file)
- `obj-$(CONFIG_X)` → the value of `CONFIG_X` (from your `.config`) is either `y`, `m`, or empty — so this line dynamically decides built-in vs. module vs. excluded, all in one line
- `feature-y := a.o b.o` → tells Kbuild that the *composite object* `feature.o` is actually linked from `feature_core.o` + `feature_helper.o`

## Where `$(obj)` and `$(src)` come from

Kbuild recursively **descends** into each directory listed by its parent (`obj-y += subdir/`), and re-invokes Make with `$(obj)` and `$(src)` set to that subdirectory. This is why kernel Makefiles never hardcode paths — they're relative and re-usable no matter how deep the tree goes.

## Kconfig — where the `CONFIG_*` values come from

Every directory can also carry a `Kconfig` file describing options as a menu tree:

```
config MY_FEATURE
    tristate "Enable my cool feature"
    depends on NET
    help
      This turns on the cool feature. Say M to build as a module.
```

`tristate` means the option can be `y` (built-in), `m` (module), or `n` (excluded) — this is precisely what feeds `obj-$(CONFIG_MY_FEATURE)` above. Run `make menuconfig` and you're editing exactly this tree interactively, producing a `.config` file.

## Hands-on: read a real one

```bash
cd linux/drivers/char
cat Makefile | head -30
# then trace one entry:
grep -rn "CONFIG_TTY" Kconfig | head
```

## Why it matters for your project

When you write an out-of-tree LKM (Chapter 6), you'll write a *tiny* Kbuild-style Makefile yourself that invokes the *kernel's own* build system against your single file — this is the "`make -C /lib/modules/$(uname -r)/build M=$(pwd) modules`" incantation you'll type constantly. Understanding *why* that command works (it's re-entering the kernel's Kbuild tree, treating your directory as `$(M)`, an external module dir) will save you a lot of copy-paste-without-understanding.

---

# Chapter 5 — Kernel Compilation

## Concept

Compiling a full kernel is "just" running Kbuild top to bottom: resolve config → compile everything marked `obj-y`/`obj-m` for that config → link into a bootable image → build the module tree separately. The main reason to do this yourself (rather than trust your distro's kernel) for a security project: you need **debug symbols**, **specific config options enabled** (e.g., `CONFIG_KPROBES`, `CONFIG_BPF_SYSCALL`, `CONFIG_DEBUG_INFO`), or you're testing a patched kernel.

## Hands-on: full build, step by step

```bash
git clone --depth 1 https://github.com/torvalds/linux.git
cd linux

# 1. Get a baseline config — don't start from scratch
cp /boot/config-$(uname -r) .config
make olddefconfig     # fills in new options with sane defaults vs. your old config

# 2. (Optional but recommended for your project) enable security/debug relevant options
make menuconfig
#   → General setup -> enable "Kernel debugging"
#   → Kernel hacking -> CONFIG_DEBUG_INFO, CONFIG_KGDB (Ch. 8)
#   → enable CONFIG_BPF_SYSCALL, CONFIG_BPF_JIT (needed for Ch. 10, eBPF)
#   → enable CONFIG_KPROBES, CONFIG_KPROBES_ON_FTRACE

# 3. Build. -j$(nproc) parallelizes across all cores (kernel builds are CPU-bound)
make -j$(nproc)

# 4. Build modules separately, then install everything
sudo make modules_install
sudo make install       # installs vmlinuz + updates bootloader (grub) entries

# 5. Reboot into it
sudo reboot
# at grub, or after boot:
uname -r    # confirm you're running your freshly built kernel
```

## Mechanics: what actually happens during `make`

1. **Kconfig resolution** — `.config` is read; every `obj-$(CONFIG_X)` line across the whole tree is resolved to `y`/`m`/nothing.
2. **Recursive descent** — Kbuild walks every directory listed via `obj-y += subdir/`, compiling each `.c` → `.o` per the pattern rules from Chapter 4.
3. **Built-in objects get archived** into per-directory `built-in.a`, which get progressively linked upward until one giant `vmlinux` (uncompressed kernel ELF binary) exists at the top.
4. **`vmlinux` → `bzImage`** — compressed + wrapped with a small real-mode bootstrap stub so BIOS/UEFI can load it. This is the file the bootloader actually boots.
5. **Modules** (`obj-m` entries) are linked **separately** into individual `.ko` files, not part of `vmlinux` at all — this is *why* modules can be loaded/unloaded at runtime (Chapter 6): they were never statically linked into the image in the first place.

```
    Kconfig (.config)
          │
          ▼
   resolve obj-y / obj-m for every directory
          │
   ┌──────┴───────┐
   ▼               ▼
built-in (obj-y)   modules (obj-m)
   │                   │
   ▼                   ▼
 vmlinux            foo.ko, bar.ko ...
   │
   ▼
 bzImage  (compressed, bootable)
```

## Common pain points (save yourself the debugging time)

- **Missing headers/tools**: `make` will complain and tell you exactly what package it needs (`libssl-dev`, `libelf-dev`, `flex`, `bison`, `bc` are the common ones for Debian/Ubuntu).
- **Disk space**: a full build + debug symbols can eat 15-20GB.
- **Secure Boot**: if your VM/machine has Secure Boot enabled, your self-built (unsigned) kernel may refuse to boot — either disable Secure Boot in firmware settings for your dev VM, or sign your build (a rabbit hole you don't need for a learning project).

## Why it matters for your project

You almost never need to rebuild and boot an *entire* custom kernel just to write an LKM or eBPF program — those load into your **existing running kernel**. You *do* need a from-scratch build if: your prof's project involves patching kernel source directly (not just modules/eBPF), or you need specific debug configs (KGDB, kprobes tracing) that your distro kernel wasn't built with.

---

# Chapter 6 — Kernel Modules (LKMs)

## Concept

A Loadable Kernel Module (LKM) is compiled code that gets **linked into the running kernel's address space at runtime** — no reboot needed. This is how most drivers, filesystems, and (relevantly for you) a lot of rootkits/security tooling get into the kernel without a custom kernel build. An LKM is Ring 0 code you can insert and remove on demand; that power is exactly why LKM-based rootkits are a classic attack technique, and why kernel security courses spend real time here.

## Mechanics: the minimal LKM

```c
// hello_lkm.c
#include <linux/module.h>
#include <linux/kernel.h>
#include <linux/init.h>

static int __init hello_init(void)
{
    printk(KERN_INFO "hello_lkm: loaded\n");
    return 0;                 // 0 = success; nonzero aborts the load
}

static void __exit hello_exit(void)
{
    printk(KERN_INFO "hello_lkm: unloaded\n");
}

module_init(hello_init);
module_exit(hello_exit);

MODULE_LICENSE("GPL");            // non-GPL modules can't call GPL-only kernel symbols
MODULE_AUTHOR("Tulsi Patel");
MODULE_DESCRIPTION("Minimal example LKM");
```

- `__init` / `__exit` mark these functions to be freed from memory after use (init) or excluded entirely if the module is built-in rather than loadable — a small but real memory optimization.
- `printk` is the kernel's `printf` — output goes to the kernel ring buffer, viewable via `dmesg`, not to a terminal `stdout` (there isn't one in kernel context).
- `MODULE_LICENSE("GPL")` isn't cosmetic: many kernel symbols are exported via `EXPORT_SYMBOL_GPL()` and are **only callable from GPL-licensed modules** — the kernel checks this string at link time.

## The Makefile for an out-of-tree module

```makefile
obj-m += hello_lkm.o

all:
	make -C /lib/modules/$(shell uname -r)/build M=$(PWD) modules

clean:
	make -C /lib/modules/$(shell uname -r)/build M=$(PWD) clean
```

This re-enters the *kernel's own* Kbuild tree (`-C /lib/modules/.../build` is a symlink to your kernel headers) and tells it: "treat my current directory as an external module directory (`M=$(PWD))`, and build just the `obj-m` targets in it." Everything from Chapter 4 applies — it's the exact same `obj-m` mechanism, just pointed at your file instead of an in-tree driver.

## Hands-on: build, load, inspect, unload

```bash
sudo apt install build-essential linux-headers-$(uname -r)
make
sudo insmod hello_lkm.ko
dmesg | tail                 # see your printk output
lsmod | grep hello_lkm       # confirm it's loaded
sudo rmmod hello_lkm
dmesg | tail                 # see the unload message
```

## Passing parameters & exporting symbols

```c
static int count = 1;
module_param(count, int, 0644);   // /sys/module/hello_lkm/parameters/count
MODULE_PARM_DESC(count, "number of times to greet");
```

```bash
sudo insmod hello_lkm.ko count=5
```

## Why it matters for your project (and the honest security caveat)

Almost every classic **kernel rootkit technique** is "a normal LKM mechanism, abused":
- Hooking the syscall table (Chapter 7) to hide files/processes
- Hiding the module itself from `lsmod` by unlinking it from the kernel's internal module list
- Hooking VFS `readdir`/`getdents` to hide directory entries

Understanding *legitimate* LKM writing is the prerequisite for understanding *detection* — which is almost certainly closer to what your prof's kernel-security project actually wants (e.g., detecting a hidden/hooked module via `kallsyms` cross-checks, or via eBPF-based integrity monitoring in Chapter 10, rather than writing an offensive rootkit).

---

# Chapter 7 — System Calls

## Concept

A system call is the **only sanctioned gate** from user space (Ring 3) into kernel space (Ring 0). `open()`, `read()`, `fork()`, `mmap()` — every one of these libc functions is a thin wrapper that ultimately executes a special CPU instruction to trap into the kernel, look up a numbered table, and jump to the corresponding kernel function.

## Mechanics: from libc call to kernel function

```
your code: read(fd, buf, len)
       │
       ▼
glibc wrapper: sets up registers, executes `syscall` instruction (x86_64)
       │             (older 32-bit: `int 0x80`; ARM64: `svc #0`)
       ▼
CPU traps into Ring 0, jumps to a fixed kernel entry point
       │
       ▼
kernel: entry point looks up syscall NUMBER in the syscall table
       │
       ▼
kernel: dispatches to sys_read() / do_read() etc.
       │
       ▼
kernel: copies result back to user buffer via copy_to_user(), returns
```

The syscall table on x86_64 lives (in source) at `arch/x86/entry/syscalls/syscall_64.tbl` — literally a text file mapping numbers to names:

```
# arch/x86/entry/syscalls/syscall_64.tbl (excerpt)
0    common   read     sys_read
1    common   write    sys_write
2    common   open     sys_open
59   common   execve   sys_execve
```

This is a big security-relevant fact: **that table is a fixed-size array of function pointers in kernel memory at runtime.** Overwriting an entry (classic rootkit technique) redirects every future call to that syscall system-wide — which is exactly why the table is normally read-only (mapped via `set_memory_ro()`), and why detecting table tampering is a real kernel-security technique (compare live pointers against `/proc/kallsyms` expected addresses, or use eBPF/kprobes to monitor instead of hooking directly).

## Anatomy of a syscall implementation

```c
// simplified real example, fs/read_write.c
SYSCALL_DEFINE3(read, unsigned int, fd, char __user *, buf, size_t, count)
{
    return ksys_read(fd, buf, count);
}
```

`SYSCALL_DEFINE3` is a macro that expands into the correct `asmlinkage long sys_read(...)` signature plus boilerplate for argument marshalling — you rarely write raw `asmlinkage` functions by hand anymore. Note `char __user *buf` — the `__user` annotation is a **static-analysis marker** (checked by `sparse`) flagging that this pointer comes from user space and must never be dereferenced directly; you must go through `copy_from_user()`/`copy_to_user()`, which validate the address range and handle page faults safely.

## Hands-on: adding your own syscall (mainline kernel, for learning)

```c
// 1. add to arch/x86/entry/syscalls/syscall_64.tbl:
451   common   hello_syscall   sys_hello_syscall

// 2. declare prototype in include/linux/syscalls.h:
asmlinkage long sys_hello_syscall(void);

// 3. implement it, e.g. in kernel/hello_syscall.c:
#include <linux/syscalls.h>
#include <linux/kernel.h>

SYSCALL_DEFINE0(hello_syscall)
{
    printk(KERN_INFO "hello_syscall: called by pid %d\n", current->pid);
    return 0;
}

// 4. add kernel/hello_syscall.o to kernel/Makefile's obj-y list
// 5. rebuild kernel (Chapter 5), reboot into it
```

```c
// user-space caller (no glibc wrapper exists for your custom syscall, so use syscall(2) directly)
#include <stdio.h>
#include <sys/syscall.h>
#include <unistd.h>

int main(void) {
    long ret = syscall(451);
    printf("returned: %ld\n", ret);
    return 0;
}
```

## Why it matters for your project

Custom syscalls require a **full kernel rebuild + reboot** (unlike LKMs/eBPF), which makes them clumsy for iterative security research — this is precisely *why* modern kernel security tooling (auditd, Falco, eBPF-based EDR) avoids adding new syscalls and instead **observes existing ones** via tracepoints, kprobes, or eBPF programs attached to syscall entry/exit (Chapter 10). Understanding the raw syscall mechanism, though, is what lets you understand what those tracing tools are actually hooking into.

---

# Chapter 8 — Kernel Debugging

## Concept

Debugging kernel code is fundamentally harder than user-space debugging because a crash can take down the *entire machine* — there's no OS left underneath to catch the fault and print a backtrace to your terminal. So kernel debugging relies on a mix of: passive logging, static analysis, and (for deep bugs) a second machine/VM acting as a debugger over a serial/network link.

## Tier 1: `printk` and `dmesg` (your daily driver)

```c
printk(KERN_INFO "my_module: value=%d\n", value);
printk(KERN_ERR  "my_module: failed to allocate memory\n");
```

```bash
dmesg -w              # follow the kernel ring buffer live, like `tail -f`
dmesg -T | grep my_module
cat /proc/kmsg        # raw stream (needs root)
```

Log levels (`KERN_EMERG` down to `KERN_DEBUG`) control whether a message also gets echoed to the console immediately vs. just buffered — tune via `/proc/sys/kernel/printk`.

## Tier 2: `ftrace` — the built-in kernel tracer

`ftrace` can trace **every function call** in the kernel with almost no overhead, without needing a second machine. Massively useful for "what actually happened, in order" questions.

```bash
cd /sys/kernel/debug/tracing
echo function > current_tracer
echo do_sys_open > set_ftrace_filter    # trace just this function
echo 1 > tracing_on
cat trace                                # see the captured calls
echo 0 > tracing_on
```

## Tier 3: `kprobes` — dynamic breakpoints anywhere in the kernel

A kprobe lets you attach a handler to (almost) any kernel instruction address at runtime, without recompiling — this is the mechanism a lot of security monitoring tools (and your eBPF work in Ch. 10) build on top of.

```c
#include <linux/kprobes.h>

static struct kprobe kp = { .symbol_name = "do_sys_open" };

static int handler_pre(struct kprobe *p, struct pt_regs *regs)
{
    printk(KERN_INFO "do_sys_open called, pid=%d\n", current->pid);
    return 0;
}

static int __init kp_init(void) {
    kp.pre_handler = handler_pre;
    return register_kprobe(&kp);
}
static void __exit kp_exit(void) { unregister_kprobe(&kp); }

module_init(kp_init);
module_exit(kp_exit);
MODULE_LICENSE("GPL");
```

## Tier 4: KGDB — a real interactive debugger, over serial/network

For genuinely hard bugs (memory corruption, race conditions), KGDB lets you attach `gdb` from a *second* machine to a kernel running on a VM/target machine, set breakpoints, step through kernel code, and inspect memory — exactly like debugging a userspace program, just across a connection.

```bash
# on the target kernel's boot line (grub), add:
kgdboc=ttyS0,115200 kgdbwait

# on the debugging host, connect over the serial line:
gdb ./vmlinux
(gdb) target remote /dev/ttyS0
(gdb) break do_sys_open
(gdb) continue
```

(For VM-to-VM setups this is often done over a virtual serial port; QEMU/VirtualBox both support this cleanly — worth setting up once as a lab exercise even if you don't need it daily.)

## Tier 5: `panic()`/Oops analysis

When something does go wrong badly enough, the kernel prints an **Oops** (non-fatal, tries to continue) or a full **panic** (fatal) with a **call stack**. Learning to read these is a core skill:

```
BUG: unable to handle kernel NULL pointer dereference at 0000000000000008
RIP: 0010:my_broken_function+0x12/0x40 [my_module]
Call Trace:
 caller_function+0x55/0x80
 do_one_initcall+0x50/0x230
 ...
```

`RIP` tells you exactly which instruction faulted; the call trace tells you the chain that got you there. `addr2line` or `scripts/decode_stacktrace.sh` (shipped with the kernel source) can resolve these addresses back to source lines if you have matching debug symbols.

## Why it matters for your project

For a security project you'll mostly live in Tier 1 (printk) and Tier 3 (kprobes) — kprobes in particular are the direct ancestor of eBPF's `kprobe`/`kretprobe` program types (Chapter 10), so getting comfortable with raw kprobes first will make eBPF click much faster.

---

# Chapter 9 — Kernel Security

## Concept

Everything in Chapters 1-8 was "how the kernel works." This chapter is "how the kernel decides who's allowed to do what" — the actual mechanisms your prof's project is almost certainly asking you to understand, extend, or monitor.

## Layer 1: Discretionary Access Control (DAC) — the baseline

The classic Unix model: every file has an owner, group, and permission bits (`rwx` × owner/group/other), checked at every `open()`/`write()` etc. **Discretionary** because the *owner* decides who else gets access (`chmod`, `chown`) — there's no central policy overriding them. Root (`uid 0`) bypasses almost all of it, which is precisely DAC's weakness: one root-owned bug, and DAC provides zero defense in depth.

## Layer 2: Capabilities — splitting up "root"

Historically root was all-or-nothing. **Capabilities** split root's power into ~40 discrete privileges, so a process can have exactly the power it needs:

```bash
getcap /usr/bin/ping
# /usr/bin/ping cap_net_raw=ep     ← ping needs raw sockets, not full root

# view a process's current capability set
cat /proc/$$/status | grep Cap
```

Common ones: `CAP_NET_ADMIN` (network config), `CAP_SYS_ADMIN` (a worryingly broad grab-bag — a common target in privilege escalation research), `CAP_SYS_MODULE` (load kernel modules — exactly the capability an attacker wants if they're trying to load a malicious LKM from Chapter 6).

## Layer 3: LSM (Linux Security Modules) — the real enforcement framework

LSM is a **hook framework** built into core kernel code: at every security-relevant decision point (file open, socket creation, process exec, ptrace, etc.), the kernel calls out to a registered LSM hook *before* proceeding, and the hook can allow or deny.

```c
// simplified illustration of the pattern used throughout the kernel, e.g. fs/open.c
int ret = security_file_open(file);   // ← this is the LSM hook call
if (ret)
    return ret;                        // denied — LSM said no
// ... proceed with the actual open
```

`security_file_open()` is defined in `security/security.c` and dispatches to whichever LSM(s) are active: **SELinux** (label-based mandatory access control — every file/process gets a security *context*, and policy rules decide what contexts can touch what), **AppArmor** (simpler, path-based confinement profiles), **Yama** (restricts `ptrace` scope — directly relevant to anti-debugging/anti-injection hardening), or **seccomp** (restricts which *syscalls* a process may even invoke, independent of file/label logic).

```bash
# check what's active on your system
cat /sys/kernel/security/lsm     # e.g. "lockdown,capability,yama,apparmor,bpf"

# SELinux status
sestatus
# AppArmor status
sudo aa-status
```

Notice `bpf` in that list — as of recent kernels, **eBPF programs can themselves act as LSM hooks** (`BPF_PROG_TYPE_LSM`), meaning you can write security policy in eBPF instead of a monolithic module. This is the direct bridge into Chapter 10 and very likely the actual mechanism your prof wants for the project.

## Layer 4: seccomp — syscall filtering per-process

```c
#include <seccomp.h>
scmp_filter_ctx ctx = seccomp_init(SCMP_ACT_KILL);      // default: kill on any syscall
seccomp_rule_add(ctx, SCMP_ACT_ALLOW, SCMP_SYS(read), 0);
seccomp_rule_add(ctx, SCMP_ACT_ALLOW, SCMP_SYS(write), 0);
seccomp_rule_add(ctx, SCMP_ACT_ALLOW, SCMP_SYS(exit_group), 0);
seccomp_load(ctx);
// from this point, this process can ONLY call read/write/exit_group — everything
// else (fork, execve, socket...) gets killed by the kernel immediately.
```

This is exactly how container runtimes (Docker, containerd) and browser sandboxes (Chrome's renderer process) limit blast radius — restrict a compromised process to the *minimum* syscall surface it needs.

## Putting it together: defense in depth

```
Attacker gets code execution in a user process
        │
        ▼
   seccomp filter?  ──deny──► killed
        │ allow
        ▼
   DAC permission check? ──deny──► EACCES
        │ allow
        ▼
   LSM hook (SELinux/AppArmor/eBPF-LSM)? ──deny──► EPERM
        │ allow
        ▼
   capability check (e.g. CAP_SYS_ADMIN)? ──deny──► EPERM
        │ allow
        ▼
   operation proceeds
```

Each layer is a genuinely independent barrier — a real kernel-hardening project (yours) typically means adding or strengthening **one of these layers**, most practically the LSM/eBPF layer since it doesn't require kernel recompilation or reboot.

## Why it matters for your project

If your prof's framing is "kernel security" for a DRDO-adjacent / indigenous-defence context, the realistic, demonstrable deliverable is very likely one of: (a) an eBPF-based syscall/LSM monitor that detects anomalous behavior (e.g., unexpected `execve` chains, unauthorized `CAP_SYS_MODULE` usage, module-load attempts), or (b) a seccomp/LSM policy hardening a specific service. Chapter 10 gives you the concrete tooling for (a).

---

# Chapter 10 — eBPF (extended Berkeley Packet Filter)

*(Your prof explicitly asked for this — it's the modern default tool for exactly the kind of kernel security work described in Chapter 9, so treat this as the capstone chapter.)*

## Concept

eBPF lets you run **sandboxed, verified programs inside the kernel**, attached to hooks (syscalls, network events, function entry/exit, LSM decision points) — **without writing an LKM, without recompiling the kernel, and without the "one bug panics the whole box" risk of Chapter 6's LKMs.** This is the single biggest reason eBPF has replaced kernel modules for most modern observability/security tooling (Falco, Cilium, Tetragon, Datadog's kernel-level tracing all run on it).

The key architectural trick: you don't get to run *arbitrary* code in the kernel. You write in a **restricted C subset**, compile to **eBPF bytecode**, and the kernel's **verifier** statically proves your program will terminate, won't read out-of-bounds memory, and won't crash the kernel — *before* it's allowed to load. This verifier is the whole reason eBPF is considered safe enough to expose to (privileged, but non-root-shell) users at all.

## Mechanics: the pipeline

```
   your_program.bpf.c   (restricted-C, uses BPF helper functions)
          │  compile with clang -target bpf
          ▼
   eBPF bytecode (.o file)
          │  bpf() syscall loads it
          ▼
   KERNEL VERIFIER  ──rejects──►  "invalid: unbounded loop" etc.
          │ passes
          ▼
   JIT-compiled to native machine code
          │
          ▼
   attached to a HOOK:
     - kprobe/kretprobe   (any kernel function, like Ch.8's kprobes)
     - tracepoint          (stable, kernel-maintained trace points)
     - syscall entry/exit  (exactly Ch.7's syscall table, but observed not hooked)
     - XDP / TC            (network packet processing, very high performance)
     - LSM hook            (BPF_PROG_TYPE_LSM — literally Ch.9's security_* functions)
          │
          ▼
   communicates results to user space via a MAP (shared kernel/userspace data structure)
```

## Hands-on: the simplest useful example — trace `execve`

Modern eBPF development uses **libbpf + CO-RE** (Compile Once, Run Everywhere — avoids needing kernel headers matching every target machine exactly). Toolchain setup:

```bash
sudo apt install clang llvm libbpf-dev linux-tools-common linux-tools-$(uname -r) bpftool
```

```c
// trace_exec.bpf.c  — kernel-side eBPF program
#include <vmlinux.h>
#include <bpf/bpf_helpers.h>
#include <bpf/bpf_tracing.h>

SEC("tp/sched/sched_process_exec")
int trace_exec(struct trace_event_raw_sched_process_exec *ctx)
{
    pid_t pid = bpf_get_current_pid_tgid() >> 32;
    char comm[16];
    bpf_get_current_comm(&comm, sizeof(comm));
    bpf_printk("EXEC: pid=%d comm=%s\n", pid, comm);
    return 0;
}

char LICENSE[] SEC("license") = "GPL";
```

```bash
clang -O2 -g -target bpf -c trace_exec.bpf.c -o trace_exec.bpf.o

# quick load-and-attach for testing (production tools use libbpf's skeleton API instead)
sudo bpftool prog load trace_exec.bpf.o /sys/fs/bpf/trace_exec
sudo bpftool prog tracelog     # see bpf_printk() output live — like dmesg, but for eBPF
```

For a real project you'd use `bpftool gen skeleton` to generate a matching user-space loader in C (or use Python's `bcc` library / Rust's `aya` for faster iteration than raw libbpf).

## Maps — how eBPF talks to user space

A **BPF map** is a kernel-resident key/value store both your eBPF program and a user-space program can read/write — this is how you get data (counters, flagged PIDs, event logs) *out* of the kernel without printk/tracelog:

```c
struct {
    __uint(type, BPF_MAP_TYPE_HASH);
    __uint(max_entries, 1024);
    __type(key, u32);      // pid
    __type(value, u64);    // count
} exec_counts SEC(".maps");
```

```c
// inside your eBPF program:
u32 pid = bpf_get_current_pid_tgid() >> 32;
u64 *count = bpf_map_lookup_elem(&exec_counts, &pid);
u64 new_count = count ? *count + 1 : 1;
bpf_map_update_elem(&exec_counts, &pid, &new_count, BPF_ANY);
```

## The LSM program type — eBPF as security policy

This is the direct link to Chapter 9's LSM framework:

```c
SEC("lsm/bprm_check_security")
int BPF_PROG(check_exec, struct linux_binprm *bprm)
{
    // inspect bprm->filename, bprm->file etc.
    // return -EPERM to DENY the exec, 0 to allow
    if (some_condition_indicating_malicious_exec(bprm))
        return -EPERM;
    return 0;
}
```

Requires the kernel booted with `CONFIG_BPF_LSM=y` and `lsm=...,bpf` on the kernel command line (recall `cat /sys/kernel/security/lsm` from Chapter 9 — `bpf` needs to appear in that list). This lets you write **enforceable, in-kernel security policy in a memory-safe, verified language**, loaded/updated without rebooting — genuinely the state of the art for what your prof is describing.

## Debugging & introspecting eBPF programs

```bash
sudo bpftool prog list                 # every loaded eBPF program, by ID
sudo bpftool prog dump xlated id <N>   # see the verifier-processed bytecode
sudo bpftool map dump id <N>           # inspect a map's live contents
sudo bpftool prog tracelog             # bpf_printk() output stream
```

## Practical libraries worth knowing (in increasing order of "closer to raw kernel")

| Tool | Language | Best for |
|---|---|---|
| **bcc** | Python + embedded C | fastest prototyping, huge library of ready-made tracing tools (`execsnoop`, `opensnoop`) |
| **bpftrace** | its own DSL (awk-like) | one-liners for quick investigation, no compile step |
| **libbpf + CO-RE** | C | production tools, portable across kernel versions |
| **aya** | Rust | modern, memory-safe userspace side, growing security-tooling ecosystem |

Practical starting point: `sudo apt install bpfcc-tools`, then run `sudo execsnoop-bpfcc` and `sudo opensnoop-bpfcc` — these are exactly the kind of "eBPF-based security monitor" your project is likely aiming toward, and reading their (short, readable) source is one of the best ways to see real eBPF-for-security code.

## Why it matters for your project

eBPF gives you the *outcome* kernel security work usually wants — visibility into and control over kernel-level events — **without** the risks of Chapter 6's raw LKMs (verifier-enforced safety) or Chapter 7's syscall additions (no reboot, no kernel patching). If your prof's deliverable is a monitoring/detection system (likely, given the DRDO/indigenous-defence framing of your other projects), eBPF + `BPF_PROG_TYPE_LSM` or tracepoint-based monitoring is almost certainly the intended implementation path, and everything in Chapters 1-9 is the background needed to understand *why* it's built the way it is.

---

# Suggested order of hands-on practice (given everything above)

1. Chapter 3 exercise (plain Make) — 20 minutes, just to get muscle memory
2. Chapter 6: build, load, unload the minimal LKM on a VM — get comfortable with `insmod`/`rmmod`/`dmesg`
3. Chapter 8: add a kprobe to the same VM, watch it fire on a real syscall
4. Chapter 9: run `cat /sys/kernel/security/lsm`, `getcap`, `sestatus`/`aa-status` on your own machine — ground the theory in what's actually enabled around you right now
5. Chapter 10: install `bpfcc-tools`, run `execsnoop-bpfcc` while you open a few programs — watch real eBPF output before writing your own program
6. Only then: write your own `tp/sched/sched_process_exec` tracer (Chapter 10's hands-on section) and extend it toward whatever specific detection logic your prof's project needs
7. Chapter 5 (full kernel compile) is the one step you can genuinely skip until/unless the project specifically requires kernel-source patches rather than LKM/eBPF-level work

