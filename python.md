---
title: Python
nav_order: 9
---

# Python

Python on IBM i is the easiest of the three big language ecosystems to install and run, but there are three things that surprise people:

1. There's no Python 2; only Python 3 — and you'll see `python3` everywhere, not `python`.
2. Some packages on PyPI ship pre-built wheels only for x86; on PowerPC, `pip install` falls back to building from source. Most of the time this works; sometimes it doesn't.
3. The IBM-shipped `python3-itoolkit` and `python3-ibm_db` give you direct DB2 and RPG access, and they're worth installing even if you don't think you need them yet.

This chapter walks through installation, virtualenvs, and the PASE-specific gotchas.

---

## Install Python from `yum`

```bash
yum install -y python3 python3-pip python3-devel
```

Verify:

```bash
python3 --version
pip3 --version
```

The IBM repos ship a recent Python 3 (3.9 or 3.11 depending on when your bootstrap happened — `yum info python3` shows the exact version). It's an actual CPython 3 build, not a PASE fork; if you can do something on a generic Linux Python 3, you can do it here.

There's no `python2` package on IBM i. There's no `python` symlink to `python3` either; you have to type `python3`. If you want a `python` shortcut, drop one in your own `~/.local/bin/`:

```bash
mkdir -p ~/.local/bin
ln -s /QOpenSys/pkgs/bin/python3 ~/.local/bin/python
echo 'export PATH=$HOME/.local/bin:$PATH' >> ~/.bashrc
```

…but think twice. New Python tutorials assume `python3`; the shortcut quickly becomes confusing.

---

## The IBM i Python ecosystem

Three packages from `yum` are worth knowing:

```bash
yum install -y python3-itoolkit       # Calling RPG / IBM i system functions from Python
yum install -y python3-ibm_db          # DB2 connectivity (the native ibm_db driver)
yum install -y python3-pyodbc          # The standard pyodbc, works against IBM i ODBC
```

These are the IBM-blessed paths. The `itoolkit` package is the Python equivalent of the PHP toolkit: it lets you call ILE programs, read data queues, run CL commands. `ibm_db` is the native DB2 driver. `pyodbc` is the cross-platform ODBC driver, and it's increasingly the strategic answer for the same reasons PDO_ODBC is the strategic answer for PHP.

A canonical "connect to DB2 via ODBC" snippet:

```python
import pyodbc

conn = pyodbc.connect(
    "DRIVER={IBM i Access ODBC Driver};"
    "SYSTEM=localhost;"
    "UID=K3SAPP;"
    "PWD=...;"
    "NAM=1;"            # name format: 1 = SQL naming
    "DBQ=K3SDATA;"      # default schema / library
    "CCSID=1208;"       # UTF-8 negotiation; see the CCSID chapter
    "UNICODESQL=1;"
)
cur = conn.cursor()
cur.execute("SELECT custno, name FROM custmast FETCH FIRST 5 ROWS ONLY")
for row in cur:
    print(row.custno, row.name)
```

The connection string options largely match the PHP/PDO_ODBC ones described in the [toolkit guide](https://odbcphp.k3s.com/connecting-to-db2).

---

## Virtual environments

`venv` works on IBM i exactly the way it works on Linux:

```bash
cd /home/jesse/projects/myproject
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
```

The activated environment has its own `python` and `pip` symlinks pointing at the venv's internal copies. `which python` confirms.

{: .note }
> `venv` creates symlinks back into `/QOpenSys/pkgs/bin/python3` by default. If you ever uninstall and reinstall Python via `yum`, your venvs may end up with broken symlinks. Worth re-creating venvs after a major Python upgrade.

For project dependency management, **`pip` plus `requirements.txt`** is the simplest path and works the same everywhere. **`pipx`** is also fine if you want to install CLI tools (like `httpie` or `black`) globally without polluting the system Python:

```bash
yum install -y python3-pipx        # if available; else pip install --user pipx
pipx install black
```

**`poetry`** and **`uv`** also work, but you're more likely to hit a "no pre-built wheel for ppc64" issue with their dependency resolvers than with plain `pip`. Stick with `pip` + `venv` until you have a reason to step up.

---

## The "no pre-built wheel for ppc64" problem

PyPI hosts wheels (pre-compiled binary packages) for Linux x86_64, Linux ARM, macOS, and Windows. It does **not** host wheels for PowerPC. This means when you `pip install pandas`, `pip` looks at PyPI, sees no ppc wheel, and tries to build from source.

Most of the time, building from source works. You need a C compiler (`yum install gcc`) and the relevant `-devel` packages, and the build runs in the venv. For pure-Python packages there's no source build needed; for C-extension packages (numpy, pandas, lxml, cryptography, pillow), the source build needs the dependencies present.

When it doesn't work — usually a missing system library or a build script that assumes Linux-specific paths — the options are:

1. Install the dependency from `yum` if available (e.g., `yum install libxml2-devel` for `lxml`).
2. Pin to an older version of the package that's known to build on PowerPC (often a comment in the package's GitHub issues will tell you which version).
3. Use a different package that does the same thing.
4. As a last resort, build the package somewhere else and copy the resulting wheel onto the partition.

The [IBM i OSS docs](https://ibmi-oss-docs.readthedocs.io/) maintain a list of Python packages with known PowerPC build issues; check there before you spend time fighting one.

{: .warning }
> Don't pip install with `sudo` (or as `QSECOFR`) into the system Python (`/QOpenSys/pkgs/lib/python3.x/site-packages/`). That installs system-wide, breaks `yum` upgrades, and pollutes other users' environments. Use venvs. Always.

---

## CCSID and Python

Python 3 strings are Unicode internally. The places CCSID matters are the boundaries:

- **Reading source files.** Python reads source as UTF-8 by default. Make sure your `.py` files are CCSID 1208 (see [CCSID sanity](ccsid-sanity.html)).
- **Reading and writing data files.** `open()` defaults to the locale's encoding; with `LANG=EN_US.UTF-8` set, that's UTF-8. Always pass `encoding=` explicitly when reading non-UTF-8 files: `open('legacy.txt', encoding='cp037')` for an EBCDIC file, for example.
- **Connecting to DB2 via ODBC.** Set `CCSID=1208` and `UNICODESQL=1` in the connection string, as in the example above.
- **Writing logs and stdout.** `LC_ALL=EN_US.UTF-8` in the environment ensures Python encodes its output as UTF-8.

---

## Running Python as a long-lived service

For a Python web service (Flask, FastAPI, or anything WSGI/ASGI-shaped), the K3S pattern is to run it under a process supervisor that's already on the partition: typically `supervisord` (from `yum install supervisor`) or, for production, behind an NGINX or Apache front-end with the Python process spawned via SBMJOB into a dedicated subsystem.

A minimal `supervisord` config for a FastAPI app:

```ini
[program:myapi]
command=/home/k3sadmin/projects/myapi/.venv/bin/uvicorn main:app --host 127.0.0.1 --port 8000
directory=/home/k3sadmin/projects/myapi
user=k3sapp
environment=PASE_LANG="EN_US.UTF-8",LC_ALL="EN_US.UTF-8",PATH="/QOpenSys/pkgs/bin:%(ENV_PATH)s"
autostart=true
autorestart=true
stdout_logfile=/var/log/myapi/stdout.log
stderr_logfile=/var/log/myapi/stderr.log
```

Then NGINX in front, proxying `location /api/ { proxy_pass http://127.0.0.1:8000; }`.

---

## Where next

Python sorted. **[Node.js](node.html)** is next, with the same shape: install via `yum`, npm caveats, native modules, a few PASE-specific things.
