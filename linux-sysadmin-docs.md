# Linux kernel docs — the "System Administrator" reading track

Compiled from `README` lines 98–108, which lists six documents for the System Administrator role. All six are concatenated below, converted from RST to Markdown.

---

## What this collection actually is

First, a framing that the README doesn't give you: **four of the six are index pages** — tables of contents, not content. Only `perf-security.rst` and the prose half of `kernel-parameters.rst` are documents you can sit and read.

So this is a **map, not a syllabus.** Its value is knowing what exists and where, so that at 03:00 you know a document exists for the thing that's broken.

Second, the six sort cleanly into three jobs:

| | Documents | What it controls |
|---|---|---|
| **How you change the system** | Kernel Parameters, Sysctl | Boot-time and run-time tuning |
| **How you see the system** | Tracing, Hardware Monitoring | Software behaviour and physical truth |
| **Who is allowed to see it** | Performance Security | Access control over observation |

With the Admin Guide index sitting above all of them as the master table of contents.

---

## The concepts, document by document

### 1. Admin Guide index — the master map

The top-level index for everything user- and administrator-facing. Its own preamble is refreshingly honest: *"there is, as yet, little overall order or organization here — this material was not written to be a single, coherent document!"*

Its section headings are the useful part, because they're a decent taxonomy of what a sysadmin ever needs from the kernel: general administration, booting, **tracking down and identifying problems**, core-kernel subsystems, block/filesystem, device-specific guides.

Note what's filed under "core-kernel subsystems": `cgroup-v2`, `cpu-isolation`, `mm/index`, `pm/index`. Those four are the heart of performance work.

### 2. Kernel Parameters — the boot-time control surface

The prose half of this document is genuinely worth reading once, and it's short. The key concepts:

**Where parameters come from.** They're implemented by four macros — `__setup()`, `early_param()`, `core_param()` and `module_param()`. That tells you something useful: a "kernel parameter" isn't one mechanism, it's four, and they're processed at different stages of boot.

**The `--` boundary.** The kernel parses the command line up to `--`; anything it doesn't recognise (and that has no `.` in it) gets passed on to `init`. Parameters with `=` become environment variables for init, others become its arguments. This is why a typo in a boot parameter doesn't always error — it can silently become an init argument.

**Module parameters have two paths.** `usbcore.blinkenlights=1` on the command line, or `modprobe usbcore blinkenlights=1`. Built-in (non-module) drivers can *only* take the command-line form. And crucially: `/sys/module/<name>/parameters/` exposes them at runtime, and some are writable:

```
echo -n ${value} > /sys/module/${modulename}/parameters/${parm}
```

**Hyphens and underscores are equivalent.** `log_buf_len` and `log-buf-len` are the same parameter. Saves an argument one day.

**CPU list syntax** — directly relevant to isolation work. Beyond `1,2,10-20` there's a stride form most people have never seen:

```
isolcpus=1,2,10-20,100-2000:2/25
```

The last item means "take 2 CPUs from the start of each group of 25", giving 100,101,125,126,150,151… Also `N` for the last CPU (`16-N`) and `all` (`nohz_full=all`). The document warns that `N` is *dynamic* — the same boot line on a smaller machine can become an invalid range like `16-3`.

**`[KMG]` suffixes are binary**, not decimal — K is 2^10, not 1000.

**There's a length limit.** `COMMAND_LINE_SIZE`, between 256 and 4096 characters depending on architecture. A long isolation + hugepage + mitigation command line can genuinely hit this.

The actual parameter list is `kernel-parameters.txt`, pulled in by an RST `include` — **8,709 lines**, deliberately not inlined here.

### 3. Sysctl — the run-time control surface

Where kernel parameters are fixed until reboot, sysctl is live. The interface is a filesystem: `/proc/sys/`, one subtree per kernel area, so no special tooling is required — `cat` and `echo` are sufficient (`sysctl` is a convenience wrapper).

The subdirectory map is the concept worth keeping:

| Subtree | Area |
|---|---|
| `kernel/` | Global kernel tuning, scheduler, security/LSM |
| `vm/` | Memory management, buffer and cache behaviour |
| `net/` | Networking (documented over in `Documentation/networking/`) |
| `fs/` | Filehandles, inodes, dentries, quotas |
| `user/` | Per-user-namespace limits |
| `abi/`, `crypto/`, `debug/`, `dev/`, `sunrpc/`, `xen/` | Narrower areas |

The document is also a period piece — Rik van Riel's 1998 "legal blurb" warns you that if you wreck your system with it, *"I might even laugh at you."* Worth knowing that sysctls are a genuinely sharp tool with no seatbelts: a bad value takes effect immediately, system-wide.

