# Millennium — Linux Low Latency Systems Engineer (Hong Kong)

## Learning note & interview prep

`REQ-30783` · Latency Critical Trading team · sourced from the JD PDF

---

## 0. Read the job correctly before you study for it

This is **an infrastructure / SRE role**, not kernel development and not C++ quant development. The evidence is in the JD's own verbs:

> "Support and maintain Linux, virtualization, and related infrastructure" · "Identify, prioritize, and resolve complex incidents and service requests" · "participate in off-hours on-call support" · Ansible · VMware ESXi/vCenter · DNS

What makes it *this* job rather than a normal Linux SRE job is the latency depth stacked on top. Note also that the same team is described as bringing together "experienced C++, Linux, networking, and SRE professionals" — you are being hired as the Linux/SRE leg of that, working *alongside* the C++ people, not as one of them.

**Consequence for how you prepare:** they will test operational depth under pressure — *"the feed is 200µs slower than the B side this morning, what do you do?"* — not *"implement a lock-free ring buffer."* Every topic below should be learned to the level of "I can diagnose this at 07:00 before the open", not "I can recite the theory."

---

## 1. Gap map

Requirements straight from the JD, against the material we already built (`linux-lowlatency-hft-concepts.pdf`):

| JD requirement | Status |
|---|---|
| Performance tuning, kernel parameters, BIOS-level tuning | ✅ **Covered** — Parts I & II are exactly this |
| Low-latency networking | 🟡 **Partly** — concepts covered; multicast not |
| Solarflare OpenOnload | ❌ **Gap** — named explicitly. Highest-value gap |
| Multicast networking | ❌ **Gap** — this is how market data arrives |
| cgroups / cpusets | ❌ **Gap** — and the modern form matters (see §2D) |
| NTP, PTP, PPS, WR | ❌ **Gap** — four named technologies, zero coverage |
| VMware ESXi / vCenter | ❌ **Gap** |
| NVIDIA GPU / CUDA | ❌ **Gap** — lowest weight |
| Python & Bash, Ansible | ⚙️ **Your call** — JD-critical, you know your own level |
| AI assistants in daily engineering | ✅ **You're doing it now** — turn it into a story (§5) |
| RHCE (a plus) | ⚙️ Optional, but see §2H |

**Read of the weighting:** the single densest bullet is *"performance tuning, low-latency networking, Solarflare OpenOnload, multicast networking, cgroups/cpusets, kernel parameters, and BIOS-level tuning."* That one line is the job. Two-thirds of it you now have; Onload, multicast and cpusets are the missing third.

Time sync (NTP/PTP/PPS/WR) is the second densest, and it's an entire discipline you currently have nothing on. Weight your study there.

---

## 2. The study blocks

### A. Kernel, CPU, NUMA and BIOS tuning — ✅ mostly done

You have this in the existing PDF. To convert knowledge into interview answers, do the practical half:

- Run `turbostat` on any Linux box and read the C-state residency columns until they're obvious
- Run `cyclictest` and produce a histogram; change one knob; produce another
- Be able to say out loud *why* p99.9 is the number, in one sentence

**Interview shape:** they will ask you to justify a knob, not list knobs. "Why `skew_tick=1`?" has a real answer; be ready with it.

---

### B. Solarflare OpenOnload — ❌ highest-value gap

**Why it matters:** it is named by product in the JD, which means they run it in production. This is a keyword you either have or don't.

**What it is:** a userspace TCP/UDP stack from Solarflare (now AMD). The `onload` wrapper `LD_PRELOAD`s itself in front of an unmodified application and intercepts socket calls, so the app talks to the NIC directly — no syscall, no softirq, no copy. It is §6 and §7 of your concepts doc made into a product.

**Learn, in this order:**

1. The acceleration model — `onload ./app`, what gets accelerated and what silently falls back to the kernel
2. **`onload_stackdump`** — the diagnostic tool. Listing stacks, reading packet and event counters, spotting an app that *thinks* it's accelerated but isn't. This is the single most interview-relevant piece
3. Spin tuning — `EF_POLL_USEC`, `EF_UDP_RECV_SPIN`, `EF_TCP_RECV_SPIN`. This is the polling-versus-interrupts tradeoff (§6) exposed as env vars
4. `EF_PREFAULT_PACKETS` and buffer pre-allocation — ties to "never `malloc` in the hot path"
5. The layer below: **ef_vi** (raw layer-2 API) and **TCPDirect/zf** — know what they are and when a desk reaches for them, even if you never write to them

