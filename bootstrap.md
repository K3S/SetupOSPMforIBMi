---
title: Bootstrapping OSPM
nav_order: 3
---

# Bootstrapping OSPM

`yum` is the package manager that delivers every open-source RPM on IBM i. To use `yum` you need `yum` installed. If `yum` isn't installed, you're in the bootstrap problem.

There are three ways out, depending on what your partition can talk to:

1. **ACS Open Source Package Management** (the easy, IBM-supported path) — when you have SSH and your partition can reach the public IBM file server.
2. **The SQL / FTP recipe** — when SSH is not yet running on the partition. This is the original recipe this repository was created for; it still works.
3. **Offline / dark network** — when the partition cannot reach the public Internet at all. You build a local repo mirror on a system that can, then point the partition at it.

Pick whichever matches your situation.

---

## 1. The ACS path (easy, when it works)

This is what IBM documents at [Getting Started with Open Source Package Management](https://www.ibm.com/support/pages/getting-started-open-source-package-management-ibm-i-acs). Short version:

1. Open **ACS** on a workstation that can reach both the IBM i and `public.dhe.ibm.com`.
2. Open **Tools → Open Source Package Management**.
3. ACS prompts for an SSH connection to the IBM i. You'll need:
   - SSHD running on the IBM i (`STRTCPSVR *SSHD`).
   - A user profile with `*ALLOBJ` or at least sufficient authority to install into `/QOpenSys/pkgs`.
   - A working network path from the IBM i to `public.dhe.ibm.com:443`.
4. If `yum` is not yet installed, ACS will detect that and offer to bootstrap it. Click **Yes**.

When this works, you're done in a couple of minutes. ACS handles the entire bootstrap behind the scenes.

{: .note }
> The pre-condition that fails most often is **SSH is not running**, or the SSH user doesn't have a valid `HOMEDIR` set. See [SSH and the VS Code workflow](ssh-and-vscode.html) for the SSH setup itself; come back here once `STRTCPSVR *SSHD` runs cleanly.

---

## 2. The SQL / FTP recipe (when SSH is not available)

If you don't have SSH yet — common on partitions that have never been used for open source — you can bootstrap `yum` by FTPing the bootstrap files from `public.dhe.ibm.com` and executing the IBM-shipped `bootstrap.sh` from QSH.

This is the script this repo was originally built around. It still works. Paste it into **ACS → Run SQL Scripts** and run it:

```sql
create or replace table qtemp.ftpcmd(cmd char(240)) on replace delete rows;
create or replace table qtemp.ftplog(line char(240)) on replace delete rows;

insert into qtemp.ftpcmd(CMD) values
   ('anonymous anonymous@example.com')
  ,('namefmt 1')
  ,('lcd /tmp')
  ,('cd /software/ibmi/products/pase/rpms')
  ,('bin')
  ,('get README.md (replace')
  ,('get bootstrap.tar.Z (replace')
  ,('get bootstrap.sh (replace')
 with nc;

CL:OVRDBF FILE(INPUT)  TOFILE(QTEMP/FTPCMD) MBR(*FIRST) OVRSCOPE(*JOB);
CL:OVRDBF FILE(OUTPUT) TOFILE(QTEMP/FTPLOG) MBR(*FIRST) OVRSCOPE(*JOB);

CL:FTP RMTSYS('public.dhe.ibm.com');

CL:QSH CMD('touch -C 819 /tmp/bootstrap.log; /QOpenSys/usr/bin/ksh /tmp/bootstrap.sh > /tmp/bootstrap.log 2>&1');

select
  case when (message_tokens = X'00000000')
    then 'Bootstrapping successful! Review /tmp/README.md for more info'
    else 'Bootstrapping failed.  Consult /tmp/bootstrap.log for more info'
  end as result
from table(qsys2.joblog_info('*')) x
where message_id = 'QSH0005'
order by message_timestamp desc
fetch first 1 rows only;
```

When the script finishes, `yum` will be installed at `/QOpenSys/pkgs/bin/yum`, and `/QOpenSys/pkgs/bin` will be a usable directory full of binaries.

{: .warning }
> Read `/tmp/bootstrap.log` *before* you assume the bootstrap worked, especially the last 30 lines. The SQL above prints "successful" based on the QSH message ID, but a non-fatal error during the install (e.g., one RPM with a missing dependency) won't prevent that message from appearing.

After the bootstrap, the next thing you almost certainly want to do is install SSH itself so you don't have to do this again:

```bash
/QOpenSys/pkgs/bin/yum install openssh
```

Then start the SSH server (`STRTCPSVR *SSHD` from CL) and proceed with [SSH and the VS Code workflow](ssh-and-vscode.html).

### What the recipe actually does

For people who like to know what they're running:

1. Builds two scratch tables (`QTEMP/FTPCMD`, `QTEMP/FTPLOG`).
2. Stuffs the `FTPCMD` table with the FTP commands to log in anonymously and fetch three files.
3. Overrides `INPUT` and `OUTPUT` so the IBM i `FTP` command reads from `FTPCMD` and writes to `FTPLOG`.
4. Runs `FTP` against `public.dhe.ibm.com`, which downloads the files into `/tmp`.
5. Runs `bootstrap.sh` under QSH, redirecting all output to `/tmp/bootstrap.log`.
6. Reads back the joblog to display whether QSH exited cleanly.

Nothing magical. It's a creative way to drive `FTP` and `QSH` from a single SQL script, which is how you can bootstrap a system you can only reach via ACS.

---

## 3. The offline / dark-network case

Some partitions can't reach `public.dhe.ibm.com` at all. Maybe they're in an isolated VLAN, maybe outbound HTTPS is locked down, maybe the security review hasn't approved external internet access yet.

The IBM-documented solution is to build a local mirror on a system that *can* reach the public file server, then serve the RPMs from your own internal HTTP server. The full procedure is documented in the [IBM i OSS Docs](https://ibmi-oss-docs.readthedocs.io/en/latest/yum/README.html#offline-or-dark-network) under "Offline or dark network." The shape of it:

1. On a system with internet access (could be a Linux box, another IBM i, or a workstation with `reposync` available), run:

   ```bash
   yum install yum-utils createrepo
   reposync -p /path/to/repo/root -r ibm
   createrepo /path/to/repo/root/ibm
   ```

2. Serve `/path/to/repo/root` over HTTP from somewhere your locked-down IBM i *can* reach (an internal Apache, NGINX, or even Python's `http.server` for a one-shot).

3. On the locked-down IBM i, edit `/QOpenSys/etc/yum/repos.d/ibm.repo` to point `baseurl=` at your internal mirror.

4. Bootstrap `yum` itself the same way, but using the local mirror — IBM ships an offline bootstrap variant that takes a tarball you copy in via FTP, NetServer, or whatever transport you do have.

This is the path most regulated shops eventually want to be on regardless, because it gives you control over what version of what package shows up when somebody runs `yum update`. See [Package managers in 2026](package-managers.html) for the repo configuration.

{: .tip }
> If you're going to run a local mirror, keep the `ibm`, `seiden_stable`, and (if you use it) `zend` repos in sync at the same cadence. Mismatched cadences cause subtle "this RPM updated but the one it depends on didn't" issues.

---

## Verifying the bootstrap

Regardless of which path you took, run this from PASE (SSH, Qp2term, or `STRQSH`) to confirm:

```bash
/QOpenSys/pkgs/bin/yum --version
/QOpenSys/pkgs/bin/yum repolist
```

You should see a version number (around 3.4.x — IBM i is still on yum, see [Package managers in 2026](package-managers.html)) and at least one repo listed. If `repolist` shows no enabled repos, your `/QOpenSys/etc/yum/repos.d/` is empty or all repos are `enabled=0`. Re-add the IBM base repo:

```bash
yum-config-manager --add-repo http://public.dhe.ibm.com/software/ibmi/products/pase/rpms/repo
```

(That requires `yum-utils`, which the bootstrap usually installs. If it doesn't, drop the `.repo` file in by hand — see the package-managers chapter.)

---

## Adding `/QOpenSys/pkgs/bin` to your PATH

Once `yum` is installed, you'll want every PASE shell — interactive, SSH, batch — to find the binaries without typing the full path every time. Add a system-wide profile fragment:

```bash
echo 'export PATH=/QOpenSys/pkgs/bin:$PATH' >> /QOpenSys/etc/profile
```

For batch jobs that don't read `/QOpenSys/etc/profile` (notably some web-server-spawned jobs), set `PASE_PATH` directly on the calling job. There's more on this in [Service users and authority](service-users-and-authority.html).

---

## Where next

`yum` works. **[Package managers in 2026](package-managers.html)** explains the `yum` vs `dnf` situation (IBM i is still on `yum`, AIX has moved to `dnf`, this matters when you read documentation), the repo files you'll be editing, and the most-used commands.
