---
title: Package managers in 2026
nav_order: 4
---

# Package managers in 2026

If you've spent time on AIX recently, you've been told to use `dnf`. If you read modern Red Hat / Fedora documentation, you'll see `dnf` everywhere. This makes the IBM i answer surprising:

**IBM i is still on `yum`.**

Not "yum aliased to dnf" the way RHEL 8+ does it. Actual `yum`, the Python 2-era utility. As of early 2026, the IBM-shipped bootstrap installs `yum` 3.4.x, and the [IBM i OSS Docs](https://ibmi-oss-docs.readthedocs.io/en/latest/yum/README.html) and the [Seiden Group docs](https://www.seidengroup.com/php-documentation/updating-open-source-packages/) both reference `yum` exclusively.

This isn't a bug. AIX moved to `dnf` because they hit a hard requirement to support modern Python. IBM i hasn't followed yet. Don't run AIX `dnf` setup scripts on IBM i; they'll either fail or, worse, leave you with two confused package managers fighting over the same RPM database.

{: .note }
> **Future-proofing.** IBM has signaled at COMMON and PowerUp that `dnf` will eventually come to IBM i, but as of this writing there is no announced date. When it does, the migration path will mirror what AIX did: a transition script, both available side-by-side for a while, then `dnf` as the default. Until then: `yum`.

---

## What `yum` looks like on IBM i

After bootstrap, the binaries you care about are at:

```
/QOpenSys/pkgs/bin/yum
/QOpenSys/pkgs/bin/yum-config-manager   # (after `yum install yum-utils`)
/QOpenSys/pkgs/bin/repoquery
/QOpenSys/pkgs/bin/rpm
```

The configuration lives at:

```
/QOpenSys/etc/yum/yum.conf
/QOpenSys/etc/yum/repos.d/*.repo
```

The RPM database — the place `yum` records what it has installed — lives at `/QOpenSys/var/lib/rpm`. You almost never touch this directly. If you do find yourself reaching for `rpm --rebuilddb`, something has gone seriously wrong; consider opening a TSS case.

---

## The repo files

Each `*.repo` file in `/QOpenSys/etc/yum/repos.d/` defines a repository. After a fresh bootstrap, you'll see at least:

- `ibm.repo` — the IBM base repository at `public.dhe.ibm.com`. This is what supplies `bash`, `python3`, `nodejs`, `git`, `openssh`, `openssl`, `gcc`, and most of the rest of the open-source stack.

You'll likely want to add one or both of:

- `zend.repo` — Zend (Perforce) PHP RPMs at `repos.zend.com/ibmiphp`.
- `seiden.repo` — Seiden PHP+ at `repos.seidengroup.com`.

A canonical IBM repo file looks like:

```ini
[ibm]
name=IBM i Open Source RPMs
baseurl=http://public.dhe.ibm.com/software/ibmi/products/pase/rpms/repo
failovermethod=priority
enabled=1
gpgcheck=0
```

A Zend repo file:

```ini
[zend]
name=Zend PHP RPMs for IBM i
baseurl=http://repos.zend.com/ibmiphp
enabled=1
gpgcheck=0
```

A Seiden repo file (after registering at the Seiden Group install page to get the URL — the configuration steps will be on their site):

```ini
[seiden_stable]
name=Seiden PHP+ Stable
baseurl=https://...        # supplied by Seiden Group
enabled=1
gpgcheck=0
```

You can drop these in by hand, or let `yum-config-manager --add-repo <url>` create them for you.

{: .warning }
> **`gpgcheck=0` deserves a moment of attention.** The IBM and Zend repos historically don't sign their packages, so GPG checking is off in the canonical configs. That means the only thing protecting you from a bad RPM is HTTPS (which you should be using; flip the URLs from `http://` to `https://` if your IBM i can do TLS to those hosts). It's worth understanding the trust model here, especially if you're in a regulated industry.

---

## The 25 commands you'll actually use

In order of how often you'll reach for them:

```bash
yum search <term>                 # find a package by name or description
yum info <package>                # what is this, what version, who depends
yum install <package> [more...]   # install (asks for confirmation)
yum install -y <package>          # install without prompting
yum update                        # update everything that has updates
yum update <package> [more...]    # update only the named packages
yum update "php-*"                # wildcard updates (only updates installed)
yum remove <package>              # uninstall (asks for confirmation)
yum list installed                # everything you have installed
yum list installed | grep <term>  # quick "do I have X installed?"
yum repolist                      # which repos are enabled
yum repolist all                  # including disabled ones
yum clean metadata                # force re-download of repo metadata
yum clean all                     # nuke the entire cache
yum history                       # what have I done recently
yum history undo <id>             # undo a specific transaction (powerful, careful)
yum provides */<filename>         # which package provides this file
yum-config-manager --enable <id>  # turn a repo on
yum-config-manager --disable <id> # turn a repo off
yum-config-manager --add-repo <url>
repoquery --requires <package>    # what does this need
repoquery --whatrequires <package> # what depends on this
rpm -qa | grep <term>             # raw RPM-DB query
rpm -ql <package>                 # list every file the RPM owns
rpm -qf <path>                    # which package owns this file
```

The handful that surprise people:

- **`yum update "php-*"`** only updates *installed* `php-*` packages. It does not install new PHP packages. This is exactly the behavior you want when applying security updates.
- **`yum history undo`** is shockingly capable. If you `yum install`'d a tree of packages and now want to back it all out, `yum history` to find the transaction ID, `yum history undo N`, done. It's still possible to break things (especially if you've installed manually-built RPMs in between), but it's the closest thing IBM i open source has to a "ctrl-Z."
- **`yum provides */filename`** answers "what RPM do I install to get this binary or library." Very useful when a `composer install` complains about a missing `.so`.

---

## The repos worth knowing about

In rough order of how often K3S shops touch them:

| Repo                  | What's in it                            | Source                                                                                                                       |
| --------------------- | --------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------- |
| `ibm`                 | Core OSS: bash, git, python, node, gcc, openssh, openssl, sqlite, curl, etc. | [IBM OSS Docs](https://ibmi-oss-docs.readthedocs.io/en/latest/yum/README.html)                                              |
| `zend`                | Zend PHP RPMs (PHP 7.4 / 8.x family)   | [repos.zend.com/ibmiphp](http://repos.zend.com/ibmiphp)                                                                      |
| `seiden_stable`       | Seiden PHP+ — PHP RPMs with extras     | [seidengroup.com](https://www.seidengroup.com)                                                                              |
| Community third-party | Various                                 | [3rd Party Open Source Repos for IBM i](https://ibmi-oss-docs.readthedocs.io/en/latest/yum/3RD_PARTY_REPOS.html)            |

Read the third-party list before you add anything from it. Some of those repos are well-maintained; some are one person's hobby; almost none are signed. Pick what you need, and pin to specific repos rather than flipping `enabled=1` on everything you see.

---

## When `yum` breaks

Two failure modes happen often enough to be worth naming:

### "Unable to retrieve repo metadata"

This is almost always a network issue. From PASE, can the partition reach the repo host?

```bash
curl -I http://public.dhe.ibm.com/software/ibmi/products/pase/rpms/repo/repodata/repomd.xml
```

If that fails, the problem is firewall, proxy, DNS, or routing. `yum` itself is fine. If your partition needs an HTTP proxy to reach the public internet, set `proxy=http://...` in `/QOpenSys/etc/yum/yum.conf` (and also `http_proxy`/`https_proxy` in your environment for `curl`, `git`, etc. — `yum` reads its proxy from `yum.conf`, not from env).

### "Loaded repos list is empty"

`yum-config-manager` lost track of the repos, usually after an upgrade or after someone deleted `/QOpenSys/etc/yum/repos.d/ibm.repo`. Re-add it:

```bash
yum install yum-utils       # if not already
yum-config-manager --add-repo http://public.dhe.ibm.com/software/ibmi/products/pase/rpms/repo
yum clean all
yum repolist
```

If `yum-config-manager` doesn't exist either, `yum-utils` got blown away too. Drop the `ibm.repo` file in by hand using the example above.

---

## A practical day-one install list

For a fresh IBM i partition where you've just bootstrapped `yum`, the K3S baseline is roughly:

```bash
yum install -y \
  yum-utils \
  openssh \
  git \
  bash \
  ca-certificates-mozilla \
  curl \
  wget \
  vim-enhanced \
  rsync \
  unzip \
  tmux \
  ksh
```

Then the language-specific ones, depending on what you'll actually run:

```bash
# Python
yum install -y python3 python3-pip python3-itoolkit python3-ibm_db

# Node
yum install -y nodejs

# PHP — pick one path; see the PHP chapter
# yum install -y php php-cli php-pdo php-pdo-odbc ...   (Zend)
# yum install -y seiden-php php-cli ...                 (Seiden PHP+)
```

Each of these is covered in its own chapter.

---

## Where next

You have `yum` and a baseline of utilities. **[SSH and the VS Code workflow](ssh-and-vscode.html)** is next — turning the partition into a place you'd actually want to develop on, rather than something you SSH into to run commands.
