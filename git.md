---
title: Git on IBM i
nav_order: 11
---

# Git on IBM i

`git` works on IBM i. It's the same `git` you'd run on Linux, installed from `yum`, with the same commands, the same config file format, and the same SSH-key flow for talking to GitHub or your internal Git host.

The things that surprise people are CCSID interactions, line endings, and the fact that `git` repos should *never* live under `/QSYS.LIB` — only under the IFS. This chapter covers the install, the configuration that prevents day-one pain, and the workflow for editing source members in a git-tracked way.

---

## Install git

```bash
yum install -y git
```

Verify:

```bash
git --version
```

That's the entire install. The IBM repos ship a recent, normal `git`.

---

## Identity

Set your name and email, globally for your user:

```bash
git config --global user.name "Jesse King Harrison IV"
git config --global user.email "jesse@k3s.com"
```

This writes to `~/.gitconfig`. Verify:

```bash
git config --global --list
```

For service users that run `git pull` from a deploy script, you can either set per-repo identity via `git -C <repo> config user.email ...`, or use the slightly more elegant pattern of letting deploys go through a real human's identity (the deploy runs as `K3SAPP`, but the commit-author identity on the partition is whatever the human's local `git config` says — since you're rarely *committing* on the partition).

---

## Line endings

The default `git` behavior on PASE is to leave line endings alone (`core.autocrlf=false`). This is the right answer almost always. Don't enable `autocrlf=true` on IBM i; it converts CRLF to LF on commit and LF to CRLF on checkout, which causes problems when the same repo is also used on Linux/macOS.

Make this explicit anyway:

```bash
git config --global core.autocrlf input
git config --global core.eol lf
```

`input` means "convert CRLF to LF on commit, but leave LF alone on checkout." This is the right setting for a mixed-platform repo where the canonical line ending is LF but your collaborators on Windows might commit a CRLF file by accident.

For repos containing IBM i source members brought into the IFS via `Rfile` or VS Code's source-member browser, those typically arrive as LF already. Confirm with:

```bash
file myfile.rpgle           # should not say "with CRLF line terminators"
```

---

## SSH keys for GitHub

Generate a key dedicated to this partition (don't reuse your laptop key):

```bash
ssh-keygen -t ed25519 -C "jesse@ibmi.k3s.com" -f ~/.ssh/id_ed25519_github
```

Add it to your SSH config:

```bash
# ~/.ssh/config
Host github.com
  HostName github.com
  User git
  IdentityFile ~/.ssh/id_ed25519_github
  IdentitiesOnly yes
```

Then upload the *public* key (`~/.ssh/id_ed25519_github.pub`) to GitHub at **Settings → SSH and GPG keys → New SSH key**.

Test:

```bash
ssh -T git@github.com
# Should respond: "Hi <your-username>! You've successfully authenticated, ..."
```

After that:

```bash
git clone git@github.com:K3S/myrepo.git
```

…just works.

{: .note }
> Use a deploy key (read-only), not your personal account key, for production deploys on the partition. GitHub deploy keys are scoped to a single repo and don't give cross-repo access if the IBM i is ever compromised.

---

## CCSID and git

`git` itself is CCSID-agnostic; it stores bytes. The CCSID concerns are at the IFS boundaries:

- **Files in your working tree** should be CCSID 1208 (UTF-8). Use `setccsid 1208` on directories before you `git clone` into them, and tag any file you create explicitly. See [CCSID sanity](ccsid-sanity.html).
- **`git diff` output** is whatever your terminal expects. With `LANG=EN_US.UTF-8`, the output is UTF-8.
- **Commit messages** with non-ASCII characters: `git` defaults to UTF-8. As long as your editor (set via `core.editor`) writes UTF-8, you're fine. For IBM i, set:

  ```bash
  git config --global core.editor "vim"
  ```

  …or whatever editor you use. Avoid `EDIT FILE` / `STRSEU`-flavored editors, which write EBCDIC.

---

## Where to put the repos

Convention:

```
/home/jesse/projects/myrepo/        # personal / dev clones
/home/k3sadmin/projects/myapp/      # production deploys, owned by K3SADMIN
```

**Never** under `/QSYS.LIB`. Git's `.git/` directory is unhappy in QSYS-style storage; even if it appears to work, performance is poor and case-sensitivity is fragile.

For applications that include both modern source (PHP/Python/Node, in the IFS) and IBM i source members (RPG/CL, in QSYS libraries), the standard pattern is:

- The `git` repo sits in the IFS.
- A subdirectory (`src/rpg/`, `src/cl/`) holds the *file representation* of the RPG/CL source.
- A build script copies / pushes those files into source physical files at compile time.

The Code for IBM i extension supports this pattern natively — it can edit IFS files and source members in the same VS Code window, and the `iproj` project format ties them together.

---

## A useful global gitignore

The patterns that end up in every IBM-i project's `.gitignore`:

```
# OS / IDE
.DS_Store
Thumbs.db
.vscode/
.idea/

# Language ecosystems
node_modules/
__pycache__/
*.pyc
.venv/
venv/
vendor/
*.log

# Build outputs
dist/
build/
*.savf
*.SAVF

# IBM i / IFS noise
.svn/
*.iws
```

Set it globally:

```bash
git config --global core.excludesfile ~/.gitignore_global
```

…and put the contents above in `~/.gitignore_global`.

---

## Where next

Git is installed and configured. **[ODBC and PDO](odbc.html)** is a short chapter pointing at the sibling toolkit guide for the full DB2-connectivity story.
