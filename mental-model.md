---
title: The mental model
nav_order: 2
---

# The mental model

Before installing anything, internalize how IBM i actually presents itself to open source. Almost every confusion in this space is a quadrant mistake — running an ILE thing as if it were PASE, or expecting library list to behave like `$PATH`. Three pages here, and the rest of the guide makes sense.

---

## PASE and ILE

IBM i has two completely separate userland environments running on top of the same kernel.

**ILE** (Integrated Language Environment) is the native side — RPG, COBOL, CL, native SQL stored procedures, the library list, the `*PGM` and `*SRVPGM` objects, the world that lives in libraries. It's what you see from a 5250 command line. It's where your business logic lives.

**PASE** (Portable Application Solutions Environment) is an AIX-compatible runtime built into IBM i. AIX binaries run inside it, mostly unmodified. `bash`, `yum`, `php`, `python`, `node`, `git`, `openssl`, `ssh` — everything in this guide — all run in PASE.

PASE is a *userland*, not a separate machine. PASE programs share the same OS, the same kernel, the same DB2, the same job table. You can call from one to the other in both directions:

- **From ILE to PASE:** `QP2SHELL`, `QP2TERM`, `system` API, the various `Qp2*` runtime APIs.
- **From PASE to ILE:** the XMLSERVICE Toolkit, ILE program calls via `system()` in C/PHP, `qsh -c` for CL, JDBC/ODBC for SQL.

The single sentence to remember: **PASE is where open source runs, ILE is where the business runs, and the database is shared between them.**

{: .note }
> If you've used AIX before, PASE will feel exactly familiar. If you haven't, every `bash`-and-`yum` tutorial you find online is roughly correct — just translate every "AIX" to "PASE on IBM i." Most of the time it Just Works.

---

## The two filesystems of thinking

IBM i exposes a single integrated file system (IFS), but you should mentally divide it into three regions:

### The library file system: `/QSYS.LIB`

Every IBM i library appears as a `.LIB` directory under `/QSYS.LIB`. Files, programs, and other objects appear inside as `.FILE`, `.PGM`, etc. PASE programs can reach in and read DB2 tables this way, but it's clunky and rarely the right answer.

For day-to-day open-source work, treat `/QSYS.LIB` as **the place where ILE lives**. Don't put your PHP code there. Don't put your `git` repos there. Don't put `node_modules` there.

### The PASE region: `/QOpenSys`

Everything from `yum` lives under `/QOpenSys/pkgs`. Specifically:

```
/QOpenSys/pkgs/bin/        # binaries (php, python3, node, git, yum, ...)
/QOpenSys/pkgs/lib/        # shared libraries
/QOpenSys/pkgs/include/    # development headers
/QOpenSys/pkgs/share/      # arch-independent data
/QOpenSys/etc/             # config (php.ini, sshd_config, ...)
/QOpenSys/var/             # state, logs, mail spools, run files
```

`/QOpenSys` is **case-sensitive** and uses CCSID **1208 (UTF-8)**. This is fundamentally different from the rest of the IFS, and is the single most common source of CCSID problems. See [CCSID sanity](ccsid-sanity.html).

### The "regular IFS": `/home`, `/tmp`, `/www`, `/etc`, anything else

Everything not under `/QSYS.LIB` or `/QOpenSys` is the IFS proper. It's case-sensitive in newer releases, but its CCSID is whatever IBM picked when the directory was created — usually 37 (US English EBCDIC) or 819 (ISO 8859-1 Latin-1). PASE programs will read and write here happily, but the CCSID-tagging behavior is different from `/QOpenSys`. See [CCSID sanity](ccsid-sanity.html).

{: .tip }
> **Rule of thumb.** Open-source code (PHP apps, Python scripts, Node services, Git repos) lives somewhere under `/home/<user>/` or a project-root directory you choose. Keep it consistent across your shop. Avoid putting application code under `/QOpenSys` — that's reserved for IBM- and vendor-installed packages.

---

## Library list vs $PATH

