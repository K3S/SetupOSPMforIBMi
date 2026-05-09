---
title: Service users and authority
nav_order: 7
---

# Service users and authority

The default IBM i open-source experience is to do everything as `QSECOFR`. It works, it's the path of least resistance, and it's a bad habit. This chapter is about how to set up the partition so that:

- Daemons (Apache, NGINX, php-fpm, node servers) run as their own users.
- Application code runs as a non-`*ALLOBJ` user.
- The actual humans on your team SSH in as themselves and `sudo` (or rather, swap profiles) only when they need to.
- Authority on `/QOpenSys` and the IFS doesn't drift over time.

This isn't security theater; it's how you make it possible to read a job log and know what was running.

---

## The "do everything as QSECOFR" antipattern

It's tempting because `QSECOFR` has every authority. SSH in as `QSECOFR`, install something, run it, debug it, ship it. Everything works.

It works until:

- A bug in your PHP application writes to a file and now that file is owned by `QSECOFR` with `*PUBLIC *EXCLUDE`, and nobody else can read it.
- A `composer install` `chown`s a `vendor/` directory to `QSECOFR`, and now the web server (running as a different user) can't read it.
- Someone files an audit ticket asking who modified a production config, and the answer is "QSECOFR" — which is true and useless.
- A compromised PHP request now has `*ALLOBJ` and can do literally anything on the partition.

Don't do this. The setup below takes maybe an hour to do once and prevents months of pain.

---

## The K3S service-user model

The convention K3S uses on customer partitions:

| User      | Purpose                                    | Authority pattern                              |
| --------- | ------------------------------------------ | ---------------------------------------------- |
| Humans    | Each developer / admin has their own       | `*USER` class, with `*ALLOBJ` for those who need it |
| `K3SWEB`  | Owns and runs the web server (Apache/NGINX) | `*USER`; granted to web-served directories     |
| `K3SAPP`  | Runs the application (php-fpm pool, node)  | `*USER`; granted to application code & data    |
| `K3SBAT`  | Runs scheduled batch jobs                  | `*USER`; granted to job inputs/outputs         |
| `K3SADMIN` | Owns the application code itself          | `*USER`; can deploy / modify code              |

This separates four concerns: serving HTTP, running application code, running batch, and deploying code. The "owns code" user (`K3SADMIN`) has write access to the application directory; the "runs application" user (`K3SAPP`) has read-and-execute. A compromised application can read its own files but can't modify them.

Whether you call them `K3SWEB` and `K3SAPP` or something else doesn't matter. Pick a naming convention and use it consistently.

---

## Creating a service user

A service user is just an IBM i user profile with no interactive access:

```cl
CRTUSRPRF USRPRF(K3SAPP)                                       +
          PASSWORD(*NONE)                                      +
          INLPGM(*NONE)                                        +
          INLMNU(*SIGNOFF)                                     +
          LMTCPB(*YES)                                         +
          USRCLS(*USER)                                        +
          SPCAUT(*NONE)                                        +
          TEXT('K3S application runtime user')                 +
          CCSID(1208)                                          +
          HOMEDIR('/home/k3sapp')
```

What each line buys you:

- **`PASSWORD(*NONE)`** — nobody can sign on as this user via 5250 or password-based SSH. They can only run jobs as this user via SBMJOB, profile-swap, or SSH-key.
- **`INLPGM(*NONE)`, `INLMNU(*SIGNOFF)`, `LMTCPB(*YES)`** — even if someone did manage to attach to this profile interactively, they can't navigate anywhere.
- **`USRCLS(*USER)`, `SPCAUT(*NONE)`** — minimal authority. We grant authority to specific objects below.
- **`CCSID(1208)`** — see [CCSID sanity](ccsid-sanity.html).
- **`HOMEDIR('/home/k3sapp')`** — for PASE work; create it next.

Then in PASE:

```bash
mkdir -p /home/k3sapp
chown k3sapp /home/k3sapp
chmod 700 /home/k3sapp
```

Repeat for `K3SWEB`, `K3SBAT`, and any others your shop needs.

---

## Granting authority by directory

The next decision: which user can do what to which IFS directories. The pattern that scales:

- **Application code** lives in `/home/k3sadmin/projects/myapp/`. Owned by `K3SADMIN`.
  - `K3SADMIN` has `*RWX` (read, write, execute).
  - `K3SAPP` has `*RX` (read, execute) — can run the code, can't modify it.
  - `K3SWEB` has `*RX` if it serves static assets directly.
  - `*PUBLIC` has `*EXCLUDE`.

