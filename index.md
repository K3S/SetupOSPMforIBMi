---
layout: default
title: Setup OSPM for IBM i
description: How to install yum and RPMs on IBM i without SSH access, using only ACS Run SQL Scripts.
---

# How To Setup Open Source Package Manager if you don't have SSH access

If your IBM i doesn't have SSH access — or you haven't set it up yet — you can still bootstrap the Open Source Package Manager (yum) using nothing but ACS and a SQL script. Cut and paste the script below into ACS under **Run SQL Scripts** and run it.

You should have OSPM working when it finishes, or a log telling you why not.

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
  with nc
;

CL:OVRDBF FILE(INPUT) TOFILE(QTEMP/FTPCMD) MBR(*FIRST) OVRSCOPE(*JOB);
CL:OVRDBF FILE(OUTPUT) TOFILE(QTEMP/FTPLOG) MBR(*FIRST) OVRSCOPE(*JOB);

CL:FTP RMTSYS('public.dhe.ibm.com');

CL:QSH CMD('touch -C 819 /tmp/bootstrap.log; /QOpenSys/usr/bin/ksh /tmp/bootstrap.sh > /tmp/bootstrap.log 2>&1');

select
case when (message_tokens = X'00000000')
 then 'Bootstrapping successful! Review /tmp/README.md for more info'
 else 'Bootstrapping failed. Consult /tmp/bootstrap.log for more info'
end as result
from table(qsys2.joblog_info('*')) x
where message_id = 'QSH0005'
order by message_timestamp desc
fetch first 1 rows only;
```

## Why this works

ACS Run SQL Scripts can issue CL commands but can't directly run PASE binaries, and the OSPM bootstrap (`bootstrap.sh`) is a PASE script — yum and the RPM ecosystem live entirely in PASE under `/QOpenSys/pkgs/`. The script bridges that gap in four steps:

1. **FTP** pulls IBM's bootstrap files from `public.dhe.ibm.com` into `/tmp`.
2. **QSH** is used as a one-line bridge from CL into PASE. The `touch -C 819` pre-creates the log file with CCSID 819 (ASCII / ISO-8859-1) so the PASE process's stdout writes back readable. Without that tag, QSH would create the stream file as EBCDIC by default and the log would render as garbage.
3. **`/QOpenSys/usr/bin/ksh`** is the PASE Korn shell — not QSH. The full path is the explicit handoff into PASE, where `bootstrap.sh` actually needs to run.
4. **The final SELECT** checks the joblog for QSH's completion message and reports the result back in the SQL output.

For a deeper explanation of QSH vs PASE and when to bridge between them, see [QSH vs PASE on IBM i: A Practical Guide](https://technical.k3s.com/docs/utilities/qsh-vs-pase/) on our docs site.

## About K3S (King III Solutions, Inc)

[K3S](https://k3s.com) is a software development company that specializes in inventory management and procurement solutions for the distribution industry. Their applications and solutions are developed to run on the IBM i OS (the best enterprise level OS!) and interface with any ERP application on any platform.

K3S open sources many of its [Guides &amp; Utilities](https://technical.k3s.com/docs/utilities/) in an effort to improve the IBM i community.
