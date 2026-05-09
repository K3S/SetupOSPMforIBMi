---
title: ODBC and PDO
nav_order: 12
---

# ODBC and PDO

ODBC is the strategically-preferred way to talk to DB2 from PHP, Python, Node, and basically anything that isn't RPG. The driver ships with IBM i (the **IBM i Access ODBC Driver**), it works the same way from a PASE program on the partition as it does from Linux or Windows over TCP, and it's the path IBM is consistently pointing toward for new development.

The full treatment — connection-string keywords, how to install `unixODBC`, all the PDO_ODBC and `pyodbc` and `odbc.js` examples — lives in the companion guide:

**[PHP / PDO / ODBC Toolkit Setup → Connecting to DB2](https://odbcphp.k3s.com/connecting-to-db2)**

This chapter is the short version: enough to know what you're installing, what it does, and where the long version is.

---

## What ships with IBM i, and what comes from `yum`

The **IBM i Access ODBC Driver** is part of IBM i Access Client Solutions and is shipped on the partition as part of standard IBM i. You don't `yum install` it; it's already there at `/QOpenSys/usr/lib/libcwbodbc.so` (or wherever your release puts it).

What you do `yum install` is the **driver manager** — the layer that PASE programs use to find and load the IBM driver:

```bash
yum install -y unixODBC unixODBC-devel
```

This gives you `/QOpenSys/pkgs/bin/odbcinst`, the configuration files at `/QOpenSys/etc/odbcinst.ini` and `/QOpenSys/etc/odbc.ini`, and the `libodbc.so` that PHP, Python, Node, and others link against.

Plus, depending on the language:

```bash
# PHP
yum install -y php-pdo php-pdo-odbc

# Python
yum install -y python3-pyodbc

# Node — npm install in your project:
# npm install odbc
```

---

## A canonical connection string

The form that works for almost everyone:

```
DRIVER={IBM i Access ODBC Driver};SYSTEM=localhost;UID=K3SAPP;PWD=...;NAM=1;DBQ=K3SDATA;CCSID=1208;UNICODESQL=1
```

The keywords that matter:

| Keyword       | Value                  | Meaning                                                       |
| ------------- | ---------------------- | ------------------------------------------------------------- |
| `DRIVER`      | `{IBM i Access ODBC Driver}` | The IBM-shipped driver. Note the curly braces and exact name. |
| `SYSTEM`      | `localhost` (on-box) or hostname | The IBM i to connect to.                                |
| `UID` / `PWD` | Your credentials       | Use a service user (`K3SAPP`), never `QSECOFR`.               |
| `NAM`         | `1` for SQL naming, `0` for system naming | SQL naming (`schema.table`) is what most modern code expects.  |
| `DBQ`         | A library name         | Default schema / library for unqualified references.          |
| `CCSID`       | `1208`                 | Connection-level character encoding negotiation. UTF-8.       |
| `UNICODESQL`  | `1`                    | Tells the driver that bound parameters and results are Unicode. |

The other 50+ ODBC keywords are documented (and sometimes critical for performance) in the toolkit guide. Read [Connecting to DB2](https://odbcphp.k3s.com/connecting-to-db2) when you need them.

---

## ODBC in PHP, Python, Node — at a glance

**PHP (PDO_ODBC):**

```php
$dsn = 'odbc:DRIVER={IBM i Access ODBC Driver};SYSTEM=localhost;UID=K3SAPP;PWD=...;NAM=1;DBQ=K3SDATA;CCSID=1208;UNICODESQL=1';
$pdo = new PDO($dsn);
$stmt = $pdo->query('SELECT custno, name FROM custmast FETCH FIRST 5 ROWS ONLY');
foreach ($stmt as $row) {
    echo $row['CUSTNO'] . ' ' . $row['NAME'] . "\n";
}
```

**Python (pyodbc):**

```python
import pyodbc
conn = pyodbc.connect(
    'DRIVER={IBM i Access ODBC Driver};SYSTEM=localhost;UID=K3SAPP;PWD=...;'
    'NAM=1;DBQ=K3SDATA;CCSID=1208;UNICODESQL=1'
)
for row in conn.cursor().execute('SELECT custno, name FROM custmast FETCH FIRST 5 ROWS ONLY'):
    print(row.custno, row.name)
```

**Node (odbc):**

```javascript
import odbc from 'odbc';
const conn = await odbc.connect(
  'DRIVER={IBM i Access ODBC Driver};SYSTEM=localhost;UID=K3SAPP;PWD=...;' +
  'NAM=1;DBQ=K3SDATA;CCSID=1208;UNICODESQL=1'
);
const result = await conn.query('SELECT custno, name FROM custmast FETCH FIRST 5 ROWS ONLY');
console.log(result);
```

All three call into the same IBM i Access driver, with the same connection string keywords, with the same CCSID and library-list semantics. That's the ODBC promise.

---

## Where next

For everything beyond the basics — the full keyword list, calling RPG from PHP through the toolkit on top of an existing PDO connection, performance tuning, prestart jobs, exit programs, custom subsystems for ODBC isolation — go to **[PHP / PDO / ODBC Toolkit Setup](https://odbcphp.k3s.com)**.

Otherwise, **[What belongs in IBM i OSS](what-belongs.html)** is next.