**The answer they want to hear:** *"An app is `onload`-wrapped but no faster — first thing I do is `onload_stackdump` to confirm it actually has an accelerated stack, because the wrapper fails open and falls back to the kernel silently."*

---

### C. Multicast networking — ❌ gap, and it's how the money arrives

**Why it matters:** exchange market data is UDP multicast. Every feed handler on that platform is a multicast receiver. "Low latency networking" at a trading firm largely *means* multicast receive.

**Host side:**
- IGMP v2 vs v3, and **SSM** (source-specific multicast) — `MCAST_JOIN_SOURCE_GROUP` vs plain `IP_ADD_MEMBERSHIP`
- `net.ipv4.igmp_max_memberships` — a real production limit when one host joins hundreds of groups (`Documentation/networking/ip-sysctl.rst:1872`)
- `net.ipv4.conf.*.force_igmp_version` (`:1907`) — forcing v2 when a switch misbehaves
- **`rp_filter`** (`:2078`) — the classic multi-homed-host trap: reverse-path filtering silently drops your feed when the data arrives on a different NIC than the route back. Know this one cold, it's a favourite interview question
- `ip maddr show`, `netstat -g`, `ss -u`
- Per-socket vs per-interface joins; `SO_REUSEADDR` / `SO_REUSEPORT` with multiple listeners