The practical piece the document omits: changes via `/proc/sys` are lost on reboot. Persistence is `/etc/sysctl.d/*.conf`.

### 4. Tracing — how you see what the kernel is doing

The largest and most operationally valuable of the six. The distinction that matters:

- **Tracepoints** — static instrumentation compiled into the kernel at chosen points. Cheap, stable, always there.
- **kprobes / fprobes** — dynamic instrumentation. Attach to (almost) any kernel function at runtime, no recompile.
- **uprobes / user_events** — the same idea for user-space programs.
- **ftrace** — the framework that ties these together; the function tracer and the tracing filesystem.
- **Event tracing + histograms** — don't just record events, aggregate them in-kernel. This is how you count or bucket millions of events without drowning in output.

**Three tracers in this index are direct jitter-hunting tools**, and they deserve calling out:

- **`hwlat_detector`** — the Hardware Latency Detector. Its own doc says it detects *"large system latencies induced by the behavior of certain underlying hardware or firmware, independent of Linux itself… originally to detect SMIs (System Management Interrupts)."* SMIs are the firmware stealing your CPU behind the OS's back — invisible to every normal tool, and a classic cause of unexplained latency spikes.
- **`osnoise-tracer`** — measures *Operating System Noise*: "the interference experienced by an application due to activities inside the operating system… NMIs, IRQs, SoftIRQs, and any other system thread." This is a direct measurement of the thing CPU isolation is trying to eliminate.
- **`timerlat-tracer`** — sets a periodic timer and measures the wakeup latency of the woken thread, *"like cyclictest"*, but with tracing attached so you can see the cause rather than just the number.

If you're tuning for latency, these three are how you find out whether your tuning worked, and what's left.

### 5. Performance Security — who may observe

An unusual document: it's about the security risk of *observability itself*.

**The core problem.** `perf` collects four categories of data, escalating in sensitivity: (1) hardware/software configuration, (2) process names, PIDs, module load addresses, (3) counter contents, and (4) **architectural register contents and process memory addresses and data**. Category 4 can contain another process's secrets. So performance monitoring is a privilege, not a right.

**`CAP_PERFMON`.** Rather than requiring root, Linux splits out a capability specifically for performance monitoring. The document is emphatic that this is the modern, correct mechanism and that using `CAP_SYS_ADMIN` for it — though still supported for compatibility — is discouraged. `CAP_PERFMON` is least-privilege applied to observability. Since Linux 5.9 it's also sufficient on its own; `CAP_SYS_PTRACE` is no longer needed.

**`perf_event_paranoid`** is the dial for unprivileged users, and it's worth memorising:

| Value | Scope allowed |
|---|---|
| `-1` | No restrictions at all. Least secure; even the mlock limit is ignored |
| `>= 0` | Per-process *and* system-wide, but no raw tracepoints or ftrace function tracepoints |
| `>= 1` | Per-process only. No system-wide monitoring |
| `>= 2` | Per-process, **user-space only**. No kernel-space events |

**Resource limits.** Two that bite in practice: `RLIMIT_NOFILE`, because perf opens one file descriptor *per event per CPU* (a big event list on a big machine hits `ulimit -n` fast), and `perf_event_mlock_kb`, which caps the memory available for perf's ring buffers — per CPU, so on 8 cores a 516 KiB setting gives 4128 KiB total, and the *first* monitoring process will take all of it unless you divide it with `--mmap-pages`.

**The pattern it teaches** is broader than perf: give a group a capability on a specific binary (`setcap`, `chgrp`, `chmod o-rwx`) rather than handing out root. The document walks through exactly that with a `perf_users` group, and a `capsh`-based fallback for filesystems mounted `nosuid`.

### 6. Hardware Monitoring — physical ground truth

`hwmon` is the kernel's interface to temperature, voltage, current, power and fan sensors, exposed through sysfs (this is what `lm-sensors` reads).

The index is dominated by ~289 per-chip driver pages, which are reference material — you look up your board's chip, you don't read them. The generically useful entries are at the top: `sysfs-interface` (the ABI), `userspace-tools`, `hwmon-kernel-api`, and `pmbus-core` (the power-management bus standard used by server PSUs and VRMs).

**Why a sysadmin cares:** this is where you catch thermal throttling. A CPU that quietly drops frequency because it's hot produces exactly the symptom profile of a software performance regression, and no amount of tracing will explain it. Hardware monitoring is how you rule the physical world in or out.

---

## The through-line

Read in order, the six documents describe one loop:

