---
title: CCSID sanity
nav_order: 6
---

# CCSID sanity

CCSID problems are the most-reported, most-misdiagnosed, and most-googled class of IBM i open-source bug. The good news is that there are only about six concepts to learn, and once you've internalized them, you can debug any CCSID issue in under five minutes.

This chapter gives you the concepts, in order, and ends with a copy-pasteable diagnostic checklist.

---

## What a CCSID actually is

A **CCSID** (Coded Character Set Identifier) is IBM's name for what the rest of the world calls a "character encoding." Every byte on an IBM i has, in principle, a CCSID associated with it that tells the OS how to interpret it as text.

The four you'll encounter every day:

| CCSID | What it is                              | Where it shows up                                           |
| ----- | --------------------------------------- | ----------------------------------------------------------- |
| 37    | US English EBCDIC                       | Default for ILE / native source members; default `JOBCCSID` for many users |
| 819   | ISO 8859-1 (Latin-1)                    | Common for IFS files; what `bootstrap.sh` writes its log in |
| 1208  | UTF-8                                   | What `/QOpenSys/pkgs/*` uses; what you want for source code, configs, anything modern |
| 65535 | "Binary / no conversion / leave alone"  | The CCSID that means "I refuse to declare a CCSID"          |

The fifth, less common, that matters in some shops:

| 1252  | Windows-1252 (Latin-1 superset)         | Files that came in via Windows tools                         |

That's most of what you need. Yes, there are hundreds of CCSIDs; in 25 years of running open source on IBM i you'll see maybe a dozen.

---

## CCSID 65535: the one that ruins your day

Files tagged with CCSID **65535** mean "treat as binary; don't translate." This is fine for ZIP files, JPEGs, executables. It's a disaster for text files.

When PASE writes a text file into the IFS *outside* `/QOpenSys`, it sometimes ends up tagged 65535. When you then try to read that file from ILE — or from a PASE program with `JOBCCSID(65535)` — IBM i refuses to translate, and you get bytes pretending to be characters.

Check the CCSID of any IFS file:

```bash
attr /home/jesse/somefile.txt CCSID
```

If it says 65535 and the file is actually text, fix it:

```bash
setccsid 1208 /home/jesse/somefile.txt
```

Or for a whole tree:

```bash
find /home/jesse/projects -type f -name "*.php" -exec setccsid 1208 {} \;
```

`setccsid` only re-tags the file. It does **not** convert the bytes. If the file was actually written as Latin-1 but tagged 65535, `setccsid 1208` will tag it as UTF-8 but the bytes are still Latin-1, and you'll see the same garbage. To actually convert, use `iconv`:

```bash
iconv -f ISO-8859-1 -t UTF-8 < old.txt > new.txt
setccsid 1208 new.txt
```

---

## `/QOpenSys` is the easy region

`/QOpenSys` is special: it's a UNIX-style filesystem that always uses CCSID **1208 (UTF-8)**. Files written there are tagged 1208 automatically. PASE programs that operate inside `/QOpenSys` see and write UTF-8 by default.

This is why most of your `yum`-installed software Just Works without CCSID intervention: it lives in `/QOpenSys`, it reads UTF-8, it writes UTF-8, IBM i agrees with it.

The CCSID complications start the moment your PHP / Python / Node code reads or writes a file *outside* `/QOpenSys` — in `/home`, `/tmp`, `/www`, `/etc`. Those are normal IFS, where the CCSID of newly-created files depends on the parent directory's CCSID, the job's `JOBCCSID`, and the program's environment.

The K3S convention: keep application source code under `/home/<user>/` or a project-root directory, and explicitly set those directories to CCSID 1208 at creation:

```bash
mkdir /home/jesse/projects
setccsid 1208 /home/jesse/projects
```

Files created in there by PASE programs running with `JOBCCSID(1208)` will inherit 1208.

---

## `JOBCCSID`: the job's notion of text

Every IBM i job has a `JOBCCSID`. It's set from (in order):

1. The job description's `CCSID` parameter, if not `*USRPRF`.
2. The user profile's `CCSID` parameter, if not `*SYSVAL`.
3. The system value `QCCSID`.

Most older systems have `QCCSID = 65535`. Most user profiles have `CCSID(*SYSVAL)`. So the inherited `JOBCCSID` is often **65535**, which causes the problems described above.

You don't have to change the system value (and probably shouldn't without thinking carefully). You *can* change individual user profiles, which is the right granularity for open-source work:

