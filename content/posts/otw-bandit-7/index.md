---
title: "Bandit | Levels 27-33 | Git"
date: 2026-04-12T08:42:29+02:00
lastmod: 2026-04-12T08:42:29+02:00
description: "Walkthrough of Levels 27–33 in the OverTheWire Bandit wargame. Learn essential git commands for security work: cloning, log inspection, branch exploration, tags, stash, and push, then escape an uppercase shell."
summary: "Walkthrough of Levels 27–33 in the OverTheWire Bandit wargame. Learn essential git commands for security work: cloning, log inspection, branch exploration, tags, stash, and push, then escape an uppercase shell."
categories: ["WarGames", "OverTheWire", "Linux"]
series: ["OverTheWire - Bandit"] # If you write several articles on the same subject
seriesOrder: 7
showSummary: true
showTableOfContents: true
showTaxonomies: true
showReadingTime: true
showWordCount: false
showDate: true
draft: false
---

# Introduction

Levels 27 through 32 are entirely focused on git. Version control is not just a developer tool, git repositories frequently contain leaked credentials, deleted sensitive files, recoverable from history and evidence of how a project evolved. Understanding git from an investigative perspective is a genuine DFIR and OSINT skill. level 33 closes the series with one final escape challenge.

> These levels are cloned and completed on the **local machine**, not from the Bandit server.

---

## Level 27 -> 28

**Level Goal**

> Clone the repository at `ssh://bandit27-git@bandit.labs.overthewire.org:2220/home/bandi27-git/repo` and find the password.

**Solution**

```
[greycipher@remnant ~]$ mktemp -d && cd /tmp/tmp.git27

[greycipher@remnant tmp.git27]$ git clone ssh://bandit27-git@bandit.labs.overthewire.org:2220/home/bandit27-git/repo
Cloning into 'repo'...
bandit27-git@bandit.labs.overthewire.org's password: [bandit27 password]

[greycipher@remnant tmp.git27]$ cat repo/README
The password to the next level is: [REDACTED]
```

**Notes**

`git clone` downloads a full copy of a repository including its complete history. The password for `bandit27-git` is the name as your current level password. In OSINT and red team work, cloning public repositories and searching them for secrets is a standard reconnaissance step.

---

## Level 28 -> 29

**Level Goal**

> Clone the repository and find the password, it may have been removed from the current version.

**Solution**

```
[greycipher@remnant ~]$ mktemp -d && cd /tmp/tmp.git28

[greycipher@remnant tmp.git28]$ git clone ssh://bandit28-git@bandit.labs.overthewire.org:2220/home/bandit28-git/repo

[greycipher@remnant tmp.git28]$ cat repo/README.md
# Bandit Notes
## credentials
- username: bandit29
- password: xxxxxxxxxx
```

The password has been redacted in the current version. Check the commit history:

```
[greycipher@remnant tmp.git28]$ cd repo

[greycipher@remnant tmp.git28]$ git log --oneline
adc7f88 (HEAD -> master, origin/master, origin/HEAD) fix info leak
a3437bd add missing data
cb630ec initial commit of README.md

[greycipher@remnant repo]$ git shot a3437bd
commit a3437bddd447f2d496731658e86b98cbea9d3c98
Author: Morla Porla <morla@overthewire.org>
Date:   Fri Apr 3 15:17:54 2026 +0000

    add missing data

diff --git a/README.md b/README.md
index 7ba2d2f..d4e3b74 100644
--- a/README.md
+++ b/README.md
@@ -4,5 +4,5 @@ Some notes for level29 of bandit.
 ## credentials

 - username: bandit29
-- password: <TBD>
+- password: [REDACTED]
```

**Notes**

`git log --oneline` shows a compact commit history. `git show <hash>` displays the diff introduced by that commit. Credentials that were "deleted" from a repository are still fully recoverable from history, this is one of the most common real-world secrets exposure vectors. Tools like `trufflehog` and `gitleaks` automate this search across entire repositories.

---

## Level 29 -> 30

**Level Goal**

> Clone the repository and find the password, it may be on a different branch.

**Solution**