**Configure** (kernel parameters at boot, sysctl at runtime) → **observe** (tracing for software behaviour, hwmon for physical reality) → **control who may observe** (perf security) — with the Admin Guide index as the map over all of it.

That loop *is* systems administration. Everything else is detail.

---

*Below: the six documents, concatenated and converted from RST. The 8,709-line `kernel-parameters.txt` and 272 of the 289 per-chip hwmon driver pages are referenced rather than inlined.*

---

# 1. Admin Guide

`Documentation/admin-guide/index.rst`

## The Linux kernel user's and administrator's guide

The following is a collection of user-oriented documents that have been
added to the kernel over time.  There is, as yet, little overall order or
organization here — this material was not written to be a single, coherent
document!  With luck things will improve quickly over time.

### General guides to kernel administration

This initial section contains overall information, including the README
file describing the kernel as a whole, documentation on kernel parameters,
etc.

- `README`
- `devices`

   features

A big part of the kernel's administrative interface is the /proc and sysfs
virtual filesystems; these documents describe how to interact with tem

- `sysfs-rules`
- `sysctl/index`
- `cputopology`
- `abi`

Security-related documentation:

- `hw-vuln/index`
- `LSM/index`
- `perf-security`

### Booting the kernel

- `bootconfig`
- `kernel-parameters`
- `efi-stub`
- `initrd`

### Tracking down and identifying problems

Here is a set of documents aimed at users who are trying to track down
problems and bugs in particular.

- `reporting-issues`
- `reporting-regressions`
- `quickly-build-trimmed-linux`
- `verify-bugs-and-bisect-regressions`
- `bug-hunting`
- `bug-bisect`
- `tainted-kernels`
- `ramoops`
- `dynamic-debug-howto`
- `init`
- `kdump/index`
- `perf/index`
- `pstore-blk`
- `clearing-warn-once`
- `kernel-per-CPU-kthreads`
- `lockup-watchdogs`
- `RAS/index`
- `sysrq`

### Core-kernel subsystems

These documents describe core-kernel administration interfaces that are
likely to be of interest on almost any system.

- `cgroup-v2`
- `cgroup-v1/index`
- `cpu-isolation`
- `cpu-load`
- `mm/index`
- `module-signing`
- `namespaces/index`
- `numastat`
- `pm/index`
- `syscall-user-dispatch`

Support for non-native binary formats.  Note that some of these
documents are ... old ...

- `binfmt-misc`
- `java`
- `mono`

### Block-layer and filesystem administration

- `bcache`
- `binderfs`
- `blockdev/index`
- `cifs/index`
- `device-mapper/index`
- `ext4`
- `filesystem-monitoring`
- `nfs/index`
- `iostats`
- `jfs`
- `md`
- `ufs`
- `xfs`

### Device-specific guides

How to configure your hardware within your Linux system.

- `acpi/index`
- `aoe/index`
- `auxdisplay/index`
- `braille-console`
- `btmrvl`
- `dell_rbu`
- `edid`
- `gpio/index`
- `hw_random`
- `laptops/index`
- `lcd-panel-cgram`
- `media/index`
- `nvme-multipath`
- `parport`
- `pnp`
- `rapidio`
- `rtc`
- `serial-console`
- `svga`
- `thermal/index`
- `thunderbolt`
- `vga-softcursor`
- `video-output`

### Workload analysis

This is the beginning of a section with information of interest to
application developers and system integrators doing analysis of the
Linux kernel for safety critical applications. Documents supporting
analysis of kernel interactions with applications, and key kernel
subsystems expectations will be found here.

- `workload-tracing`

### Everything else

A few hard-to-categorize and generally obsolete documents.

- `ldm`
- `unicode`

---

# 2. Kernel Parameters

`Documentation/admin-guide/kernel-parameters.rst`

## The kernel's command-line parameters

The following is a consolidated list of the kernel parameters as implemented
by the __setup(), early_param(), core_param() and module_param() macros
and sorted into English Dictionary order (defined as ignoring all
punctuation and sorting digits before letters in a case insensitive
manner), and with descriptions where known.

The kernel parses parameters from the kernel command line up to "`--`";
if it doesn't recognize a parameter and it doesn't contain a '.', the
parameter gets passed to init: parameters with '=' go into init's
environment, others are passed as command line arguments to init.
Everything after "`--`" is passed as an argument to init.

Module parameters can be specified in two ways: via the kernel command
line with a module name prefix, or via modprobe, e.g.:

```
(kernel command line) usbcore.blinkenlights=1
(modprobe command line) modprobe usbcore blinkenlights=1
```

