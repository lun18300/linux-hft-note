# Low-Latency Linux Tuning for HFT

*The concepts behind the knobs — oriented to the in-tree kernel documentation.*

Every technique here answers one question: **what could interrupt my thread, and how do I remove it?** Paths referenced below were verified present in `Documentation/` of the kernel tree.

---

# Part I — Concepts

## 1. You don't care about latency — you care about jitter

A system averaging 5µs that occasionally spikes to 500µs is worse for trading than one steadily at 8µs. The average is a fiction; nobody gets filled at the average.

> **Consequence:** report p99, p99.9 and max — never the mean. A vendor who quotes a mean is hiding the tail. Every section below is about removing variance, not raw speed.

## 2. Your CPU is not yours

You think your thread owns a core. It does not. Four things take it away:

| Thief | What it is |
|---|---|
| **Interrupts (IRQs)** | Hardware signals "packet arrived" — the CPU drops your code mid-stream to service it |
| **The timer tick** | The kernel historically interrupts every core 100–1000×/sec just to ask "anything to do?" |
| **Other threads** | The scheduler preempts you in favour of someone else |
| **Kernel threads** | kworkers, RCU callbacks, timers — the kernel's own housekeeping |

Each steal costs a **context switch**: save registers, swap page tables, restore — roughly a microsecond directly. The real damage is that your L1/L2 cache now holds someone else's data, so your next hundred memory accesses miss. That is the hidden bill.

**The concepts that fix it:**

- **CPU isolation** (`isolcpus`) — tell the scheduler this core does not exist for general work
- **Tickless operation** (`nohz_full`) — if exactly one thread runs on a core, the tick is pointless; disable it and the core runs genuinely uninterrupted
- **IRQ affinity** — interrupts are steerable; point them at housekeeping cores, away from trading cores
- **RCU offload** (`rcu_nocbs`) — RCU's memory-reclamation callbacks otherwise fire on whichever core touched the data

> **Mental model:** you are not making the core faster. You are evicting everybody else from the room.

**In the tree:**
- `Documentation/admin-guide/kernel-per-CPU-kthreads.rst` — the single best low-latency document
- `Documentation/admin-guide/cpu-isolation.rst`
- `Documentation/timers/no_hz.rst`
- `Documentation/core-api/irq/irq-affinity.rst`

## 3. Idle CPUs are slow CPUs

The most commonly missed concept. Modern CPUs aggressively save power:

- **C-states** — idle states. C0 is running; C1, C3, C6 are progressively deeper sleep. Deeper sleep saves more power and *takes longer to wake*, ranging from near-zero to tens of microseconds.
- **P-states** — frequency and voltage while running. The CPU downclocks when it believes you are not busy.

> ⚠️ **The trap:** your trading process is fast. It handles a packet in 2µs then waits. The CPU sees an idle core and puts it to sleep. The next packet — the one that matters — pays the full wake-up cost. **Your quietest, most important packets become your slowest**, and a load test that hammers the box continuously will never show it.

The remedies — limiting C-states, the `performance` governor, or holding `/dev/cpu_dma_latency` open — all express one idea: pay the power bill to keep the core awake and at full speed. In HFT that trade is always worth taking.

**In the tree:**
- `Documentation/admin-guide/pm/cpuidle.rst`, `intel_idle.rst`
- `Documentation/admin-guide/pm/cpufreq.rst`, `intel_pstate.rst`

## 4. Memory is a hierarchy, and distance is time

Orders of magnitude only — absolutes vary by hardware, but the ratios hold:

| Where the data is | Approximate cost |
|---|---|
| L1 cache | ~1 ns |
| L2 | ~4 ns |
| L3 (shared) | ~15–40 ns |
| Local DRAM | ~80–100 ns |
| **Another socket's DRAM** | **~150–200 ns** |

### NUMA

A two-socket box is really two computers sharing a cable. If your NIC sits on socket 0, your thread on socket 1, and your buffer wherever the allocator chose, every packet crosses that cable repeatedly. **Keep NIC, core and memory on one NUMA node.** Free, large, and routinely gotten wrong.

