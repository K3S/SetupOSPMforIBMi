---
title: Reference
nav_order: 14
---

# Reference

Links, repos, and acknowledgments worth coming back to.

---

## IBM-published documentation

- **[IBM i OSS Docs](https://ibmi-oss-docs.readthedocs.io/)** — the canonical reference for `yum`, RPMs, the bootstrap, third-party repos, and IBM-shipped Python / Node / PHP details.
- **[Getting Started with Open Source Package Management](https://www.ibm.com/support/pages/getting-started-open-source-package-management-ibm-i-acs)** — the IBM ACS-side instructions.
- **[IBM i Open Source Software Support](https://www.ibm.com/support/pages/ibm-i-open-source-software-support)** — IBM's TSS support offering for OSS on IBM i.
- **[Power Toolbox / 5733-OPS Lifecycle](https://ibmi-oss-docs.readthedocs.io/en/latest/yum/README.html)** — the RPM-vs-5733-OPS transition and what's where.

---

## Repository configurations

| Repo            | URL                                                              | What's in it                                       |
| --------------- | ---------------------------------------------------------------- | -------------------------------------------------- |
| `ibm`           | `http://public.dhe.ibm.com/software/ibmi/products/pase/rpms/repo` | Core IBM-shipped OSS                              |
| `zend`          | `http://repos.zend.com/ibmiphp/`                                 | Zend (Perforce) PHP RPMs                          |
| `seiden_stable` | (per Seiden Group registration at [seidengroup.com](https://www.seidengroup.com)) | Seiden PHP+                       |
| Third-party     | [3rd Party Open Source Repos for IBM i](https://ibmi-oss-docs.readthedocs.io/en/latest/yum/3RD_PARTY_REPOS.html) | Community-maintained extras                       |

The IBM repo is configured automatically by the bootstrap. The others you add via `yum-config-manager --add-repo` or by dropping a `.repo` file in `/QOpenSys/etc/yum/repos.d/`.

---

## Vendor and community pages

- **[Seiden Group](https://www.seidengroup.com)** — PHP+, IBM i open-source services, support.
- **[IBM i OSS Slack](http://ibm.biz/ibmioss-chat-join)** — the active community chat. Free to join.
- **[Code for IBM i](https://marketplace.visualstudio.com/items?itemName=IBM.code-for-ibmi)** — the VS Code extension. The right home for daily IBM i development in 2026.
- **[COMMON](https://www.common.org/)** — the IBM i / Power user group. PowerUp and NEUGC conferences.

---

## Companion guides published by K3S

- **[PHP / PDO / ODBC Toolkit Setup](https://odbcphp.k3s.com)** — the deep dive on running PHP on IBM i, connecting PHP to DB2 via ODBC, and calling RPG from PHP.
- **[RPG Tutorial](https://rpgtutorial.k3s.com)** — practical RPG programming through VS Code.
- **[IBM i AI Workers](https://ibmi-ai-workers.k3s.com)** — calling LLMs at scale from RPG.

---

## Troubleshooting quick-reference

### "yum: command not found"

Either `yum` was never installed (see [Bootstrapping OSPM](bootstrap.html)), or `/QOpenSys/pkgs/bin` isn't in your `$PATH`. Use the full path:

```bash
/QOpenSys/pkgs/bin/yum --version
```

If that works, fix `$PATH`:

```bash
echo 'export PATH=/QOpenSys/pkgs/bin:$PATH' >> ~/.profile
```

### "Unable to retrieve repo metadata"

Network problem reaching the repo URL. From PASE:

```bash
curl -I http://public.dhe.ibm.com/software/ibmi/products/pase/rpms/repo/repodata/repomd.xml
```

If that fails, your partition can't reach the public IBM file server — check firewall, proxy, DNS. If it succeeds and `yum` still fails, run `yum clean metadata; yum repolist`.

### Garbled characters in PHP / Python / Node output

CCSID. See [CCSID sanity](ccsid-sanity.html). Run the diagnostic checklist there.

### "PHP Warning: Module 'ibm_db2' already loaded"

Two `extension=ibm_db2.so` lines, usually one in `php.ini` and one in `conf.d/`. Remove the one in `php.ini`; let `conf.d/` own it.

### `composer install` fails on a native extension build

Confirm `gcc`, `make`, and the relevant `-devel` packages are installed:

```bash
yum install -y gcc gcc-c++ make autoconf automake libtool
```

### `npm install` fails downloading a pre-built binary

The package is trying to fetch a PowerPC binary that doesn't exist. Either find a pure-JS alternative (often there is one), pin to an older version that builds from source cleanly, or remove the package as a dependency.

### SSH connection works, but "Permission denied (publickey)"

Permissions on `~/.ssh` and `~/.ssh/authorized_keys` are wrong. From PASE:

```bash
chmod 700 ~/.ssh
chmod 600 ~/.ssh/authorized_keys
ls -la ~/.ssh
```

The directory must be `drwx------`, the file `-rw-------`, and both owned by the SSH user.

### `WRKACTJOB` shows the web server running as `QSECOFR`

The web-server instance was started as `QSECOFR` and never reconfigured. Set the dedicated user (`K3SWEB`) in the Apache `httpd.conf` (`User K3SWEB`), in NGINX `nginx.conf` (`user k3sweb;`), or php-fpm pool config. Restart the daemon and verify in `WRKACTJOB`. See [Service users and authority](service-users-and-authority.html).

### A PTF removed something `yum` provided

Rare, but it happens — a Group PTF can replace a file that `yum` thought it owned. Symptom: `yum verify` shows the file as modified or missing. Resolution: `yum reinstall <package>` after the PTF settles, or pin the package version if you've intentionally chosen a specific build.

---

## Acknowledgments

This guide stands on the work of many in the IBM i community:

- **Jesse Gorzinski** and the IBM Rochester open-source team, who have made all of this possible by carrying open-source delivery to IBM i for the last decade.
- **Alan Seiden** and the **Seiden Group** team, whose blog posts, talks, and direct support have answered more PHP-on-IBM-i questions over the years than anyone else.
- **Liam Allan** and the **Code for IBM i** team, who turned VS Code into a first-class IBM i development environment.
- The **IBM i OSS Slack community**, where so many of the hard problems get solved between people who've already solved them.
- The K3S customers and engineers who've reported, debugged, and fixed the issues that this guide quietly captures.

Errors in this guide are my own. Corrections, suggestions, and pull requests are welcome via the **Edit this page on GitHub** link at the bottom of any chapter.

---

*Maintained by [King III Solutions, Inc.](https://k3s.com)*
