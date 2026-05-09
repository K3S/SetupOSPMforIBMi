# IBM i Open Source Setup

A practical guide to setting up and running open source software on IBM i — `yum`, RPMs, SSH, the editor workflow, and the conventions that make the difference between an environment you trust and one you tolerate.

**Read it at: [ospm.k3s.com](https://ospm.k3s.com)**

---

## What's here

This repository is the source for the guide site. The chapters cover:

- **The mental model** — PASE vs ILE, `/QOpenSys` vs the rest of the IFS, and why the distinction matters every day.
- **Bootstrapping OSPM** — the IBM-supported ACS path, the no-SSH FTP path, and the dark-network case where the partition can't reach the public IBM file server at all.
- **Package managers in 2026** — `yum` is still what IBM i ships. `dnf` is an AIX thing. What that means in practice.
- **SSH and the VS Code workflow** — Code for IBM i, key setup, terminal CCSID, and how to stop dropping into Qp2term.
- **CCSID sanity** — 819, 1208, 65535, `JOBCCSID`, `PASE_LANG`, and the file-tagging rules that prevent half a day of "why is this `é` showing up as `Ã©`."
- **Service users and authority** — running daemons as something other than `QSECOFR`, `*PUBLIC` defaults, and `/QOpenSys` authority oddities.
- **PHP** — Zend (Perforce) RPMs, Seiden PHP+, and how to choose. Cross-links to the [PHP / PDO / ODBC toolkit guide](https://odbcphp.k3s.com).
- **Python and Node** — installing them through OSPM, virtualenvs, native modules, and the version-pinning conversation.
- **Git on IBM i** — installation, identity, line endings, and SSH keys.
- **What belongs in IBM i OSS and what does not** — a rule of thumb for when to reach for `yum install` versus a PTF, a licensed program product, or just RPG.
- **Reference** — repos, links, troubleshooting, acknowledgments.

The site is a Jekyll project using the [just-the-docs](https://just-the-docs.github.io/just-the-docs/) theme, served from GitHub Pages with a custom domain.

---

## Companion guides

This is one of several K3S-published guides on IBM i development:

- **[PHP / PDO / ODBC Toolkit Setup](https://odbcphp.k3s.com)** — running PHP on IBM i, connecting to DB2, calling RPG from PHP.
- **[RPG Tutorial](https://rpgtutorial.k3s.com)** — practical RPG programming through VS Code.
- **[IBM i AI Workers](https://ibmi-ai-workers.k3s.com)** — calling LLMs at scale from RPG.

Each is independent. Together they describe the K3S architecture: RPG owns business logic; PHP and a modern web framework own presentation and queries; AI is integrated as a worker pattern; and all of it lives on top of a properly set-up open-source environment.

---

## Contributing

Issues and pull requests welcome. Use the **Edit this page on GitHub** link at the bottom of any page on the live site to jump straight to the source for a chapter.

---

## License

- **Prose** — [Creative Commons Attribution 4.0 International (CC BY 4.0)](https://creativecommons.org/licenses/by/4.0/). See `LICENSE-PROSE`.
- **Code samples** — [MIT License](https://opensource.org/licenses/MIT). See `LICENSE`.

---

*Maintained by [King III Solutions Inc.](https://k3s.com)*