Parameters for modules which are built into the kernel need to be
specified on the kernel command line.  modprobe looks through the
kernel command line (/proc/cmdline) and collects module parameters
when it loads a module, so the kernel command line can be used for
loadable modules too.

This document may not be entirely up to date and comprehensive. The command
"modinfo -p ${modulename}" shows a current list of all parameters of a loadable
module. Loadable modules, after being loaded into the running kernel, also
reveal their parameters in /sys/module/${modulename}/parameters/. Some of these
parameters may be changed at runtime by the command
`echo -n ${value} > /sys/module/${modulename}/parameters/${parm}`.

### Special handling

Hyphens (dashes) and underscores are equivalent in parameter names, so:

```
log_buf_len=1M print-fatal-signals=1
```

can also be entered as:

```
log-buf-len=1M print_fatal_signals=1
```

Double-quotes can be used to protect spaces in values, e.g.:

```
param="spaces in here"
```

#### cpu lists

Some kernel parameters take a list of CPUs as a value, e.g.  isolcpus,
nohz_full, irqaffinity, rcu_nocbs.  The format of this list is:

	<cpu number>,...,<cpu number>

or

	<cpu number>-<cpu number>
	(must be a positive range in ascending order)

or a mixture

<cpu number>,...,<cpu number>-<cpu number>

Note that for the special case of a range one can split the range into equal
sized groups and for each group use some amount from the beginning of that
group:

	<cpu number>-<cpu number>:<used size>/<group size>

For example one can add to the command line following parameter:

	isolcpus=1,2,10-20,100-2000:2/25

where the final item represents CPUs 100,101,125,126,150,151,...

The value "N" can be used to represent the numerically last CPU on the system,
i.e "foo_cpus=16-N" would be equivalent to "16-31" on a 32 core system.

Keep in mind that "N" is dynamic, so if system changes cause the bitmap width
to change, such as less cores in the CPU list, then N and any ranges using N
will also change.  Use the same on a small 4 core system, and "16-N" becomes
"16-3" and now the same boot input will be flagged as invalid (start > end).

The special case-tolerant group name "all" has a meaning of selecting all CPUs,
so that "nohz_full=all" is the equivalent of "nohz_full=0-N".

The semantics of "N" and "all" is supported on a level of bitmaps and holds for
all users of bitmap_parselist().

#### Metric suffixes

The [KMG] suffix is commonly described after a number of kernel
parameter values. 'K', 'M', 'G', 'T', 'P', and 'E' suffixes are allowed.
These letters represent the _binary_ multipliers 'Kilo', 'Mega', 'Giga',
'Tera', 'Peta', and 'Exa', equaling 2^10, 2^20, 2^30, 2^40, 2^50, and
2^60 bytes respectively. Such letter suffixes can also be entirely omitted.

### Kernel Build Options

The parameters listed below are only valid if certain kernel build options
were enabled and if respective hardware is present. This list should be kept
in alphabetical order. The text in square brackets at the beginning
of each description states the restrictions within which a parameter
is applicable.

Parameters denoted with BOOT are actually interpreted by the boot
loader, and have no meaning to the kernel directly.
Do not modify the syntax of boot loader parameters without extreme
need or coordination with `Documentation/arch/x86/boot.rst`.

There are also arch-specific kernel-parameters not documented here.

Note that ALL kernel parameters listed below are CASE SENSITIVE, and that
a trailing = on the name of any parameter states that the parameter will
be entered as an environment variable, whereas its absence indicates that
it will appear as a kernel argument readable via /proc/cmdline by programs
running once the system is up.

The number of kernel parameters is not limited, but the length of the
complete command line (parameters including spaces etc.) is limited to
a fixed number of characters. This limit depends on the architecture
and is between 256 and 4096 characters. It is defined in the file
./include/uapi/asm-generic/setup.h as COMMAND_LINE_SIZE.

> *(RST include: `kernel-parameters.txt` — see note above)*

   :literal:

---

# 3. Sysctl Tuning

`Documentation/admin-guide/sysctl/index.rst`

## Documentation for /proc/sys

Copyright (c) 1998, 1999,  Rik van Riel <riel@nl.linux.org>

---

'Why', I hear you ask, 'would anyone even _want_ documentation
for them sysctl files? If anybody really needs it, it's all in
the source...'

Well, this documentation is written because some people either
don't know they need to tweak something, or because they don't
have the time or knowledge to read the source code.

Furthermore, the programmers who built sysctl have built it to
be actually used, not just for the fun of programming it :-)

---

Legal blurb:

As usual, there are two main things to consider:

