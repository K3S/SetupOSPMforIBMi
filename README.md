# How To Set Up Open Source Package Manager Without SSH Access

Copy and paste the script below into ACS under 'Run SQL Scripts' and run it. 

You should have Open Source Package Manager now working (or an error log telling you why not). 

```
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

# About K3S (King III Solutions, Inc)

[K3S](https://k3s.com) is a software development company that specializes in inventory management and procurement solutions for the distribution industry. Their applications and solutions are developed to run on the IBM i OS (the best enterprise level OS!) and interface with any ERP application on any platform. 

As well K3S open sources many of its [Guides & Utilities ](https://technical.k3s.com/docs/utilities/) in an effort to improve the IBM i community. 
