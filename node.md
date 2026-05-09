---
title: Node.js
nav_order: 10
---

# Node.js

Node on IBM i works well. The runtime ships from the IBM repos, `npm` works the way it does everywhere else, and the IBM i community has put significant effort into making the two big native modules — `idb-connector` (DB2) and `itoolkit` (RPG / system calls) — work cleanly under PASE.

The friction points are version selection, native modules that try to download pre-built binaries from non-PowerPC sources, and the same CCSID / environment-variable considerations that apply to PHP and Python.

---

## Install Node from `yum`

```bash
yum install -y nodejs
```

This installs a single major version of Node. Which version depends on what the IBM repos currently ship — check with:

```bash
yum info nodejs
```

You'll typically see something like Node 20 (LTS) at any given moment. The IBM-shipped Node is real Node, built for PowerPC PASE; not a polyfill, not a fork.

If you need a specific version that the IBM repos don't ship (e.g., an older Node 18 for compatibility, or a newer non-LTS to test against), there are two options:

- **`nvm` (Node Version Manager)** runs in user-space, downloads pre-built Node binaries, and switches between them per-shell. It works on PASE *if* you can find pre-built PowerPC binaries for the version you want, which is hit-or-miss. Worth trying for development; not recommended for production.
- **`yum install`** of a specific version. Run `yum --showduplicates list nodejs` to see what versions are in the repo, then `yum install nodejs-20.x.y`.

For production, stay on whatever the IBM repo ships. For development, `nvm` is fine.

Verify:

```bash
node --version
npm --version
```

---

## The IBM i Node ecosystem

Two npm packages from IBM are worth knowing:

```bash
npm install idb-connector       # DB2 connectivity
npm install itoolkit             # Calling RPG / IBM i system functions
```

The `idb-connector` is the native DB2 driver. `itoolkit` is the same conceptual layer as the PHP and Python toolkits.

If you don't need RPG-call capability and would rather use the cross-platform ODBC route:

```bash
npm install odbc
```

Combined with the IBM i Access ODBC Driver (which ships with IBM i), `odbc` works the same on Linux, Windows, and IBM i. This is the strategically-preferred direction.

A canonical "connect via ODBC" snippet:

```javascript
import odbc from 'odbc';

const connStr = [
  'DRIVER={IBM i Access ODBC Driver}',
  'SYSTEM=localhost',
  'UID=K3SAPP',
  'PWD=...',
  'NAM=1',
  'DBQ=K3SDATA',
  'CCSID=1208',
  'UNICODESQL=1',
].join(';');

const conn = await odbc.connect(connStr);
const result = await conn.query('SELECT custno, name FROM custmast FETCH FIRST 5 ROWS ONLY');
console.log(result);
await conn.close();
```

---

## npm and the native-module problem

`npm install` works the same way it does everywhere. The wrinkles are at the edges, all related to native modules — packages that include C/C++ code that has to be compiled or downloaded as a pre-built binary.

The two failure modes:

1. **The package tries to download a pre-built binary** for your platform from a CDN. PowerPC PASE binaries usually aren't there. The package sees the missing binary and either falls back to building from source (good) or fails (bad).
2. **The package builds from source**, but the build script assumes a Linux-specific compiler invocation, library path, or system call.

Common offenders: `node-sass` (build-from-source heavyweight, often gives way to `sass` which is pure JavaScript), `sharp` (image processing, pre-built binaries needed), `canvas` (Cairo bindings, hard on any non-Linux). For each one, the workaround is package-specific:

- **`node-sass`** → switch to `sass` (the pure-JS Dart Sass port). Drop-in replacement.
- **`sharp`** → may require building libvips from source first; some shops use ImageMagick (`yum install ImageMagick`) and call it via `child_process` instead.
- **`canvas`** → often replaceable with `node-canvas` alternatives or a server-side approach using ImageMagick.

If a `npm install` fails with a native-module compile error, the first questions are:

1. Is there a pure-JS alternative? (Frequently yes.)
2. Do I have `gcc`, `make`, and `python3` installed for `node-gyp`? (`yum install gcc make python3`.)
3. Is the package known to work on PowerPC? Check the [IBM i OSS Docs](https://ibmi-oss-docs.readthedocs.io/) and the package's GitHub issues.

---

## CCSID and Node

Node's defaults are friendly: it reads files as UTF-8 unless told otherwise, and emits UTF-8 to stdout. The CCSID concerns are at the same boundaries as Python:

- **Source files**: tag your `.js`/`.ts`/`.mjs` files as CCSID 1208.
- **DB2 connections**: pass `CCSID=1208` and `UNICODESQL=1` in the ODBC connection string.
- **Reading non-UTF-8 files**: pass an `encoding` to `fs.readFile`, or use `iconv-lite` for anything PASE / EBCDIC-flavored.

Set `LANG`, `LC_ALL`, and `PASE_LANG` in the service user's environment, the same way as everywhere else. See [CCSID sanity](ccsid-sanity.html).

---

## Running Node as a long-lived service

The K3S production pattern, mirroring the Python one:

- A dedicated user (`K3SAPP`).
- A process supervisor (`supervisord`, or PM2 if you want a Node-native answer).
- Listening on `127.0.0.1` plus a port; NGINX in front terminating TLS and routing.
- A dedicated subsystem if you want isolation in `WRKACTJOB`.

A minimal supervisord config:

```ini
[program:myapi]
command=/QOpenSys/pkgs/bin/node /home/k3sadmin/projects/myapi/dist/server.js
directory=/home/k3sadmin/projects/myapi
user=k3sapp
environment=NODE_ENV="production",PASE_LANG="EN_US.UTF-8",LC_ALL="EN_US.UTF-8",PATH="/QOpenSys/pkgs/bin:%(ENV_PATH)s"
autostart=true
autorestart=true
stdout_logfile=/var/log/myapi/stdout.log
stderr_logfile=/var/log/myapi/stderr.log
```

PM2 is also fine — it's purely a process-supervisor written in Node — but `supervisord` is what most multi-language K3S partitions end up running because it can manage Python services in the same place.

---

## Where next

Node is installed and running. **[Git on IBM i](git.html)** covers the install, identity, line endings, and SSH keys for using `git` against GitHub or another remote from PASE.
