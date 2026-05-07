---
title: "Security and Isolation — SELinux, AppArmor, seccomp, auditd, Capabilities"
date: 2026-05-07
updated: 2026-05-07
tags: [linux, selinux, apparmor, seccomp, capabilities, auditd]
---

# Security and Isolation — SELinux, AppArmor, seccomp, auditd, Capabilities

**Date:** 2026-05-07 | **Updated:** 2026-05-07
**Tags:** `linux` `selinux` `apparmor` `seccomp` `capabilities` `auditd`

---

## Table of Contents

- [Summary](#summary)
- [1. SELinux — Modes, Policies, and Common audit2allow Loops](#1-selinux--modes-policies-and-common-audit2allow-loops)
  - [1.1 DAC vs MAC](#11-dac-vs-mac)
  - [1.2 Type Enforcement and Contexts](#12-type-enforcement-and-contexts)
  - [1.3 Modes and the targeted Policy](#13-modes-and-the-targeted-policy)
  - [1.4 Booleans, restorecon, chcon](#14-booleans-restorecon-chcon)
  - [1.5 The audit2allow Loop](#15-the-audit2allow-loop)
  - [1.6 Why "just disable SELinux" Is the Wrong Answer](#16-why-just-disable-selinux-is-the-wrong-answer)
- [2. AppArmor — Profile-Based MAC, the Ubuntu/SUSE Default](#2-apparmor--profile-based-mac-the-ubuntususe-default)
  - [2.1 Path-Based vs Label-Based MAC](#21-path-based-vs-label-based-mac)
  - [2.2 Profile Syntax and /etc/apparmor.d/](#22-profile-syntax-and-etcapparmord)
  - [2.3 Complain vs Enforce; aa-status, aa-genprof, aa-logprof](#23-complain-vs-enforce-aa-status-aa-genprof-aa-logprof)
  - [2.4 Abstractions, Child Profiles, and Stacking](#24-abstractions-child-profiles-and-stacking)
  - [2.5 When AppArmor Is Appropriate](#25-when-apparmor-is-appropriate)
- [3. seccomp — Syscall Allowlists for Containers](#3-seccomp--syscall-allowlists-for-containers)
  - [3.1 What seccomp Does](#31-what-seccomp-does)
  - [3.2 STRICT vs FILTER (BPF)](#32-strict-vs-filter-bpf)
  - [3.3 Docker and Kubernetes Defaults](#33-docker-and-kubernetes-defaults)
  - [3.4 Generating Profiles by Tracing](#34-generating-profiles-by-tracing)
  - [3.5 Trade-offs vs an LSM](#35-trade-offs-vs-an-lsm)
- [4. auditd — Audit Trail and Filesystem Watch Rules](#4-auditd--audit-trail-and-filesystem-watch-rules)
  - [4.1 Kernel Audit Subsystem vs Userspace auditd](#41-kernel-audit-subsystem-vs-userspace-auditd)
  - [4.2 Rule Syntax — `-w`, `-a`, `-S`, `-k`](#42-rule-syntax---w--a--s--k)
  - [4.3 ausearch and aureport](#43-ausearch-and-aureport)
  - [4.4 Volume, Throttling, and SIEM Integration](#44-volume-throttling-and-siem-integration)
- [5. Capabilities — POSIX File Capabilities and Container Drops](#5-capabilities--posix-file-capabilities-and-container-drops)
  - [5.1 The Capability Model](#51-the-capability-model)
  - [5.2 Common Capabilities Worth Knowing](#52-common-capabilities-worth-knowing)
  - [5.3 The Five Capability Sets](#53-the-five-capability-sets)
  - [5.4 File Capabilities — getcap, setcap](#54-file-capabilities--getcap-setcap)
  - [5.5 Container Default-Drop Lists and `--cap-add` / `--cap-drop`](#55-container-default-drop-lists-and---cap-add----cap-drop)
  - [5.6 Capabilities and User Namespaces](#56-capabilities-and-user-namespaces)
- [6. Operator Walkthrough](#6-operator-walkthrough)
- [Related](#related)
- [References](#references)

---

## Summary

This doc is the operator's view of the five security primitives a Linux backend engineer actually has to deal with: SELinux and AppArmor (Linux Security Modules implementing Mandatory Access Control on top of file/process permissions), seccomp (a per-process BPF syscall filter), auditd (the userspace consumer of the kernel audit subsystem), and POSIX capabilities (the split-up superset of root). The goal: when SELinux denies a service, when a Docker container needs a custom seccomp profile, when "non-root with `CAP_NET_BIND_SERVICE`" actually means something — you know which knob to turn.

---

## 1. SELinux — Modes, Policies, and Common audit2allow Loops

### 1.1 DAC vs MAC

Discretionary Access Control (DAC) is the classic UNIX permission model: file owner decides who can read/write, root bypasses everything. The user with control over the object decides — that's the "discretionary" part.

Mandatory Access Control (MAC) layers a second, kernel-enforced policy on top. The owner cannot waive it. Root cannot waive it (by `chmod`). Even if `/etc/shadow` were `chmod 0666`, an SELinux policy that says "only processes in `passwd_t` can read `shadow_t`" still denies your `cat`. Both DAC and MAC must allow an action; either can deny it.

The Linux Security Module (LSM) framework is the kernel hook layer that implements MAC. SELinux, AppArmor, Smack, TOMOYO, Yama, LoadPin, SafeSetID, IPE, and Landlock are all LSMs; the active set on a running system is in `/sys/kernel/security/lsm` (kernel.org LSM admin guide).

### 1.2 Type Enforcement and Contexts

SELinux's primary access-control model is **Type Enforcement**. Every process has a *domain* (a type). Every object (file, port, IPC) has a *type*. A policy is a giant matrix of "domain X may do operation Y on type Z." Allow the pair, the syscall proceeds. Otherwise the kernel denies it and writes an Access Vector Cache (AVC) record into the audit log.

A full SELinux **security context** is `user:role:type:level`. Day-to-day, the `type` is what matters.

```bash
# Process contexts
ps -eZ | head
# system_u:system_r:init_t:s0   1 ?  00:00:01 systemd
# system_u:system_r:sshd_t:s0-s0:c0.c1023 871 ?  00:00:00 sshd
# system_u:system_r:httpd_t:s0  1042 ?  00:00:00 httpd

# File contexts
ls -Z /var/www/html/index.html
# unconfined_u:object_r:httpd_sys_content_t:s0 /var/www/html/index.html

# Port contexts
semanage port -l | grep ^http_port_t
# http_port_t       tcp     80, 81, 443, 488, 8008, 8009, 8443, 9000
```

The `httpd` daemon runs in `httpd_t`. The HTML file is `httpd_sys_content_t`. Policy says `httpd_t` may read `httpd_sys_content_t` — request allowed.

### 1.3 Modes and the targeted Policy

Three modes:

| Mode | Behavior |
|------|----------|
| `enforcing` | Policy is consulted; denials are enforced *and* logged. |
| `permissive` | Policy is consulted; denials are logged but *not* enforced. Useful for diagnosing without breaking the service. |
| `disabled` | The LSM is not loaded. No policy. No logs. |

Switching at runtime (does not survive reboot for `disabled`):

```bash
getenforce          # current mode
setenforce 0        # to permissive
setenforce 1        # to enforcing
sestatus            # full status, including loaded policy
```

The default policy on RHEL/Fedora/CentOS Stream is `targeted`. "Targeted" means most user processes run in `unconfined_t` and the policy primarily confines specific system services (httpd, sshd, postgresql, nginx, etc.). It is the trade-off chosen by Red Hat to keep the policy maintainable while still confining the daemons that actually face the network. The Red Hat "Using SELinux" guide for RHEL 9 documents the modes, the targeted policy, and the troubleshooting workflow.

### 1.4 Booleans, restorecon, chcon

**Booleans** are policy-author-provided knobs that flip a chunk of policy on or off without compiling a new module. Examples: `httpd_can_network_connect`, `httpd_can_sendmail`, `nis_enabled`. List and toggle:

```bash
getsebool -a | head
setsebool -P httpd_can_network_connect on   # -P persists across reboots
```

When SELinux denies something a service should be allowed to do, **first check whether a boolean already exists** for that pattern. It almost always does for common cases.

File contexts are stored as the `security.selinux` extended attribute on each file, but the *intended* context for any path is computed from the policy's file-contexts spec. Two commands matter:

| Tool | Purpose |
|------|---------|
| `restorecon` | Restore the file's context to what policy says it should be, by consulting the file_contexts spec. |
| `chcon` | Change the context to an arbitrary value, ignoring policy. **Not persistent across `restorecon`.** |

```bash
restorecon -Rv /var/www/html      # recursively restore policy-defined contexts
chcon -t httpd_sys_content_t /tmp/file.html   # one-off override (will be reverted by restorecon)
semanage fcontext -a -t httpd_sys_content_t '/srv/web(/.*)?'   # add a permanent rule
restorecon -Rv /srv/web
```

Use `semanage fcontext` + `restorecon` to make a context change durable. Use `chcon` only for diagnosis; never for a long-lived fix.

### 1.5 The audit2allow Loop

The standard troubleshooting loop, when a denial is real and no boolean fixes it:

1. **Reproduce the denial** in `permissive` mode (or `enforcing` — denials are logged either way).
2. **Find the AVC** in the audit log:

   ```bash
   ausearch -m AVC -ts recent
   # or
   ausearch -m AVC,USER_AVC -ts today -i
   ```

   `-m AVC` filters on Access Vector Cache messages (SELinux denials). `-ts recent` is "the last ten minutes" per the `ausearch(8)` man page. `-i` interprets numeric IDs into names.

3. **Generate a policy module** from the denial:

   ```bash
   ausearch -m AVC -ts recent | audit2allow -M mywebapp_local
   # produces mywebapp_local.te (source) and mywebapp_local.pp (binary)
   ```

4. **Read the `.te`** before installing. `audit2allow` will happily generate `allow X Y:Z manage_all_perms;` rules that are far too broad — review every line. If the proposed rule looks dangerous (for example, `allow httpd_t shadow_t:file read;`), stop and rethink: the daemon should not be touching that object at all.

5. **Install the module:**

   ```bash
   semodule -i mywebapp_local.pp
   semodule -l | grep mywebapp_local
   ```

6. **Re-test in enforcing.** If a new denial appears, repeat.

If the loop runs more than two or three iterations, the daemon is doing something fundamentally outside its expected behavior — a misconfigured path, a wrong port, a non-standard log location. Fixing the configuration (a `semanage fcontext` rule, a `semanage port -a` to add a port) is almost always cleaner than adding policy.

### 1.6 Why "just disable SELinux" Is the Wrong Answer

Disabling SELinux removes the second of the two access-control layers, with no replacement. On a RHEL-family system the targeted policy was written assuming services run confined; disabling it not only removes confinement, it removes the constant signal of "something tried to do an unexpected thing" that AVC denials provide. The right response to a denial is one of:

1. Flip a boolean.
2. Fix a label with `semanage fcontext` + `restorecon`.
3. Add a port type with `semanage port`.
4. Run in permissive *only* during diagnosis.
5. As a last resort, generate a reviewed local policy module.

Setting `SELINUX=disabled` in `/etc/selinux/config` is the response of a tired operator at 02:00, not a fix. Treat AVC denials the way you treat a panicking pod: read the log, then act.

---

## 2. AppArmor — Profile-Based MAC, the Ubuntu/SUSE Default

### 2.1 Path-Based vs Label-Based MAC

SELinux is **label-based**: every object has an extended attribute that identifies its type, and the policy is written against types. Move a file with `mv -Z` and the label travels.

AppArmor is **path-based**: rules are written against pathnames. `/usr/sbin/nginx` running with the `nginx` profile is allowed to read `/var/www/**` — the rule references the path, not a label on the inode. Move the data outside `/var/www` and the rule no longer applies, regardless of what the file "is."

| Aspect | SELinux (label) | AppArmor (path) |
|--------|-----------------|-----------------|
| Mental model | Type Enforcement matrix | Per-program file rules |
| Policy unit | Compiled policy module | Single-file profile |
| Authoring difficulty | Steep — full type system | Closer to a `.htaccess` file in feel |
| Bind-mounts and chroots | Label travels with inode | Rules can be misled by remounts |
| Default on | RHEL/Fedora/CentOS Stream | Ubuntu, SUSE, Debian (apparmor.d(5) on Ubuntu) |

### 2.2 Profile Syntax and /etc/apparmor.d/

Profiles live in `/etc/apparmor.d/`. A profile is a name, optional flags, and a brace-enclosed rule set. The `apparmor.d(5)` man page describes the structure:

```text
#include <tunables/global>

/usr/sbin/nginx {
  #include <abstractions/base>
  #include <abstractions/nameservice>

  capability net_bind_service,
  capability setuid,
  capability setgid,

  network inet stream,
  network inet6 stream,

  /etc/nginx/** r,
  /var/log/nginx/*.log w,
  /var/www/** r,
  /run/nginx.pid rw,

  /usr/sbin/nginx mr,
}
```

File permissions are leading or trailing flags after the path: `r` read, `w` write, `a` append, `l` link, `k` lock, `m` mmap as executable, `ix`/`px`/`ux` various execute transitions (apparmor.d(5)).

### 2.3 Complain vs Enforce; aa-status, aa-genprof, aa-logprof

A profile can be in **enforce** (deny on rule violation, log it) or **complain** (log only — equivalent to SELinux permissive for that program).

```bash
sudo aa-status                # summary: loaded/enforced/complaining counts
sudo aa-complain /etc/apparmor.d/usr.sbin.nginx
sudo aa-enforce  /etc/apparmor.d/usr.sbin.nginx
```

The Ubuntu `aa-status(8)` man page documents the `--enforced` and `--complaining` counters.

Profile-authoring tools:

| Tool | Purpose |
|------|---------|
| `aa-genprof` | Run a program, log all access attempts, interactively build a profile from them. |
| `aa-logprof` | Read existing logs, propose additions to an existing profile. |

Both walk the operator through each access decision (allow/deny/glob/abstraction) and write the result back to `/etc/apparmor.d/`. They are imperfect — review the result.

Reload after editing:

```bash
sudo apparmor_parser -r /etc/apparmor.d/usr.sbin.nginx
```

### 2.4 Abstractions, Child Profiles, and Stacking

**Abstractions** in `/etc/apparmor.d/abstractions/` are reusable rule clusters: `<abstractions/base>` (libc, ld.so, common locales), `<abstractions/nameservice>` (resolver), `<abstractions/ssl_certs>`, etc. Pull them in with `#include` instead of repeating. From `apparmor.d(5)`: "AppArmor provides an easy abstraction mechanism to group common access requirements."

**Child profiles** (subprofiles, hats) confine a portion of a program differently. A web server's request handler can `change_hat` into a hat with stricter rules for the duration of that request (`aa_change_hat(2)`).

**Stacking** lets multiple profiles apply at once — useful for systems where a container runtime profile and an application profile both need to confine a process.

### 2.5 When AppArmor Is Appropriate

- Default and most-tested LSM on Ubuntu, SUSE, Debian.
- Snap confinement is built on AppArmor.
- Docker ships a default AppArmor profile (`docker-default`) on Ubuntu hosts; profiles can be loaded by name.

If your fleet is RHEL-family, you are almost certainly using SELinux and there is no reason to switch. If your fleet is Ubuntu, AppArmor is the expected control and switching to SELinux is a multi-month project. Match the distro.

---

## 3. seccomp — Syscall Allowlists for Containers

### 3.1 What seccomp Does

seccomp ("secure computing mode") is a per-thread filter on the syscall interface. After a thread enters seccomp mode, the kernel evaluates a filter on every syscall before dispatching it. If the filter returns "kill," the kernel terminates the thread; if "errno," the syscall returns an error without executing; if "allow," the syscall proceeds normally. Filters are evaluated *in-kernel* — there is no userspace round trip per syscall.

It is **not** an LSM. It does not look at the object the syscall is about (with the narrow exception of comparing scalar arguments). It only looks at the syscall number and immediate arguments.

### 3.2 STRICT vs FILTER (BPF)

Two modes (`seccomp(2)` man page on man7.org):

| Mode | Allowed syscalls |
|------|------------------|
| `SECCOMP_MODE_STRICT` | Exactly four: `read(2)`, `write(2)`, `_exit(2)` (note: not `exit_group`), `sigreturn(2)`. |
| `SECCOMP_MODE_FILTER` | Whatever a Berkeley Packet Filter (BPF) program attached via `seccomp(2)` says is allowed. |

`STRICT` is largely a curiosity outside very narrow sandboxes. `FILTER` is what every modern container runtime uses.

The kernel `Documentation/userspace-api/seccomp_filter.rst` documents the BPF return values in precedence order, including:

| Return value | Effect |
|--------------|--------|
| `SECCOMP_RET_KILL_PROCESS` / `SECCOMP_RET_KILL_THREAD` | Terminate with `SIGSYS`. |
| `SECCOMP_RET_ERRNO` | Don't run the syscall; return the encoded errno to userspace. |
| `SECCOMP_RET_TRAP` / `SECCOMP_RET_TRACE` / `SECCOMP_RET_LOG` / `SECCOMP_RET_USER_NOTIF` | Various developer-facing alternatives. |
| `SECCOMP_RET_ALLOW` | Run the syscall normally. |

The kernel doc explicitly notes BPF programs cannot dereference pointers, which is why seccomp can match `open()`'s flags but not the path string the pointer points to — the path is in userspace memory.

### 3.3 Docker and Kubernetes Defaults

**Docker:** every container gets a default seccomp profile unless overridden. Per docs.docker.com, the profile is an allowlist that disables roughly 44 syscalls of 300+, including kernel module ops (`init_module`, `delete_module`), `ptrace`, `mount`/`umount`, time-of-day mutation (`clock_settime`, `settimeofday`), I/O privilege (`ioperm`, `iopl`), and key-management (`add_key`, `keyctl`). The reference profile lives at `https://github.com/moby/profiles/blob/main/seccomp/default.json`. Docker's docs explicitly recommend not changing it.

```bash
# inspect the profile a running container is using
docker inspect --format '{{ .HostConfig.SecurityOpt }}' <container>

# run with no seccomp profile (do this only for diagnosis)
docker run --security-opt seccomp=unconfined ...

# run with a custom profile
docker run --security-opt seccomp=/path/to/profile.json ...
```

**Kubernetes:** seccomp is set in `securityContext.seccompProfile`. Three `type` values (kubernetes.io seccomp tutorial):

| `type` | Effect |
|--------|--------|
| `RuntimeDefault` | The container runtime's default profile (matches Docker default on a containerd/cri-o host). |
| `Localhost` | A custom profile loaded on the node, referenced by `localhostProfile`. |
| `Unconfined` | No profile. |

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: web
spec:
  securityContext:
    seccompProfile:
      type: RuntimeDefault
  containers:
  - name: nginx
    image: nginx:1.27
```

The Kubernetes docs note privileged containers always run as `Unconfined` regardless of profile, so combining `privileged: true` with a seccomp profile is meaningless.

### 3.4 Generating Profiles by Tracing

Hand-writing a seccomp profile that allowlists exactly the right syscalls is painful. Tools exist:

- `strace -c -f -e trace=all <cmd>` produces a syscall-frequency summary; the column of names is a starting allowlist.
- `bcc`'s `syscount` or `ttysnoop` plus the older `seccomp` tooling can record the syscalls a workload uses.
- The Kubernetes Security Profiles Operator can record per-pod seccomp profiles automatically.

Workflow:

1. Run the workload under a permissive profile or in `SECCOMP_RET_LOG` mode.
2. Collect every syscall it issued during a representative load.
3. Generate an allowlist profile.
4. Switch to enforcing.
5. Watch for `SIGSYS` kills or audit `SECCOMP` records and iterate.

### 3.5 Trade-offs vs an LSM

| Property | seccomp | SELinux / AppArmor |
|----------|---------|--------------------|
| Granularity | Syscall + scalar arguments only | Operation on a labeled or path-named object |
| Path / label awareness | None | Full |
| Composition with namespaces | Per-thread, inherited at fork | Same |
| Authoring | BPF allowlist of syscall numbers | Domain/type matrix or path rules |
| Performance | One BPF eval per syscall, in-kernel | LSM hook checks, in-kernel |
| Best at | Reducing kernel attack surface (you literally cannot reach the buggy syscall) | Confining what an authorized syscall is allowed to act on |

They are complementary. A hardened container uses both: seccomp blocks syscalls the workload should never need; the LSM blocks reaching files or sockets it should never need. Defense in depth.

---

## 4. auditd — Audit Trail and Filesystem Watch Rules

### 4.1 Kernel Audit Subsystem vs Userspace auditd

The kernel has an audit subsystem that emits structured records for syscalls, AVC denials, and explicit watch points. Records are delivered to userspace via a netlink socket. `auditd(8)` is the userspace daemon that reads from that socket and persists records — typically to `/var/log/audit/audit.log`. The man page describes auditd as "the userspace component to the Linux Auditing System" and notes that rules from `/etc/audit/audit.rules` "are read by auditctl and loaded into the kernel" at startup.

If `auditd` is not running, audit records may be lost (the kernel buffer is bounded), but the kernel still emits AVC denials to the kernel ring buffer where `dmesg` will show them.

```bash
systemctl status auditd
ls -lh /var/log/audit/audit.log
auditctl -s         # status: enabled, pid, lost records, backlog
```

### 4.2 Rule Syntax — `-w`, `-a`, `-S`, `-k`

Rules are loaded with `auditctl` at runtime and persisted in `/etc/audit/rules.d/*.rules` (compiled into `/etc/audit/audit.rules` by `augenrules`). The `auditctl(8)` man page defines the flags:

| Flag | Meaning (man page) |
|------|--------------------|
| `-w PATH` | "Place a watch on path." File or directory. |
| `-a action,filter` | "Append rule to the end of list with action." Common forms: `-a always,exit` for syscall-completion events, `-a always,user` for user-context events. |
| `-S syscall` | "Any syscall name or number may be used. The word 'all' may also be used." Only meaningful on syscall-list rules. |
| `-k key` | A short tag attached to matched events, used later by `ausearch -k`. |

Examples:

```bash
# Watch for any modification of the sudoers file
auditctl -w /etc/sudoers -p wa -k sudo_changes

# Audit every execve from non-root accounts
auditctl -a always,exit -F arch=b64 -S execve -F auid>=1000 -F auid!=-1 -k execs

# Audit every connect() from a specific binary
auditctl -a always,exit -F arch=b64 -S connect -F path=/usr/bin/curl -k curl_net

# List active rules
auditctl -l

# Make persistent
cat > /etc/audit/rules.d/99-sudo.rules <<'EOF'
-w /etc/sudoers -p wa -k sudo_changes
EOF
augenrules --load
```

### 4.3 ausearch and aureport

`ausearch` queries the log; `aureport` summarizes it.

```bash
# all events tagged sudo_changes today
ausearch -k sudo_changes -ts today -i

# all SELinux denials in the last ten minutes
ausearch -m AVC,USER_AVC -ts recent -i

# everything from one specific PID
ausearch -p 12345 -i

# summary by syscall
aureport -s --summary

# summary by user
aureport -u --summary
```

### 4.4 Volume, Throttling, and SIEM Integration

Audit rules cost CPU and disk. A rule of `-S all` matches every syscall a busy server makes, which on a 32-core machine doing 400k req/s is millions of events per second. The kernel will throttle and drop. **Tight rules only.** Specific syscalls, specific paths, specific UIDs.

For SIEM integration, the standard pattern is `audisp` (the audit dispatcher, a sub-process of auditd) forwarding to `syslog` or to a remote collector via `audisp-remote`. Tune `auditd.conf` (`max_log_file`, `num_logs`, `disk_full_action`) before pointing this at production. If `disk_full_action = halt` (a real option), an audit-log overflow can panic the box.

---

## 5. Capabilities — POSIX File Capabilities and Container Drops

### 5.1 The Capability Model

Traditional UNIX has one privileged identity — UID 0, root — that can do anything. Linux capabilities split that single power into about 40 named privileges. A process can hold any subset; the kernel checks the specific capability for each operation rather than "are you root?"

This makes it possible to run a non-root daemon that can still do *exactly one* privileged thing — for example, bind to port 80 — without granting it the entire kernel.

### 5.2 Common Capabilities Worth Knowing

From `capabilities(7)` on man7.org:

| Capability | What it lets you do |
|------------|---------------------|
| `CAP_NET_BIND_SERVICE` | "Bind a socket to Internet domain privileged ports (port numbers less than 1024)." |
| `CAP_NET_RAW` | Use RAW and PACKET sockets, bind to any address (transparent proxying). |
| `CAP_SYS_ADMIN` | The "kitchen sink" capability — mount/umount, namespace creation, quotas, many device ops. Hand it out and you have effectively granted root. |
| `CAP_DAC_OVERRIDE` | Bypass file read/write/execute permission checks. |
| `CAP_SETUID` | Arbitrary UID changes via `setuid(2)`. |
| `CAP_SYS_PTRACE` | Trace any process; read its memory. |
| `CAP_KILL` | Send signals to any process regardless of UID. |
| `CAP_CHOWN` | Change file ownership arbitrarily. |
| `CAP_AUDIT_WRITE` | Write to the kernel audit log. |
| `CAP_MKNOD` | Create special files via `mknod(2)`. |

`CAP_SYS_ADMIN` deserves a healthy fear. Granting it to a container is roughly equivalent to dropping it on the host as root. Treat it as an exception that needs justification, not a default.

### 5.3 The Five Capability Sets

`capabilities(7)` defines five per-thread sets:

| Set | Role |
|-----|------|
| **Permitted** | Limiting superset; capabilities the thread *may* assume into Effective. |
| **Effective** | What the kernel actually checks against on each privileged op. |
| **Inheritable** | Preserved across `execve(2)` for processes the thread spawns. |
| **Bounding** | Per-thread ceiling; capabilities here limit what `execve(2)` can grant via file caps. |
| **Ambient** | (Linux 4.3+) Preserved across `execve(2)` even for unprivileged programs without file caps. |

The arithmetic at `execve(2)` is intricate. The shorthand most operators need: **bounding set is the upper bound on what a binary can ever pick up at exec.** Container runtimes shape the bounding set when launching containers, which is what `--cap-drop` and `--cap-add` actually adjust.

### 5.4 File Capabilities — getcap, setcap

POSIX file capabilities are stored as the `security.capability` xattr on a binary. When that binary is `execve`'d, the kernel can grant capabilities into the process — without the binary needing to be setuid root.

```bash
# Inspect a binary
getcap /usr/bin/ping
# /usr/bin/ping cap_net_raw=ep   (on many distros)

# Grant a capability to a binary
sudo setcap 'cap_net_bind_service=+ep' /usr/local/bin/myserver

# Remove
sudo setcap -r /usr/local/bin/myserver
```

The `setcap(8)` man page references `cap_text_formats(7)` for the assignment grammar. The pieces:

| Token | Meaning |
|-------|---------|
| `cap_net_bind_service` | The capability name. |
| `=` | Assignment (replace existing). |
| `+ep` (or `=ep`) | Place in the **e**ffective and **p**ermitted sets. `i` would add inheritable. |

`+ep` is the typical operator pattern: the capability is in Permitted (so it can be assumed) and Effective (so the kernel grants it immediately at exec). No setuid bit, no run-as-root; the binary just gets that one privilege.

`getpcaps PID` reads a *running process's* capability sets from `/proc/PID/status` and prints them.

### 5.5 Container Default-Drop Lists and `--cap-add` / `--cap-drop`

Docker containers do **not** run with all capabilities even when the entrypoint is UID 0. Docker drops a substantial set by default and keeps a small list of permitted capabilities. The exact list varies slightly across versions; consult the current Docker docs and `containerd` defaults for an authoritative list.

CLI shape:

```bash
# drop everything, then add only what you need
docker run --cap-drop=ALL --cap-add=NET_BIND_SERVICE myimage

# Kubernetes equivalent
# securityContext:
#   capabilities:
#     drop: ["ALL"]
#     add:  ["NET_BIND_SERVICE"]
```

`--cap-add` and `--cap-drop` are the runtime's way of shaping the container's bounding (and other) sets. The combination of `runAsNonRoot: true` plus `capabilities.drop: [ALL]` plus `capabilities.add: [NET_BIND_SERVICE]` is a very common production pattern: the container is non-root, has no extra power, but can bind to port 80/443.

### 5.6 Capabilities and User Namespaces

Inside a user namespace, a process can have all 40 capabilities — but only against resources owned by that namespace. `CAP_SYS_ADMIN` inside a user namespace does not let you mount over `/etc` on the host; it lets you mount things owned by the namespace's mappings. This is why rootless containers can do things that look like privileged operations without being privileged on the host.

Combined with capabilities, user namespaces are how container runtimes deliver "feels like root, isn't actually root" — the container's UID 0 maps to a high unprivileged UID on the host, and capabilities only mean what they mean *inside* the namespace.

---

## 6. Operator Walkthrough

A reproducible sequence on a throwaway VM (RHEL 9 family for SELinux pieces, any Linux for the rest). Adjust paths/versions to taste.

```bash
# --- 1. Read a process security context ---
ps -eZ | grep sshd
# system_u:system_r:sshd_t:s0-s0:c0.c1023  ...  /usr/sbin/sshd -D
#                  ^^^^^^ the type that policy keys on

# --- 2. Trigger an SELinux denial in permissive mode ---
sudo setenforce 0
sudo getenforce
# Permissive

# Place a file outside any web type, then make httpd try to read it.
sudo bash -c 'echo hello > /root/secret.html'
sudo chcon -t admin_home_t /root/secret.html  # ensure a "wrong" label
sudo ln -s /root/secret.html /var/www/html/secret.html  || true

# Install httpd if needed; start it.
sudo dnf install -y httpd
sudo systemctl enable --now httpd
curl -sS http://localhost/secret.html  >/dev/null 2>&1  || true

# Read the resulting AVC.
sudo ausearch -m AVC -ts recent -i | head -40
# ... type=AVC msg=audit(...): avc:  denied  { read } for  pid=...
#     comm="httpd" name="secret.html" dev="..." ino=...
#     scontext=system_u:system_r:httpd_t:s0
#     tcontext=unconfined_u:object_r:admin_home_t:s0  tclass=lnk_file ...

# Generate a candidate module and READ IT before installing.
sudo ausearch -m AVC -ts recent | sudo audit2allow -M demo_local
cat demo_local.te
# (review carefully — almost certainly do NOT install this in real life;
#  the right fix is to relocate the file or label it correctly)

# Re-enable enforcing.
sudo setenforce 1

# --- 3. Inspect a Docker container's seccomp profile ---
docker run -d --name webdemo nginx:1.27
docker inspect --format '{{json .HostConfig.SecurityOpt }}' webdemo
# [] when using runtime default; an item like
# ["seccomp=...path..."] if explicitly set.

# Confirm the container is using the runtime default.
docker inspect --format '{{ .AppArmorProfile }}' webdemo
docker inspect --format '{{ .HostConfig.CapDrop }} | {{ .HostConfig.CapAdd }}' webdemo

docker rm -f webdemo

# --- 4. Drop a binary cap so a non-root user can bind port 80 ---
# Build a trivial server (node here; any language with a bind() call works).
cat > /tmp/srv.js <<'EOF'
const http = require('http');
http.createServer((_,res)=>res.end('ok\n')).listen(80, () => {
  console.log('bound 80 as', process.getuid());
});
EOF

# Without the capability, port 80 is forbidden for non-root.
sudo -u nobody node /tmp/srv.js
# Error: listen EACCES: permission denied 0.0.0.0:80   <-- expected

# Grant the file capability to the node binary (use a copy in real life, not /usr/bin/node).
NODE_BIN=$(readlink -f $(command -v node))
sudo cp "$NODE_BIN" /usr/local/bin/node-cap
sudo setcap 'cap_net_bind_service=+ep' /usr/local/bin/node-cap
getcap /usr/local/bin/node-cap
# /usr/local/bin/node-cap cap_net_bind_service=ep

# Now bind as a non-root user.
sudo -u nobody /usr/local/bin/node-cap /tmp/srv.js &
sleep 1
curl -sS http://localhost/
# ok

# Verify the running process's caps.
PID=$(pgrep -f node-cap | head -1)
getpcaps "$PID"
# pid 12345: cap_net_bind_service=ep

# Clean up.
sudo kill "$PID"
sudo rm /usr/local/bin/node-cap
```

What just happened, in one sentence per step:

1. `ps -eZ` showed the SELinux *type* of `sshd` — that type is what `targeted` policy keys all of sshd's rules on.
2. We let `httpd` request a file labeled outside the web types and watched the kernel write an AVC denial; `audit2allow` produced a candidate `.te` we then declined to install (the right fix is relabel/relocate, not policy growth).
3. `docker inspect` showed where the container's seccomp/AppArmor/capabilities knobs live; default seccomp is the runtime default unless `--security-opt seccomp=...` is set.
4. `setcap cap_net_bind_service=+ep` gave a single non-setuid binary one specific privilege, letting a non-root user bind a privileged port; `getpcaps` confirmed the live process's effective set.

That is the operator surface of the security primitives. Everything else in this doc is detail under one of those four moves.

---

## Related

- [../../security/fundamentals/03-authentication-vs-authorization.md](../../security/fundamentals/03-authentication-vs-authorization.md) — the authn/authz model these MAC primitives enforce at the kernel level.
- [../../security/fundamentals/02-threat-modeling-stride.md](../../security/fundamentals/02-threat-modeling-stride.md) — STRIDE categories that capabilities and seccomp directly mitigate (Elevation of Privilege, Tampering).
- [../../kubernetes/security/pod-security.md](../../kubernetes/security/pod-security.md) — how Kubernetes' Pod Security Standards compose `securityContext`, seccomp, and capabilities.
- [../../kubernetes/security/secrets-and-supply-chain.md](../../kubernetes/security/secrets-and-supply-chain.md) — adjacent surface for the same threat model.
- [../../operating-systems/fundamentals/05-syscalls-and-the-abi.md](../../operating-systems/fundamentals/05-syscalls-and-the-abi.md) — the syscall interface seccomp filters; understand the boundary before writing BPF for it.
- [../../operating-systems/fundamentals/01-process-thread-model.md](../../operating-systems/fundamentals/01-process-thread-model.md) — capability sets are *per-thread*; the process model is the prerequisite.

## References

- man7.org, `capabilities(7)` — https://man7.org/linux/man-pages/man7/capabilities.7.html
- man7.org, `seccomp(2)` — https://man7.org/linux/man-pages/man2/seccomp.2.html
- man7.org, `auditctl(8)` — https://man7.org/linux/man-pages/man8/auditctl.8.html
- man7.org, `auditd(8)` — https://man7.org/linux/man-pages/man8/auditd.8.html
- man7.org, `ausearch(8)` — https://man7.org/linux/man-pages/man8/ausearch.8.html
- man7.org, `setcap(8)` — https://man7.org/linux/man-pages/man8/setcap.8.html
- man7.org, `restorecon(8)` — https://man7.org/linux/man-pages/man8/restorecon.8.html
- kernel.org, *Seccomp BPF (SECure COMPuting with filters)* — https://www.kernel.org/doc/html/latest/userspace-api/seccomp_filter.html
- kernel.org, *Linux Security Modules* admin guide — https://www.kernel.org/doc/html/latest/admin-guide/LSM/index.html
- Red Hat, *Using SELinux* (RHEL 9) — https://docs.redhat.com/documentation/en-us/red_hat_enterprise_linux/9/html/using_selinux/index
- Docker, *Seccomp security profiles for Docker* — https://docs.docker.com/engine/security/seccomp/
- Kubernetes, *Restrict a Container's Syscalls with seccomp* — https://kubernetes.io/docs/tutorials/security/seccomp/
- Ubuntu manpages, `aa-status(8)` — https://manpages.ubuntu.com/manpages/jammy/en/man8/aa-status.8.html
- Ubuntu manpages, `apparmor.d(5)` — https://manpages.ubuntu.com/manpages/jammy/en/man5/apparmor.d.5.html