In ILE, you find objects through the **library list**: `*LIBL`, `*USRLIBL`, `*SYSLIBL`. In PASE, you find binaries through `$PATH` and shared libraries through `LIBPATH` (the AIX equivalent of Linux's `LD_LIBRARY_PATH`).

These two systems are independent. A PASE program calling `system("CALL PGM(MYLIB/MYPGM)")` will look up `MYLIB` through the *job's* library list, not the `$PATH`. A 5250 user running `CALL QP2TERM` will land in PASE with `$PATH` set from the user's PASE environment, not anything in the library list.

The handoff that bites people: when you call PASE from RPG or CL via `Qp2RunPase` or `system`, the PASE process *inherits the same job's* environment, including `$PATH` if you've set it through `PASE_PATH` or via an environment file. If `$PATH` doesn't include `/QOpenSys/pkgs/bin`, your PASE binaries are invisible. See [Service users and authority](service-users-and-authority.html) for environment-file conventions.

---

## $HOME and per-user environment

Each IBM i user profile has a `HOMEDIR` attribute (`DSPUSRPRF`, then look at "Home directory"). For a brand-new user, this is unset, and PASE will fall back to `/`, which is the root of the IFS — almost certainly not what you want.

Set a real home directory for any user that will SSH in or run PASE work:

```cl
CHGUSRPRF USRPRF(JESSE) HOMEDIR('/home/jesse')
```

Then create the directory in PASE (it won't exist yet):

```bash
mkdir -p /home/jesse
chown jesse /home/jesse
chmod 700 /home/jesse
```

User-level PASE environment files live at `~/.profile` (read by `bash`/`ksh` on interactive login). Set per-user `$PATH`, `PASE_LANG`, prompt, and aliases there. The system-wide equivalents are `/QOpenSys/etc/profile` (and `/etc/profile`, which is also read on most setups).

{: .warning }
> Do **not** edit `/QOpenSys/etc/profile` to add huge environment changes that affect every PASE session including IBM-shipped jobs. Use per-user `~/.profile` for personal preferences, and reserve the system-wide profile for things every user genuinely needs (like adding `/QOpenSys/pkgs/bin` to `$PATH`).

---

## Jobs, subsystems, and PASE

Every PASE process runs inside an IBM i *job*, which lives inside a *subsystem*. The subsystem decides routing, default user, default library list, default `JOBCCSID`, and a lot more.

Most PASE work falls into one of four buckets:

1. **Interactive jobs** in `QINTER` — you logged in via 5250 and called `QP2TERM`.
2. **SSH jobs** in `QUSRWRK` (by default) — you connected over SSH and got a PASE shell.
3. **Web server jobs** in `QHTTPSVR` (or a custom subsystem you set up) — Apache, NGINX, php-fpm, node servers.
4. **Batch jobs** routed wherever your job description says.

Where a job runs determines what authority it has, what its `JOBCCSID` is, and which subsystem's monitors and logs catch it. **The most common production mistake is running PASE workloads in the wrong subsystem.** The [Service users and authority](service-users-and-authority.html) chapter describes the conventions K3S uses; you can adapt them or pick your own, but pick something.

---

## A diagnostic checklist

When something open-source isn't behaving on IBM i, work through this list in order:

1. **Which quadrant am I in?** ILE or PASE? Library list or `$PATH`?
2. **Which user is the job running as?** `WRKACTJOB` and look at "User", or `whoami` in PASE.
3. **What's the `JOBCCSID`?** `WRKJOB` -> option 2, or `system 'DSPJOB'` from PASE. See [CCSID sanity](ccsid-sanity.html).
4. **What's the `PASE_LANG`?** `echo $PASE_LANG`. Should be something like `EN_US.UTF-8`.
5. **Which subsystem is it running in?** `WRKACTJOB` again, or check `WRKSBSJOB`.
6. **Which `$PATH` does the PASE process see?** `echo $PATH`. Is `/QOpenSys/pkgs/bin` in it?
7. **What's the home directory?** `echo $HOME`. Is it set to a real path?

If steps 1–7 all check out, the problem is genuinely in the open-source software and you can debug it the way you would on Linux.

---

## Where next

Now that the layout makes sense, install `yum` itself. **[Bootstrapping OSPM](bootstrap.html)** covers the three flavors of bootstrap: ACS-supported (the easy path), no-SSH (the original SQL/FTP recipe), and the dark-network case.
