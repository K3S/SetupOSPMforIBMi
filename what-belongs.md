---
title: What belongs in IBM i OSS
nav_order: 13
---

# What belongs in IBM i OSS, and what does not

The single most useful piece of judgment for a long-lived IBM i shop is knowing *when* to reach for `yum install`, *when* to install a PTF, *when* to license a vendor product, and *when* to just write RPG. There's no algorithm; the trade-offs depend on what you already have and what you're optimizing for.

This chapter is one shop's rules of thumb, with reasons. Disagree with them productively.

---

## Use `yum` (open source) for

**Cross-platform tooling.** Anything you'd install on a Linux box for the same job. `git`, `bash`, `vim`, `curl`, `rsync`, `tmux`, `jq`, `htop`. These are commodity utilities; the open-source versions are exactly as good as anywhere else.

**Web stacks and language runtimes.** PHP, Python, Node, Apache, NGINX, Redis (where supported). The open-source versions are the canonical implementations; the IBM-licensed alternatives (Zend Server in its old form, IBM HTTP Server) are increasingly thin wrappers around the same code.

**Newer technologies that don't have an IBM equivalent.** Kafka clients, OAuth libraries, message queue tooling, observability agents. If you need it on the IBM i for an integration, `yum` it (or build it from source with `gcc` if necessary).

**Anything you're going to use on day one and tear out on day six.** Quick-experiment territory. `yum install`, do the experiment, `yum remove`. No commitment.

---

## Don't use `yum` for

**Database operations on DB2.** SQL access from PHP/Python/Node should go through ODBC or the IBM-shipped database libraries — not through some `yum`-installed alternative DB layer. The IBM-shipped path has decades of investment behind it; the open-source alternatives don't know about library lists, journaling, commitment control, or the QSYS schema model.

**Replacing IBM-licensed software for capabilities you've already paid for.** If you have BRMS, use BRMS. Don't try to reproduce its capabilities with `tar` and `cron`. If you have Db2 Mirror, use it. Don't reproduce it with `rsync`.

**Critical infrastructure that has an LPP or PTF equivalent.** If IBM ships a thing as a licensed program product or a PTF, that's a path with IBM support behind it. The open-source version is one community update away from breaking.

**Running RPG-equivalent business logic.** Don't rewrite a working RPG program in Python because Python feels modern. Use the right tool. RPG is a great language for the kind of thing RPG does; PHP/Python/Node are great for the kind of thing PHP/Python/Node do; mixing them with the toolkit is the K3S architecture.

---

## Use a PTF for

**Bug fixes to IBM-shipped software.** Db2 SQL fixes, RPG compiler fixes, OS-level patches. If IBM has a PTF for the bug you're hitting, install the PTF.

**Security updates to the OS.** The IBM i Group PTFs are how IBM ships security updates. Apply them on a regular cadence; don't wait for things to break.

**New features in IBM-shipped tooling.** `WRKACTJOB` improvements, new SQL services, new Db2 functions. These come through PTFs. They're worth the install for what they add.

**TR-level releases (Technology Refreshes).** These are bundles of new capability. They include open-source enhancements (like the `5733-OPS` to RPM transition). Apply them at the cadence your shop is comfortable with.

---

## Use a licensed program product (LPP) for

**Capabilities IBM specifically ships.** Db2 Mirror, IBM Rational Developer for i (RDi), IBM HTTP Server, BRMS, IBM i Access. These are paid-for; if you've paid for them, use them. If you haven't, evaluate whether you should.

**Things with a regulatory / compliance requirement** that names the IBM product. Auditors are happier with "running BRMS for backups" than "running a hand-rolled `rsync` script for backups."

**Long-running workloads where IBM's level of support and stability is the value.** If your batch posting job has run nightly without incident for fifteen years, that's the value of the platform. Don't gratuitously rewrite it.

---

## Use a third-party / vendor product for

**Capabilities the IBM-shipped offering and the OSS world both cover poorly.** ERP, EDI, WMS, certain industry-specific applications. There's a long-tail of IBM i ISV ecosystem here that's genuinely valuable.

**Where a vendor's expertise is the product.** Seiden Group's PHP+ is a "PHP RPMs plus support" product. The product is partly the RPMs and partly the people behind them. Same with Halcyon, Profound Logic, and others — what you're buying is the company.

---

## Just write RPG (or CL, or SQL) for

**Anything that's primarily about manipulating Db2 tables on this partition.** RPG and SQL stored procedures are unbeatable for this. They run in the same job as the data; there's no network round-trip, no serialization tax, no driver layer.

**Things that need to interoperate with the existing RPG codebase.** If your codebase is mostly RPG, adding more RPG is the path of least friction. Don't introduce Python because it's "more modern" — introduce it when you have a reason that's specific to the work.

**Things that need access to native IBM i features.** Library list manipulation, dataqueues, user spaces, output queues, message queues. Native APIs from RPG are direct; from PASE they go through the toolkit and add a hop.

---

## A two-question test

When you're not sure where something belongs:

1. **Is the IBM-supplied path (LPP, PTF, native) materially worse for this specific job?** If no, take the IBM-supplied path. If yes, why?
2. **Will I be able to staff this in five years?** Open source widens the talent pool; RPG narrows it. Both are real considerations, and the right answer depends on what your shop already has.

The K3S house position is roughly: *use what IBM ships when it's good (Db2, the JVM, the OS, ODBC), use open source for the things open source does better (web frameworks, dependency management, modern tooling), keep RPG owning the database-shaped business logic, and resist the temptation to rewrite working code on aesthetic grounds.*

That's the architecture R7 is built on. It's also the architecture the K3S [PHP/PDO/ODBC](https://odbcphp.k3s.com) and [RPG Tutorial](https://rpgtutorial.k3s.com) guides assume.

---

## Where this chapter is wrong (in your shop)

Every shop has constraints this chapter doesn't know about:

- A regulatory environment that mandates specific products.
- A skills mix that makes one path much cheaper than another.
- An ownership change that requires a migration off the IBM i in N years and so makes "write more RPG" the wrong call.
- An ISV relationship that pre-decides parts of the stack.

Use the rules of thumb as a starting point, override them where your situation calls for it, and document why.

---

## Where next

The last chapter, **[Reference](reference.html)**, collects the links, repos, and acknowledgments you'll come back to.
