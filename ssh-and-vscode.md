---
title: SSH and the VS Code workflow
nav_order: 5
---

# SSH and the VS Code workflow

Almost everything in this guide assumes you can SSH into the partition. SSH on IBM i is solid; it's been there for years; and once it's set up, the VS Code workflow it enables is by far the best development experience for IBM i in 2026 — better than Qp2term, better than RDi for most tasks (RPG-debugging excepted), better than 5250 for anything that touches the IFS.

This chapter gets you from "SSH might be running" to "I can edit, browse, and run things from VS Code over SSH against the partition."

---

## 1. Install and start the SSH server

`openssh` ships in the IBM base repo:

```bash
yum install openssh
```

Then start the SSHD server from CL or QSH:

```cl
STRTCPSVR *SSHD
```

To make SSHD start automatically when TCP/IP starts (the normal case after IPL), set the autostart attribute:

```cl
CHGTCPSVR SVRSPCVAL(*SSHD) AUTOSTART(*YES)
```

Verify:

```cl
NETSTAT *CNN
```

Look for a `*Listen` entry on local port 22. If it's there, the daemon is up. From your workstation:

```bash
ssh jesse@your-ibmi.example.com
```

If you land at a `bash-5.x$` prompt, you're done with the basics.

{: .note }
> The SSH server reads `/QOpenSys/etc/ssh/sshd_config`. Default settings are reasonable. The two changes worth making early: `PasswordAuthentication no` (after you've set up keys), and `PermitRootLogin no` (which on IBM i means: refuse SSH for `QSECOFR`, the closest analogue to `root`).

---

## 2. Set up SSH keys

Password authentication works but you don't want to type your IBM i password into VS Code thirty times an hour. Generate a key on your workstation:

```bash
ssh-keygen -t ed25519 -C "jesse@workstation"
```

Then copy the public key to the IBM i:

```bash
ssh-copy-id jesse@your-ibmi.example.com
```

If `ssh-copy-id` isn't available on your workstation (Windows without WSL, for example), copy it manually:

```bash
# On the IBM i, in PASE:
mkdir -p ~/.ssh
chmod 700 ~/.ssh
echo "ssh-ed25519 AAAA... jesse@workstation" >> ~/.ssh/authorized_keys
chmod 600 ~/.ssh/authorized_keys
```

The `chmod` matters. SSHD enforces strict permissions: if `~/.ssh` is group- or world-writable, or if `authorized_keys` is, SSHD silently refuses to use the file and falls back to password auth.

{: .warning }
> A common failure mode: a user's `HOMEDIR` is set to `/home/jesse`, but `/home/jesse` doesn't exist or has wrong ownership. Confirm with `DSPUSRPRF` from CL, then in PASE:
> ```bash
> ls -lad /home/jesse /home/jesse/.ssh /home/jesse/.ssh/authorized_keys
> ```
> The directory and the file should be owned by `jesse` (the IBM i user), not `qsecofr`, not `qhttpsvr`, not `root`.

---

## 3. Set $HOME and a real shell

Every user that will SSH in needs:

- A valid `HOMEDIR` set on the user profile.
- A real home directory in the IFS, owned by them.
- A shell that exists. PASE bash lives at `/QOpenSys/pkgs/bin/bash`.

From CL:

```cl
CHGUSRPRF USRPRF(JESSE) HOMEDIR('/home/jesse')
```

From PASE (as a user with authority — `qsecofr` or somebody with `*ALLOBJ`):

```bash
mkdir -p /home/jesse
chown jesse /home/jesse
chmod 700 /home/jesse
```

To set bash as the default login shell, drop a `.profile` into the home directory that exec's bash:

```bash
# /home/jesse/.profile
if [ -x /QOpenSys/pkgs/bin/bash ] && [ -z "$BASH_VERSION" ]; then
  export PATH=/QOpenSys/pkgs/bin:$PATH
  exec bash -l
fi
```

The `exec bash -l` re-launches bash as a login shell, which then reads `/etc/profile`, `~/.bash_profile`, and `~/.bashrc` the way users expect.

{: .tip }
> If you'd rather use `ksh` (the IBM i default and what `chsh` would point you at), that's fine — it's a perfectly capable shell. But every modern tutorial assumes bash, and many `composer` and `npm` post-install scripts have bashisms that ksh doesn't handle. Save yourself the friction; use bash.

---

## 4. Terminal CCSID

When you SSH in, your terminal session has a CCSID. Get this wrong and PASE will mangle every non-ASCII character you type or read.

The combination that works for almost everyone in 2026:

- **PASE side**: `PASE_LANG=EN_US.UTF-8`, set in `/QOpenSys/etc/profile` or your `~/.profile`.
- **Job side**: `JOBCCSID=1208` (UTF-8) for the SSH job, set via the user profile or job description.
- **Your terminal emulator**: Set to UTF-8 on the workstation side (the default for almost every modern terminal).

Putting that into a per-user profile fragment:

```bash
# ~/.profile or ~/.bash_profile
export PASE_LANG=EN_US.UTF-8
export LC_ALL=EN_US.UTF-8
export LANG=EN_US.UTF-8
export PATH=/QOpenSys/pkgs/bin:$PATH
```

The `JOBCCSID` part needs CL. Set it on the user profile so every SSH job from this user starts with it:

```cl
CHGUSRPRF USRPRF(JESSE) CCSID(1208)
```

See [CCSID sanity](ccsid-sanity.html) for the full picture of why these three settings have to agree.

---

## 5. Code for IBM i (VS Code extension)

The Halcyon/IBM-blessed VS Code extension is **Code for IBM i**, available in the VS Code marketplace as `IBM.code-for-ibmi`. Install it. Then:

1. **Add a connection** — sidebar icon, "Connect to a remote IBM i".
2. **Authenticate** — username, and either a password or (recommended) an SSH key file.
3. **Connect.** The extension probes the partition, finds `yum`, and offers to install any helpers it needs.

Once connected, you get:

- A **PASE filesystem browser** rooted wherever you point it (typically `/home/<user>` and `/QOpenSys`).
- A **library / object browser** for IFS-resident objects and library list contents.
- An **integrated terminal** that opens a PASE shell inside VS Code.
- **Source member editing** for QSYS source physical files (the older RPG/CL style) — VS Code edits them like normal files; the extension handles the conversion.
- **Compile commands** wired to right-click context menus.

For this guide's purposes, the two features that matter most are the PASE filesystem browser and the integrated terminal. Once you have those, VS Code becomes the natural editor for everything PHP, Python, Node, and shell-script-shaped on the partition.

{: .note }
> **Code for IBM i works over SSH.** It uses the same SSH connection you set up above, plus an internal SQL connection over the same SSH tunnel for some of the database-aware features. It doesn't need anything extra on the IBM i beyond `openssh` and (for some features) `python3-itoolkit`.

---

## 6. The actual day-to-day workflow

Once it's all set up, the loop looks like this:

1. **Open VS Code on your workstation.** The Code for IBM i sidebar shows your saved connections.
2. **Click the connection.** It opens a remote workspace; the VS Code window's color/title indicates remote mode (just like SSH-remote VS Code).
3. **Navigate to a project folder** under `/home/jesse/projects/something/`. Open files, edit, save. Saves write directly to the IBM i over SSH.
4. **Open a terminal** (Ctrl-` in VS Code). You're at a PASE shell on the IBM i. Run `composer install`, `npm test`, `python -m pytest`, `git status` — whatever the project needs.
5. **For RPG**, use the source member browser to open library-list-resident source files. The extension handles compile through right-click → "Compile".

That's the loop. It's the same loop you'd have on Linux or macOS with VS Code's Remote-SSH extension, just specialized for IBM i.

---

## 7. The things that bite people on day one

**`/dev/null` permissions.** On rare older systems, `/QOpenSys/dev/null` ends up with weird authority and SSHD-spawned shells fail in confusing ways. If your bash session immediately dies after login, `ls -la /dev/null` and confirm it's `crw-rw-rw-`. Fix with `chmod 666 /dev/null` and restart SSHD.

**`HOMEDIR` not actually a directory.** `DSPUSRPRF` says `/home/jesse`, but `/home/jesse` doesn't exist. SSHD logs you in, drops you somewhere, and `cd ~` goes nowhere reasonable. Always `mkdir -p` and `chown` after `CHGUSRPRF`.

**Profile inherited from `QSECOFR`.** If you copied a user from `QSECOFR`, the new user inherits `JOBCCSID(*USRPRF)` chained back to `QSYSVAL` 65535. Set `CCSID` on the new user explicitly — `CHGUSRPRF USRPRF(JESSE) CCSID(1208)`.

**SSHD running, but iptables/firewall in front.** If `STRTCPSVR *SSHD` reports success but `ssh` from your workstation hangs or refuses, the partition is up but a firewall in front of it isn't letting port 22 through. This is a network-team conversation; `NETSTAT *CNN` on the IBM i confirms the daemon is listening.

**Two SSH servers.** Some shops install both IBM's OpenSSH and a third-party SSH and end up with one trying to bind to port 22 while the other is already running. `WRKACTJOB SBS(QUSRWRK)` — look for one and only one `SSHD` job.

---

## Where next

SSH works, VS Code is connected, and your terminal speaks UTF-8. The next chapter, **[CCSID sanity](ccsid-sanity.html)**, is the one that prevents 80% of "this character looks wrong" bug reports. It's worth reading even if everything seems fine.