**Network side (know the vocabulary, you won't own it):** IGMP snooping, querier, PIM-SM/SSM, rendezvous point.

**Trading-specific:** A/B feed arbitration (exchanges send two identical streams over separate paths), sequence-gap detection, line arbitration, and why a "slow" feed is often actually a *dropped packet + retransmit*, not latency.

**Diagnostics:** `ethtool -S` drop counters, `/proc/net/snmp` and `/proc/net/udp` receive errors, `netdev_max_backlog` overflow, `dropwatch`.

---

### D. cgroups / cpusets — ❌ gap, and the modern form is the point

**Why it matters:** the JD lists `cgroups/cpusets` *separately* from `kernel parameters`. That strongly implies they isolate CPUs at runtime, not only at boot — which is the grown-up way to run a mixed fleet.

**The key fact, verified in this tree** (`Documentation/admin-guide/cgroup-v2.rst:2663`):

> `cpuset.cpus.partition` accepts `"isolated"` — *"Partition root without load balancing"* — and `:2718` confirms those CPUs are *"in an isolated state without any load balancing from the scheduler."*

That is **`isolcpus` at runtime, without a reboot.** There is also `cpuset.cpus.isolated` (`:2656`) showing the current isolated set.

**Learn:**
- cgroup v2 hierarchy, controllers, `cgroup.subtree_control`
- `cpuset.cpus`, `cpuset.mems` — CPU *and* NUMA-node confinement in one place (ties straight to §4 of your concepts doc)
- `cpuset.cpus.partition` = `member` / `root` / `isolated`
- systemd as the front end: `AllowedCPUs=`, `AllowedMemoryNodes=`, slices, `systemd-run --slice=`
- How this interacts with `isolcpus` — and why the boot parameter is increasingly considered legacy

**Interview gold:** *"I'd prefer a cpuset partition in `isolated` mode over `isolcpus`, because I can re-shape isolation per workload without rebooting a trading host."* That sentence says you're current.

---

### E. Time synchronisation: NTP → PTP → PPS → WR — ❌ biggest untouched discipline

**Why it matters:** four named technologies in one bullet. Two independent drivers:
1. **Measurement** — you cannot compare timestamps between two hosts more precisely than they are synchronised. Your entire §9 (measure the tail) collapses without it.
2. **Compliance** — MiFID II RTS 25 requires HFT timestamps traceable to UTC within 100µs. That is a regulatory obligation someone in that team owns.

**The ladder, in accuracy order:**

| Tech | Accuracy | What it is |
|---|---|---|
| **NTP** | ~ms | `chrony` / `ntpd`, over ordinary network |
| **PTP** (IEEE 1588) | sub-µs | Hardware timestamping in the NIC |
| **PPS** | ns-level edge | A 1-pulse-per-second electrical signal, usually from GPS/GNSS |
| **WR** (White Rabbit) | sub-ns | PTP + SyncE + phase detection over fibre; CERN-originated |

**Learn concretely — `linuxptp` is the toolchain:**
- **`ptp4l`** — disciplines the NIC's PTP Hardware Clock (PHC, `/dev/ptp0`) to a grandmaster
- **`phc2sys`** — bridges PHC ↔ system clock. *Nearly every "PTP is broken" ticket is actually phc2sys.*
- **`ts2phc`** — disciplines a PHC from a PPS/GNSS source
- **`pmc`** — management client; query grandmaster identity, port state, offsets
- `ethtool -T ethX` — does this NIC even do hardware timestamping? (Same command as §9 of your concepts doc)
- Clock roles: ordinary / **boundary** / **transparent**; one-step vs two-step
- What to watch: offset-from-master, path delay, servo state `s0→s1→s2`
- Failure modes: asymmetric paths, a switch that isn't a boundary clock, GPS antenna loss, holdover

**In this tree:** `Documentation/driver-api/ptp.rst`, `Documentation/driver-api/pps.rst`, `Documentation/ABI/testing/sysfs-ptp`, and `Documentation/networking/timestamping.rst` (which you already know from §9).

**White Rabbit:** niche. Knowing *what* it is and *why* (deterministic sub-ns over fibre, combining PTP with SyncE physical-layer syntonisation) is sufficient. Don't sink a week into it.

---

### F. VMware ESXi / vCenter for latency — ❌ gap

Counter-intuitive pairing with low latency, which is exactly why it's worth knowing: they clearly run latency-sensitive workloads virtualised.

- **`Latency Sensitivity = High`** — the master switch. Gives exclusive physical-CPU affinity and stops vCPU time-sharing. Requires full memory *and* CPU reservation
- **SR-IOV / DirectPath I/O** — NIC passthrough. Effectively mandatory if Onload runs inside a VM
- Host power policy → High Performance (§0 of your checklist, one layer up)
- vNUMA alignment — §4 applies to VMs too, and gets silently broken by VM resizing
- vNIC coalescing disabled (`ethernetX.coalescingScheme = disabled`)
- `esxtop` — `%RDY` (ready time) and `%CSTP` are your jitter indicators

---

### G. NVIDIA GPU / CUDA — ❌ gap, lowest weight

The JD frames this as *"for Linux infrastructure and performance-sensitive workloads"* — i.e. you support GPU boxes (research, backtesting, possibly ML), you don't write CUDA kernels.

- Driver and toolkit install, version/ABI matching, DKMS
- `nvidia-smi` day-to-day; **persistence mode** (`nvidia-smi -pm 1`) so the driver doesn't re-init per process
- `nvidia-smi topo -m` — PCIe/NVLink topology and NUMA affinity (§4 again)
- MIG and MPS — what they are, when you'd use them
- Common tickets: ECC errors, thermal throttling, Xid errors in `dmesg`

Timebox this. Two focused days, not a week.

---

### H. Ansible, Python, Bash, RHCE — ⚙️ your call, but here's the angle

The JD wants *"stable, repeatable, high-performing Linux systems through automation and code-driven engineering practices."*

**The strongest possible portfolio artifact for this role:** take your own Part II checklist and make it an **Ansible role** — kernel cmdline, sysctls, `ethtool` settings, IRQ affinity, cpuset partitions — with `--check` mode, idempotency, and a verification play that asserts the tuning actually landed (turbostat residency, `/proc/interrupts`, `ethtool -c`). That single repo demonstrates: Linux depth, automation, reproducibility at fleet scale, and the verification discipline. It maps to four JD bullets at once.

**RHCE** (EX294) is Ansible-based, so it and the above reinforce each other. Listed as "a plus" — pursue only if you have spare time after the gaps above.

---

### I. AI assistants — ✅ you are literally doing this

> *"Experience using AI assistants in daily engineering work, including troubleshooting, coding/scripting, terminal workflows, and operational automation"*

This is an unusual bullet and it's a gift, because you have a concrete story rather than a claim. This very session is one: you used an agent in the terminal to navigate the kernel tree, **verified flag names against the source instead of trusting recall**, and caught two pieces of widely-repeated folklore that are wrong — `isolcpus` flag set, and `tcp_low_latency` being a dead no-op.

Tell it that way. The point isn't "I use AI", it's *"I use AI and I verify its output against primary sources"* — which is exactly the judgement an on-call infrastructure engineer needs, and exactly what distinguishes a good answer from a buzzword.

---

## 3. A four-week plan

Weighted by JD density × your current gap.

**Week 1 — Onload + multicast** (the densest bullet)
Onload architecture, `onload_stackdump`, spin env vars. Multicast: IGMP versions, SSM, `rp_filter`, the sysctls above. Build a local multicast sender/receiver pair and deliberately break it with `rp_filter=1` on a multi-homed host until the failure is muscle memory.

**Week 2 — Time sync** (the second densest, and entirely new)
NTP → PTP → PPS → WR ladder. Install `linuxptp`, run `ptp4l` + `phc2sys` even without a grandmaster, read `pmc` output, learn the servo states. Be able to draw the PHC↔system-clock relationship on a whiteboard.

**Week 3 — cgroups/cpusets + ESXi + GPU**
cgroup v2 cpuset partitions hands-on (this one you can practise on any Linux box today). ESXi latency settings — reading depth is fine if you lack a host. GPU: two days, `nvidia-smi` fluency.

**Week 4 — Consolidate and build**
The Ansible role from §2H. Then rehearse §4 out loud. Re-read your own Part I so the *concepts* are what you speak from, not the knob lists.

---

## 4. Questions you will be asked

With the shape of a strong answer — don't memorise these, understand why each one is being asked.

**"A feed handler's p99 latency doubled this morning. Walk me through it."**
They're testing *method*, not a lucky guess. Structure: what changed (deploy? kernel? BIOS? switch?) → is it latency or packet loss and retransmit → check the B feed for comparison → `turbostat` for C-state entry → `/proc/interrupts` for IRQ drift → `onload_stackdump` for fallback to kernel → NIC drop counters. **Say out loud that you compare against the A/B pair and against yesterday** — that's the trading-specific instinct they're listening for.

**"How do you prove a core is actually isolated?"**
Not "I set `isolcpus`". Answer: `/proc/interrupts` LOC row flat, `ps -eLo psr,comm` showing nothing else scheduled there, turbostat residency, and — the senior answer — *"and I'd check whether `irqbalance` is running, because it will quietly undo it."*

**"An app is wrapped in `onload` but isn't faster. Why?"**
`onload_stackdump` first — confirm it's actually accelerated rather than silently fallen back. Then: is it a syscall pattern Onload can't accelerate, is spin time too low, is the NIC even a Solarflare, is it in a VM without passthrough.

**"NTP or PTP, and how would you prove your timestamps to compliance?"**
PTP with NIC hardware timestamping, GPS/PPS-disciplined grandmaster, `phc2sys` bridging to system clock, monitored offset-from-master with alerting and retained history. Mention traceability to UTC and the 100µs MiFID II figure — it shows you understand *why* the requirement exists.

**"Multicast works on host A, not host B. Same config."**
Route back / `rp_filter`, IGMP version mismatch, switch snooping/querier, `igmp_max_memberships` ceiling, wrong source in an SSM join, NIC not in the right VLAN.

**"Why might you *not* disable hyperthreading?"**
Tests whether you cargo-cult. Real answer: it halves core count, and if you have more isolated-core demand than physical cores you may prefer SMT off on trading cores but on elsewhere — it's a per-host decision driven by measurement.

**"You want `sched_rt_runtime_us = -1`. Convince me."**
The right answer includes the risk unprompted: a spinning RT thread can wedge the box, so it only goes on with isolation in place and out-of-band access available. **Volunteering the downside is the answer.**

**"How does this tuning stay correct across 200 hosts?"**
Ansible, `--check` drift detection, and verification that asserts the *outcome* (residency, IRQ placement) rather than the file contents. Then: configuration drift is a latency incident waiting to happen.

---

## 5. Things to do that aren't studying

- **The Ansible role** (§2H). One repo, four bullets.
- **Write up one latency investigation you've actually done** — structure, hypothesis, measurement, resolution. Real war stories beat knob lists in every interview.
- **Prepare the AI story** (§2I) as a 60-second answer with a specific example.
- **Look at the sibling posting** — the JD page lists the same role in Singapore and a *Quantitative Developer, C++ / Low-Latency Systems* in New York. Useful context for how the team is structured, and worth mentioning that you understand where your role sits relative to the C++ side.

---

## 6. Honest caveats

**A learning note cannot get you an offer.** It closes the technical-knowledge gap. The JD also asks for *5+ years in financial technology services, including 3+ years supporting low-latency trading systems* — that's a fact about your CV, not something study changes. If you're under that bar, the lever is how specifically you evidence the low-latency work you *have* done, not more reading.

**I don't know your current level** on any of these. The four-week plan assumes all the ❌ items are genuinely new; skip whole blocks if they aren't, and tell me which — I'll re-weight it.

**I have not verified the vendor-specific details** (Onload env vars, ESXi setting names, linuxptp specifics) against primary documentation the way I verified the kernel flags — those came from general knowledge, and vendor docs change. Check them against AMD's Onload user guide, VMware's performance docs and the linuxptp man pages before you quote any of them in an interview. The kernel-tree citations in §2C, §2D and §2E *are* verified against the source in this repo.

**Treat the confidence levels honestly in the room too.** "I've read about White Rabbit but never run it" is a better answer than a bluff that unravels in two follow-ups. Interviewers at this level are calibrating how much they can trust your reports at 3am.
