---
title: "QSH vs PASE on IBM i: A Practical Guide"
description: A practical guide to QSH and PASE on IBM i — what they actually are, why the boundary matters, and when you need to bridge between them.
---

# QSH vs PASE on IBM i: A Practical Guide

If you've spent any time installing open-source software on IBM i, you've eventually hit a moment where something works in one shell and breaks in the other — or you copy a recipe from a forum and it does almost-but-not-quite the right thing. The cause is almost always the same: **QSH and PASE are two completely different runtime environments that happen to both give you a shell prompt.**

The IBM docs split the topic across half a dozen pages, the redbooks treat it as background, and most forum answers gloss over it. This guide pulls the practical side together: what each environment actually is, why the boundary matters in real work, when you have to bridge between them, and what the failure modes look like when you don't.

## TL;DR

- **QSH (Qshell)** is a POSIX-style shell written natively for IBM i. It runs in ILE, uses EBCDIC by default, and is tightly coupled to the IBM i job structure — library lists, job logs, IFS authorities. Started with `STRQSH` or `QSH`.
- **PASE (Portable Application Solutions Environment)** is a real AIX runtime grafted onto the i. The binaries running in it are AIX binaries, ASCII by default. Anything you install via `yum` from the IBM RPM repo runs here.
- **`CALL QP2TERM`** is one way to get into PASE — the interactive 5250 terminal. **`QP2SHELL`** and **`QP2SHELL2`** are the spawn variants you call from CL or RPG when you want to launch a PASE process non-interactively.
- They are **not interchangeable.** Most "weird IBM i shell problems" trace back to confusion about which one you're in.

## What QSH actually is

Qshell is IBM's native POSIX-ish shell for the i. It's compiled to ILE, runs as an IBM i job, and the utilities that ship with it (`ls`, `cat`, `grep`, `awk`, `sed`, `find`, etc.) are IBM's own ports — not the GNU coreutils.

What that means in practice:

- **It's an IBM i job.** It honors your library list, your CCSID, your job description, your auditing settings. `system` from QSH calls native IBM i commands the way you'd expect. `liblist` works. The job log is a real QSYS job log.
- **It's EBCDIC by default.** Stream files created by QSH are tagged with your job's CCSID unless you tell it otherwise.
- **Its utilities have IBM i extensions.** The `touch` in QSH has `-C` for CCSID. The `ls` shows IFS authorities. There are commands like `system`, `Rfile`, `db2` (the QSH wrapper) that don't exist anywhere else.
- **It can call native programs directly.** `CALL MYPGM` from QSH works. `qsh -c '...'` is the natural way to script CL-adjacent work.

QSH is the right tool when you're orchestrating native i objects, scripting CL-flavored work, or doing something that's tightly coupled to library-list behavior.

## What PASE actually is

PASE is an AIX binary runtime running inside an IBM i job. When you start PASE, the i loads an AIX system call emulator, and from that point on the process is, for all practical purposes, AIX. Native AIX binaries — Apache, Node, PHP, Python, git, openssl, bash, all of it — run unmodified in PASE.

Three ways to enter PASE:

- **`CALL QP2TERM`** — interactive 5250 terminal session. What you'd use from a green-screen for ad hoc PASE work.
- **`CALL QP2SHELL` / `QP2SHELL2`** — non-interactive PASE process spawned from CL. `QP2SHELL2` is the modern variant; the older `QP2SHELL` exists for compatibility.
- **SSH** — if you have SSH set up, the default shell drops you straight into PASE bash.

What that means in practice:

- **It's ASCII by default.** Stream files written from PASE are tagged ASCII. Bytes flow through stdin/stdout/stderr without any per-syscall translation.
- **It's where the open-source ecosystem lives.** Anything from `yum install` lands in `/QOpenSys/pkgs/` and runs in PASE. Modern bash, modern openssl, Node, Python, PHP runtimes — all PASE.
- **It can still call back into ILE.** Via `/QOpenSys/usr/bin/system` (a PASE wrapper that runs native CL commands), or via the QSYS API set, or via `db2` (the PASE one). But it's a foreign-function call, not a native operation.
- **It doesn't honor library lists naturally.** Your library list still exists on the parent IBM i job, but PASE programs don't read or modify it the way QSH does. You typically pass things in via environment variables.

