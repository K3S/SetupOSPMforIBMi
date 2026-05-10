---
title: Home
nav_order: 1
---

# IBM i Open Source Setup

A practical guide to setting up the open-source environment on IBM i — `yum` and the RPM packages, SSH and the editor workflow, and the conventions that separate an environment you trust from one you tolerate.

Read it the way you'd read a runbook: skip to whatever you're stuck on, ignore the rest.


[Get started](#start-here){: .btn .btn-primary .fs-5 .mb-4 .mb-md-0 .mr-2 }
[View on GitHub](https://github.com/K3S/SetupOSPMforIBMi){: .btn .fs-5 .mb-4 .mb-md-0 }


---

## Who this is for

You administer or develop on an IBM i partition, and you want to install and use modern open-source software on it. Maybe you've already got `yum` working but you're hitting CCSID weirdness; maybe you've got a partition with no SSH and you've been told you can't bootstrap; maybe you've inherited a system where everything runs as `QSECOFR` and you'd like to fix that.

This guide assumes:

- You can get to **5250** (whether through ACS, a third-party emulator, or `tn5250`).
- You have **ACS (Access Client Solutions)** on your workstation.
- You have **`*ALLOBJ`** authority, or someone who does is sitting next to you for the parts that need it.

You do **not** need to be an experienced UNIX/Linux administrator. The chapters explain the IBM i quirks where they come up.

---

## Start here

The site is organized as a linear path, but each chapter stands alone:

- **[The mental model](mental-model.html)** — PASE vs ILE, `/QOpenSys` vs the rest of the IFS, library list vs `$PATH`. Three pages that prevent a thousand confusions.
- **[Bootstrapping OSPM](bootstrap.html)** — installing `yum` itself. The ACS path, the no-SSH path (the original `ftp` + SQL recipe), and the dark-network case.
- **[Package managers in 2026](package-managers.html)** — `yum` is what IBM i ships; `dnf` is an AIX thing. What that means for you, and the repos worth knowing about.
- **[SSH and the VS Code workflow](ssh-and-vscode.html)** — enabling `*SSHD`, key setup, the Code for IBM i extension, and how to stop dropping into Qp2term.
- **[CCSID sanity](ccsid-sanity.html)** — 819, 1208, 65535, `JOBCCSID`, `PASE_LANG`, the `attr` and `setccsid` rituals.
- **[Service users and authority](service-users-and-authority.html)** — running daemons as something other than `QSECOFR`, `*PUBLIC` defaults, and `/QOpenSys` authority oddities.
- **[PHP](php.html)** — Zend RPMs vs Seiden PHP+, and how to choose. Cross-links to the sibling toolkit guide.
- **[Python](python.html)** — `python3` from `yum`, virtualenvs, and the few gotchas that PASE adds.
- **[Node.js](node.html)** — `nodejs` from `yum`, `npm` caveats, native modules, and the version conversation.
- **[Git on IBM i](git.html)** — installation, identity, line endings, SSH keys, and the things that bite people on day one.
- **[ODBC and PDO](odbc.html)** — short chapter; the long version lives in the [PHP / PDO / ODBC toolkit guide](https://odbcphp.k3s.com).
- **[What belongs in IBM i OSS](what-belongs.html)** — when to reach for `yum install`, when for a PTF, when for a licensed program product, when for RPG.
- **[Reference](reference.html)** — repos, links, troubleshooting, acknowledgments.

---

## The four-quadrant mental model

The single most useful thing to internalize before you start installing things is that IBM i has two userlands and two filesystems-of-thinking, and they cross-cut:

|                       | **ILE (native side)**                            | **PASE (AIX-style userland)**                      |
| --------------------- | ------------------------------------------------ | -------------------------------------------------- |
| **Lives in**          | Libraries, `*PGM` / `*MODULE` / `*SRVPGM` objects | `/QOpenSys/pkgs/` and the rest of the IFS         |
| **Run by**            | `QSYS` jobs, subsystems, library list             | PASE jobs, `$PATH`, environment files              |
| **Talks to DB2 via**  | Embedded SQL, native I/O, RPG database ops       | ODBC, JDBC, `ibm_db2`, native shared libraries     |
| **Examples**          | RPG, COBOL, CL, native SQL stored procedures     | `yum`, `php`, `python`, `node`, `git`, `bash`     |

Almost every IBM i open-source confusion can be traced to forgetting which quadrant you're in. The [mental model chapter](mental-model.html) walks through it in detail.

---

## What this guide is not

- **Not a generic Linux tutorial.** When `yum`, `bash`, or `git` work the way they do everywhere else, the guide says so and moves on.
- **Not a sales pitch for a particular vendor.** Both Zend (Perforce) and Seiden Group make distributions worth running. The guide describes the trade-offs and lets you choose.
- **Not exhaustive.** Where the IBM-published documentation is good — and increasingly it is — the guide links to it rather than duplicating it.

---

## Where next

Start with **[The mental model](mental-model.html)** if you're new to the IBM i / PASE split, or jump straight to **[Bootstrapping OSPM](bootstrap.html)** if you already know that landscape and just need `yum` installed.
