# Linux Documentation Index — Learning Path

A progressive path through practical Linux for backend engineers. Distinct from the [Operating Systems learning path](../operating-systems/INDEX.md) — that path is theory (process model, virtual memory, schedulers, syscalls); this path is the operator's view (systemd, networking, diagnostics, shell). The two are designed to be read together: the OS path tells you *why*, the Linux path tells you *what to type and what to look at when things break*.

Cross-references to the [Operating Systems learning path](../operating-systems/INDEX.md) (theory counterpart — process/thread model, VM, page cache), the [Networking learning path](../networking/INDEX.md) (the protocols behind `ip`, `ss`, `tcpdump`), the [Kubernetes learning path](../kubernetes/INDEX.md) (cgroups, namespaces, container runtimes — Linux primitives that K8s composes), the [Observability learning path](../observability/INDEX.md) (`journalctl`, structured logs, host-level metrics), the [Performance learning path](../performance/INDEX.md) (`perf`, `bpftrace`, flame graphs from the system view), the [Security learning path](../security/INDEX.md) (SELinux/AppArmor, seccomp, audit), and the [Web Scraping learning path](../web-scraping/INDEX.md) (scraper hosts, egress IP control) where topics overlap.

**Markers:** **★** = core must-learn (everyday on-call and operator work). **○** = supporting deep-dive (specialized subsystems, distro-specific tooling). Internalize all ★ before going deep on ○.

---

## Tier 1 — Process and Service Management

How services run, supervised, and survive reboots on a modern Linux box.

- ★ [Process and Service Management — systemd, journalctl, /proc, Signals](process-service/process-and-service-management.md) _(2026-05-08)_
  - Covers: systemd units/targets/dependencies, `journalctl` querying & persistence, process inspection (`ps`, `top`/`htop`, `/proc`, `lsof`), signals & process groups & job control

---

## Tier 2 — Networking Tools

The operator's view of the network. Successor commands to the deprecated `net-tools` (`ifconfig`, `route`, `netstat`).

- ★ [Networking Tools — ip, ss, iptables/nftables, tcpdump, DNS, netns](networking/networking-tools.md) _(2026-05-07)_
  - Covers: `iproute2` (`ip`, `ss`), `iptables`/`nftables` filtering and NAT, `tcpdump` capture & read, DNS tools (`dig`, `getent`, `/etc/nsswitch.conf`), network namespaces (`ip netns`)

---

## Tier 3 — Filesystem and Storage

Mounts, block devices, and how to not lose data.

- ★ [Filesystem and Storage — Mounts, LVM, Disk Tools, ext4/XFS/Btrfs](filesystem/filesystem-and-storage.md) _(2026-05-07)_
  - Covers: VFS layer & `fstab`, LVM (PV/VG/LV) mechanics, disk tooling (`df`, `du`, `lsblk`, `blkid`, `fstrim`), practical differences between ext4, XFS, and Btrfs

---

## Tier 4 — Diagnostics and Tracing

When something is wrong and you don't know what.

- ○ [Diagnostics and Tracing — strace, perf, bpftrace, ftrace, Memory](diagnostics/diagnostics-and-tracing.md) _(2026-05-07)_
  - Covers: `strace`/`ltrace` for syscall and library tracing, `perf` CPU profiling and PMU counters, `bpftrace`/bcc eBPF tracing, `ftrace` and `/sys/kernel/debug/tracing`, memory diagnostics (`free`, `smem`, `/proc/meminfo`, OOM killer)

---

## Tier 5 — Security and Isolation

The operator-visible surface of the security primitives.

- ○ [Security and Isolation — SELinux, AppArmor, seccomp, auditd, Capabilities](security/security-and-isolation.md) _(2026-05-07)_
  - Covers: SELinux modes/policies/`audit2allow`, AppArmor profile-based MAC, `seccomp` syscall allowlists for containers, `auditd` audit trail and watch rules, POSIX file capabilities and container drops

---

## Tier 6 — Shell and Automation

Bash deeply, the small tools that beat scripts in another language, and how to schedule them.

- ★ [Shell and Automation — Bash Internals, awk/sed, xargs/parallel, Scheduling](shell-automation/shell-and-automation.md) _(2026-05-07)_
  - Covers: bash quoting/expansion/process substitution/`set -euo pipefail`, `awk` and `sed` text processing, composition with `xargs`/GNU parallel/`find -exec`, scheduling comparison across `cron`, systemd timers, and `at`

---

## Quick Reference by Topic

### Process and Service