### TLB and hugepages

Virtual-to-physical translations are cached in the TLB. With 4KB pages a 2GB buffer needs ~500,000 entries; the TLB holds perhaps a thousand. Constant misses, each an extra memory walk. 2MB hugepages cut the entry count by 512×.

> ⚠️ **Nuance:** *transparent* hugepages can hurt, because the kernel may stall your thread while compacting memory to produce one. Many shops pre-allocate hugepages explicitly and disable THP. Same feature, opposite result — it depends on whether the work happens at startup or in your hot path.

### Cache-line contention

Two cores writing to variables sharing a 64-byte line ping-pong ownership between caches. It looks like independent code and performs like a lock. `perf c2c` finds it.

**In the tree:** `Documentation/admin-guide/mm/hugetlbpage.rst`, `transhuge.rst`, `numaperf.rst`

## 5. The packet's actual journey

```
wire → NIC → DMA into ring buffer in RAM → IRQ → softirq / NAPI
     → protocol stack (IP, TCP) → socket buffer → read() → your code
```

- **Ring buffer + DMA** — the NIC writes packets straight into memory without the CPU, then must tell someone.
- **Interrupt coalescing** — the NIC can delay the interrupt to batch packets. A pure throughput-versus-latency dial: batching is efficient per packet and terrible for the first one. HFT turns it down or off, accepting more CPU burn.
- **NAPI** — the kernel's hybrid: take one interrupt, then poll for more while traffic is heavy. Good for throughput, and it means latency under load differs from latency when quiet.
- **Softirq** — interrupt handlers must be short, so real work is deferred. Which core runs it is decided by packet steering.
- **RSS / RPS / RFS** — RSS hashes flows to hardware queues in silicon; RPS does the same in software; RFS steers a packet to the core where the application that will `read()` it is actually running, so data lands in the right cache.
- **The copy** — classic sockets copy kernel buffer to user buffer: time, plus cache pollution.

**In the tree:** `Documentation/networking/napi.rst`, `scaling.rst`, `Documentation/admin-guide/sysctl/net.rst`

## 6. Polling versus interrupts

The deepest idea in the topic.

- **Interrupt-driven** — sleep until told. Efficient, but you pay wake-up and scheduling on every message, and you are back to the C-state problem in §3.
- **Polling** — spin asking "anything yet?". Burns 100% of a core doing nothing. But there is no wake-up, no context switch, no scheduling decision.

HFT chooses polling almost everywhere. That is what `busy_poll` / `SO_BUSY_POLL` expose in the kernel, and it is the founding design principle of every bypass stack.

> **The mindset flip:** CPU cycles are cheap, jitter is expensive. Burning an entire core to avoid a 10µs wake-up is a trade you take every time. This is why HFT tuning looks irrational to a normal sysadmin — you deliberately waste resources to buy determinism.

## 7. Kernel bypass

Once you accept polling, the logical end point: why involve the kernel at all? Bypass maps the NIC's queues directly into process memory. The application talks to the hardware — no syscall, no softirq, no copy, no scheduler.

| Approach | What it is |
|---|---|
| **AF_XDP** | The kernel's own in-tree version |
| **DPDK** | Full userspace drivers; the NIC disappears from the OS |
| **Onload / VMA** | Vendor libraries intercepting the socket API, so unmodified code gets bypass |

Ballpark: the kernel stack is roughly 5–15µs wire-to-application; bypass is roughly 1–2µs.

> ⚠️ **Answer this first:** find out whether your firm's hot path already bypasses the kernel. If it does, §4 and §9 still apply in full, and much of §5's softirq tuning does not. It decides how much of this document is relevant to you.

**In the tree:** `Documentation/networking/af_xdp.rst`

## 8. Scheduling

| Class | Behaviour |
|---|---|
| `SCHED_OTHER` | Default (CFS). Fair, and it will preempt you |
| `SCHED_FIFO` / `RR` | Real-time. You run until you yield |
| `SCHED_DEADLINE` | Deadline-driven |

"Real-time" means **predictable, not fast**. A guaranteed 8µs beats usually-3µs-sometimes-400µs.

