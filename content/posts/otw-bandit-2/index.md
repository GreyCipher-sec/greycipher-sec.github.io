---
title: "Bandit | Levels 5-10 | Finding & Filtering"
date: 2026-03-08T15:59:51+01:00
lastmod: 2026-03-08T15:59:51+01:00
description: "Walkthrough of Levels 5–10 in the OverTheWire Bandit wargame. Learn how to search for files by properties, filter text with grep, sort and deduplicate data, extract readable strings from binary files, and decode base64."
summary: "Walkthrough of Levels 5–10 in the OverTheWire Bandit wargame. Learn how to search for files by properties, filter text with grep, sort and deduplicate data, extract readable strings from binary files, and decode base64."
categories: ["WarGames", "OverTheWire", "Linux"]
series: ["OverTheWire - Bandit"] # If you write several articles on the same subject
seriesOrder: 2
showSummary: true
showTableOfContents: true
showTaxonomies: true
showReadingTime: true
showWordCount: false
showDate: true
draft: false
---

# Introduction

Levels 5 through 10 shift the focus from simply reading files to finding them. The challenges introduce filtering by file properties, searching within file contents, and working with encoded data.

---

## Level 5 -> 6

**Level Goal**

> The password for the next level in stored in a file somewhere under the **inhere** directory and has all of the following properties:
> - human-readable
> - 1033 bytes in size
> - not executable

**Solution**

```
[greycipher@remnant ~]$ ssh bandit5@bandit.labs.overthewire.org -p 2220
```

Instead of checking every file manually, `find` lets us filter by all three properties at once:

```
bandit5@bandit:~$ find inhere/ -type f -readable ! -executable -size 1033c
inhere/maybehere07/.file2

bandit5@bandit:~$ cat inhere/maybehere07/.file2
[REDACTED]
```

**Notes**

`find` is one of the most useful commands on Linux. The flags used here:
- `-type f` limits results to files only.
- `-readable` matches files readable by the current user.
- `! -executable` excludes executable files.
- `-size 1033c` matches exactly 1033 bytes (`c` stands for bytes).

---

## Level 6 -> 7

**Level Goal**

> The password for the next level is stored somewhere on the server and has all of the following properties:
> - owned by user bandit7
> - owned by group bandit6
> - 33 bytes in size

**Solution**
```
[greycipher@remnant ~]$ ssh bandit6@bandit.labs.overthewire.org -p 2220
```

The file could be anywhere on the server, so we search from the root.
Redirecting errors to `/dev/null` keeps the output clean:

```
bandit6@bandit:~$ find / -type f -user bandit7 -group bandit6 -size 33c 2>/dev/null
/var/lib/dpkg/info/bandit7.password

bandit6@bandit:~$ cat /var/lib/dpkg/info/bandit7.password
[REDACTED]
```

**Notes**

- `-user` and `-group` filter by file ownership.
- `2>/dev/null` redirects `stderr` (permission denied errors) to `/dev/null`, discarding them so only valid results are shown.
- Searching from `/` covers the entire filesystem.

---

## Level 7 -> 8

**Level Goal**

> The password for the next level is stored in the file **data.txt** next to the word **millionth**.

**Solution**

```
[greycipher@remnant ~]$ ssh bandit7@bandit.labs.overthewire.org -p 2220
```

`data.txt` contains thousands of lines. `grep` finds the one we need instantly:

```
bandit7@bandit:~$ grep "millionth" data.txt
millionth	[REDACTED]
```

**Notes**

`grep` searches for a pattern inside a file and prints matching lines.
It is one of the most used commands in log analysis and forensics, searching through large files for a specific string, IP, username, or keyword is a daily task in SOC work.

---

## Level 8 -> 9

**Level Goal**

> The password for the next level is stored in the file **data.txt** and is the only line of text that occurs only once.


**Solution**

```
[greycipher@remnant ~]$ ssh bandit8@bandit.labs.overthewire.org -p 2220
```

`sort` organises identical lines together, then `uniq -u` filters to lines that appear exactly once:

```
bandit8@bandit:~$ sort data.txt | uniq -u
[REDACTED]
```

**Notes**

- `uniq` only detects duplicates on adjacent lines, which is why `sort` must come first.
- The `|` pipe passes the output of one command directly into the next.
- `-u` tells `uniq` to print only lines with no duplicates.
- This pattern `sort | uniq` is a standard one-liner for deduplication and frequency analysis.

---

## Level 9 -> 10

**Level Goal**

> The password for the next level is stored in the file **data.txt** in one of the few human-readable strings, preceded by several `=` characters.

**Solution**

```
[greycipher@remnant ~]$ ssh bandit9@bandit.labs.overthewire.org -p 2220
```

`data.txt` is a binary file. `strings` extracts human-readable text from it, then `grep` filters for lines with `=`:

```
bandit9@bandit:~$ strings data.txt | grep "==="
========== [REDACTED]
```

**Notes**

`strings` extracts printable character sequences from any file, including binaries. This is one of the first steps in static malware analysis — running `strings` on a suspicious executable often reveals hardcoded URLs, registry keys, error messages, and other indicators before you even open a disassembler.