PASE is the right tool when you're running anything from the modern open-source world, doing performance-sensitive Unix-style work, or invoking software (PHP, Node, Python, git, curl) that was written for a Unix environment.

## Why the boundary matters

### Encoding is the one that bites everyone

This is the single most common source of "why doesn't this script work" pain on IBM i. The default behavior:

- QSH creates IFS files with your job's CCSID (usually EBCDIC, e.g. 37).
- PASE creates IFS files tagged ASCII (typically CCSID 819 / ISO-8859-1).
- The IFS does on-the-fly translation between CCSIDs at read/write boundaries based on those tags.

If you create a file in QSH (EBCDIC-tagged) and then have a PASE process write ASCII bytes into it, the bytes go in raw — but the tag still says EBCDIC. The IFS translation layer thinks it needs to translate ASCII bytes as if they were EBCDIC. Result: a log file that displays as gibberish, or worse, that displays fine in one viewer and as gibberish in another, which is the maximally confusing failure mode.

The fix is to pre-tag the file with the right CCSID before the PASE process writes to it:

```sh
touch -C 819 /tmp/some.log
```

The `-C 819` is a QSH `touch` extension. It creates the stream file already tagged ASCII, so when PASE writes its stdout into it, no translation happens and the bytes survive readable. This is exactly what the OSPM bootstrap script does on its critical line — it's not a stylistic choice, it's required correctness.

### Library lists and native objects

QSH inherits and modifies your job's library list. PASE doesn't. If your script relies on `*LIBL` resolution to find a file or program, that script needs to run in QSH, or you need to construct the qualified name explicitly before handing off to PASE.

The PASE `system` command (`/QOpenSys/usr/bin/system`) lets you fire CL commands from inside PASE, including library-list manipulation, but it spawns a new IBM i job context to do it. That's fine for one-shot calls but expensive in a loop.

### PATH conventions

The two environments have different conventions for where binaries live:

- **QSH leans on:** `/usr/bin`, `/QOpenSys/usr/bin`, `/QIBM/ProdData/...`
- **PASE leans on:** `/QOpenSys/pkgs/bin` (yum-installed software), `/QOpenSys/usr/bin` (AIX-style legacy), and whatever you set in `.profile`

When you bridge from one to the other, **always use full paths.** Don't rely on PATH inheritance. `/QOpenSys/usr/bin/ksh` is unambiguous — it's the PASE ksh. `ksh` alone could resolve to either depending on context.

### Performance

PASE is generally faster for Unix-style workloads because there's no per-syscall translation. QSH is faster for native-i operations because there's no foreign-function-call overhead. The crossover point is usually obvious — if you're running a tight loop calling `system` from PASE, you're paying that overhead every iteration and would be better off staying in QSH or doing the work in a native program.

### Tooling availability

The big practical one: anything from yum is PASE-only. If you `yum install nodejs` and then try to `node` from QSH, you're either getting an error or QSH is silently invoking the PASE binary on your behalf — but you've lost most of the things QSH was useful for in the process. Run modern open-source tooling in PASE.

QSH-specific tooling (`Rfile`, the QSH `system` builtin, the QSH `db2` wrapper, CCSID-aware `touch` and `cp`) is not available in PASE. If you need those, stay in QSH.

## The bridge pattern

A surprising amount of IBM i scripting work consists of bridging from CL or RPG into PASE, doing the actual work, and capturing output. The pattern that works reliably:

```cl
QSH CMD('touch -C 819 /tmp/work.log; /QOpenSys/usr/bin/<pase-binary> <args> > /tmp/work.log 2>&1')
```

That's it. QSH is the bridge. The four pieces, in order:

1. **`touch -C 819`** — pre-creates the log file with ASCII CCSID so PASE's stdout doesn't get translated.
2. **`/QOpenSys/usr/bin/<binary>`** — explicit full path to a PASE binary. Never rely on PATH at the bridge boundary.
3. **`> /tmp/work.log 2>&1`** — standard shell redirection of stdout and stderr. QSH supports the same redirection syntax as bash, which is the convenience that makes this pattern worthwhile.
4. **The CL caller** can check the joblog for QSH completion (message ID `QSH0005`) to know the bridge ran cleanly.