```cl
CHGUSRPRF USRPRF(JESSE) CCSID(1208)
```

For SSH-only service accounts (web server runs as, batch jobs run as), do the same:

```cl
CHGUSRPRF USRPRF(K3SWEB) CCSID(1208)
CHGUSRPRF USRPRF(K3SAPP) CCSID(1208)
```

Verify on a running job:

```cl
WRKJOB JOB(0/JESSE/QPADEV0001) OPTION(*DFNATR)
```

Look for `Coded character set identifier`. (From PASE: `system 'DSPJOB OPTION(*DFNATR)'`.)

---

## `PASE_LANG`: PASE's notion of text

PASE has its own concept of locale, separate from `JOBCCSID`. The environment variable is `PASE_LANG`, and the value you almost always want is:

```bash
export PASE_LANG=EN_US.UTF-8
```

Set this at three levels:

- System-wide for all PASE jobs: append to `/QOpenSys/etc/profile`.
- Per-user: in `~/.profile` or `~/.bash_profile`.
- Per-script: at the top of any shell script that runs in a context where the others might not apply (e.g., cron jobs, SBMJOB-launched shells).

Plus the GNU userland variables that most modern utilities also check:

```bash
export LANG=EN_US.UTF-8
export LC_ALL=EN_US.UTF-8
```

---

## The complete environment fragment

A canonical `~/.profile` for an IBM i open-source user:

```bash
# Path
export PATH=/QOpenSys/pkgs/bin:$PATH

# Locale / CCSID for PASE
export PASE_LANG=EN_US.UTF-8
export LANG=EN_US.UTF-8
export LC_ALL=EN_US.UTF-8

# Editor preferences
export EDITOR=vim
export VISUAL=vim

# Common aliases
alias ll='ls -la'
alias l.='ls -d .*'

# Re-exec into bash if we landed in something else
if [ -x /QOpenSys/pkgs/bin/bash ] && [ -z "$BASH_VERSION" ]; then
  exec bash -l
fi
```

Pair that with `CHGUSRPRF USRPRF(JESSE) CCSID(1208)` from CL, and the user is set up.

---

## The VS Code complication

VS Code over SSH runs whatever `bash` you've configured. If your SSH-side environment is correct (UTF-8 everywhere), VS Code's terminal will be too, and the editor will save files as UTF-8 by default. The Code for IBM i extension respects your locale settings.

Two things to double-check:

1. In VS Code settings: `"files.encoding": "utf8"` (the default).
2. The PASE-side `JOBCCSID` of the SSH job. From a VS Code terminal:

   ```bash
   system "DSPJOB OPTION(*DFNATR)" | grep -i ccsid
   ```

   You want **1208**.

---

## Diagnostic checklist

When something looks wrong with characters — bad accents, garbage in logs, "why does my PHP write UTF-8 to the database but read it back as Latin-1" — work this list in order:

1. **What CCSID is the source file?**
   ```bash
   attr myfile.php CCSID
   ```
2. **What's the job's `JOBCCSID`?**
   ```bash
   system "DSPJOB OPTION(*DFNATR)" | grep -i ccsid
   ```
3. **What's `PASE_LANG`?**
   ```bash
   echo $PASE_LANG
   ```
4. **What's `LANG` and `LC_ALL`?**
   ```bash
   echo $LANG $LC_ALL
   ```
5. **For files: was the file written with the *current* settings, or by something earlier with different settings?**
   - `find` with `-newer` and check creation timestamps.
6. **For DB2: what's the column's CCSID and what's the connection's CCSID negotiation?**
   - For columns: `SELECT CCSID FROM SYSCOLUMNS WHERE TABLE_NAME = 'MYTABLE' AND COLUMN_NAME = 'MYCOL'`.
   - For ODBC connections: see the ODBC chapter and the [PHP/PDO/ODBC toolkit guide](https://odbcphp.k3s.com).

In K3S experience, 90% of CCSID complaints resolve to one of:

- A file tagged 65535 that should be 1208.
- A user profile with `CCSID(*SYSVAL)` chained back to `QCCSID = 65535`.
- An ODBC connection that doesn't have `UNICODESQL=1` set.
- A web-server-spawned PASE job that doesn't read `/QOpenSys/etc/profile` and so has `PASE_LANG` unset.

---

## Where next

CCSID is handled. **[Service users and authority](service-users-and-authority.html)** is the next foundation chapter — it covers running daemons as something other than `QSECOFR`, IFS authority, and the conventions that keep a multi-user IBM i partition manageable.