1. you get what you pay for
2. it's free

The consequences are that I won't guarantee the correctness of
this document, and if you come to me complaining about how you
screwed up your system because of wrong documentation, I won't
feel sorry for you. I might even laugh at you...

But of course, if you _do_ manage to screw up your system using
only the sysctl options used in this file, I'd like to hear of
it. Not only to have a great laugh, but also to make sure that
you're the last RTFMing person to screw up.

In short, e-mail your suggestions, corrections and / or horror
stories to: <riel@nl.linux.org>

Rik van Riel.

---

## Introduction

Sysctl is a means of configuring certain aspects of the kernel
at run-time, and the /proc/sys/ directory is there so that you
don't even need special tools to do it!
In fact, there are only four things needed to use these config
facilities:

- a running Linux system
- root access
- common sense (this is especially hard to come by these days)
- knowledge of what all those values mean

As a quick 'ls /proc/sys' will show, the directory consists of
several (arch-dependent?) subdirs. Each subdir is mainly about
one part of the kernel, so you can do configuration on a piece
by piece basis, or just some 'thematic frobbing'.

This documentation is about:

=============== ===============================================================
abi/		execution domains & personalities
<$ARCH>		tuning controls for various CPU architecture (e.g. csky, s390)
crypto/		cryptographic subsystem
debug/		debugging features
dev/		device specific information (e.g. dev/cdrom/info)
fs/		specific filesystems
		filehandle, inode, dentry and quota tuning
		binfmt_misc `Documentation/admin-guide/binfmt-misc.rst`
kernel/		global kernel info / tuning
		miscellaneous stuff
		some architecture-specific controls
		security (LSM) stuff
net/		networking stuff, for documentation look in:
		`Documentation/networking/`
proc/		<empty>
sunrpc/		SUN Remote Procedure Call (NFS)
user/		Per user namespace limits
vm/		memory management tuning
		buffer and cache management
xen/		Xen hypervisor controls
=============== ===============================================================

These are the subdirs I have on my system or have been discovered by
searching through the source code. There might be more or other subdirs
in another setup. If you see another dir, I'd really like to hear about
it :-)

- `abi`
- `crypto`
- `debug`
- `fs`
- `kernel`
- `net`
- `sunrpc`
- `user`
- `vm`
- `xen`

---

# 4. Tracing / Debugging

`Documentation/trace/index.rst`

## Linux Tracing Technologies Guide

Tracing in the Linux kernel is a powerful mechanism that allows
developers and system administrators to analyze and debug system
behavior. This guide provides documentation on various tracing
frameworks and tools available in the Linux kernel.

### Introduction to Tracing

This section provides an overview of Linux tracing mechanisms
and debugging approaches.

- `debugging`
- `tracepoints`
- `tracepoint-analysis`
- `ring-buffer-map`

### Core Tracing Frameworks

The following are the primary tracing frameworks integrated into
the Linux kernel.

- `ftrace`
- `ftrace-design`
- `ftrace-uses`
- `kprobes`
- `kprobetrace`
- `fprobetrace`
- `eprobetrace`
- `fprobe`
- `ring-buffer-design`

### Event Tracing and Analysis

A detailed explanation of event tracing mechanisms and their
applications.

- `events`
- `events-kmem`
- `events-power`
- `events-nmi`
- `events-msr`
- `events-landlock`
- `events-pci`
- `events-pci-controller`
- `boottime-trace`
- `histogram`
- `histogram-design`

### Hardware and Performance Tracing

This section covers tracing features that monitor hardware
interactions and system performance.

- `intel_th`
- `stm`
- `sys-t`
- `coresight/index`
- `rv/index`
- `hisi-ptt`
- `mmiotrace`
- `hwlat_detector`
- `osnoise-tracer`
- `timerlat-tracer`

### User-Space Tracing

These tools allow tracing user-space applications and
interactions.

- `user_events`
- `uprobetracer`

### Remote Tracing

This section covers the framework to read compatible ring-buffers, written by
entities outside of the kernel (most likely firmware or hypervisor)

- `remotes`

### Additional Resources

For more details, refer to the respective documentation of each
tracing tool and framework.

---

# 5. Performance Security

`Documentation/admin-guide/perf-security.rst`

## Perf events and tool security

### Overview

Usage of Performance Counters for Linux (perf_events) [1] , [2] , [3]
can impose a considerable risk of leaking sensitive data accessed by
monitored processes. The data leakage is possible both in scenarios of
direct usage of perf_events system call API [2] and over data files
generated by Perf tool user mode utility (Perf) [3] , [4] . The risk
depends on the nature of data that perf_events performance monitoring
units (PMU) [2] and Perf collect and expose for performance analysis.
Collected system and performance data may be split into several
categories:

