---
title: "Process and Service Management — systemd, journalctl, /proc, Signals"
date: 2026-05-08
updated: 2026-05-08
tags: [linux, systemd, journalctl, processes, signals]
---

# Process and Service Management — systemd, journalctl, /proc, Signals

**Date:** 2026-05-08 | **Updated:** 2026-05-08
**Tags:** `linux` `systemd` `journalctl` `processes` `signals`

---

## Table of Contents

- [Summary](#summary)
- [1. systemd Units, Targets, and Dependencies](#1-systemd-units-targets-and-dependencies)
  - [1.1 Unit types and what they manage](#11-unit-types-and-what-they-manage)
  - [1.2 Anatomy of a `.service` file](#12-anatomy-of-a-service-file)
  - [1.3 Dependency directives — `Requires`, `Wants`, `After`, friends](#13-dependency-directives--requires-wants-after-friends)
  - [1.4 Targets — the modern replacement for runlevels](#14-targets--the-modern-replacement-for-runlevels)
  - [1.5 Drop-ins — overriding without forking the unit file](#15-drop-ins--overriding-without-forking-the-unit-file)
  - [1.6 `systemctl` lifecycle and querying](#16-systemctl-lifecycle-and-querying)
  - [1.7 Reading `systemctl status`](#17-reading-systemctl-status)
  - [1.8 Transient units — `systemd-run`](#18-transient-units--systemd-run)
- [2. journalctl — Querying, Filtering, and Persisting Logs](#2-journalctl--querying-filtering-and-persisting-logs)
  - [2.1 Storage modes — volatile, persistent, auto, none](#21-storage-modes--volatile-persistent-auto-none)
  - [2.2 Filters that matter on call](#22-filters-that-matter-on-call)
  - [2.3 Structured fields — trusted vs user-supplied](#23-structured-fields--trusted-vs-user-supplied)
  - [2.4 Output formats](#24-output-formats)
  - [2.5 Retention — `journald.conf`, vacuuming, rotation](#25-retention--journaldconf-vacuuming-rotation)
  - [2.6 Forwarding to syslog](#26-forwarding-to-syslog)
- [3. Process Inspection — ps, top/htop, /proc, lsof](#3-process-inspection--ps-tophtop-proc-lsof)
  - [3.1 `ps` — three syntax styles, one tool](#31-ps--three-syntax-styles-one-tool)
  - [3.2 `top` interactive keys and the `%Cpu` line](#32-top-interactive-keys-and-the-cpu-line)
  - [3.3 `/proc/[pid]/` — the process as a filesystem](#33-procpid--the-process-as-a-filesystem)
  - [3.4 `lsof` — open files and listening sockets](#34-lsof--open-files-and-listening-sockets)
  - [3.5 `pidstat` — per-process metrics over time](#35-pidstat--per-process-metrics-over-time)
- [4. Signals, Process Groups, and Job Control](#4-signals-process-groups-and-job-control)
  - [4.1 Signal table and default dispositions](#41-signal-table-and-default-dispositions)
  - [4.2 What is and isn't catchable](#42-what-is-and-isnt-catchable)
  - [4.3 `kill`, `pkill`, `killall` — and the `kill -0` existence trick](#43-kill-pkill-killall--and-the-kill--0-existence-trick)
  - [4.4 Process groups, sessions, controlling terminals](#44-process-groups-sessions-controlling-terminals)
  - [4.5 `nohup` vs `disown` vs `setsid` vs `systemd-run --scope`](#45-nohup-vs-disown-vs-setsid-vs-systemd-run---scope)
- [5. Operator Walkthrough](#5-operator-walkthrough)
  - [5.1 Write a minimal `.service` and start it](#51-write-a-minimal-service-and-start-it)
  - [5.2 Query its logs](#52-query-its-logs)
  - [5.3 Send a signal that triggers graceful shutdown](#53-send-a-signal-that-triggers-graceful-shutdown)
  - [5.4 Watch the kernel ring buffer for OOM-style events](#54-watch-the-kernel-ring-buffer-for-oom-style-events)
  - [5.5 Inspect `/proc/[pid]/status` for the running process](#55-inspect-procpidstatus-for-the-running-process)
- [Related](#related)
- [References](#references)

## Summary

The on-call operator's view of a Linux box: how services run under systemd, how their logs land in the journal, how to inspect a running process via `/proc`, and how signals actually propagate. Aimed at a backend dev who can already use a shell but wants to answer "why won't this service start, and what is it doing right now?" without guessing. Reproducible commands at the end demonstrate the full loop on a throwaway VM or container.

---

## 1. systemd Units, Targets, and Dependencies

systemd is PID 1 on essentially every modern Linux distribution outside minimal container images. It supervises services, mounts filesystems, runs timers, and orders the boot graph. Everything it manages is a **unit**, defined by an INI-style file.

### 1.1 Unit types and what they manage

The unit type is encoded in the filename suffix:

| Suffix | Manages |
|---|---|
| `.service` | A long-running or one-shot process |
| `.socket` | An IPC or network socket; can socket-activate a paired `.service` |
| `.timer` | A scheduled trigger, replacement for cron entries |
| `.mount` / `.automount` | A filesystem mount point |
| `.target` | A logical grouping point — analogous to a SysV runlevel |
| `.device` | A device unit, populated automatically from udev |
| `.path` | A path watcher that triggers another unit on change |
| `.slice` | A node in the cgroup hierarchy used for resource control |
| `.scope` | A group of externally-created processes adopted into a unit |
| `.swap` | A swap area |

The full list is documented in `systemd.unit(5)`.

### 1.2 Anatomy of a `.service` file

A service has three sections: `[Unit]` (metadata, dependencies), `[Service]` (how to run the process), `[Install]` (how it's wired into targets when enabled).

```ini
[Unit]
Description=Demo HTTP echo service
Documentation=https://example.com/docs
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
ExecStart=/usr/local/bin/demo-echo --port=8080
Restart=on-failure
RestartSec=2s
TimeoutStopSec=10s
User=demo
Group=demo

[Install]
WantedBy=multi-user.target
```

Key `[Service]` directives, per `systemd.service(5)`:

| Directive | Purpose | Notes |
|---|---|---|
| `Type=` | Tells systemd how to consider the service "started" | Default is `simple` when `ExecStart=` is given. `notify` requires `sd_notify(READY=1)`. `forking` requires the daemon to fork and is paired with `PIDFile=`. |
| `ExecStart=` | The command to run | One per service for non-`oneshot` types. |
| `ExecStartPre=` / `ExecStop=` / `ExecReload=` | Pre/stop/reload hooks | Multiple `ExecStartPre` are allowed; failure aborts startup. |
| `Restart=` | When to auto-restart | `no` (default), `on-failure`, `on-abnormal`, `on-watchdog`, `on-abort`, `always`, `on-success`. |
| `KillSignal=` | Stop signal | Default is `SIGTERM`. |
| `TimeoutStopSec=` | Grace period before escalating to `SIGKILL` | After this, systemd sends `SIGKILL`. |
| `WatchdogSec=` | Liveness watchdog timeout | Service must call `sd_notify(WATCHDOG=1)` periodically; missing it triggers `SIGABRT`. |

### 1.3 Dependency directives — `Requires`, `Wants`, `After`, friends

The most-misunderstood part of systemd: **dependency** and **ordering** are independent. `Requires=foo.service` does *not* mean "start `foo` first"; it means "if I'm started, `foo` must also be started." You need `After=foo.service` to actually order the start. Per `systemd.unit(5)`:

| Directive | Effect |
|---|---|
| `Requires=` | Activates the listed units alongside this one. If a required unit fails to start, this unit fails too. |
| `Wants=` | Weaker `Requires=`. Failure of the listed unit does not invalidate this one. The default for most cases. |
| `Requisite=` | The listed units must already be active; otherwise this unit fails immediately (does not start them). |
| `BindsTo=` | Stronger `Requires=`. If the bound unit stops unexpectedly, this unit stops too. |
| `PartOf=` | One-way: `systemctl stop`/`restart` on the parent propagates to this unit. |
| `Upholds=` | Continuously restarts the listed unit while this one is running. |
| `After=` / `Before=` | Pure ordering. `After=foo` means start *after* `foo` reaches its activation point. |
| `Conflicts=` | Starting this unit stops the listed ones, and vice versa. |

Mental model: `Requires` answers *"what must come along?"*; `After` answers *"in what order?"*. Use both when you need both.

### 1.4 Targets — the modern replacement for runlevels

Targets are synchronization points. Boot reaches `multi-user.target` (no GUI) or `graphical.target` (GUI). These successors to SysV runlevels are units with no `ExecStart=`; they exist so other units can declare `WantedBy=multi-user.target`.

When you `systemctl enable foo.service`, systemd reads the `[Install]` section and creates a symlink at `/etc/systemd/system/multi-user.target.wants/foo.service -> /etc/systemd/system/foo.service`. That symlink is what makes `foo.service` start on boot. `disable` removes it. `WantedBy=` populates `<target>.wants/`; `RequiredBy=` populates `<target>.requires/`. `Alias=` and `Also=` create alternative names and pull in additional units at enable time.

### 1.5 Drop-ins — overriding without forking the unit file

Distribution-shipped units live under `/usr/lib/systemd/system/`. Don't edit them — the next package update overwrites your changes. Instead, create a drop-in directory `/etc/systemd/system/<name>.service.d/` containing one or more `.conf` files. systemd merges them on top of the original at load time. `systemctl edit foo.service` opens an editor that writes to `override.conf` under that directory; `--full` makes a complete copy (rarely what you want). Drop-in files in `/etc/` take precedence over `/run/`, which override `/usr/lib/`.

To clear a list-valued setting (like `ExecStart=`) before redefining it, set it to empty:

```ini
[Service]
ExecStart=
ExecStart=/usr/local/bin/foo --new-flag
```

After any unit-file change, run `systemctl daemon-reload` so systemd re-reads the unit graph.

### 1.6 `systemctl` lifecycle and querying

| Command | Effect |
|---|---|
| `systemctl start foo` | Activate the unit now. Does not affect boot. |
| `systemctl stop foo` | Deactivate now. |
| `systemctl restart foo` | Stop, then start. |
| `systemctl reload foo` | Send the configured reload action (often `SIGHUP`). |
| `systemctl enable foo` | Create startup symlinks. Survives reboot. |
| `systemctl enable --now foo` | Enable for boot *and* start immediately. |
| `systemctl disable --now foo` | Remove symlinks *and* stop. |
| `systemctl mask foo` | Symlink to `/dev/null` so the unit cannot be started, even by dependency. |
| `systemctl daemon-reload` | Re-read all unit files after edits. |
| `systemctl cat foo` | Print the merged unit file (original + drop-ins). |
| `systemctl show foo` | Print all properties as `key=value` (machine-readable). |
| `systemctl is-active foo` | Exit 0 if active. |
| `systemctl is-enabled foo` | Exit 0 if enabled. |
| `systemctl is-failed foo` | Exit 0 if in failed state. |

`--user` operates on the per-user manager instead of the system manager. Useful for desktop services and unprivileged developer workflows.

### 1.7 Reading `systemctl status`

```text
● demo-echo.service - Demo HTTP echo service
     Loaded: loaded (/etc/systemd/system/demo-echo.service; enabled; preset: enabled)
     Active: active (running) since Tue 2026-05-08 10:14:22 UTC; 4min 12s ago
   Main PID: 18432 (demo-echo)
      Tasks: 4 (limit: 4915)
     CGroup: /system.slice/demo-echo.service
             └─18432 /usr/local/bin/demo-echo --port=8080
```

- **Loaded** — was the unit file parsed? Values include `loaded`, `error`, `not-found`, `bad-setting`, `masked`. If this isn't `loaded`, nothing else matters.
- **Active** — high-level state: `active`, `inactive`, `activating`, `deactivating`, `failed`. A unit enters `failed` after a non-zero exit, signal kill, or timeout (subject to `Restart=` and `SuccessExitStatus=`).
- **Main PID** — the supervised process. `systemctl kill` targets this PID by default.
- **CGroup** — the cgroup the service's processes live under. All children inherit it; reading `/sys/fs/cgroup/system.slice/demo-echo.service/cgroup.procs` shows the full process set. This is how systemd kills "the whole service" cleanly even when it forks subprocesses.
- The last 10 journal lines for the unit are appended below this header.

### 1.8 Transient units — `systemd-run`

`systemd-run` creates a unit on the fly without writing a file. It's the right tool for one-off jobs, ad-hoc resource limits, and capturing existing processes into a cgroup.

```bash
# Transient service unit, captured in the journal
systemd-run --unit=backup-once.service --description='one-off backup' /usr/local/bin/backup.sh

# Transient scope unit: the calling shell stays the parent, but the
# process is placed in a systemd-managed cgroup with limits applied.
systemd-run --scope --slice=batch.slice -p MemoryMax=512M ./long-job

# Schedule a transient timer
systemd-run --on-active=30s --unit=delayed.service /usr/local/bin/job.sh
```

A **service unit** is forked and supervised by systemd. A **scope unit** wraps already-running processes (the caller stays the parent) but places them in a cgroup so resource limits and `systemctl stop` work. **Transient** means: created at runtime, not persisted to disk, garbage-collected when finished.

---

## 2. journalctl — Querying, Filtering, and Persisting Logs

`systemd-journald` is the journal daemon. It captures stdout/stderr from every supervised service, kernel messages (kmsg), syslog entries, and direct submissions via `sd_journal_send`. Records are structured: each entry is a set of key=value fields, not a free-form line.

### 2.1 Storage modes — volatile, persistent, auto, none

`Storage=` in `/etc/systemd/journald.conf` decides where logs go:

| Value | Behavior |
|---|---|
| `volatile` | In-memory only at `/run/log/journal/`. Lost on reboot. |
| `persistent` | Disk at `/var/log/journal/`, falling back to `/run/log/journal/` early in boot. |
| `auto` | Persistent if `/var/log/journal/` exists, volatile otherwise. (Default on many distros.) |
| `none` | Drop all storage. Forwarding (e.g., to syslog) still works. |

To switch a box from volatile to persistent without touching config: `sudo mkdir -p /var/log/journal && sudo systemctl restart systemd-journald`. The directory's existence flips `auto` mode into persistent.

### 2.2 Filters that matter on call

| Flag | Effect |
|---|---|
| `-u UNIT` | Logs from a systemd unit. Repeatable. |
| `-p PRIO` | Filter by priority (0–7 or names: `emerg`, `alert`, `crit`, `err`, `warning`, `notice`, `info`, `debug`). |
| `--since`, `--until` | Time bounds. Accepts absolute times and relative forms like `"1 hour ago"`, `yesterday`, `-1h`. |
| `-k` | Kernel ring buffer messages. Equivalent to a `_TRANSPORT=kernel` match. |
| `-b [N]` | Filter by boot. `-b 0` is the current boot, `-b -1` the previous, etc. |
| `-f` | Follow (like `tail -f`). |
| `-n N` | Last N entries. Default 10. |
| `-e` | Jump to the end inside the pager. |
| `--user` | Read the calling user's per-user journal. |

Combine freely:

```bash
journalctl -u nginx.service -p err --since "2 hours ago"
journalctl -u demo-echo.service -f
journalctl -k --since "10 minutes ago"
```

### 2.3 Structured fields — trusted vs user-supplied

Every journal entry is a set of fields (`systemd.journal-fields(7)`). User-supplied: `MESSAGE`, `PRIORITY`, `SYSLOG_IDENTIFIER`. Trusted (set by journald, unforgeable): `_PID`, `_UID`, `_GID`, `_COMM`, `_EXE`, `_CMDLINE`, `_SYSTEMD_UNIT`, `_BOOT_ID`, `_MACHINE_ID`, `_HOSTNAME`, `_TRANSPORT` (`journal` / `stdout` / `kernel` / `syslog` / `audit` / `driver`).

The leading underscore marks **trusted** fields. An attacker cannot fake `_PID` or `_SYSTEMD_UNIT`, but `MESSAGE` is whatever the program wrote.

Filter by any field with `FIELD=value`:

```bash
journalctl _PID=18432
journalctl _SYSTEMD_UNIT=demo-echo.service _TRANSPORT=stdout
```

Same field repeated → OR. Different fields → AND. So `_PID=100 _PID=200 _UID=0` means *(PID 100 OR PID 200) AND UID 0*.

### 2.4 Output formats

| Format | Use case |
|---|---|
| `short` | Default, syslog-like single line. |
| `cat` | Just `MESSAGE`. Useful for piping into other tools. |
| `verbose` | Every field, indented. |
| `json` / `json-pretty` | Machine-readable. Long fields appear as `null` once they exceed the size limit. |
| `export` | Binary archive transfer format. |

### 2.5 Retention — `journald.conf`, vacuuming, rotation

Per `journald.conf(5)`:

| Setting | Default | Meaning |
|---|---|---|
| `SystemMaxUse=` | 10% of FS, capped at 4 GiB | Max total disk for persistent journal. |
| `SystemKeepFree=` | 15% of FS, capped at 4 GiB | Reserve free space on the disk for non-journal. |
| `SystemMaxFileSize=` | 1/8 of `SystemMaxUse=` | Max size per individual journal file before rotation. |
| `MaxRetentionSec=` | 0 (disabled) | Hard time-based retention. |
| `Compress=` | yes (threshold 512 B) | Compress field data above the threshold. |

Operator-side commands:

```bash
journalctl --disk-usage                     # how much disk are we using?
sudo journalctl --vacuum-size=500M          # delete oldest archived files until <= 500M
sudo journalctl --vacuum-time=2weeks        # delete files older than 2 weeks
sudo journalctl --rotate                    # close active file, start a new one
sudo journalctl --verify                    # check journal integrity
```

`--vacuum-*` only deletes *archived* files; the active journal is never deleted by vacuum.

### 2.6 Forwarding to syslog

`journald.conf` can push entries elsewhere: `ForwardToSyslog=yes` pipes to rsyslog or syslog-ng over a socket; `ForwardToKMsg=yes` pushes into the kernel ring buffer (rare); `ForwardToConsole=yes` echoes to the system console; `ForwardToWall=yes` wall-messages high-priority entries to logged-in users (default on). In practice: leave the journal as the source of truth, enable `ForwardToSyslog=yes` if rsyslog is shipping to a remote collector, and let rsyslog do the network work.

---

## 3. Process Inspection — ps, top/htop, /proc, lsof

### 3.1 `ps` — three syntax styles, one tool

procps `ps` accepts three independent option families:

| Style | Form | Example |
|---|---|---|
| BSD | No dash | `ps aux` |
| UNIX | Single dash | `ps -ef` |
| GNU long | Double dash | `ps --sort=-%mem` |

They can be mixed but the rules of which dash conflicts with which take a while to internalize. Two phrases worth memorizing: `ps aux` (every process, user-oriented columns including `%CPU` and `%MEM`) and `ps -ef` (every process, full format with `PPID`).

Tree view:

```bash
ps auxf            # BSD forest
ps -ejH            # UNIX hierarchy
```

Key columns:

| Column | Meaning |
|---|---|
| `PID` / `PPID` | Process ID and parent PID. |
| `USER` | Effective user. |
| `%CPU`, `%MEM` | Percent of CPU and percent of physical memory (RSS / total RAM). |
| `VSZ` | Virtual memory size (KiB). Includes mapped-but-unfaulted regions. |
| `RSS` | Resident set size (KiB). Physical memory currently held. |
| `TTY` | Controlling terminal, or `?` if none (daemon). |
| `STAT` | State + flags. |
| `TIME` | Cumulative CPU time. |
| `START` | Process start time. |

`STAT` codes:

| Code | Meaning |
|---|---|
| `R` | Running or runnable. |
| `S` | Interruptible sleep (waiting for an event). |
| `D` | Uninterruptible sleep, usually I/O. Cannot be killed; usually a stuck disk. |
| `Z` | Zombie — exited but parent has not reaped it. |
| `T` | Stopped (job control or trace). |
| `I` | Idle kernel thread. |
| `<` | High priority (negative nice). |
| `N` | Low priority (positive nice). |
| `s` | Session leader. |
| `l` | Multi-threaded. |
| `+` | In the foreground process group on its TTY. |
| `L` | Has pages locked into memory (mlocked). |

Custom columns with `-o`:

```bash
ps -eo pid,ppid,user,stat,rss,vsz,comm,cmd --sort=-rss | head
```

### 3.2 `top` interactive keys and the `%Cpu` line

The `%Cpu(s):` line breaks CPU time into: `us` (user, un-niced), `sy` (kernel), `ni` (niced user), `id` (idle), `wa` (I/O wait), `hi` (hardware IRQ), `si` (software IRQ), `st` (stolen by hypervisor; nonzero only on VMs). High `wa` means disk; high `sy` with mediocre `us` means kernel-side overhead (often syscalls in tight loops); high `st` on a VM means the host is contended.

Useful interactive keys:

| Key | Effect |
|---|---|
| `h` / `q` | Help / quit. |
| `1` | Toggle per-CPU breakdown vs aggregate. |
| `M` / `P` / `T` | Sort by `%MEM` / `%CPU` / cumulative `TIME+`. |
| `c` | Toggle program name vs full command line. |
| `V` | Forest (tree) view. |
| `u` | Filter by user. |
| `f` | Field management — pick visible columns, choose sort key. |
| `k` | Kill (prompts for PID and signal; defaults to `SIGTERM`). |
| `r` | Renice. |
| `W` | Persist the current configuration. |

Per-process columns: `PR` (priority; `rt` = real-time), `NI` (nice), `VIRT` (= `VSZ`), `RES` (= `RSS`), `SHR` (shared portion of RSS), `S` (state), `%CPU`, `%MEM`, `TIME+`. `htop` is a third-party reimplementation with mouse support and an easier kill UI; it reads the same `/proc` data.

### 3.3 `/proc/[pid]/` — the process as a filesystem

Every running process has a directory under `/proc/`. This is where every monitoring tool ultimately reads from.

| Path | Contents |
|---|---|
| `/proc/[pid]/status` | Human-readable summary: name, state, UIDs, memory, signals, capabilities. |
| `/proc/[pid]/stat` | Single-line machine-readable counterpart. Schema in `proc(5)`. |
| `/proc/[pid]/cmdline` | NUL-separated argv. |
| `/proc/[pid]/environ` | NUL-separated environment. Owner-readable only. |
| `/proc/[pid]/exe` | Symlink to the executable. |
| `/proc/[pid]/cwd` | Symlink to current working directory. |
| `/proc/[pid]/root` | Symlink to the process's filesystem root (different inside containers / `chroot`). |
| `/proc/[pid]/fd/` | One symlink per open file descriptor. |
| `/proc/[pid]/maps` | Memory map: each mapped region with permissions and backing file. |
| `/proc/[pid]/smaps` | `maps` plus per-region RSS, PSS, swap, dirty pages. |
| `/proc/[pid]/cgroup` | The cgroup paths the process belongs to. |
| `/proc/[pid]/ns/` | One symlink per namespace (mnt, pid, net, ipc, uts, user, cgroup). |
| `/proc/[pid]/limits` | The process's `getrlimit` view. |
| `/proc/[pid]/oom_score`, `oom_score_adj` | OOM killer score and tunable bias. |
| `/proc/[pid]/task/<tid>/` | Per-thread subdirectories with the same shape. |

Notable fields in `/proc/[pid]/status`:

| Field | Meaning |
|---|---|
| `Name` | First 16 chars of the command (`comm`). |
| `State` | `R`/`S`/`D`/`Z`/`T`/`X` plus a label. |
| `Tgid` | Thread group ID = the PID a user thinks of. |
| `Pid` | Per-thread ID. For a single-threaded process, equals `Tgid`. |
| `PPid` | Parent PID. |
| `TracerPid` | Non-zero if `strace`/`gdb`/another tracer is attached. |
| `Uid` / `Gid` | Real, effective, saved-set, filesystem IDs. |
| `Threads` | Number of threads. |
| `VmRSS`, `VmSize`, `VmPeak`, `VmData`, `VmStk`, `VmExe`, `VmLib` | Memory accounting. |
| `SigPnd`, `ShdPnd`, `SigBlk`, `SigIgn`, `SigCgt` | Pending (per-thread/shared), blocked, ignored, caught signal masks. |
| `CapInh`, `CapPrm`, `CapEff` | Inheritable / permitted / effective POSIX capabilities. |
| `Cpus_allowed`, `Mems_allowed` | CPU and NUMA affinity bitmaps. |
| `voluntary_ctxt_switches`, `nonvoluntary_ctxt_switches` | Voluntary = blocked on its own; nonvoluntary = preempted. A high nonvoluntary count signals CPU contention. |

### 3.4 `lsof` — open files and listening sockets

"Everything is a file" pays off here: `lsof` lists open files of every kind — regular, directory, pipe, socket, device.

| Flag | Use |
|---|---|
| `-p PID` | Files held by a PID. |
| `-i` / `-i :8080` / `-i TCP` | All Internet sockets / by port / by protocol. |
| `-u USER` / `-c COMM` | Filter by user / by command-name prefix. |
| `+D DIR` | Walk a directory tree (slow). |
| `+L1` | Files with link count < 1: deleted but still held open (the classic "disk full but `du` shows nothing" cause). |
| `-n` / `-P` | Skip DNS / skip `/etc/services` port-to-name lookup. |

The `FD` column shows: `cwd` (current working dir), `rtd` (filesystem root), `txt` (executable text), `mem` (mmap'd file), or a numbered fd with a mode suffix (`r` read, `w` write, `u` read+write). Two recurring on-call patterns:

```bash
sudo lsof -i :443 -nP                  # who is on 443?
sudo lsof +L1 | head                   # who's holding deleted files open?
```

### 3.5 `pidstat` — per-process metrics over time

`pidstat`, from the `sysstat` package, samples per-process counters at an interval — the per-PID equivalent of `vmstat`/`iostat`. Flags: `-u` CPU, `-r` memory and page faults, `-d` block I/O, `-w` context switches, `-t` per-thread, `-p PID` (or `ALL`). Example — sample CPU and I/O for PID 18432 every 2 s, 5 times:

```bash
pidstat -u -d -p 18432 2 5
```

Friendlier than re-reading `/proc/[pid]/stat` and diffing by hand.

---

## 4. Signals, Process Groups, and Job Control

A signal is an asynchronous notification delivered to a process. The kernel may queue it; the process may handle, ignore, or block it (for most signals).

### 4.1 Signal table and default dispositions

From `signal(7)`. Defaults: **Term** = terminate, **Core** = terminate + core dump, **Ign** = ignore, **Stop** = pause, **Cont** = resume.

| Signal | x86 # | Default | Typical use |
|---|---|---|---|
| `SIGHUP` | 1 | Term | Controlling terminal closed; conventionally repurposed to mean "reload config." |
| `SIGINT` | 2 | Term | Keyboard `Ctrl-C`. |
| `SIGQUIT` | — | Core | Keyboard `Ctrl-\`. Forces a core dump. |
| `SIGKILL` | 9 | Term | Cannot be caught, blocked, or ignored. Last resort. |
| `SIGTERM` | 15 | Term | Polite shutdown. Default for `kill`. |
| `SIGSTOP` | 19 | Stop | Cannot be caught, blocked, or ignored. Pauses. |
| `SIGCONT` | — | Cont | Resume a stopped process. |
| `SIGCHLD` | — | Ign | Child changed state (default ignored; usually handled by parents). |
| `SIGUSR1` / `SIGUSR2` | — | Term | User-defined. |
| `SIGPIPE` | — | Term | Wrote to a pipe with no readers (the classic broken-pipe early-exit). |
| `SIGSEGV` | — | Core | Invalid memory access. |

Numeric values for `SIGTERM`, `SIGKILL`, `SIGINT`, `SIGHUP`, `SIGSTOP` are stable on Linux x86. Other architectures may differ; prefer symbolic names in scripts.

### 4.2 What is and isn't catchable

**`SIGKILL` and `SIGSTOP` cannot be caught, blocked, or ignored.** They are kernel-enforced. This is why `kill -9` is the hammer — and why a process stuck in `D` state (uninterruptible sleep) will not respond even to `SIGKILL` until the kernel returns from the syscall it's blocked in.

Every other signal can be:
- **Caught** — the process registered a handler (`signal()`/`sigaction()`).
- **Ignored** — explicitly set to `SIG_IGN`.
- **Blocked** — temporarily masked; the kernel queues it and delivers later.

`SigBlk`, `SigIgn`, `SigCgt` in `/proc/[pid]/status` reveal exactly which signals a process is blocking, ignoring, or catching. Each is a hex bitmap; `bit n` set = signal n+1 is in that set.

### 4.3 `kill`, `pkill`, `killall` — and the `kill -0` existence trick

| Command | What it does |
|---|---|
| `kill PID` | Send `SIGTERM` to PID. |
| `kill -9 PID` / `kill -KILL PID` / `kill -s KILL PID` | Send `SIGKILL`. |
| `kill -l` | List signal names. |
| `kill -- -PGID` | Send to every process in process group `PGID`. The leading `--` and negative number are required. |
| `kill 0` | Send to every process in the caller's own process group. |
| `kill -0 PID` | **Don't send anything**. Just check that PID exists and the caller has permission to signal it. Returns 0 on success, non-zero otherwise. |
| `pkill -f pattern` | Match against full command line, then signal. |
| `pkill -u alice nginx` | Match by user *and* name. |
| `killall name` | Signal every process with this exact name. Different from `pkill` in matching semantics. |

`kill -0` is the canonical way to write a "is this PID alive?" health check in a shell script.

The `kill(2)` syscall (`man 2 kill`) defines the negative-PID semantics: `pid < -1` sends to the process group whose ID is `-pid`; `pid == -1` broadcasts to every signalable process except init.

### 4.4 Process groups, sessions, controlling terminals

Per `credentials(7)`: a **process group** is a set of related processes sharing a PGID — shells create one per pipeline so `Ctrl-C` can signal them all at once via `kill -- -PGID`. A **session** is a set of process groups sharing an SID; it maps to a single login or a daemon's standalone world. A **session leader** is the process that called `setsid()`; its PID becomes the SID. A **controlling terminal** is the TTY associated with a session; job control (`Ctrl-Z`, fg, bg) flows through it. A daemon, by definition, has no controlling terminal — it (or its starter, like systemd) called `setsid()` so the parent shell's hangup can't reach it. View PGID and SID with `ps -o pid,pgid,sid,tty,stat,cmd`.

### 4.5 `nohup` vs `disown` vs `setsid` vs `systemd-run --scope`

All four address "let this process survive my shell exiting," but they're different:

| Tool | What it actually does |
|---|---|
| `nohup CMD` | Sets `SIGHUP` to ignore for `CMD`, redirects stdout/stderr to `nohup.out` if they're a TTY. Still shares the shell's session. |
| `disown` (bash builtin) | Removes a job from the shell's job table so the shell won't send `SIGHUP` on exit. Doesn't change the kernel signal mask. |
| `setsid CMD` | Starts `CMD` in a new session with no controlling terminal. The shell exiting can't deliver `SIGHUP` because the relationship is gone. |
| `systemd-run --scope CMD` | Same goal but the process lands in a systemd-managed cgroup, is logged to the journal, and is inspectable as a unit. The right tool on a systemd box. |

Prefer `systemd-run --scope` (or `tmux`/`screen`) for long jobs on a server. `nohup ... &` is the answer when you don't have those, but it leaves orphans that are awkward to clean up.

---

## 5. Operator Walkthrough

Reproducible on a throwaway VM (Debian/Ubuntu/Fedora) or a privileged container with systemd available (`fedora` images run systemd; minimal Debian images don't by default — use a VM if unsure).

### 5.1 Write a minimal `.service` and start it

A two-second loop that logs to stdout and handles `SIGTERM` cleanly. Save as `/usr/local/bin/demo-loop`:

```bash
#!/usr/bin/env bash
set -eu
cleanup() { echo "demo-loop: caught SIGTERM, draining 1s"; sleep 1; exit 0; }
trap cleanup TERM
echo "demo-loop: starting, pid=$$"
i=0
while true; do i=$((i + 1)); echo "demo-loop: tick #$i"; sleep 2; done
```

`sudo install -m 0755 demo-loop /usr/local/bin/demo-loop`. Then the unit, at `/etc/systemd/system/demo-loop.service`:

```ini
[Unit]
Description=Demo heartbeat loop
After=network-online.target

[Service]
Type=simple
ExecStart=/usr/local/bin/demo-loop
Restart=on-failure
RestartSec=2s
TimeoutStopSec=5s

[Install]
WantedBy=multi-user.target
```

Load and start:

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now demo-loop.service
systemctl status demo-loop.service
```

Expected status header:

```text
● demo-loop.service - Demo heartbeat loop
     Loaded: loaded (/etc/systemd/system/demo-loop.service; enabled)
     Active: active (running) since Tue 2026-05-08 11:02:14 UTC; 3s ago
   Main PID: 19821 (demo-loop)
     CGroup: /system.slice/demo-loop.service
             ├─19821 /bin/bash /usr/local/bin/demo-loop
             └─19842 sleep 2
```

### 5.2 Query its logs

```bash
journalctl -u demo-loop.service -n 5 --no-pager
```

```text
May 08 11:02:14 host01 systemd[1]: Started demo-loop.service - Demo heartbeat loop.
May 08 11:02:14 host01 demo-loop[19821]: demo-loop: starting, pid=19821
May 08 11:02:14 host01 demo-loop[19821]: demo-loop: tick #1
May 08 11:02:16 host01 demo-loop[19821]: demo-loop: tick #2
May 08 11:02:18 host01 demo-loop[19821]: demo-loop: tick #3
```

Follow live; structured output:

```bash
journalctl -u demo-loop.service -f
journalctl -u demo-loop.service -n 1 --output=json-pretty
```

Filter to last 5 minutes and only what came in via stdout:

```bash
journalctl -u demo-loop.service --since "5 minutes ago" _TRANSPORT=stdout
```

### 5.3 Send a signal that triggers graceful shutdown

Send `SIGTERM` via `systemctl` (which finds the main PID and respects the cgroup):

```bash
sudo systemctl kill --signal=SIGTERM demo-loop.service
journalctl -u demo-loop.service -n 4 --no-pager
```

Expected last entries:

```text
demo-loop[19821]: demo-loop: tick #95
demo-loop[19821]: demo-loop: caught SIGTERM, draining for 1s, then exiting
systemd[1]: demo-loop.service: Deactivated successfully.
systemd[1]: demo-loop.service: Consumed ... CPU time.
```

The trap handler ran because `SIGTERM` is catchable. Try `SIGKILL` instead and notice no drain log appears — the kernel kills the process before bash runs the trap:

```bash
sudo systemctl start demo-loop.service
sleep 4
sudo systemctl kill --signal=SIGKILL demo-loop.service
journalctl -u demo-loop.service -n 3 --no-pager
```

### 5.4 Watch the kernel ring buffer for OOM-style events

`journalctl -k` is the modern `dmesg`. To synthesize an OOM-killer scenario in a memory-capped scope:

```bash
# Allocate way more than the cap. The cgroup OOM killer terminates the scope.
sudo systemd-run --scope -p MemoryMax=64M --unit=oom-demo.scope \
    bash -c 'a=(); while true; do a+=( $(head -c 1M /dev/urandom | base64) ); done'

journalctl -k --since "1 minute ago" | grep -iE 'oom|killed'
```

Typical lines (formatting varies by kernel version):

```text
kernel: Memory cgroup out of memory: Killed process 20184 (bash) total-vm:...kB, anon-rss:...kB ...
kernel: Out of memory: Killed process ...
```

`/proc/[pid]/oom_score` shows the kernel's per-process OOM ranking; `oom_score_adj` (range -1000 to 1000) lets you bias it. systemd exposes this as `OOMScoreAdjust=` in `[Service]`.

### 5.5 Inspect `/proc/[pid]/status` for the running process

Re-start the unit, then read the kernel's view:

```bash
sudo systemctl start demo-loop.service
PID=$(systemctl show -p MainPID --value demo-loop.service)
sudo grep -E '^(Name|State|Tgid|PPid|Uid|Threads|VmRSS|SigBlk|SigIgn|SigCgt|voluntary_ctxt_switches|nonvoluntary_ctxt_switches):' /proc/$PID/status
```

Sample output:

```text
Name:	demo-loop
State:	S (sleeping)
Tgid:	21002
PPid:	1
Uid:	0	0	0	0
Threads:	1
VmRSS:	   3848 kB
SigBlk:	0000000000000000
SigIgn:	0000000000384004
SigCgt:	0000000000010002
voluntary_ctxt_switches:	18
nonvoluntary_ctxt_switches:	0
```

`State: S` = sleeping in `sleep 2`. `PPid: 1` = systemd is the parent, as expected for a service. `SigCgt` has bits set for the signals bash catches — `SIGTERM` is among them, confirming the trap is wired up. Other useful pokes: `ls -l /proc/$PID/fd` (open fds), `cat /proc/$PID/cgroup` (confirms membership in `demo-loop.service`), `cat /proc/$PID/limits`, `readlink /proc/$PID/exe`, `lsof -p $PID`.

This is the on-call workflow in miniature. When a service won't start: `systemctl status` first (loaded? active state? exit code?), then `journalctl -u <unit> -n 100` for the actual error. If it *is* running but misbehaving, drop into `/proc/[pid]/` and `lsof -p` to see what it's doing.

---

## Related

- [Operating Systems — Process and Thread Model](../../operating-systems/fundamentals/01-process-thread-model.md) — theory behind PIDs, TGIDs, and the task struct `/proc` exposes.
- [Operating Systems — Syscalls and the ABI](../../operating-systems/fundamentals/05-syscalls-and-the-abi.md) — `kill(2)`, `signal()`, and how signal delivery crosses the user/kernel boundary.
- [Operating Systems — File Descriptors and ulimits](../../operating-systems/fundamentals/06-file-descriptors-and-ulimits.md) — what `lsof` is enumerating.
- [Operating Systems — CPU Scheduling (CFS)](../../operating-systems/fundamentals/04-cpu-scheduling-cfs.md) — what the `R`/`S`/`D` states mean to the scheduler.
- [Observability — Structured Logging](../../observability/fundamentals/05-structured-logging.md) — the journal's structured-fields model is the systemd-native take on the same idea.
- [Performance — Flame Graphs (CPU and Off-CPU)](../../performance/fundamentals/05-flame-graphs-cpu-and-off-cpu.md) — next step once `top` and `pidstat` point at a hot process.
- [Kubernetes — Pods and Deployments](../../kubernetes/workloads/pods-and-deployments.md) — pods are cgroup + namespace bundles; these primitives are what kubelet ultimately drives.

## References

- `systemctl(1)`, man7.org: <https://man7.org/linux/man-pages/man1/systemctl.1.html>
- `systemd.unit(5)`, man7.org: <https://man7.org/linux/man-pages/man5/systemd.unit.5.html>
- `systemd.service(5)`, man7.org: <https://man7.org/linux/man-pages/man5/systemd.service.5.html>
- `systemd-run(1)`, man7.org: <https://man7.org/linux/man-pages/man1/systemd-run.1.html>
- `journalctl(1)`, man7.org: <https://man7.org/linux/man-pages/man1/journalctl.1.html>
- `journald.conf(5)`, man7.org: <https://man7.org/linux/man-pages/man5/journald.conf.5.html>
- `systemd.journal-fields(7)`, man7.org: <https://man7.org/linux/man-pages/man7/systemd.journal-fields.7.html>
- `signal(7)`, man7.org: <https://man7.org/linux/man-pages/man7/signal.7.html>
- `kill(2)`, man7.org: <https://man7.org/linux/man-pages/man2/kill.2.html>
- `kill(1)`, man7.org: <https://man7.org/linux/man-pages/man1/kill.1.html>
- `credentials(7)`, man7.org: <https://man7.org/linux/man-pages/man7/credentials.7.html>
- `ps(1)` (procps-ng), man7.org: <https://man7.org/linux/man-pages/man1/ps.1.html>
- `top(1)`, man7.org: <https://man7.org/linux/man-pages/man1/top.1.html>
- `lsof(8)`, man7.org: <https://man7.org/linux/man-pages/man8/lsof.8.html>
- `pidstat(1)`, man7.org: <https://man7.org/linux/man-pages/man1/pidstat.1.html>
- `proc_pid_status(5)`, man7.org: <https://man7.org/linux/man-pages/man5/proc_pid_status.5.html>