> ⚠️ **Gotcha worth knowing in advance:** `sched_rt_runtime_us` throttles RT tasks to roughly 95% of each period, as a safety valve so a runaway RT thread cannot wedge the machine. Hit it and your carefully pinned thread is descheduled periodically — this costs people a day to diagnose.

**Pinning** (`taskset`, `sched_setaffinity`) is the partner concept: isolation removes others from the core, pinning nails you to it so you never migrate and never lose a warm cache.

**In the tree:** `Documentation/scheduler/sched-rt-group.rst`, `sched-deadline.rst`

## 9. Measurement — the non-negotiable one

Everything above is a hypothesis until measured, and measurement has its own traps:

- **Clock sources.** TSC versus HPET matters enormously. A poor clocksource makes timestamping itself a syscall costing microseconds, so the measurement perturbs what it measures.
- **Hardware timestamping.** The NIC stamps the packet on arrival, before any software sees it. This is the only honest wire-to-application number; software timestamps hide precisely the delay you are hunting.
- **Histograms, never averages.** See §1.
- **Measure the production path under production load.** C-state behaviour, NAPI mode and cache pressure all differ between a quiet box and a busy one — usually in the direction that flatters your benchmark.

### Tools that ship in the kernel tree

- `tools/power/x86/turbostat` — actual achieved frequency and real per-core C-state residency. This is how you verify §3 rather than hoping.
- `tools/perf` — `perf sched` for scheduling latency, `perf c2c` for cache-line contention.

**In the tree:** `Documentation/networking/timestamping.rst`, `Documentation/admin-guide/perf/index.rst`

## 10. How it all fits together

Everything above reduces to three moves:

| | |
|---|---|
| **Exclusivity** | Get everything else off this core — isolation, pinning, IRQ affinity, tickless operation |
| **Readiness** | Never be asleep, slow or cold when work arrives — C-states, P-states, polling, warm caches |
| **Shortest path** | Fewest hops from wire to your code — NUMA locality, hugepages, coalescing off, bypass |

And one discipline underneath all three: **measure the tail, on the real path, with hardware timestamps.**

---

# Part II — The Checklist

*Every setting maps back to a concept in Part I. Flag names verified against this kernel tree.*

**Legend:** ✅ safe default (low risk on a dedicated trading host) · ⚠️ measure first (depends on your hardware) · 🔒 needs sign-off (trades away safety margin)

Worked example assumes **cores 0–1 housekeeping, cores 2–15 trading**, NIC on NUMA node 0. Adjust to your own topology (`lstopo`, `numactl -H`).

> ⚠️ **Two corrections to the folklore.** `isolcpus=` accepts exactly three flags in this kernel — `nohz`, `domain`, `managed_irq`. And `net.ipv4.tcp_low_latency` is **dead**: `Documentation/networking/ip-sysctl.rst` calls it "a legacy option, it has no effect anymore", and the variable is wired to the sysctl table but never read. It appears in most HFT tuning guides online. Setting it does nothing.

## 0. BIOS first — or everything below is theatre

The OS cannot override firmware. Do this before touching Linux.

| Setting | Value | |
|---|---|---|
| Power profile | Max Performance / OS Control | ✅ |
| C-States, C1E | Disabled | ✅ |
| P-States / SpeedStep | Disabled, or OS-controlled | ⚠️ |
| Turbo Boost | See note | ⚠️ |
| NUMA / node interleaving | NUMA on, interleaving off | ✅ |
| PCIe ASPM | Disabled | ✅ |
| Hyperthreading (SMT) | Usually off for HFT | ⚠️ |

**Turbo** raises peak clock but introduces a *variable* clock. Some shops disable it for determinism, others keep it — measure, this is a real tradeoff rather than a known answer.

**SMT** siblings share L1/L2 and execution units, so a noisy sibling jitters your trading thread; disabling halves your core count, and if you keep it on, never schedule work on a trading core's sibling.

## 1. Kernel command line