- **Application data** (logs, uploads, caches) lives in `/var/k3sapp/`. Owned by `K3SAPP`.
  - `K3SAPP` has `*RWX`.
  - `K3SADMIN` has `*RX` (can inspect; doesn't usually write).
  - `*PUBLIC` has `*EXCLUDE`.

- **Web-served static** (anything Apache/NGINX directly returns) at `/var/www/myapp/`. Owned by `K3SWEB`.
  - `K3SWEB` has `*RWX`.
  - `*PUBLIC` has `*EXCLUDE` or `*RX` depending on threat model.

Setting these is one of the few times you reach for `CHGAUT` or `CHGOWN` from CL, since the IBM commands handle authority *and* the IBM-i-native authorization list semantics:

```cl
CHGOWN OBJ('/home/k3sadmin/projects/myapp')   NEWOWN(K3SADMIN) SUBTREE(*ALL)
CHGAUT OBJ('/home/k3sadmin/projects/myapp')   USER(K3SAPP)     DTAAUT(*RX) SUBTREE(*ALL)
CHGAUT OBJ('/home/k3sadmin/projects/myapp')   USER(*PUBLIC)    DTAAUT(*EXCLUDE) SUBTREE(*ALL)
```

PASE `chown`/`chmod` work too, but they map to the underlying IBM i authority model in a slightly lossy way (no concept of `*USE` vs `*RX` distinction, no authorization list awareness). For application directories, the CL forms are clearer.

{: .tip }
> Run `WRKAUT` (or PASE `ls -la`) on the directory after you set authorities, to confirm what you actually got. The first few times, the result is rarely what you expected.

---

## Authority on `/QOpenSys`

`/QOpenSys/pkgs/` is owned by IBM and updated by `yum`. **Don't change authority on `/QOpenSys/pkgs/*`.** If you `chown -R` something in there, you'll break the next `yum update`, which expects to overwrite files it owns. The same warning applies to anything under `/QOpenSys/etc/` that comes from a package.

The exception: configs you edit by hand (`/QOpenSys/etc/php.ini`, `/QOpenSys/etc/nginx/nginx.conf`, `/QOpenSys/etc/ssh/sshd_config`). Those *are* meant to be edited, and `yum` preserves your edits and writes new defaults next door as `*.rpmnew` / `*.rpmsave`. After every `yum update`, diff `*.rpmnew` against your config to see what new directives were added.

---

## Running daemons as service users

The web-server chapters of the [PHP/PDO/ODBC toolkit guide](https://odbcphp.k3s.com) cover the specifics, but the principle is universal: daemons should run as their dedicated user, not as `QSECOFR` and not as `QHTTPSVR` (which is shared across every Apache instance on the partition).

For Apache: in your instance's `httpd.conf`, set:

```
User K3SWEB
Group 0
```

For NGINX: in `nginx.conf`:

```
user k3sweb;
```

For php-fpm: in your pool config:

```ini
user = k3sapp
group = 0
```

Then make sure the daemon is *started* under those users — typically by submitting via SBMJOB with `USER(K3SWEB)`, or by configuring the IBM i HTTP admin instance to run under that user.

To confirm at runtime:

```cl
WRKACTJOB
```

The "User" column on each web-server job should show your dedicated user, not `QSECOFR` and not `QTMHHTTP`.

---

## Profile swap for development work

When a human developer needs to do something as a service user (debug a permission issue, tail a log, run a one-off as `K3SAPP`), the path that beats sharing service-user passwords is **profile swap**.

The native IBM i tool is `QWTSETP` ("set profile"), which lets a job temporarily adopt another user's profile. Wrapped in a CL command for convenience:

```cl
/* /home/k3sadmin/bin/SUASUSER.CLP */
PGM    PARM(&USER)
DCL    VAR(&USER) TYPE(*CHAR) LEN(10)
CALL   PGM(QSYS/QWTSETP) PARM(&USER)
ENDPGM
```

Compile that with adopted authority, grant to a small group of admins, and they can swap to `K3SAPP` for a session without ever knowing the (`*NONE`) password.

For a more SSH-and-bash-flavored version, see the IBM-blessed `setccsid`-style tooling, or set up an `authorized_keys` entry on the service user that's keyed to specific admins' keys.

---

## Two patterns worth avoiding

**Don't** add `*PUBLIC *USE` to `/QOpenSys` recursively to "fix" a permission problem. You'll fix today's problem and create five tomorrow.

**Don't** create a single "open source" user that owns everything PHP/Python/Node-related. You'll lose the ability to tell an HTTP-spawned process from a batch job in `WRKACTJOB`, and your audit story collapses.

---

## Where next

Service users in place, authority sane, and `/QOpenSys` left alone. Now the language chapters: **[PHP](php.html)** is next, covering the Zend-vs-Seiden choice and the install steps.
