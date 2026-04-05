---
title: "Bandit | Levels 25-27 | Escape & Restricted Shells"
date: 2026-04-05T08:45:44+02:00
lastmod: 2026-04-05T08:45:44+02:00
description: "Walkthrough of Levels 25-27 in te OverTheWire Bandit wargame. Learn how to identify and escape restricted shells, abuse the more pager to spawn a shell, and use vim as a shell escape vector."
summary: "Walkthrough of Levels 25-27 in te OverTheWire Bandit wargame. Learn how to identify and escape restricted shells, abuse the more pager to spawn a shell, and use vim as a shell escape vector."
categories: ["WarGames", "OverTheWire", "Linux"]
series: ["OverTheWire - Bandit"] # If you write several articles on the same subject
seriesOrder: 6
showSummary: true
showTableOfContents: true
showTaxonomies: true
showReadingTime: true
showWordCount: false
showDate: true
draft: false
---

# Introduction

Levels 25 through 27 are about restricted shells and escape techniques. When an account is configured with a non-standard shell, it is often an attempt to limit what the user can do. These levels teach you how to identify what shell is running, understand its constraints and find ways around them, skills that appear in CTF privilege escalation and real-world hardening audits.

---

## Level 25 -> 26

**Level Goal**

> Logging into bandit26 should be easy, but the shell for bandit26 is not **/bin/bash**. Find out what it is, how it works, and how to break out of it.

**Solution**

```
[greycipher@remnant ~]$ ssh bandit25@bandit.labs.overthewire.org -p 2220
```

A private key for bandit26 is available immediately:

```
bandit25@bandit:~$ ls
bandit26.sshkey

bandit25@bandit:~$ cat /etc/passwd | grep bandit26
bandit26:x:11026:11026::/home/bandit26:/usr/bin/showtext
```

The shell is `/usr/bin/showtext`, not bash, Check what it does:

```
bandit25@bandit:~$ cat /usr/bin/showtext
#!/bin/sh
export TERM=linux
exec more ~/text.txt
exit 0
```

It opens `more` to display a file and then exits, any normal SSH session closes immediately. The trick is to force `more` to stay open by making your terminal window small enough that it cannot display the full file at once, triggering its interactive pager mode.

**Shrink your terminal window vertically** to just a few lines, then connect:

```
[greycipher@remnant ~]$ ssh -i bandit26.sshkey bandit26@bandit.labs.overthewire.org -p 2220
```

`more` enters pager mode. From inside `more`, press `v` to open the current file in **vim**. From vim, spawn a shell:

```
:set shell=/bin/bash
:shell
```

You now have a bash shell as bandit26.

**Notes**

This escape works because `more` passes control to `$EDITOR` (vim by default) when you press `v`. Vim can execute shell commands via `:shell` or `:!command`. This is a well-known escape vector, vim, less, man, and other pager-based tools are all listed on [GTFOBins](https://gtfobins.github.io) as shell escape vectors.
Restricting a user's shell to a pager is not effective hardening.

---

## Level 26 -> 27

**Level Goal**

> You have a shell as bandit26. Now grab the password for bandit27.

**Solution**

From the vim shell you opened in the previous level, you are already inside as bandit26. There is a setuid binary in the home directory:

```
bandit26@bandit:~$ ls
bandit27-do  text.txt

bandit26@bandit:~$ ./bandit27-do cat /etc/bandit_pass/bandit27
[REDACTED]
```

**Notes**

This level is intentionally brief, it rewards completingthe previous escape by giving you direct access to the next password via the same setuid pattern seen in level 19. If your shell closed, repeat the level 25 -> 26 escape to get back in.