```
[greycipher@remnant ~]$ mktemp && cd /tmp/tmp.git29

[greycipher@remnant tmp.git29]$ git clone ssh://bandit29-git@bandit.labs.overthewire.org:2220/home/bandit29-git/repo

[greycipher@remnant tmp.tmpgit29]$ cd repo && cat README.md
# Bandit Notes
Some notes for bandit30 of bandit.

## credentials

- username: bandit30
- password: <no passwords in production!>
```

List all branches including remote ones:

```
[greycipher@remnant repo]$ git branch -a
* master
  remotes/origin/HEAD -> origin/master
  remotes/origin/dev
  remotes/origin/master
  remotes/origin/sploits-dev

[greycipher@remnant repo]$ git checkout dev

[greycipher@remnant repo]$ cat README.md
# Bandit Notes
Some notes for bandit30 of bandit.

## credentials

- username: bandit30
- password: [REDACTED]
```

**Notes**

`git branch -a` shows all branches, including remote tracking branches. Development and staging branches frquently contain credentials, debug output or features not yet sanitised for production. During a repository audit always check all branches, not just `main` or `master`.

---

## Level 30 -> 31

**Level Goal**

> Clone the repository and find the password, it may be stored in a git tag.

**Solution**

```
[greycipher@remnant ~]$ mktemp && cd /tmp/tmp.git30

[greycipher@remnant tmp.git30]$ git clone ssh://bandit30-git@bandit.labs.overthewire.org:2220/home/bandit30-git/repo

[greycipher@remnant tmp.git30]$ cd repo && cat README.md
just an empty file... muahaha
```

Check for tags:

```
[greycipher@remnant repo]$ git tag
secret

[greycipher@remnant repo]$ git show secret
[REDACTED]
```

**Notes**

Git tags mark specific points in history, often used for releases. They are separate from branches and easy to overlook. `git tag` lists all tags, `git show <tagname>` displays the tagged object. Tags can contain annotation messages which sometimes hold sentitive information behind by careless developers.

---

## Level 31 -> 32

**Level Goal**

> Clone the repository. Push a file called **key.txt** containing the text `May I come in?` to the remote on branch **master**.

**Solution**

```
[greycipher@remnant]$ mktemp && cd /tmp/tmp.git31

[greycipher@remnant tmp.git31]$ git clone ssh://bandit31-git@bandit.labs.overthewire.org:2220/home/bandit31-git/repo

[greycipher@remnant tmp.git31]$ cd repo && cat README.md
This time your task is to push a file to the remote repository.
File name: key.txt
Content: 'May I come in?'
Branch: master
```

Check `.gitignore`, `*.txt` may be ignored:

```
[greycipher@remnant repo]$ cat .gitignore
*.txt
```

Force-add the file despite the ignore rule:

```
[greycipher@remnant repo]$ git add -f key.txt
[greycipher@remnant repo]$ git commit -m "add key"
[greycipher@remnant repo]$ git push origin master
[REDACTED]
```

**Notes**

`.gitignore` tells git which files to skip when staging changes. The `-f` flag on `git add` overrides it. In a security context, `.gitignore` is worth inspecting during a repository audit, it sometimes reveals the names of sensitive files that developers intentionally excluded from version control, like `.env`, private keys or config files containing credentials.

---

## Level 32 -> 33

**Level Goal**

> After all this git stuff, it's time for another escape.

**Solution**

```
[greycipher@remnant ~]$ ssh bandit32@bandit.labs.overthewire.org -p 2220
```

Everything you type is converted to uppercase, making standard
commands fail:

```
WELCOME TO THE UPPERCASE SHELL
>> ls
sh: 1: LS: not found
```

Shell special variables are not affected by the uppercasing. `$0` expands to the name of the current shell:

```
>> $0
$ whoami
bandit33
$ cat /etc/bandit_pass/bandit33
[REDACTED]
```

**Notes**

`$0` is a special shell variable that holds the name or path of the running shell. Because it starts with `$` rather than a letter, the uppercase filter does not modify it. Expanding it directly spawns a new shell instance without the restriction. This is a good example of how constraints that seem total often have edge cases, understanding how the shell processes input at a low level reveals the gap.
