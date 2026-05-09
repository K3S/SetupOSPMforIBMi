---
title: PHP
nav_order: 8
---

# PHP

There are two practical sources for PHP RPMs on IBM i in 2026:

- **Zend (Perforce)** — the original RPM distribution, hosted at `repos.zend.com/ibmiphp`. Free. The PHP that the official IBM documentation has historically pointed at.
- **Seiden PHP+** — Seiden Group's enterprise-grade distribution (formerly known as CommunityPlus+ PHP). Free RPMs, with optional paid support.

Both produce a working PHP environment. The differences are in extensions shipped, support model, and update cadence. This chapter covers the choice, the install, and the post-install configuration. For the *application* side — connecting to DB2, calling RPG, running Apache or NGINX in front of PHP — see the companion **[PHP / PDO / ODBC Toolkit Setup](https://odbcphp.k3s.com)** guide.

{: .note }
> **Zend Server Basic was discontinued.** If you're on a partition that still has Zend Server running, that's the previous-generation thing — distinct from the current Zend RPMs. Both Zend (Perforce) and Seiden PHP+ run side-by-side with an existing Zend Server installation on the same partition; you don't have to rip-and-replace.

---

## Which one to install

The honest answer: either is fine for production. The differences:

| Concern                                | Zend (Perforce) RPMs                                            | Seiden PHP+                                                       |
| -------------------------------------- | --------------------------------------------------------------- | ----------------------------------------------------------------- |
| Cost                                   | Free                                                             | Free RPMs; paid SmartSupport optional                            |
| Where the support comes from           | Community + Perforce paid options                               | Seiden Group, founded by long-time IBM i / PHP folks              |
| Default extension set                  | Smaller; some extensions (`intl`, `zip`) are conspicuously missing in some packagings | Broader; extensions like `intl`, `zip`, `gnupg`, `imagick` typically included |
| Repo URL                               | `http://repos.zend.com/ibmiphp`                                 | Supplied by Seiden after free registration                        |
| Update cadence                         | When Zend cuts releases                                          | Often quicker on security patches and IBM-i-specific fixes        |
| Configuration                          | `/QOpenSys/etc/php/php.ini`                                     | Same                                                              |

The K3S house preference, on partitions where the customer doesn't already have a strong opinion, is **Seiden PHP+**: the broader extension set saves you from manually packaging things, the support relationship is excellent if you ever do need to escalate, and Seiden Group's IBM i specificity is genuinely useful when something goes sideways.

That said, if you're running an existing application that was built and tested against the Zend RPMs, stay on the Zend RPMs. Both work.

---

## Install path 1: Zend (Perforce) RPMs

Add the Zend repo (this requires `yum-utils`, install it first if needed):

```bash
yum install yum-utils
yum-config-manager --add-repo http://repos.zend.com/ibmiphp/
```

Then install the PHP packages. The minimum useful set:

```bash
yum install -y \
  php \
  php-cli \
  php-cgi \
  php-fpm \
  php-pdo \
  php-pdo-odbc \
  php-mbstring \
  php-xml \
  php-curl \
  php-json
```

Plus the extensions your application uses:

```bash
yum install -y \
  php-intl \
  php-zip \
  php-gd \
  php-mysqlnd \
  php-bcmath \
  php-soap \
  php-ibm_db2     # optional; the native ibm_db2 extension
```

Verify:

```bash
/QOpenSys/pkgs/bin/php -v
/QOpenSys/pkgs/bin/php -m | sort   # list loaded modules
```

---

## Install path 2: Seiden PHP+

Visit [seidengroup.com](https://www.seidengroup.com), register for the free Seiden PHP+ access, and follow their setup instructions to add the `seiden_stable` repo. The pattern is the same as adding the Zend repo, with a different URL and (if you're using the signed-package flow) a GPG key.

Then:

```bash
yum install -y seiden-php           # or whatever the current package name is
yum install -y seiden-php-cli seiden-php-fpm seiden-php-pdo-odbc ...
```

The exact package names follow Seiden's naming, which has been stable for a few releases — check the Seiden install page for the current set.

Verify the same way:

```bash
/QOpenSys/pkgs/bin/php -v
/QOpenSys/pkgs/bin/php -m | sort
```

---

## Configuration

Both distributions install the canonical `php.ini` at:

```
/QOpenSys/etc/php.ini
```

For PHP RPMs to load it (CLI and CGI both check), it should be at:

```
/QOpenSys/etc/php/php.ini
```

Per-extension configuration lives in:

```
/QOpenSys/etc/php/conf.d/
```

Each extension drops a `*.ini` file in there with its `extension=...` line. You generally don't edit these; you add your own `*.ini` files alongside if you want to tweak per-extension settings.

To see exactly which `php.ini` a given PHP process loaded:

```bash
php --ini
```

Or, more visibly, drop an `index.php` containing `<?php phpinfo();` into your web root and load it. The "Loaded Configuration File" line near the top is canonical.

{: .tip }
> When `yum` updates PHP, your edited config files are preserved. The new defaults are written next door as `php.ini.rpmnew`. After every update, `diff /QOpenSys/etc/php/php.ini /QOpenSys/etc/php/php.ini.rpmnew` and merge in any new directives the upstream added.

---

## Running multiple PHP versions side by side

A single partition can host as many PHP versions as you want, each fully isolated, using `chroot`. The toolkit guide covers this in detail at [Installing PHP — multiple versions](https://odbcphp.k3s.com/installing-php#running-multiple-php-versions-side-by-side); the short version:

```bash
yum install ibmichroot
chroot_setup -y /QOpenSys/chroots/PHP74
chroot /QOpenSys/chroots/PHP74 /QOpenSys/pkgs/bin/yum install php
```

Then run that PHP via:

```bash
chroot /QOpenSys/chroots/PHP74 /QOpenSys/pkgs/bin/php -v
```

This is invaluable when you have one application stuck on PHP 7.4 and another running on PHP 8.4 and you can't pick one.

---

## Connecting to DB2

The DB2 connectivity story is large and lives in its own guide: **[PHP / PDO / ODBC Toolkit Setup](https://odbcphp.k3s.com)**. The summary:

- **PDO_ODBC** is the strategically-preferred path. ODBC drivers come with IBM i; the connection model is well-understood; the same pattern works from Linux and Windows hosts.
- **`ibm_db2`** is the older native extension. Lots of existing code uses it; it still works; new code should prefer PDO_ODBC.
- **Calling RPG** from PHP uses the IBM i Toolkit (also known as the XML Toolkit), which can sit on top of either an `ibm_db2` or PDO_ODBC connection.

If you're starting a new application, install:

```bash
yum install -y php-pdo php-pdo-odbc unixODBC
```

…then go to the toolkit guide for connection-string examples.

---

## Composer

Composer (the PHP package manager) is not in the IBM repos. Install it the way the rest of the world does:

```bash
cd /tmp
curl -sS https://getcomposer.org/installer | /QOpenSys/pkgs/bin/php
mv composer.phar /QOpenSys/pkgs/bin/composer
chmod +x /QOpenSys/pkgs/bin/composer
```

Then:

```bash
composer --version
```

Composer Just Works on IBM i. The one IBM-i-specific thing to remember: `composer install` will create a `vendor/` directory with whatever ownership the running user has. If you ran `composer install` as `K3SADMIN` and the web server runs as `K3SWEB`, make sure `K3SWEB` has read access to `vendor/`. See [Service users and authority](service-users-and-authority.html).

---

## Where to keep your application code

Convention:

```
/home/k3sadmin/projects/<appname>/
├── public/             # web root, served by Apache/NGINX
│   └── index.php
├── src/                # application code
├── vendor/             # composer-installed
├── config/
└── composer.json
```

The web server (running as `K3SWEB`) is granted `*RX` on this whole tree. The application user (running php-fpm pools as `K3SAPP`) is granted `*RX` on the code and `*RWX` on whichever subdirectories need writes (`storage/`, `cache/`, `logs/`).

---

## Where next

PHP installed and configured. For the web-server side (Apache + FastCGI or NGINX + php-fpm), connection-string-level DB2 detail, calling RPG from PHP, performance tuning, and isolating PHP into its own subsystem, see the **[PHP / PDO / ODBC Toolkit Setup](https://odbcphp.k3s.com)** guide.

Otherwise, continue to **[Python](python.html)** for the Python install, virtualenvs, and the few PASE-specific quirks.