```
isolcpus=domain,managed_irq,nohz,2-15
nohz_full=2-15
rcu_nocbs=2-15
rcu_nocb_poll
irqaffinity=0-1
nosoftlockup
nmi_watchdog=0
audit=0
skew_tick=1
transparent_hugepage=never
default_hugepagesz=1G hugepagesz=1G hugepages=16
intel_pstate=disable
intel_idle.max_cstate=0 processor.max_cstate=1
pcie_aspm=off
numa_balancing=disable
tsc=reliable
```

| Group | Why | |
|---|---|---|
| `isolcpus` `nohz_full` `rcu_nocbs` | §2 — evict the scheduler, the tick and RCU callbacks | ✅ |
| `rcu_nocb_poll` | Offload threads poll rather than being woken by your core | ⚠️ |
| `irqaffinity=0-1` | Default IRQ target becomes the housekeeping cores | ✅ |
| `nosoftlockup` `nmi_watchdog=0` | Watchdogs periodically interrupt every core | ✅ |
| `skew_tick=1` | De-synchronises remaining ticks so cores don't contend at once | ✅ |
| hugepages | §4 — pre-allocate at boot, before memory fragments | ✅ |
| `transparent_hugepage=never` | THP defrag can stall your hot path | ✅ |
| `intel_pstate=disable` + C-state caps | §3 — the single biggest jitter source | ✅ |
| `tsc=reliable` | §9 — keeps timestamping cheap | ⚠️ |

### The two that need a conversation

> 🔒 **`idle=poll`** — keeps cores in C0 permanently. Lowest possible wake latency, but significant heat and power, and it can *reduce* available turbo headroom. Try the C-state caps first; reach for this only if `turbostat` shows you are still entering deep states.

> 🔒 **`mitigations=off`** — disables Spectre/Meltdown/MDS mitigations. Typically a large, measurable latency win, which is why it appears in every HFT guide. It is also a deliberate reduction in CPU-level security isolation, and **that is a security decision, not a tuning decision**. It is listed here because you will meet it and should understand it — not as a recommendation. It needs sign-off from whoever owns security risk, and is most defensible on a dedicated, single-tenant, non-internet-facing host. Individual mitigations can be disabled selectively rather than all at once; see `Documentation/admin-guide/hw-vuln/`.

## 2. Sysctls

```ini
# /etc/sysctl.d/99-lowlat.conf
kernel.numa_balancing = 0
kernel.timer_migration = 0
vm.swappiness = 0
vm.stat_interval = 120
net.core.busy_poll = 50
net.core.busy_read = 50
net.core.netdev_max_backlog = 8192
net.core.rmem_max = 16777216
net.core.wmem_max = 16777216
```

| Knob | Notes | |
|---|---|---|
| `numa_balancing=0` | Auto-migration moves your pages mid-trade | ✅ |
| `timer_migration=0` | Stops timers migrating onto isolated cores | ✅ |
| `swappiness=0` | Never swap a trading process | ✅ |
| `stat_interval=120` | vmstat housekeeping runs less often | ✅ |
| `busy_poll` / `busy_read` | §6 — microseconds to spin before sleeping; 50 is a starting point | ⚠️ |
| socket buffer maxima | Raise the ceiling; the application still sets its own | ✅ |

> 🔒 **`kernel.sched_rt_runtime_us = -1`** — removes the RT throttle (§8). Without it a `SCHED_FIFO` thread is descheduled for roughly 5% of every period, exactly the jitter you are trying to remove. With it, **a spinning RT thread with a bug can wedge the machine hard enough to need a power cycle.** Set it only alongside proper core isolation, and keep a housekeeping core plus out-of-band access (IPMI/iDRAC) available.

**Do not bother with:** `net.ipv4.tcp_low_latency` — no-op, see above.

## 3. NIC — per interface

```bash
# coalescing off: don't batch, don't wait  (§5)
ethtool -C eth0 adaptive-rx off adaptive-tx off rx-usecs 0 rx-frames 1 tx-usecs 0 tx-frames 1

# offloads that aggregate packets add latency
ethtool -K eth0 gro off lro off tso off gso off

# pause frames stall the link
ethtool -A eth0 rx off tx off

# one queue per trading core
ethtool -L eth0 combined 8

# ring size
ethtool -G eth0 rx 1024 tx 1024
```

