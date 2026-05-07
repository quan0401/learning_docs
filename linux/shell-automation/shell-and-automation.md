---
title: "Shell and Automation — Bash Internals, awk/sed, xargs/parallel, Scheduling"
date: 2026-05-07
updated: 2026-05-07
tags: [linux, bash, awk, sed, cron, automation]
---

# Shell and Automation — Bash Internals, awk/sed, xargs/parallel, Scheduling

**Date:** 2026-05-07 | **Updated:** 2026-05-07
**Tags:** `linux` `bash` `awk` `sed` `cron` `automation`

---

## Table of Contents

- [Summary](#summary)
- [1. Bash Internals — Quoting, Expansion, Process Substitution, set -euo pipefail](#1-bash-internals--quoting-expansion-process-substitution-set--euo-pipefail) — order of expansions, quoting, IFS, arrays, process substitution, here-docs, `set -euo pipefail`, `trap`, `PIPESTATUS`
- [2. awk and sed — The Text-Processing Pair Worth Knowing](#2-awk-and-sed--the-text-processing-pair-worth-knowing) — sed addressing, `s///`, `-i` portability; awk records/fields, BEGIN/END, associative arrays
- [3. xargs, parallel, and find -exec — Composition Patterns](#3-xargs-parallel-and-find--exec--composition-patterns) — `-print0`/`-0`, `-exec ;` vs `+`, xargs flags, GNU parallel and the citation quirk
- [4. cron, systemd Timers, and at — Scheduling Comparison](#4-cron-systemd-timers-and-at--scheduling-comparison) — crontab format, environment gotcha, systemd timer directives, `at`, side-by-side comparison
- [5. Operator Walkthrough](#5-operator-walkthrough) — bash script with trap, awk column-sum, parallel xargs pipeline, hourly systemd timer
- [Related](#related)
- [References](#references)

---

## Summary

Bash is a programming language with deeply unusual semantics — quoting and expansion rules decide whether your script handles a filename like `My Documents/log file.txt` or silently corrupts it. This doc covers the order of expansions, the `set -euo pipefail` triad, when sed/awk/xargs each earn their place, and the practical differences between cron, systemd timers, and `at` for scheduling. The goal is operator competence: write glue that survives a filename with a newline in it and a job that survives a reboot.

---

## 1. Bash Internals — Quoting, Expansion, Process Substitution, set -euo pipefail

### 1.1 The order of expansions

Per bash(1):

> The order of expansions is: brace expansion, tilde expansion, parameter and variable expansion, arithmetic expansion, and command substitution (done in a left-to-right fashion), word splitting, and pathname expansion.

The seven steps:

1. **Brace expansion** — `{a,b,c}` → `a b c`, `{1..5}` → `1 2 3 4 5`
2. **Tilde expansion** — `~` → `$HOME`, `~user` → that user's home
3. **Parameter and variable expansion** — `$VAR`, `${VAR:-default}`
4. **Arithmetic expansion** — `$((1 + 2))`
5. **Command substitution** — `$(cmd)` or backticks
6. **Word splitting** — splits the result on `IFS` characters (only for unquoted expansions)
7. **Pathname (glob) expansion** — `*.txt` → matched filenames

bash(1) also notes:

> Only brace expansion, word splitting, and pathname expansion can change the number of words of the expansion.

That is the source of most bash bugs. If you write `cp $src $dst` and `$src` contains a space, it becomes two arguments after word splitting. The fix is always `cp "$src" "$dst"`. Process substitution (`<(cmd)` / `>(cmd)`) is listed by bash(1) as an additional expansion available on systems that support FIFOs or `/dev/fd`.

### 1.2 Quoting — single, double, backtick, $()

| Form | Variable expansion | Command substitution | Backslash escapes | Use when |
|------|--------------------|----------------------|-------------------|----------|
| `'literal'` | no | no | no | Truly literal — passwords, regex with `$`, awk programs |
| `"with $var"` | yes | yes | partial | Almost always — quoting variables |
| `` `cmd` `` | n/a | yes (legacy) | tricky nesting | Avoid in new code |
| `$(cmd)` | n/a | yes | clean nesting | Always over backticks |

Backticks require backslash-escaping for nested backticks; `$(...)` does not, and nests cleanly: `$(echo $(date))`. The rule that catches Node devs: **double-quote every variable expansion unless you have a specific reason not to.** An unquoted `$var` is word-split and glob-expanded before the command runs.

```bash
# WRONG — word-split, then glob-expanded
rm $files

# RIGHT — single argument, exactly the literal value
rm "$file"

# RIGHT — array, expanded as separate quoted words
rm "${files[@]}"
```

### 1.3 Word splitting and IFS

`IFS` controls word splitting. Per bash(1): "The Internal Field Separator ... default value is `<space><tab><newline>`." Two practical patterns:

```bash
# Read a colon-delimited file safely
while IFS=: read -r user _ uid gid _; do
  printf 'user=%s uid=%s gid=%s\n' "$user" "$uid" "$gid"
done < /etc/passwd

# Force null-separated parsing for filenames with newlines
while IFS= read -r -d '' path; do
  printf 'path=%q\n' "$path"
done < <(find . -type f -print0)
```

`IFS=` (empty) plus `read -r` disables splitting and backslash-mangling; `-d ''` sets the delimiter to NUL, which is the only byte guaranteed not to appear in a path.

### 1.4 Arrays vs strings

A space-delimited string is not an array. Once a filename contains a space, a string-of-paths breaks.

```bash
# WRONG — splits on every space
files="a.txt b file.txt c.txt"
for f in $files; do rm "$f"; done   # tries to rm "b" and "file.txt"

# RIGHT — array
files=("a.txt" "b file.txt" "c.txt")
for f in "${files[@]}"; do rm "$f"; done
```

`"${arr[@]}"` expands to one quoted word per element; `"${arr[*]}"` joins with the first character of `IFS`. Use `[@]` 99% of the time.

### 1.5 Process substitution and here-docs

Process substitution turns a command's output into a filename:

```bash
# Compare two `find` outputs without temp files
diff <(find /etc -type f | sort) <(find /etc.bak -type f | sort)

# Tee output to two pipelines
cmd | tee >(grep ERROR > errors.log) >(grep WARN > warns.log) > /dev/null
```

Here-docs (`<<EOF`) feed a multiline string as stdin. The `<<-` form strips leading **tabs** (not spaces) from each line, which lets you indent the body inside a function:

```bash
deploy_message() {
    cat <<-EOF
	Deploying $1 to $2
	Started at $(date -u +%FT%TZ)
	EOF
}
```

Quote the delimiter (`<<'EOF'`) to disable variable and command substitution inside the body.

### 1.6 set -e, set -u, set -o pipefail, trap

`set -euo pipefail` together catches most silent failures. Each has gotchas.

**`set -e` (errexit).** From bash(1):

> Exit immediately if a simple command exits with a non-zero status. The shell does not exit if the command that fails is part of the command list immediately following a while or until keyword, part of the test in an if statement, part of a `&&` or `||` list, or if the command's return value is being inverted via `!`.

Consequences:

- A command in `if cmd; then ...` is allowed to fail.
- A command on the left of `&&` is allowed to fail.
- The classic surprise: `set -e; foo() { false; echo after; }; foo && echo ok` — `false` sits inside an `&&` list, so errexit does not fire. Documented behavior, not a bug.

**`set -u` (nounset).** Treats expansion of an unset variable as an error. Forces `"${VAR:-default}"` or `"${VAR-}"` for legitimately-empty variables. Catches the worst bash bug of all time:

```bash
# Without set -u, if $PREFIX is unset, this becomes `rm -rf /usr/local`
rm -rf "$PREFIX/usr/local"
```

**`set -o pipefail`.** From bash(1):

> If pipefail is enabled, the pipeline's return status is the value of the last (rightmost) command to exit with a non-zero status, or zero if all commands exit successfully.

Without pipefail, `false | true` exits 0. With pipefail, it fails. Always set when piping into `tee`, `grep`, `head`, etc.

**`trap` for cleanup.** Run a handler on signal or shell exit.

```bash
tmpdir=$(mktemp -d)
trap 'rm -rf "$tmpdir"' EXIT INT TERM
```

`EXIT` fires no matter how the script ends. `ERR` (with `set -E`) fires only on non-zero exit — use both if you want both error logging and cleanup.

### 1.7 Exit codes — $? and PIPESTATUS

`$?` is the exit code of the most recent foreground command. For pipelines, only the rightmost command's status is in `$?` (unless pipefail is on). For all stages, bash provides `PIPESTATUS` — bash(1) describes it as "An array variable ... containing a list of exit status values from the processes in the most-recently-executed foreground pipeline."

```bash
curl -s https://example.com | jq .name
# Did curl fail or did jq?
echo "curl=${PIPESTATUS[0]} jq=${PIPESTATUS[1]}"
```

`PIPESTATUS` is overwritten by the next command — copy it to a variable immediately if you need it later.

---

## 2. awk and sed — The Text-Processing Pair Worth Knowing

These two tools have survived 50 years for a reason: they handle the line-oriented 90% case in roughly one printable line. Reach for them when a one-liner saves a script.

### 2.1 sed addressing and substitution

sed processes input one line at a time, applying commands to lines that match an *address*.

| Address | Selects |
|---------|---------|
| (none) | All lines |
| `5` | Line 5 only |
| `1,10` | Lines 1 through 10 |
| `/regex/` | Lines matching the regex |
| `/start/,/end/` | From `start` line through `end` line |
| `$` | Last line |

The substitute command is `s/pattern/replacement/flags`:

```bash
# Replace first foo per line with bar
sed 's/foo/bar/'

# Replace all on each line
sed 's/foo/bar/g'

# Capture groups: \1, \2, ... refer to (...) groups
sed -E 's/^([0-9]+) (.+)$/\2 [\1]/'

# Only on lines matching /ERROR/
sed '/ERROR/ s/foo/bar/g'
```

`-E` (or `-r` on GNU) enables extended regex — no need to backslash `(`, `)`, `+`, `?`.

### 2.2 sed -i — in-place editing portability trap

The single biggest portability footgun. The macOS (BSD) sed(1) man page documents:

> -i extension — Edit files in-place similarly to -I, but treat each file independently from other files.

BSD sed **requires an extension argument** (use `''` for no backup). GNU sed accepts `-i` with no argument. So:

```bash
# Linux / GNU sed — works
sed -i 's/foo/bar/g' file.txt

# macOS / BSD sed — fails: tries to use 's/foo/bar/g' as the extension
sed -i 's/foo/bar/g' file.txt

# Portable form — works on both
sed -i '' 's/foo/bar/g' file.txt   # macOS
sed -i.bak 's/foo/bar/g' file.txt  # both, leaves file.txt.bak
```

Portable scripts usually write `sed -i.bak ...` and then `rm -f file.txt.bak`, or detect the platform.

### 2.3 awk records, fields, and built-in variables

awk reads input one *record* at a time and splits each record into *fields*. Defaults: record = line, fields separated by whitespace runs. From awk(1): "An input line is normally made up of fields separated by white space, or by the regular expression FS. The fields are denoted $1, $2, ..., while $0 refers to the entire line."

| Variable | Meaning |
|----------|---------|
| `$0` | Entire current record |
| `$1`, `$2`, ... | Field 1, 2, ... |
| `NF` | Number of fields in current record |
| `NR` | Ordinal number of current record (line number across all input) |
| `FNR` | Record number within current file |
| `FS` | Input field separator (default: whitespace, settable with `-F`) |
| `OFS` | Output field separator (default: single space) |
| `ORS` | Output record separator (default: newline) |
| `RS` | Input record separator (default: newline) |
| `FILENAME` | Name of current input file |

### 2.4 awk pattern-action, BEGIN/END, associative arrays

An awk program is a sequence of `pattern { action }` pairs. The action runs on every record where the pattern is true. awk(1): "The special patterns BEGIN and END may be used to capture control before the first input line is read and after the last."

```bash
# Print user and shell from /etc/passwd
awk -F: '{ print $1, $7 }' /etc/passwd

# Sum column 5 (file sizes from `ls -l`)
ls -l | awk '/^-/ { sum += $5 } END { print sum }'

# Group-by: count requests per status code in an access log
awk '{ count[$9]++ } END { for (s in count) print s, count[s] }' access.log

# Dedup while preserving order
awk '!seen[$0]++' input.txt
```

That last one is worth memorizing. `seen[$0]++` returns the *current* count, then increments; `!` negates. First appearance: 0 (falsy) → print; subsequently truthy → skip.

The awk(1) man page itself gives the column-sum example: `{ s += $1 } END { print "sum is", s, " average is", s/NR }`.

### 2.5 When to reach for Python or Perl instead

awk and sed are correct when the data is line-oriented and reasonably regular, and the transformation is a substitution, projection, group-by, or dedup that fits in a one-liner or a 5-line snippet.

Reach for Python (or Perl, Ruby) when you need multi-line records (JSON, XML, YAML, CSV with embedded newlines), dates or network calls, non-trivial state, or the script will exceed ~30 lines or be modified by people who don't know awk. The common mistake: "I already wrote 40 lines of awk." That's the moment the script should have been Python.

---

## 3. xargs, parallel, and find -exec — Composition Patterns

These three tools take output from one process and feed it as arguments to another. The differences between them affect both correctness (filenames with whitespace) and parallelism.

### 3.1 find -print0 and xargs -0 — the only safe way

Default `find ... | xargs ...` is *broken* for any filename containing a space, tab, newline, or quote — xargs splits on whitespace and treats quotes specially. `find -print0` separates paths with a NUL byte (the only byte that cannot appear in a path); `xargs -0` reads NUL-separated input.

find(1): "-print0 ... prints the pathname of the current file to standard output, followed by an ASCII NUL character (character code 0)." xargs(1): "-0, --null — Change xargs to expect NUL (``\0'') characters as separators ... expected to be used in concert with the -print0 function in find(1)." If you ever pipe `find` to `xargs`, use `-print0` and `-0`. Otherwise the script is broken on input you haven't seen yet.

```bash
# Safe
find . -name '*.log' -mtime +30 -print0 | xargs -0 rm

# Broken on filenames with spaces, newlines, or quotes
find . -name '*.log' -mtime +30 | xargs rm
```

### 3.2 find -exec — `;` vs `+`

`find -exec` invokes a command for each (or batched) match. find(1) documents two forms: `-exec utility {} ;` runs once per file (the "{}" is replaced by the current pathname), while `-exec utility {} +` is "Same as -exec, except that '{}' is replaced with as many pathnames as possible for each invocation of utility. This behaviour is similar to that of xargs(1)."

| Form | Invocations | When to use |
|------|-------------|-------------|
| `-exec cmd {} \;` | One per file | Command must see exactly one path |
| `-exec cmd {} +` | Batched (many paths per invocation) | Default — much faster |

```bash
# Slow — one fork per file
find . -name '*.bak' -exec rm {} \;

# Fast — rm gets many args at once
find . -name '*.bak' -exec rm {} +
```

`-exec ... +` does not need NUL escaping because find passes arguments directly via `execve`, never through the shell.

### 3.3 xargs flags that matter

| Flag | Effect |
|------|--------|
| `-0` | NUL-separated input (pair with `find -print0`) |
| `-n N` | Run the command with at most N arguments per invocation |
| `-P N` | Run up to N invocations in parallel |
| `-I REPLSTR` | Replace `REPLSTR` (commonly `{}`) with the input — one invocation per input |
| `-r` | Don't run the command at all if input is empty (GNU; FreeBSD `xargs` accepts `-r` for compatibility but does not run on empty input by default) |

xargs(1) on `-P`: "Parallel mode: run at most maxprocs invocations of utility at once. If maxprocs is set to 0, xargs will run as many processes as possible."

```bash
# Resize all PNGs in parallel, 4 at a time
find . -name '*.png' -print0 |
  xargs -0 -P 4 -I{} convert {} -resize 800x800 {}.thumb.png

# Pass exactly one arg per invocation (needed with -I or for commands that
# only accept one)
find . -name '*.bak' -print0 | xargs -0 -n 1 -P 8 gzip
```

Note: `-I{}` implies `-n 1` — each invocation gets exactly one input.

### 3.4 GNU parallel — and the citation quirk

GNU parallel is a more featureful alternative to `xargs -P`: multiple input sources, structured output (`--keep-order`), pty allocation, remote execution over SSH, progress meters. The notable quirk is the citation notice on first interactive run. Per the GNU parallel docs: "The development of GNU parallel is indirectly financed through citations, so if your users do not know they should cite then you are making it harder to finance development."

Silence it by running `parallel --citation` once (acknowledges permanently for that user), or pass `--will-cite` per invocation. The docs note that `--will-cite` in distributed scripts hides the citation notice from end users — something the maintainers ask you not to do casually.

```bash
# 4 jobs in parallel, each gets one filename
find . -name '*.log' | parallel -j 4 gzip {}

# With explicit input list, two parameter slots
parallel -j 4 'echo {1} {2}' ::: a b c ::: 1 2 3
```

**xargs vs parallel — when to choose which:**

| Need | Tool |
|------|------|
| Default plumbing | `xargs -0 -P N` (no extra dependency) |
| Multiple input sources combined | parallel |
| Ordered output regardless of completion order | parallel `--keep-order` |
| SSH out-of-the-box | parallel |
| Progress meter | parallel `--bar` |
| Don't want the citation friction in CI | xargs |

### 3.5 Pipefail interaction

`set -o pipefail` covers pipelines, but `xargs` is a single process — its exit code follows its own rules: `0` if all child invocations succeeded, `1`–`125` if any failed, `126`/`127` for "not executable" / "not found". Pipefail surfaces xargs's own exit code at the pipeline's end, but does not propagate individual child failures from inside xargs. GNU parallel's `--halt soon,fail=1` stops on first failure — xargs does not.

---

## 4. cron, systemd Timers, and at — Scheduling Comparison

Three Unix scheduling mechanisms with overlapping but distinct uses. Modern Linux servers usually have all three available; pick by what the job actually needs.

### 4.1 crontab format

A crontab line has five time fields followed by a command. Per crontab(5): "Each line has five time and date fields, followed by a user name if this is the system crontab file, followed by a command."

| Field | Range |
|-------|-------|
| minute | 0–59 |
| hour | 0–23 |
| day of month | 1–31 |
| month | 1–12 |
| day of week | 0–7 (0 and 7 are both Sunday) |

Wildcards `*`, ranges `1-5`, lists `1,15,30`, and steps `*/10` are all permitted in any field. crontab(5) on man7.org also documents nickname strings as alternatives to the five-field form:

> @reboot   — Run once after reboot.
> @yearly   — Run once a year, ie. "0 0 1 1 *".
> @annually — Run once a year, ie. "0 0 1 1 *".
> @monthly  — Run once a month, ie. "0 0 1 * *".
> @weekly   — Run once a week, ie. "0 0 * * 0".
> @daily    — Run once a day, ie. "0 0 * * *".
> @hourly   — Run once an hour, ie. "0 * * * *".

Where the crontab lives matters:

| Location | Owner | Format |
|----------|-------|--------|
| `crontab -e` (per user) | the invoking user | five fields + command (no user column) |
| `/etc/crontab` (system) | root | five fields + **user** + command |
| `/etc/cron.d/*` (system drop-ins) | root | same as `/etc/crontab` |
| `/etc/cron.{hourly,daily,weekly,monthly}/` | root | scripts run at that interval |

`crontab -e` edits your user's crontab in `$EDITOR` (or `$VISUAL`); `crontab -l` lists; `crontab -r` removes (no confirmation — handle with care).

### 4.2 The cron environment is not your shell

The single most common cron bug: "it works when I run it manually, but the cron job fails." cron starts your job with a minimal environment. Per crontab(5): "SHELL is set to /bin/sh, and LOGNAME and HOME are set from the /etc/passwd line of the crontab's owner. HOME and SHELL can be overridden by settings in the crontab; LOGNAME can not." It also reads `MAILTO` for mail notifications.

What you do **not** get: no `~/.bashrc`, no `~/.profile`, no aliases, no interactive `PATH` (commands like `node`, `kubectl`, `aws` may not be found), no `SSH_AUTH_SOCK`, no `DISPLAY`, no `TERM`.

Mitigations, in order of preference:

1. Always use absolute paths for binaries (`/usr/local/bin/node`, not `node`).
2. Set `PATH=...` at the top of the crontab.
3. Wrap the job in a script that sources the env it needs explicitly.

```text
# /etc/cron.d/backup
SHELL=/bin/bash
PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
MAILTO=ops@example.com

15 3 * * * backup /usr/local/bin/run-backup.sh
```

### 4.3 systemd timers

systemd timers replace cron with two unit files: a `.service` that defines the work and a `.timer` that fires it. Key directives from systemd.timer(5):

| Directive | Meaning (from man page) |
|-----------|------------------------|
| `OnCalendar=` | "Defines realtime (i.e. wallclock) timers with calendar event expressions." |
| `OnBootSec=` | "Defines a timer relative to when the machine was booted up." |
| `OnStartupSec=` | "Defines a timer relative to when the service manager was first started." |
| `OnUnitActiveSec=` | "Defines a timer relative to when the unit the timer unit is activating was last activated." |
| `OnUnitInactiveSec=` | Relative to when the activated unit was last deactivated. |
| `Persistent=` | "If true, the time when the service unit was last triggered is stored on disk." |
| `AccuracySec=` | "Specify the accuracy the timer shall elapse with. Defaults to 1min." |
| `RandomizedDelaySec=` | Adds a random evenly-distributed delay up to the specified value. |
| `Unit=` | The unit to activate when this timer elapses (suffix is not `.timer`). |

`OnCalendar=` uses the systemd.time(7) calendar-event syntax: `[DayOfWeek] Year-Month-Day Hour:Minute:Second`. Shorthand strings (`hourly`, `daily`, `weekly`, `monthly`, `yearly`), wildcards (`*`), comma-lists, ranges (`..`), and `/`-stepped repetitions are all permitted — strictly more expressive than cron's five fields.

The minimal pair:

```ini
# /etc/systemd/system/cleanup.service
[Unit]
Description=Clean tmp older than 7 days

[Service]
Type=oneshot
ExecStart=/usr/local/bin/cleanup-tmp.sh
```

```ini
# /etc/systemd/system/cleanup.timer
[Unit]
Description=Run cleanup hourly

[Timer]
OnCalendar=hourly
Persistent=true
Unit=cleanup.service

[Install]
WantedBy=timers.target
```

`Persistent=true` means: if the machine was off when the timer should have fired, fire it on boot once — an operationally important advantage over cron, which silently skips missed runs (the laptop-overnight problem). You also get for free: journal logs (`journalctl -u cleanup.service`), resource control (`MemoryMax=`, `CPUQuota=`, `Nice=`), dependencies (`After=network-online.target`), and `systemctl list-timers` for status.

### 4.4 at — one-off scheduling

`at` is for "run this once, at this time." Per at(1): "The at and batch utilities read commands from standard input or a specified file which are to be executed at a later time, using sh(1)." Time formats include `HH:MM`, keywords (`midnight`, `noon`, `teatime`), date forms (`Jul 31`, `DD.MM.YYYY`), and relative offsets (`now + 30 minutes`, `4pm + 3 days`).

```bash
# One-off: rotate a log at 2am
echo '/usr/local/bin/rotate-logs.sh' | at 2am tomorrow

# atq — list pending jobs
atq

# atrm <jobid> — cancel
atrm 12
```

at(1) notes that `at` preserves the invoking shell's working directory, environment (except `TERM`, `TERMCAP`, `DISPLAY`, `_`), and umask — significantly more forgiving than cron. `batch` is a sibling that runs jobs only when system load drops below a threshold. `at` is not always installed by default on minimal distro images.

### 4.5 Side-by-side comparison

| Feature | cron | systemd timer | at |
|---------|------|---------------|----|
| Recurrence | yes (5 fields + nicknames) | yes (OnCalendar, OnUnitActiveSec, ...) | no — single shot |
| Catch-up after downtime | no (skips missed runs) | yes with `Persistent=true` | no — but the wallclock time is honored if up |
| Logs | depends — usually mailed via `MAILTO` | journal — `journalctl -u <unit>` | mailed (sendmail) |
| Resource control | only via `nice`/`ulimit` in the script | full systemd cgroup directives | only via shell |
| Dependencies | none | `After=`, `Requires=`, `Wants=` | none |
| Environment | minimal, no shell rc | minimal but configurable per-service | inherits invoking shell |
| Per-user | `crontab -e` | user units in `~/.config/systemd/user/` | yes |
| Portability | every Unix | systemd-based Linux | most Unixes |
| Best for | legacy and portable scripts | anything on a modern Linux server | "remind me / run once at X" |

**On a modern systemd Linux box, prefer timers.** Use cron when the system isn't systemd or when you want one less moving part for a trivial job. Use `at` for ad-hoc one-off scheduling.

---

## 5. Operator Walkthrough

The commands below are reproducible on any Linux box (a throwaway VM or container). Where macOS behavior differs, it's called out.

### 5.1 Robust bash script with trap cleanup

Save as `/tmp/safe-merge.sh`:

```bash
#!/usr/bin/env bash
# Merge two sorted files into a third, atomically.
set -euo pipefail

if [[ $# -ne 3 ]]; then
    echo "usage: $0 <a> <b> <out>" >&2
    exit 64
fi

src_a=$1
src_b=$2
dst=$3

# Working directory cleaned up regardless of how we exit
workdir=$(mktemp -d)
trap 'rm -rf "$workdir"' EXIT

tmp="$workdir/merged"

# Pipefail makes a sort failure abort even though tee wrote to stdout
sort -m "$src_a" "$src_b" | tee "$tmp" > /dev/null

# Atomic rename — readers either see the old file or the new one
mv "$tmp" "$dst"
echo "wrote $(wc -l < "$dst") lines to $dst"
```

Run and force a failure to verify cleanup:

```bash
chmod +x /tmp/safe-merge.sh
printf 'a\nc\ne\n' > /tmp/a.txt
printf 'b\nd\nf\n' > /tmp/b.txt
/tmp/safe-merge.sh /tmp/a.txt /tmp/b.txt /tmp/out.txt   # success path

/tmp/safe-merge.sh /tmp/missing.txt /tmp/b.txt /tmp/out.txt
# sort fails, pipefail propagates, trap runs rm -rf on the tempdir
```

### 5.2 awk: sum a column from a sample log

Generate a tiny synthetic access log:

```bash
cat > /tmp/access.log <<'EOF'
10.0.0.1 - - [01/May/2026:10:00:01 +0000] "GET / HTTP/1.1" 200 1024
10.0.0.2 - - [01/May/2026:10:00:02 +0000] "GET /api HTTP/1.1" 200 2048
10.0.0.1 - - [01/May/2026:10:00:03 +0000] "GET /img HTTP/1.1" 304 0
10.0.0.3 - - [01/May/2026:10:00:04 +0000] "POST /api HTTP/1.1" 500 256
10.0.0.2 - - [01/May/2026:10:00:05 +0000] "GET /api HTTP/1.1" 200 2048
EOF
```

Total bytes served (column 10):

```bash
awk '{ total += $10 } END { print "bytes:", total }' /tmp/access.log
# bytes: 5376
```

Bytes per status code (column 9), and top IPs by request count:

```bash
awk '{ by_status[$9] += $10 }
     END { for (s in by_status) printf "%s\t%d\n", s, by_status[s] }' /tmp/access.log

awk '{ count[$1]++ }
     END { for (ip in count) printf "%-15s %d\n", ip, count[ip] | "sort -k2 -nr" }' /tmp/access.log
```

### 5.3 Parallel find/xargs pipeline

Generate 200 dummy files, then gzip them with 4-way parallelism:

```bash
mkdir -p /tmp/many
for i in $(seq 1 200); do
    head -c $((RANDOM % 8192 + 1)) /dev/urandom > "/tmp/many/file $i.bin"
done

# Note the space in filenames — -print0 / -0 is mandatory
find /tmp/many -name '*.bin' -print0 |
    xargs -0 -P 4 -n 1 gzip

ls /tmp/many | head -5
# file 1.bin.gz, file 10.bin.gz, ...
```

`-n 1` makes parallelism per-file rather than per-batch — useful for demonstrating `-P 4`. Run `pgrep -c gzip` in another shell while the pipeline runs — it should hover near 4.

### 5.4 Hourly systemd timer with journalctl history

Requires a Linux box with systemd (real VM, `systemd`-based image, or cloud instance — not a plain Docker container).

Service unit:

```ini
# /etc/systemd/system/heartbeat.service
[Unit]
Description=Heartbeat — write the current time to a file

[Service]
Type=oneshot
ExecStart=/bin/sh -c 'date -u +%FT%TZ >> /var/log/heartbeat.log'
```

Timer unit:

```ini
# /etc/systemd/system/heartbeat.timer
[Unit]
Description=Run heartbeat every hour

[Timer]
OnCalendar=hourly
Persistent=true
AccuracySec=1s
Unit=heartbeat.service

[Install]
WantedBy=timers.target
```

Install and enable:

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now heartbeat.timer

# Confirm it's scheduled
systemctl list-timers heartbeat.timer
# NEXT                          LEFT      LAST  PASSED UNIT             ACTIVATES
# Thu 2026-05-07 15:00:00 UTC   42min     n/a   n/a    heartbeat.timer  heartbeat.service

# Trigger once manually to verify the service works
sudo systemctl start heartbeat.service
cat /var/log/heartbeat.log

# Watch the journal — every fire is logged automatically
journalctl -u heartbeat.service --since '1 hour ago'
```

After a few hours, `journalctl -u heartbeat.service` shows one start/stop pair per fire — that journal entry is the operational benefit over cron, which would either email on failure or leave no trace at all. Clean up with `sudo systemctl disable --now heartbeat.timer && sudo rm /etc/systemd/system/heartbeat.{timer,service} && sudo systemctl daemon-reload`.

---

## Related

- [Process and Thread Model](../../operating-systems/fundamentals/01-process-thread-model.md) — what `fork`/`exec` actually do under every shell command
- [Syscalls and the ABI](../../operating-systems/fundamentals/05-syscalls-and-the-abi.md) — `execve`, `pipe`, `dup2` underpin pipelines and process substitution
- [File Descriptors and ulimits](../../operating-systems/fundamentals/06-file-descriptors-and-ulimits.md) — fd inheritance is what makes here-docs and `<()` work
- [Structured Logging](../../observability/fundamentals/05-structured-logging.md) — how to emit logs from cron/timer jobs that are actually queryable
- [Three Pillars — Metrics, Logs, Traces](../../observability/fundamentals/01-three-pillars-metrics-logs-traces.md) — where journal output from systemd timers fits

## References

- bash(1) — GNU Bash man page (local: `/usr/share/man/man1/bash.1`). Sections cited: order of expansions, IFS, PIPESTATUS, `set -e` (errexit), `set -o pipefail`, process substitution, `trap`.
- sed(1) — BSD sed man page (local: `/usr/share/man/man1/sed.1`). Section cited: `-i` requires an extension argument; `-i ''` for no backup.
- awk(1) — BWK/one-true-awk man page (local: `/usr/share/man/man1/awk.1`). Sections cited: fields and `$0`/`NF`/`NR`, `BEGIN`/`END`, column-sum example.
- find(1) — BSD find man page (local: `/usr/share/man/man1/find.1`). Sections cited: `-print0`, `-exec ... ;`, `-exec ... +`.
- xargs(1) — BSD xargs man page (local: `/usr/share/man/man1/xargs.1`). Sections cited: `-0`, `-n`, `-P`, `-I`, `-r`.
- crontab(1) — Vixie cron crontab(1) man page (local: `/usr/share/man/man1/crontab.1`).
- crontab(5) — man7.org Linux crontab(5): https://man7.org/linux/man-pages/man5/crontab.5.html — five-field format, `@reboot`/`@hourly`/`@daily`/etc. nicknames, `SHELL=/bin/sh` and `LOGNAME`/`HOME`/`MAILTO` defaults.
- systemd.timer(5) — Debian bookworm: https://manpages.debian.org/bookworm/systemd/systemd.timer.5.en.html — `OnCalendar`, `OnBootSec`, `OnUnitActiveSec`, `Persistent`, `AccuracySec`, `RandomizedDelaySec`, `Unit`.
- systemd.time(7) — Debian bookworm: https://manpages.debian.org/bookworm/systemd/systemd.time.7.en.html — calendar event expression syntax (`[DayOfWeek] Year-Month-Day Hour:Minute:Second`, shorthand strings).
- at(1) — local: `/usr/share/man/man1/at.1`. Sections cited: time formats, environment preservation (except `TERM`, `TERMCAP`, `DISPLAY`, `_`).
- GNU parallel docs: https://www.gnu.org/software/parallel/parallel.html — citation notice, `--will-cite` option, comparison vs xargs.