1. System hardware and software configuration data, for example: a CPU
   model and its cache configuration, an amount of available memory and
   its topology, used kernel and Perf versions, performance monitoring
   setup including experiment time, events configuration, Perf command
   line parameters, etc.

2. User and kernel module paths and their load addresses with sizes,
   process and thread names with their PIDs and TIDs, timestamps for
   captured hardware and software events.

3. Content of kernel software counters (e.g., for context switches, page
   faults, CPU migrations), architectural hardware performance counters
   (PMC) [8] and machine specific registers (MSR) [9] that provide
   execution metrics for various monitored parts of the system (e.g.,
   memory controller (IMC), interconnect (QPI/UPI) or peripheral (PCIe)
   uncore counters) without direct attribution to any execution context
   state.

4. Content of architectural execution context registers (e.g., RIP, RSP,
   RBP on x86_64), process user and kernel space memory addresses and
   data, content of various architectural MSRs that capture data from
   this category.

Data that belong to the fourth category can potentially contain
sensitive process data. If PMUs in some monitoring modes capture values
of execution context registers or data from process memory then access
to such monitoring modes requires to be ordered and secured properly.
So, perf_events performance monitoring and observability operations are
the subject for security access control management [5] .

### perf_events access control

To perform security checks, the Linux implementation splits processes
into two categories [6] : a) privileged processes (whose effective user
ID is 0, referred to as superuser or root), and b) unprivileged
processes (whose effective UID is nonzero). Privileged processes bypass
all kernel security permission checks so perf_events performance
monitoring is fully available to privileged processes without access,
scope and resource restrictions.

Unprivileged processes are subject to a full security permission check
based on the process's credentials [5] (usually: effective UID,
effective GID, and supplementary group list).

Linux divides the privileges traditionally associated with superuser
into distinct units, known as capabilities [6] , which can be
independently enabled and disabled on per-thread basis for processes and
files of unprivileged users.

Unprivileged processes with enabled CAP_PERFMON capability are treated
as privileged processes with respect to perf_events performance
monitoring and observability operations, thus, bypass *scope* permissions
checks in the kernel. CAP_PERFMON implements the principle of least
privilege [13] (POSIX 1003.1e: 2.2.2.39) for performance monitoring and
observability operations in the kernel and provides a secure approach to
performance monitoring and observability in the system.

For backward compatibility reasons the access to perf_events monitoring and
observability operations is also open for CAP_SYS_ADMIN privileged
processes but CAP_SYS_ADMIN usage for secure monitoring and observability
use cases is discouraged with respect to the CAP_PERFMON capability.
If system audit records [14] for a process using perf_events system call
API contain denial records of acquiring both CAP_PERFMON and CAP_SYS_ADMIN
capabilities then providing the process with CAP_PERFMON capability singly
is recommended as the preferred secure approach to resolve double access
denial logging related to usage of performance monitoring and observability.

Prior Linux v5.9 unprivileged processes using perf_events system call
are also subject for PTRACE_MODE_READ_REALCREDS ptrace access mode check
 [7] , whose outcome determines whether monitoring is permitted.
So unprivileged processes provided with CAP_SYS_PTRACE capability are
effectively permitted to pass the check. Starting from Linux v5.9
CAP_SYS_PTRACE capability is not required and CAP_PERFMON is enough to
be provided for processes to make performance monitoring and observability
operations.

Other capabilities being granted to unprivileged processes can
effectively enable capturing of additional data required for later
performance analysis of monitored processes or a system. For example,
CAP_SYSLOG capability permits reading kernel space memory addresses from
/proc/kallsyms file.

### Privileged Perf users groups

Mechanisms of capabilities, privileged capability-dumb files [6],
file system ACLs [10] and sudo [15] utility can be used to create
dedicated groups of privileged Perf users who are permitted to execute
performance monitoring and observability without limits. The following
steps can be taken to create such groups of privileged Perf users.

1. Create perf_users group of privileged Perf users, assign perf_users
   group to Perf tool executable and limit access to the executable for
   other users in the system who are not in the perf_users group:

```
# groupadd perf_users
# ls -alhF
-rwxr-xr-x  2 root root  11M Oct 19 15:12 perf
# chgrp perf_users perf
# ls -alhF
-rwxr-xr-x  2 root perf_users  11M Oct 19 15:12 perf
# chmod o-rwx perf
# ls -alhF
-rwxr-x---  2 root perf_users  11M Oct 19 15:12 perf
```