You can also bridge with `QP2SHELL2` directly — `CALL PGM(QP2SHELL2) PARM('/QOpenSys/usr/bin/ksh' '/path/to/script.sh')` — but you lose the convenient inline redirection and have to handle output capture differently. The QSH bridge is idiomatic and what IBM uses in its own bootstrap docs.

When **not** to use the QSH bridge: when you're already in PASE (e.g., from an SSH session or another PASE script). At that point you're just calling a PASE binary from PASE — no bridge needed. The bridge exists specifically to cross the CL → PASE boundary.

## Worked example: bootstrapping yum

The OSPM bootstrap script in our [SetupOSPMforIBMi](https://github.com/K3S/SetupOSPMforIBMi) repo is a clean worked example of every concept above. Here's the critical line:

```cl
QSH CMD('touch -C 819 /tmp/bootstrap.log; /QOpenSys/usr/bin/ksh /tmp/bootstrap.sh > /tmp/bootstrap.log 2>&1')
```

Walking through it:

- The script is run from ACS **Run SQL Scripts** — a SQL environment. SQL scripts can issue CL via the `CL:` prefix but cannot directly run PASE binaries.
- `bootstrap.sh` is IBM's PASE script that fetches the RPMs and installs yum into `/QOpenSys/pkgs/`. It has to run in PASE; there's nowhere else for it to run.
- The bridge: SQL → CL → QSH → PASE ksh. Each hop is necessary, and each is doing exactly one job.
- The `touch -C 819` makes sure the log file is ASCII-tagged so the PASE ksh process's stdout is captured readable.
- `/QOpenSys/usr/bin/ksh` is the PASE Korn shell. Critical: this is **not** QSH. The full path makes that explicit.
- `2>&1` folds stderr into stdout so any error output ends up in the same log.

After the QSH command returns, the SQL script reads back the joblog looking for QSH's completion message and reports success or failure. The whole thing is a textbook bridge.

## Quick reference

| If you're doing this... | Use this |
| --- | --- |
| Running anything from `yum install` (Node, Python, PHP, git, modern bash) | PASE |
| Calling native IBM i programs, manipulating library lists | QSH |
| CCSID-aware file work, IFS authority manipulation | QSH |
| Scripting from CL with shell redirection | QSH |
| Long-running web server, PHP-FPM, modern open-source workload | PASE |
| Bridging from CL/SQL into a PASE binary | QSH as one-line bridge |
| Interactive PASE shell from a green-screen | `CALL QP2TERM` |
| Spawning a PASE process from CL non-interactively | `QP2SHELL2` (or QSH bridge) |
| SSH session default shell | PASE |

| Common pitfall | What's actually happening | Fix |
| --- | --- | --- |
| Log file from a PASE process renders as garbage | File was created EBCDIC-tagged before PASE wrote ASCII into it | `touch -C 819` the log file before the PASE process writes |
| `node` / `python3` "works" from QSH but acts weird | QSH is silently dispatching to a PASE binary; you've lost QSH's environment | Run open-source tooling in PASE explicitly |
| Library list isn't honored from a yum-installed program | PASE programs don't read `*LIBL` natively | Pass qualified names, or wrap with `/QOpenSys/usr/bin/system` |
| Script works in green-screen but fails from SSH | Green-screen probably ran QSH; SSH dropped you into PASE | Pick one environment for the script and use full paths to all binaries |

## When in doubt

If you're not sure which environment a piece of code needs to run in, ask: **was the binary I'm running installed by yum, or is it part of base IBM i?** If yum, it's PASE. If base IBM i, almost certainly QSH (or a native program callable from QSH).

If you're writing scripts that need to run reliably across both environments, normalize on full paths to every binary, set CCSID explicitly when creating IFS files, and treat the QSH-as-bridge pattern as the default way to cross from CL into PASE. It's the pattern IBM uses themselves, and it's the one that fails the most predictably when something goes wrong.