| Setting | Notes | |
|---|---|---|
| Coalescing off | The single biggest NIC-side win | ✅ |
| `gro` / `lro` off | These exist to merge packets — pure latency cost | ✅ |
| `tso` / `gso` off | Reconsider if you also push bulk traffic over this NIC | ⚠️ |
| Flow control off | Assuming your switch agrees | ✅ |
| Ring size | Smaller means less queuing delay and more drop risk under burst | ⚠️ |
| `ethtool -T eth0` | Not a setting — run it to confirm hardware timestamping support (§9) | ✅ |

**Steering** (§5) — pin flows to the core that will read them:

```bash
ethtool -X eth0 equal 8                                  # RSS across queues
ethtool -N eth0 flow-type tcp4 dst-port 9000 action 3    # or explicit ntuple
```

## 4. IRQ affinity

```bash
systemctl stop irqbalance && systemctl disable irqbalance

# route each NIC queue IRQ to its paired trading core
grep eth0 /proc/interrupts
echo 2 > /proc/irq/<N>/smp_affinity_list
```

Disabling `irqbalance` is not optional — it is a daemon whose entire purpose is to move IRQs around, which means undoing this section while you sleep.

## 5. Per process

```bash
numactl --cpunodebind=0 --membind=0 \
  chrt -f 80 \
  taskset -c 2 ./trading_app
```

| Layer | Purpose | |
|---|---|---|
| `numactl --membind` | §4 — memory on the NIC's node | ✅ |
| `chrt -f 80` | §8 — SCHED_FIFO. Stay below 99; leave headroom for kernel RT threads | ⚠️ |
| `taskset -c 2` | §8 — pin to one isolated core | ✅ |

Inside the application: `mlockall(MCL_CURRENT|MCL_FUTURE)` so nothing pages out, `setsockopt(SO_BUSY_POLL)` per socket, and pre-fault every buffer at startup. Nothing in the hot path should ever call `malloc`.

## 6. Verify — the part people skip

| Check | Command |
|---|---|
| Cores actually staying in C0? | `turbostat --interval 5` — watch `Busy%`, `CPU%c1/c3/c6`, `Avg_MHz` |
| C-state entries per core | `cat /sys/devices/system/cpu/cpu2/cpuidle/state*/usage` — should stay flat |
| Is the tick really off? | `cat /proc/interrupts` — the `LOC` row should barely move on isolated cores |
| IRQs landing where you think | `watch -d cat /proc/interrupts` |
| Anything on isolated cores? | `ps -eLo psr,comm,cls \| sort -n` |
| Scheduling latency | `perf sched record` / `perf sched latency` |
| Cache-line contention | `perf c2c record` / `perf c2c report` |
| Wire-to-application truth | NIC hardware timestamps — `Documentation/networking/timestamping.rst` |
| Jitter baseline | `cyclictest -m -p95 -a2 -t1 -n` (from `rt-tests`, not in this tree) |

> **Order of operations:** baseline with `cyclictest` and your own histogram → change *one* group → re-measure → keep or revert. Change everything at once and you will never learn which knob mattered, or which one hurt.

## 7. Before you apply any of this

**This is a starting point, not a configuration.** The ✅ items are low risk on a dedicated trading host; the ⚠️ items genuinely depend on your NICs, CPU generation and traffic shape, and none of this was measured on your hardware.

**The three 🔒 items are among the biggest wins** and are also the ones that trade away safety margin. None of them should be set because a checklist listed them.

**If your hot path uses Onload, DPDK or AF_XDP**, §3–§5 largely do not apply to it — the bypass stack owns the NIC and does its own polling. §0–§2 and §6 still matter in full. Confirm which world you are in before spending a week here.

---

*Orders of magnitude are indicative, not measurements of your hardware — the ratios are stable, the absolutes are not. The kernel tree carries mechanism, not a tuning recipe; any "HFT settings" list handed over without measurement on your own hardware is guesswork.*
