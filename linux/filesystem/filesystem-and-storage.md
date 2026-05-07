---
title: "Filesystem and Storage — Mounts, LVM, Disk Tools, ext4/XFS/Btrfs"
date: 2026-05-07
updated: 2026-05-07
tags: [linux, filesystem, lvm, ext4, xfs, btrfs]
---

# Filesystem and Storage — Mounts, LVM, Disk Tools, ext4/XFS/Btrfs

**Date:** 2026-05-07 | **Updated:** 2026-05-07
**Tags:** `linux` `filesystem` `lvm` `ext4` `xfs` `btrfs`

---

## Table of Contents

1. [Mounts, fstab, and the VFS Layer](#1-mounts-fstab-and-the-vfs-layer)
   - 1.1 [What VFS abstracts](#11-what-vfs-abstracts)
   - 1.2 [The mount table: `/proc/mounts` vs `/etc/mtab`](#12-the-mount-table-procmounts-vs-etcmtab)
   - 1.3 [`findmnt` and `mount`](#13-findmnt-and-mount)
   - 1.4 [`/etc/fstab` columns and common options](#14-etcfstab-columns-and-common-options)
   - 1.5 [Bind mounts and tmpfs](#15-bind-mounts-and-tmpfs)
   - 1.6 [Remount read-only and `umount` busy errors](#16-remount-read-only-and-umount-busy-errors)
   - 1.7 [systemd `.mount` units](#17-systemd-mount-units)
2. [LVM — Physical Volumes, Volume Groups, Logical Volumes](#2-lvm--physical-volumes-volume-groups-logical-volumes)
   - 2.1 [The three-layer model](#21-the-three-layer-model)
   - 2.2 [Creating a stack from scratch](#22-creating-a-stack-from-scratch)
   - 2.3 [Extending a VG and growing an LV](#23-extending-a-vg-and-growing-an-lv)
   - 2.4 [Snapshots: copy-on-write semantics](#24-snapshots-copy-on-write-semantics)
   - 2.5 [Thin pools and thin provisioning](#25-thin-pools-and-thin-provisioning)
   - 2.6 [Inspecting the device-mapper tree](#26-inspecting-the-device-mapper-tree)
3. [Disk and Filesystem Tools — df, du, lsblk, blkid, fstrim](#3-disk-and-filesystem-tools--df-du-lsblk-blkid-fstrim)
   - 3.1 [`df -h` vs `df -i`: blocks vs inodes](#31-df--h-vs-df--i-blocks-vs-inodes)
   - 3.2 [`du` for finding what filled up](#32-du-for-finding-what-filled-up)
   - 3.3 [`lsblk` and `blkid`](#33-lsblk-and-blkid)
   - 3.4 [`fstrim` and SSD discard semantics](#34-fstrim-and-ssd-discard-semantics)
   - 3.5 [`iostat` and `smartctl`](#35-iostat-and-smartctl)
4. [ext4, XFS, and Btrfs — Practical Differences](#4-ext4-xfs-and-btrfs--practical-differences)
   - 4.1 [ext4: defaults and journal modes](#41-ext4-defaults-and-journal-modes)
   - 4.2 [XFS: allocation groups and no-shrink](#42-xfs-allocation-groups-and-no-shrink)
   - 4.3 [Btrfs: subvolumes, snapshots, CoW](#43-btrfs-subvolumes-snapshots-cow)
   - 4.4 [Comparison table](#44-comparison-table)
   - 4.5 [Online vs offline resize](#45-online-vs-offline-resize)
   - 4.6 [fsck behavior at boot and recovery tooling](#46-fsck-behavior-at-boot-and-recovery-tooling)
5. [Operator Walkthrough](#5-operator-walkthrough)
   - 5.1 [Setup: loop devices as fake disks](#51-setup-loop-devices-as-fake-disks)
   - 5.2 [Build PV → VG → LV, format, mount](#52-build-pv--vg--lv-format-mount)
   - 5.3 [Snapshot, fill the LV, observe `df` vs `df -i`](#53-snapshot-fill-the-lv-observe-df-vs-df--i)
   - 5.4 [Grow the LV and resize the filesystem online](#54-grow-the-lv-and-resize-the-filesystem-online)
   - 5.5 [Cleanup](#55-cleanup)
6. [Related](#related)
7. [References](#references)

## Summary

The Linux storage stack is layered: a block device (real disk, partition, or loopback) is wrapped by LVM (PV → VG → LV) or used directly, formatted with a filesystem (ext4, XFS, Btrfs), and exposed through the VFS via a mount. When a server fills up at 3 a.m., when a mount fails on boot, or when a volume needs to grow without downtime, knowing where each layer lives — and which tool inspects it — is the difference between a five-minute fix and an outage.

---

## 1. Mounts, fstab, and the VFS Layer

### 1.1 What VFS abstracts

The Virtual File System (VFS) is the kernel layer that gives every filesystem the same interface — `open`, `read`, `write`, `stat`, `unlink`. Whether the underlying filesystem is ext4 on a SATA SSD, XFS on an NVMe, NFS over the network, or tmpfs in RAM, userspace sees a single rooted directory tree.

A *mount* attaches a filesystem to a path in that tree. The same backing device can be mounted in multiple places (bind mounts), and multiple filesystems can be stacked at different paths.

### 1.2 The mount table: `/proc/mounts` vs `/etc/mtab`

There are historically two views of "what is mounted":

- `/etc/mtab` — historical userspace-maintained file written by `mount(8)`. On modern systems it is a symlink to `/proc/self/mounts` so it always reflects kernel truth.
- `/proc/mounts` — kernel's authoritative view. Always accurate even if userspace state is corrupted.

Always trust `/proc/mounts` (or `findmnt`) over `/etc/mtab` for diagnosis.

```bash
# Authoritative current mounts
cat /proc/mounts

# Tree view, much easier to read
findmnt
```

### 1.3 `findmnt` and `mount`

`findmnt(8)` reads from `/proc/self/mountinfo` and renders a tree. It is the operator-friendly replacement for `mount` with no arguments.

```bash
# Show all mounts as a tree
findmnt

# Inspect a single mount, including options
findmnt /var

# Show what would be mounted from fstab without mounting it
findmnt --verify --verbose
```

`mount(8)` without arguments still works but produces a flat list. With arguments it performs the mount itself, consulting `/etc/fstab` for any field not given on the command line.

### 1.4 `/etc/fstab` columns and common options

`fstab(5)` describes the static filesystem table. Six whitespace-separated fields per line:

```text
<spec>          <mountpoint>    <fstype>    <options>           <dump>  <pass>
UUID=abc-123    /               ext4        defaults,noatime    0       1
UUID=def-456    /home           xfs         defaults            0       2
tmpfs           /tmp            tmpfs       defaults,size=2G    0       0
/swap.img       none            swap        sw                  0       0
```

| Field | Meaning |
|---|---|
| `spec` | What to mount: `UUID=`, `LABEL=`, `/dev/...`, NFS share, etc. Prefer UUIDs over `/dev/sdaN` because device names are not stable across reboots. |
| `mountpoint` | Where to mount it. Must already exist as a directory. |
| `fstype` | Filesystem type: `ext4`, `xfs`, `btrfs`, `tmpfs`, `nfs`, `auto`. |
| `options` | Comma-separated mount options (see below). |
| `dump` | Used by `dump(8)`. Almost always `0`. |
| `pass` | `fsck` order at boot: `1` for root, `2` for others, `0` to skip. |

Commonly seen options:

| Option | Effect |
|---|---|
| `defaults` | `rw,suid,dev,exec,auto,nouser,async`. The default behavior depends on kernel and filesystem; see `fstab(5)`. |
| `noatime` | Do not update access time on reads. Significant I/O savings on busy filesystems. |
| `nodiratime` | Same, but only for directories. |
| `nofail` | Do not fail boot if the device is missing. Use for non-critical mounts. |
| `x-systemd.automount` | Lazy-mount on first access via systemd's automount; speeds boot. |
| `x-systemd.device-timeout=10s` | How long systemd waits for the device. |
| `discard` | Issue TRIM/discard inline on every free (see §3.4). |
| `ro` / `rw` | Read-only or read-write. |

`nofail` is a common operator save: a stale entry pointing at a removed disk will hang boot indefinitely without it. After editing `fstab`, validate before rebooting:

```bash
# Reload systemd's view of fstab without rebooting
sudo systemctl daemon-reload

# Try every fstab entry; report failures
sudo mount -a

# Or, with verification
findmnt --verify --verbose
```

### 1.5 Bind mounts and tmpfs

A *bind mount* exposes a directory at a second path. The kernel routes I/O to the same underlying inode — there is no copy.

```bash
# Bind-mount /data into a chroot
sudo mount --bind /data /srv/chroot/data

# Persistent in fstab
# /data /srv/chroot/data none bind 0 0
```

`tmpfs` is a RAM-backed filesystem that swaps to disk under memory pressure. Standard for `/tmp`, `/run`, and container scratch space.

```bash
sudo mount -t tmpfs -o size=512M tmpfs /mnt/scratch
```

### 1.6 Remount read-only and `umount` busy errors

When a filesystem hits I/O errors, ext4 by default remounts read-only (`errors=remount-ro`) to limit damage. You can force this manually:

```bash
sudo mount -o remount,ro /var
```

A normal `umount` fails with `target is busy` if any process holds a file open or any process's CWD is inside the mount. Find the culprit before reaching for `umount -l` (lazy unmount, which detaches but leaves processes alive):

```bash
# Processes with files open under the mount
sudo lsof +D /var/log

# Processes using the mount (faster, less recursive)
sudo fuser -mv /var/log

# Last resort: lazy unmount; do not use blindly on production
sudo umount -l /var/log
```

`umount -l` is dangerous in unattended scripts: pending writes can still complete after the mount appears gone, and resources are not freed until the last open handle closes.

### 1.7 systemd `.mount` units

systemd treats every mount as a unit. `fstab` entries are translated into `.mount` units at boot via `systemd-fstab-generator`. You can also write `.mount` and `.automount` units directly under `/etc/systemd/system/`. The unit name encodes the path: `/srv/data` → `srv-data.mount`.

```ini
# /etc/systemd/system/srv-data.mount
[Unit]
Description=Mount /srv/data
After=local-fs-pre.target

[Mount]
What=/dev/mapper/data-vg-data
Where=/srv/data
Type=ext4
Options=defaults,noatime

[Install]
WantedBy=multi-user.target
```

`.automount` units provide on-demand mounting:

```ini
# /etc/systemd/system/srv-data.automount
[Unit]
Description=Automount /srv/data

[Automount]
Where=/srv/data
TimeoutIdleSec=60

[Install]
WantedBy=multi-user.target
```

The same effect as `x-systemd.automount` in `fstab`, but with full unit semantics (dependencies, `After=`, `Requires=`).

---

## 2. LVM — Physical Volumes, Volume Groups, Logical Volumes

### 2.1 The three-layer model

LVM (Logical Volume Manager) inserts a virtualization layer between block devices and filesystems:

```text
   filesystem (ext4 / xfs / btrfs)
        ^
        | mounted on
        |
   logical volume   (LV)   <-- /dev/mapper/<vg>-<lv>
        ^
        | carved from
        |
   volume group    (VG)
        ^
        | aggregates
        |
   physical volumes (PVs)  <-- /dev/sdb1, /dev/nvme0n1, etc.
```

- **PV (Physical Volume)** — a block device blessed for LVM use (`pvcreate`). Carries a small header.
- **VG (Volume Group)** — a pool of one or more PVs.
- **LV (Logical Volume)** — a virtual block device carved from the VG, addressable as `/dev/<vg>/<lv>` or `/dev/mapper/<vg>-<lv>`.

The LV is what you format and mount. Because the LV is decoupled from physical disks, you can grow it onto new disks, snapshot it, and migrate it between PVs while it is in use.

### 2.2 Creating a stack from scratch

```bash
# Bless two disks as PVs
sudo pvcreate /dev/sdb /dev/sdc

# Create a VG named "data" using both
sudo vgcreate data /dev/sdb /dev/sdc

# Carve a 50 GB LV named "app" from it
sudo lvcreate -L 50G -n app data

# Format and mount
sudo mkfs.ext4 /dev/data/app
sudo mkdir -p /srv/app
sudo mount /dev/data/app /srv/app
```

Inspect at every layer:

```bash
sudo pvs        # one line per PV
sudo vgs        # one line per VG
sudo lvs        # one line per LV
sudo pvdisplay  # detailed PV view
sudo vgdisplay
sudo lvdisplay
```

### 2.3 Extending a VG and growing an LV

The headline LVM trick: grow live volumes without downtime.

```bash
# Add a new physical disk to the VG
sudo pvcreate /dev/sdd
sudo vgextend data /dev/sdd

# Grow the LV by 20 GB and resize the filesystem in one step
sudo lvextend -L +20G -r /dev/data/app
```

The `-r` flag tells `lvextend(8)` to call the appropriate `resize` tool for the filesystem (`resize2fs` for ext4, `xfs_growfs` for XFS). For ext4 and XFS, the resize is online — no unmount.

### 2.4 Snapshots: copy-on-write semantics

A traditional LVM snapshot is a CoW copy: when a block in the origin is written for the first time after the snapshot, the original block is copied into the snapshot's space, then the new write proceeds on the origin. Reads from the snapshot show the original data.

```bash
# 5 GB CoW snapshot of /dev/data/app
sudo lvcreate -L 5G -s -n app-snap /dev/data/app

# Mount the snapshot read-only somewhere else
sudo mkdir -p /mnt/snap
sudo mount -o ro /dev/data/app-snap /mnt/snap
```

Key gotchas:

- The snapshot LV must be sized for the *amount of change* expected, not the size of the origin.
- If the snapshot fills up, it is invalidated and dropped automatically. Workloads on the origin keep going, but the snapshot is gone.
- Traditional snapshots impose a write penalty on the origin (every first-write triggers a copy).
- Thin snapshots (§2.5) avoid most of this overhead.

### 2.5 Thin pools and thin provisioning

A *thin pool* is a special LV that backs many *thin LVs* whose sum of sizes can exceed the pool's actual capacity. Storage is allocated only when written. Snapshots of thin LVs are CoW within the pool with negligible setup cost.

```bash
# Create a 100 GB thin pool inside the VG
sudo lvcreate --type thin-pool -L 100G -n thinpool data

# Create a 200 GB thin volume (overcommit allowed)
sudo lvcreate --type thin -V 200G --thinpool data/thinpool -n app data

# Snapshot a thin LV — instant, near-zero overhead
sudo lvcreate -s -n app-snap data/app
```

Why thin matters operationally:

- **Snapshots are cheap.** You can take dozens for backup or pre-deploy checkpoints.
- **You over-provision intentionally.** Multiple LVs share a pool; you give each one a generous logical size and worry about real usage at the pool level.
- **The pool can run out before any LV does.** If the pool fills, all writes to all thin LVs in it fail. Monitor `Data%` and `Meta%` with `lvs`.

```bash
sudo lvs -o name,size,data_percent,metadata_percent
```

### 2.6 Inspecting the device-mapper tree

LVM is implemented on top of device-mapper (`dm`). When something looks wrong at the block level, drop down to `dmsetup(8)`:

```bash
# See the dm device tree
sudo dmsetup ls --tree

# Inspect a specific mapping
sudo dmsetup table data-app
sudo dmsetup info  data-app
```

Useful when an LV refuses to activate, a snapshot's chain looks wrong, or you need to confirm that what `lvs` says matches what the kernel actually sees.

---

## 3. Disk and Filesystem Tools — df, du, lsblk, blkid, fstrim

### 3.1 `df -h` vs `df -i`: blocks vs inodes

A filesystem can run out of two distinct resources: data blocks (bytes available) and inodes (file slots available). `df` shows blocks by default and inodes with `-i`.

```bash
# Block usage, human-readable
df -h

# Inode usage
df -i
```

Inode exhaustion is the classic 3 a.m. surprise. Symptom: `df -h` shows plenty of free space, but writes fail with `No space left on device`. Cause: millions of tiny files (mail spool, session files, log fragments, broken cleanup job) have used every inode.

Diagnosis:

```bash
# Find directories with the most inodes (file count)
sudo find /var -xdev -type f | awk -F/ '{print $1"/"$2"/"$3}' | sort | uniq -c | sort -rn | head
```

ext4 fixes the inode count at `mkfs` time. XFS allocates inodes dynamically and effectively does not exhaust them in normal use; this is one of XFS's operational advantages.

### 3.2 `du` for finding what filled up

```bash
# Top-level totals in /var, sorted
sudo du -h --max-depth=1 /var | sort -h

# Pinpoint the heaviest single directory tree
sudo du -sh /var/log

# Stay on one filesystem (don't descend into mounts)
sudo du -hx --max-depth=1 /
```

`-x` matters: without it, `du /` will descend into `/proc`, `/sys`, NFS mounts, and other things you don't want to count.

### 3.3 `lsblk` and `blkid`

`lsblk(8)` renders the block device tree:

```bash
# Tree with filesystem types, labels, mountpoints, UUIDs
lsblk -f

# More columns
lsblk -o NAME,SIZE,TYPE,FSTYPE,UUID,MOUNTPOINT,MODEL
```

`blkid(8)` prints filesystem UUIDs and labels — the canonical way to get the UUID for an `fstab` entry:

```bash
sudo blkid /dev/sdb1
# /dev/sdb1: UUID="abc-123-..." TYPE="ext4" PARTUUID="..."
```

Always reference disks by `UUID=` (or `LABEL=`) in `fstab`. `/dev/sdb1` can become `/dev/sdc1` after a reboot if a USB device is attached.

### 3.4 `fstrim` and SSD discard semantics

SSDs expose a `discard` (TRIM) command that tells the drive a range of blocks is free, letting the FTL erase and reclaim them. Without TRIM, write performance degrades over time. There are two strategies:

| Strategy | How it works | Tradeoff |
|---|---|---|
| **Continuous discard** | `discard` mount option; every `unlink` issues TRIM inline | Real-time reclaim; can add latency to deletes on some SSDs |
| **Batch / periodic discard** | `fstrim` run on a schedule (typically `fstrim.timer`) | One-shot, off-peak; recommended on modern systemd |

Most modern distros ship `fstrim.timer` enabled by default, which runs `fstrim --all` weekly.

```bash
# Manually trim a mounted filesystem
sudo fstrim -v /

# Check the timer
systemctl status fstrim.timer
```

The current consensus is to prefer the timer over the `discard` mount option for general workloads.

### 3.5 `iostat` and `smartctl`

`iostat` (from the `sysstat` package) reports per-device I/O statistics — the first line you read when something is "slow":

```bash
# Extended stats every 2 seconds, with device names
iostat -xz 2

# Key columns: %util, await (ms), r/s, w/s, rkB/s, wkB/s
```

`%util` near 100% with high `await` means the device is the bottleneck.

`smartctl` (from `smartmontools`) reads SMART attributes from disks — the way to catch dying hardware before it dies in production:

```bash
# Quick health verdict
sudo smartctl -H /dev/sda

# Full attribute dump
sudo smartctl -a /dev/sda

# Run a self-test
sudo smartctl -t short /dev/sda
sudo smartctl -l selftest /dev/sda
```

Pay attention to `Reallocated_Sector_Ct`, `Current_Pending_Sector`, `Offline_Uncorrectable`, and `Wear_Leveling_Count` (SSDs).

---

## 4. ext4, XFS, and Btrfs — Practical Differences

### 4.1 ext4: defaults and journal modes

ext4 is the default on Ubuntu/Debian and a known-quantity workhorse. It is journaling, extent-based, and supports online grow but **not online shrink** (offline shrink is supported via `resize2fs`).

Three journal modes (`data=` mount option):

| Mode | What is journaled | Use case |
|---|---|---|
| `data=ordered` (default) | Metadata only; data is written to disk before metadata is journaled | The default; balanced safety and speed |
| `data=journal` | Metadata *and* data | Strongest crash consistency; slowest |
| `data=writeback` | Metadata only; no ordering guarantee for data | Fastest; can expose stale data after a crash |

Most operators stay on the default. The journal lives in a reserved area of the filesystem and is replayed automatically on mount after a crash.

### 4.2 XFS: allocation groups and no-shrink

XFS divides the filesystem into independent *allocation groups* (AGs), each with its own free-space and inode metadata. This enables parallel allocation across cores under concurrent load. XFS allocates inodes dynamically — you do not run out of inodes the way you can on ext4.

Key operational facts (per `xfs(5)`):

- Online grow is supported via `xfs_growfs`.
- **Shrink is not supported.** XFS cannot be made smaller. If you need to shrink, you must back up, recreate, and restore.
- The default on RHEL/Fedora and recommended for general server workloads on those distros.

### 4.3 Btrfs: subvolumes, snapshots, CoW

Btrfs is a copy-on-write filesystem with first-class subvolumes, snapshots, checksums, and built-in multi-device support. Per the kernel docs and `btrfs.readthedocs.io`:

- A *subvolume* is a separately rootable directory tree within the same filesystem; subvolumes share extents and have independent inode namespaces.
- A *snapshot* is a subvolume pre-populated from an origin. Creation is effectively instantaneous (after a flush of dirty data).
- "A snapshot is not a backup" — origin and snapshot share the same physical extents until something is written, so a bad sector damages both.
- All data and metadata are checksummed; corruption is detected on read.

Btrfs is an excellent choice when snapshot semantics are central to the workload (e.g., systems that snapshot before every package upgrade). It has historically been more cautious territory for production databases; many shops still prefer ext4 or XFS underneath for that reason.

### 4.4 Comparison table

| Capability | ext4 | XFS | Btrfs |
|---|---|---|---|
| Journaling | Yes (modes: ordered/journal/writeback) | Yes (delayed logging) | N/A — CoW design instead |
| Inode allocation | Static (set at `mkfs`) | Dynamic | Dynamic |
| Online grow | Yes | Yes | Yes |
| Online shrink | No (offline only) | **No** | Yes |
| Native snapshots | No (use LVM) | No (use LVM) | Yes |
| CoW | No | No | Yes |
| Checksums (data) | No (metadata only) | Metadata only by default | Yes (data + metadata) |
| Multi-device / RAID | No (use mdadm/LVM) | No (use mdadm/LVM) | Yes |
| Typical default on | Debian, Ubuntu | RHEL, Fedora, CentOS Stream | openSUSE (root); opt-in elsewhere |

### 4.5 Online vs offline resize

Growing is the easy direction; shrinking is where filesystems differ:

```bash
# ext4: online grow after the LV is bigger
sudo resize2fs /dev/data/app

# ext4: shrink (must be unmounted)
sudo umount /srv/app
sudo e2fsck -f /dev/data/app
sudo resize2fs /dev/data/app 30G
sudo lvreduce -L 30G /dev/data/app
sudo mount /srv/app

# XFS: grow only, online
sudo xfs_growfs /srv/app

# Btrfs: grow or shrink, online
sudo btrfs filesystem resize +10G /srv/app
sudo btrfs filesystem resize -5G  /srv/app
```

`lvextend -r` (§2.3) automates the grow-FS step.

### 4.6 fsck behavior at boot and recovery tooling

The `fstab` `pass` field controls boot-time `fsck`. systemd runs `systemd-fsck@<dev>.service` for each entry with `pass>0`. Behavior at boot:

- **ext4**: `e2fsck` checks the journal; replays it if needed. Periodic full checks are triggered by mount count or interval (tunable with `tune2fs`).
- **XFS**: There is no `xfs_fsck`. The journal is replayed on mount and that is the normal recovery path. For deeper repair, `xfs_repair` runs offline on an unmounted filesystem.
- **Btrfs**: `btrfs check` is the offline checker. The kernel docs and Btrfs project recommend running it only when you suspect damage; routine use is not recommended.

Recovery tooling cheat sheet:

| Filesystem | Online recovery | Offline repair |
|---|---|---|
| ext4 | journal replay on mount | `e2fsck -f /dev/...` |
| XFS | log replay on mount | `xfs_repair /dev/...` (must be unmounted) |
| Btrfs | scrub (`btrfs scrub start /mnt`) | `btrfs check --repair` (read the warnings first) |

---

## 5. Operator Walkthrough

Reproducible end-to-end exercise on any throwaway VM or container with `losetup`, `lvm2`, and `e2fsprogs` installed. Uses loop devices so no real disk is needed.

### 5.1 Setup: loop devices as fake disks

```bash
# Create two 1 GB sparse files and attach as loop devices
sudo truncate -s 1G /tmp/disk1.img
sudo truncate -s 1G /tmp/disk2.img

LOOP1=$(sudo losetup --show -f /tmp/disk1.img)
LOOP2=$(sudo losetup --show -f /tmp/disk2.img)
echo "$LOOP1 $LOOP2"
# e.g. /dev/loop0 /dev/loop1

# Confirm
lsblk "$LOOP1" "$LOOP2"
```

### 5.2 Build PV → VG → LV, format, mount

```bash
# PVs
sudo pvcreate "$LOOP1" "$LOOP2"
sudo pvs

# VG using only the first PV (we'll add the second later)
sudo vgcreate demo "$LOOP1"
sudo vgs

# LV: 200 MB inside the VG
sudo lvcreate -L 200M -n web demo
sudo lvs

# Format ext4 and mount
sudo mkfs.ext4 /dev/demo/web
sudo mkdir -p /mnt/web
sudo mount /dev/demo/web /mnt/web

# Verify
findmnt /mnt/web
df -h /mnt/web
df -i /mnt/web
```

### 5.3 Snapshot, fill the LV, observe `df` vs `df -i`

```bash
# Write a real file
sudo dd if=/dev/urandom of=/mnt/web/payload.bin bs=1M count=50
df -h /mnt/web

# Take a 50 MB CoW snapshot
sudo lvcreate -L 50M -s -n web-snap /dev/demo/web
sudo lvs

# Demonstrate inode pressure: create a million tiny files
# (will fail with ENOSPC due to inodes, not bytes)
sudo bash -c 'mkdir -p /mnt/web/many && cd /mnt/web/many && \
  i=0; while :; do : > "f$i" || break; i=$((i+1)); done; echo "stopped at $i"'

# Compare:
df -h /mnt/web    # bytes free? maybe yes
df -i /mnt/web    # inodes free? probably 0 → that's why writes failed
```

This is the canonical "disk full but `df -h` says it isn't" scenario.

### 5.4 Grow the LV and resize the filesystem online

```bash
# Add the second PV to the VG
sudo vgextend demo "$LOOP2"
sudo vgs

# Grow the LV by 500 MB and resize ext4 in one step
sudo lvextend -L +500M -r /dev/demo/web

# Confirm
sudo lvs
df -h /mnt/web
df -i /mnt/web

# Without the -r flag, you would do:
#   sudo lvextend -L +500M /dev/demo/web
#   sudo resize2fs   /dev/demo/web
```

### 5.5 Cleanup

```bash
# Tear down in reverse order
sudo umount /mnt/web
sudo lvremove -f /dev/demo/web-snap
sudo lvremove -f /dev/demo/web
sudo vgremove  -f demo
sudo pvremove  -f "$LOOP1" "$LOOP2"

sudo losetup -d "$LOOP1" "$LOOP2"
sudo rm /tmp/disk1.img /tmp/disk2.img
sudo rmdir /mnt/web
```

If you do this once on a real VM, the muscle memory for the production version of the same incident is set.

---

## Related

- [../../operating-systems/fundamentals/03-page-cache-and-buffered-io.md](../../operating-systems/fundamentals/03-page-cache-and-buffered-io.md) — why writes go to the page cache before the filesystem and why `sync` matters during snapshot/unmount.
- [../../operating-systems/fundamentals/05-syscalls-and-the-abi.md](../../operating-systems/fundamentals/05-syscalls-and-the-abi.md) — the syscall layer (`open`, `read`, `mount`, `umount`) underneath the VFS abstraction.
- [../../operating-systems/fundamentals/06-file-descriptors-and-ulimits.md](../../operating-systems/fundamentals/06-file-descriptors-and-ulimits.md) — why `umount` reports "target is busy" and how `lsof +D` finds the open fd.
- [../../kubernetes/workloads/statefulsets-and-daemonsets.md](../../kubernetes/workloads/statefulsets-and-daemonsets.md) — Kubernetes PersistentVolumes and StorageClasses that map onto LVs and filesystems on each node.
- [../../performance/fundamentals/04-use-and-red-applied.md](../../performance/fundamentals/04-use-and-red-applied.md) — the USE method applied to disks: utilization (`iostat %util`), saturation (`await`), errors (SMART).

## References

- `man 5 fstab` — `/etc/fstab` format and base mount options. man7.org mirror: <https://man7.org/linux/man-pages/man5/fstab.5.html>
- `man 8 mount` — mount and remount semantics, filesystem-independent options.
- `man 8 findmnt` — tree view of `/proc/self/mountinfo`, `--verify` mode.
- `man 5 systemd.mount` — systemd `.mount` unit syntax.
- `man 5 systemd.automount` — systemd `.automount` unit syntax.
- `man 8 lvm` — LVM overview and command index.
- `man 8 pvcreate`, `man 8 vgcreate`, `man 8 lvcreate`, `man 8 lvextend`, `man 8 lvconvert` — LVM stack management.
- `man 8 dmsetup` — device-mapper inspection (`ls --tree`, `table`, `info`).
- `man 1 df`, `man 1 du`, `man 8 lsblk`, `man 8 blkid`, `man 8 fstrim` — disk and filesystem inspection.
- `man 8 iostat` (sysstat), `man 8 smartctl` (smartmontools) — I/O and SMART telemetry.
- `man 5 xfs` — XFS mount options, allocation groups, deprecated/removed options. <https://man7.org/linux/man-pages/man5/xfs.5.html>
- Linux kernel docs — ext4: <https://www.kernel.org/doc/html/latest/filesystems/ext4/index.html>
- Linux kernel docs — XFS: <https://www.kernel.org/doc/html/latest/filesystems/xfs/index.html>
- Linux kernel docs — Btrfs: <https://docs.kernel.org/filesystems/btrfs.html>
- Linux kernel docs — Device-mapper admin guide: <https://docs.kernel.org/admin-guide/device-mapper/index.html>
- Btrfs project — Subvolumes and snapshots: <https://btrfs.readthedocs.io/en/latest/Subvolumes.html>