2. Assign the required capabilities to the Perf tool executable file and
   enable members of perf_users group with monitoring and observability
   privileges [6] :

```
# setcap "cap_perfmon,cap_sys_ptrace,cap_syslog=ep" perf
# setcap -v "cap_perfmon,cap_sys_ptrace,cap_syslog=ep" perf
perf: OK
# getcap perf
perf = cap_sys_ptrace,cap_syslog,cap_perfmon+ep
```

If the libcap [16] installed doesn't yet support "cap_perfmon", use "38" instead,
i.e.:

```
# setcap "38,cap_ipc_lock,cap_sys_ptrace,cap_syslog=ep" perf
```

Note that you may need to have 'cap_ipc_lock' in the mix for tools such as
'perf top', alternatively use 'perf top -m N', to reduce the memory that
it uses for the perf ring buffer, see the memory allocation section below.

Using a libcap without support for CAP_PERFMON will make cap_get_flag(caps, 38,
CAP_EFFECTIVE, &val) fail, which will lead the default event to be 'cycles:u',
so as a workaround explicitly ask for the 'cycles' event, i.e.:

```
# perf top -e cycles
```

To get kernel and user samples with a perf binary with just CAP_PERFMON.

As a result, members of perf_users group are capable of conducting
performance monitoring and observability by using functionality of the
configured Perf tool executable that, when executes, passes perf_events
subsystem scope checks.

In case Perf tool executable can't be assigned required capabilities (e.g.
file system is mounted with nosuid option or extended attributes are
not supported by the file system) then creation of the capabilities
privileged environment, naturally shell, is possible. The shell provides
inherent processes with CAP_PERFMON and other required capabilities so that
performance monitoring and observability operations are available in the
environment without limits. Access to the environment can be open via sudo
utility for members of perf_users group only. In order to create such
environment:

1. Create shell script that uses capsh utility [16] to assign CAP_PERFMON
   and other required capabilities into ambient capability set of the shell
   process, lock the process security bits after enabling SECBIT_NO_SETUID_FIXUP,
   SECBIT_NOROOT and SECBIT_NO_CAP_AMBIENT_RAISE bits and then change
   the process identity to sudo caller of the script who should essentially
   be a member of perf_users group:

```
# ls -alh /usr/local/bin/perf.shell
-rwxr-xr-x. 1 root root 83 Oct 13 23:57 /usr/local/bin/perf.shell
# cat /usr/local/bin/perf.shell
exec /usr/sbin/capsh --iab=^cap_perfmon --secbits=239 --user=$SUDO_USER -- -l
```

2. Extend sudo policy at /etc/sudoers file with a rule for perf_users group:

```
# grep perf_users /etc/sudoers
%perf_users    ALL=/usr/local/bin/perf.shell
```

3. Check that members of perf_users group have access to the privileged
   shell and have CAP_PERFMON and other required capabilities enabled
   in permitted, effective and ambient capability sets of an inherent process:

```
$ id
uid=1003(capsh_test) gid=1004(capsh_test) groups=1004(capsh_test),1000(perf_users) context=unconfined_u:unconfined_r:unconfined_t:s0-s0:c0.c1023
$ sudo perf.shell
[sudo] password for capsh_test:
$ grep Cap /proc/self/status
CapInh:        0000004000000000
CapPrm:        0000004000000000
CapEff:        0000004000000000
CapBnd:        000000ffffffffff
CapAmb:        0000004000000000
$ capsh --decode=0000004000000000
0x0000004000000000=cap_perfmon
```

As a result, members of perf_users group have access to the privileged
environment where they can use tools employing performance monitoring APIs
governed by CAP_PERFMON Linux capability.

This specific access control management is only available to superuser
or root running processes with CAP_SETPCAP, CAP_SETFCAP [6]
capabilities.

### Unprivileged users

perf_events *scope* and *access* control for unprivileged processes
is governed by perf_event_paranoid [2] setting:

-1:
     Impose no *scope* and *access* restrictions on using perf_events
     performance monitoring. Per-user per-cpu perf_event_mlock_kb [2]
     locking limit is ignored when allocating memory buffers for storing
     performance data. This is the least secure mode since allowed
     monitored *scope* is maximized and no perf_events specific limits
     are imposed on *resources* allocated for performance monitoring.

>=0:
     *scope* includes per-process and system wide performance monitoring
     but excludes raw tracepoints and ftrace function tracepoints
     monitoring. CPU and system events happened when executing either in
     user or in kernel space can be monitored and captured for later
     analysis. Per-user per-cpu perf_event_mlock_kb locking limit is
     imposed but ignored for unprivileged processes with CAP_IPC_LOCK
 [6] capability.