- [systemd](process-service/process-and-service-management.md#1-systemd-units-targets-and-dependencies) _(2026-05-08)_
- [journalctl](process-service/process-and-service-management.md#2-journalctl--querying-filtering-and-persisting-logs) _(2026-05-08)_
- [Process Inspection](process-service/process-and-service-management.md#3-process-inspection--ps-tophtop-proc-lsof) _(2026-05-08)_
- [Signals](process-service/process-and-service-management.md#4-signals-process-groups-and-job-control) _(2026-05-08)_

### Networking

- [ip and ss](networking/networking-tools.md#1-ip-and-ss--the-iproute2-replacements-for-ifconfignetstat) _(2026-05-07)_
- [iptables / nftables](networking/networking-tools.md#2-iptables-and-nftables--packet-filtering-and-nat) _(2026-05-07)_
- [tcpdump](networking/networking-tools.md#3-tcpdump-and-wireshark--capturing-and-reading-traffic) _(2026-05-07)_
- [DNS Tools](networking/networking-tools.md#4-dns-tools--dig-drill-getent-etcnsswitchconf) _(2026-05-07)_
- [Network Namespaces](networking/networking-tools.md#5-network-namespaces--ip-netns-and-the-container-networking-primitive) _(2026-05-07)_

### Filesystem

- [Mounts and fstab](filesystem/filesystem-and-storage.md#1-mounts-fstab-and-the-vfs-layer) _(2026-05-07)_
- [LVM](filesystem/filesystem-and-storage.md#2-lvm--physical-volumes-volume-groups-logical-volumes) _(2026-05-07)_
- [Disk Tools](filesystem/filesystem-and-storage.md#3-disk-and-filesystem-tools--df-du-lsblk-blkid-fstrim) _(2026-05-07)_
- [Filesystem Comparison](filesystem/filesystem-and-storage.md#4-ext4-xfs-and-btrfs--practical-differences) _(2026-05-07)_

### Diagnostics

- [strace and ltrace](diagnostics/diagnostics-and-tracing.md#1-strace-and-ltrace--system-call-and-library-call-tracing) _(2026-05-07)_
- [perf](diagnostics/diagnostics-and-tracing.md#2-perf--cpu-profiling-and-hardware-counters) _(2026-05-07)_
- [bpftrace](diagnostics/diagnostics-and-tracing.md#3-bpftrace-and-bcc--ebpf-for-production-tracing) _(2026-05-07)_
- [ftrace](diagnostics/diagnostics-and-tracing.md#4-ftrace-and-the-syskerneldebugtracing-tree) _(2026-05-07)_
- [Memory Diagnostics](diagnostics/diagnostics-and-tracing.md#5-memory-diagnostics--free-smem-procmeminfo-oom-killer-logs) _(2026-05-07)_

### Security

- [SELinux](security/security-and-isolation.md#1-selinux--modes-policies-and-common-audit2allow-loops) _(2026-05-07)_
- [AppArmor](security/security-and-isolation.md#2-apparmor--profile-based-mac-the-ubuntususe-default) _(2026-05-07)_
- [seccomp](security/security-and-isolation.md#3-seccomp--syscall-allowlists-for-containers) _(2026-05-07)_
- [auditd](security/security-and-isolation.md#4-auditd--audit-trail-and-filesystem-watch-rules) _(2026-05-07)_
- [Capabilities](security/security-and-isolation.md#5-capabilities--posix-file-capabilities-and-container-drops) _(2026-05-07)_

### Shell and Automation

- [Bash Internals](shell-automation/shell-and-automation.md#1-bash-internals--quoting-expansion-process-substitution-set--euo-pipefail) _(2026-05-07)_
- [awk and sed](shell-automation/shell-and-automation.md#2-awk-and-sed--the-text-processing-pair-worth-knowing) _(2026-05-07)_
- [xargs and parallel](shell-automation/shell-and-automation.md#3-xargs-parallel-and-find--exec--composition-patterns) _(2026-05-07)_
- [Scheduling](shell-automation/shell-and-automation.md#4-cron-systemd-timers-and-at--scheduling-comparison) _(2026-05-07)_

---

## Bug Spotting

Active-recall practice docs will land here once the tiers above are populated. Pattern matches the rest of the repo: 22+ broken snippets organized by difficulty (Easy / Subtle / Senior trap), one-line `<details>` hints inline, full root-cause + fix in a Solutions section. Every bug cites a real reference (CVE, manpage gotcha, kernel-mailing-list thread, distro changelog, postmortem).

---

## Related Learning Paths

- [Operating Systems — process/thread model, VM, page cache](../operating-systems/INDEX.md) — the theory this path operationalizes
- [Networking — TCP, DNS, HTTP](../networking/INDEX.md) — the protocols the tools in Tier 2 inspect
- [Kubernetes — cgroups, namespaces, container runtime](../kubernetes/INDEX.md) — Linux primitives composed into a platform
- [Observability — RED/USE, structured logs](../observability/INDEX.md) — host-level metrics and journal pipelines
- [Performance — flame graphs, profiling](../performance/INDEX.md) — system-view counterpart to Tier 4
- [Security — OWASP, threat modeling, secret management](../security/INDEX.md) — SELinux/AppArmor sit beneath the Spring Security/Vault patterns there
