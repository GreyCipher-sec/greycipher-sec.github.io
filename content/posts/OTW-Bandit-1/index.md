---
title: "Bandit | Levels 0-4 | Reading Files"
date: 2026-02-28T10:12:15+01:00
lastmod: 2026-02-28T10:12:15+01:00
description: "A beginner-friendly walkthrough of Levels 0–4 in the Bandit wargame from OverTheWire. Learn essential Linux command-line skills, SSH basics, file navigation, and how to handle hidden and oddly named files — all while building your cybersecurity foundation through hands-on practice."
categories: ["WarGames", "OverTheWire", "Linux"]
series: ["OverTheWire - Bandit"] # If you write several articles on the same subject
seriesOrder: 1
showSummary: true
showTableOfContents: true
showTaxonomies: true
showReadingTime: true
showWordCount: false
showDate: true
draft: false
---

# Introduction

If you're just getting started with Linux, cybersecurity, or Capture The Flag (CTF) challenges, the Bandit wargame from OverTheWire is one of the best hands-on introductions you can find. Designed specifically for beginners, Bandit walks you through the fundamentals of the Linux command line while subtly building the mindset needed for penetration testing and security research.

In this post, we’ll walk through Levels 0–4, breaking down not only how to solve them, but also why each command works.

## Level 0 -> 1

**Level Goal**

> The password for the next level is stored in a file called **readme** located in the home directory. Use this password to log into bandit1 using SSH. Whenever you find a password for a level, use SSH (on port 2220) to log into that level and continue the game.

**Solution**

Connect to the server as `bandit0`:

```plaintext
[greycipher@remnant ~]$ ssh bandit0@bandit.labs.overthewire.org -p 2220
```

Once inside, list the directory and read the file:
```plaintext
bandit0@bandit:~$ ls
readme

bandit0@bandit:~$ cat readme
Congratulations on your first steps into the bandit game!!
Please make sure you have read the rules at [...]

The password you are looking for is: [REDACTED]
```

**Notes**

- `cat` reads the contents of a file and prints it to standard output.
- `ls` lists the files in the current directory.

These are the two most basic commands you will use throughout this series.

---

## Level 1 -> 2

**Level Goal**

> The password for the next level is stored in a file called **-** located in the home directory

**Solution**

Connect to the server using the password previously acquired and the username `bandit1`:

```plaintext
[greycipher@remnant ~]$ ssh bandit1@bandit.labs.overthewire.org -p 2220
```

From here list the directory and read the content of the target file:

```plaintext
bandit1@bandit:~$ ls
-

bandit1@bandit:~$ cat ./-
[REDACTED]
```

**Notes**

Files starting with `-` are treated as flags. Use `./` to reference them as a path.

---

## Level 2 -> 3

**Level Goal**

> The password for the next level is stored in a file called *--spaces in this filename--* located in the home directory

**Solution**

Using the username `bandit2` and the password from the previous level connect to the server:
```plaintext
[greycipher@remnant ~]$ ssh bandit2@bandit.labs.overthewire.org -p 2220
```

Read the content of the file by either using quotes `""` or by referencing it as a path.
```plaintext
bandit2@bandit:~$ ls
--spaces in this filename--

bandit2@bandit:~$ cat ./--spaces\ in\ this\ filename--
[REDACTED]
```

**Notes**

Filenames with spaces need to be quoted or escaped. Either wrap the whole name in quotes or use a backslash before each space: `cat spaces\ in\ this\ filename`

---

## Level 3 -> 4

**Level Goal**

> The password for the next level is stored in a hidden file in the inhere directory.

**Solution**

Using the username `bandit3` and the password 

```plaintext

```