>=1:
     *scope* includes per-process performance monitoring only and
     excludes system wide performance monitoring. CPU and system events
     happened when executing either in user or in kernel space can be
     monitored and captured for later analysis. Per-user per-cpu
     perf_event_mlock_kb locking limit is imposed but ignored for
     unprivileged processes with CAP_IPC_LOCK capability.

>=2:
     *scope* includes per-process performance monitoring only. CPU and
     system events happened when executing in user space only can be
     monitored and captured for later analysis. Per-user per-cpu
     perf_event_mlock_kb locking limit is imposed but ignored for
     unprivileged processes with CAP_IPC_LOCK capability.

### Resource control

##### Open file descriptors

The perf_events system call API [2] allocates file descriptors for
every configured PMU event. Open file descriptors are a per-process
accountable resource governed by the RLIMIT_NOFILE [11] limit
(ulimit -n), which is usually derived from the login shell process. When
configuring Perf collection for a long list of events on a large server
system, this limit can be easily hit preventing required monitoring
configuration. RLIMIT_NOFILE limit can be increased on per-user basis
modifying content of the limits.conf file [12] . Ordinarily, a Perf
sampling session (perf record) requires an amount of open perf_event
file descriptors that is not less than the number of monitored events
multiplied by the number of monitored CPUs.

##### Memory allocation

The amount of memory available to user processes for capturing
performance monitoring data is governed by the perf_event_mlock_kb [2]
setting. This perf_event specific resource setting defines overall
per-cpu limits of memory allowed for mapping by the user processes to
execute performance monitoring. The setting essentially extends the
RLIMIT_MEMLOCK [11] limit, but only for memory regions mapped
specifically for capturing monitored performance events and related data.

For example, if a machine has eight cores and perf_event_mlock_kb limit
is set to 516 KiB, then a user process is provided with 516 KiB * 8 =
4128 KiB of memory above the RLIMIT_MEMLOCK limit (ulimit -l) for
perf_event mmap buffers. In particular, this means that, if the user
wants to start two or more performance monitoring processes, the user is
required to manually distribute the available 4128 KiB between the
monitoring processes, for example, using the --mmap-pages Perf record
mode option. Otherwise, the first started performance monitoring process
allocates all available 4128 KiB and the other processes will fail to
proceed due to the lack of memory.

RLIMIT_MEMLOCK and perf_event_mlock_kb resource constraints are ignored
for processes with the CAP_IPC_LOCK capability. Thus, perf_events/Perf
privileged users can be provided with memory above the constraints for
perf_events/Perf performance monitoring purpose by providing the Perf
executable with CAP_IPC_LOCK capability.

### Bibliography

- [1] https://lwn.net/Articles/337493/
- [2] http://man7.org/linux/man-pages/man2/perf_event_open.2.html
- [3] http://web.eece.maine.edu/~vweaver/projects/perf_events/
- [4] https://perf.wiki.kernel.org/index.php/Main_Page
- [5] https://www.kernel.org/doc/html/latest/security/credentials.html
- [6] http://man7.org/linux/man-pages/man7/capabilities.7.html
- [7] http://man7.org/linux/man-pages/man2/ptrace.2.html
- [8] https://en.wikipedia.org/wiki/Hardware_performance_counter
- [9] https://en.wikipedia.org/wiki/Model-specific_register
- [10] http://man7.org/linux/man-pages/man5/acl.5.html
- [11] http://man7.org/linux/man-pages/man2/getrlimit.2.html
- [12] http://man7.org/linux/man-pages/man5/limits.conf.5.html
- [13] https://sites.google.com/site/fullycapable
- [14] http://man7.org/linux/man-pages/man8/auditd.8.html
- [15] https://man7.org/linux/man-pages/man8/sudo.8.html
- [16] https://git.kernel.org/pub/scm/libs/libcap/libcap.git/

---

# 6. Hardware Monitoring

`Documentation/hwmon/index.rst`

## Hardware Monitoring

- `hwmon-kernel-api`
- `pmbus-core`
- `submitting-patches`
- `sysfs-interface`
- `userspace-tools`

## Hardware Monitoring Kernel Drivers

- `abituguru`
- `abituguru3`
- `acbel-fsg032`
- `acpi_power_meter`
- `ad7314`
- `adc128d818`
- `adm1025`
- `adm1026`
- `adm1031`
- `adm1177`
- `adm1266`
- `adm1275`
- *… and 272 further per-chip driver pages, omitted here for length — the full list is in the file itself*
