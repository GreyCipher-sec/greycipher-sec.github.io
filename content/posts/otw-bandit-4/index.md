---
title: "Bandit | Levels 16-20 | Networking & Privileges"
date: 2026-03-22T15:37:38+01:00
lastmod: 2026-03-22T15:37:38+01:00
description: "Walkthrough of Levels 16–20 in the OverTheWire Bandit wargame. Learn port scanning, SSL/TLS service identification, file diffing, bypassing modified shell configs, and Linux setuid binaries."
summary: "Walkthrough of Levels 16–20 in the OverTheWire Bandit wargame. Learn port scanning, SSL/TLS service identification, file diffing, bypassing modified shell configs, and Linux setuid binaries."
categories: ["WarGames", "OverTheWire", "Linux"]
series: ["OverTheWire - Bandit"]
seriesOrder: 4
showSummary: true
showTableOfContents: true
showTaxonomies: true
showReadingTime: true
showWordCount: false
showDate: true
draft: false
---

# Introduction

Levels 16 through 20 shift from data manipulation into networking and privileges mechanisms. You will scan for open ports, identify SSL services, compare files to find changes, bypass a tampered shell configuration, and use a setuid binary to read files as another user. These concepts map directly to real-world enumeration, persistence detection and privilege escalation.

---

## Level 15 -> 16

**Level Goal**

> The password for the next level can be retrieved by submitting the password of the current level to port 30001 on localhost using SSL/TLS encryption.

**Solution**

```
[greycipher@remnant ~]$ ssh bandit15@bandit.labs.overthewire.org -p 2220
```

Plain `nc` does not support SSL. Use `openssl s_client` instead:

```
bandit15@bandit:~$ openssl s_client -connect localhost:30001
[TLS handshake output...]
---
[REDACTED]
Correct!
[REDACTED]
```

> If you see `DONE`, `RENEGOTIATING`, or `KEYUPDATE` after sending the password, add `-ign_eof` to the command: `openssl s_client -connect localhost:30001 -ign_eof`.

**Notes**

`openssl s_client` is a diagnostic tool for testing SSL/TLS connection. It is the equivalent of `nc` but with encryption. In security work it is useful for inspecting certificates, testing cipher suites and verifying TLS configuration on servers. The difference between this level and the previous one, plain TCP vs SSL/TLS, is the same difference between HTTP and HTTPS.

---

## Level 16 -> 17

**Level Goal**

> Submit the current password to a port in the range 31000-32000 on localhost. Only one post speaks SSL/TLS and will return credentials. The others echo back whatever you send.

**Solution**

```
[greycipher@remnant ~]$ ssh bandit16@bandit.labs.overthewire.org -p 2220
```

First scan the port range to find open ports:

```
bandit16@bandit:~$ nmap -sV localhost -p 31000-32000
Starting Nmap 7.94SVN ( https://nmap.org ) at 2026-03-22 15:32 UTC
Nmap scan report for localhost (127.0.0.1)
Host is up (0.00016s latency).
Not shown: 996 closed tcp ports (conn-refused)
PORT      STATE SERVICE     VERSION
31046/tcp open  echo
31518/tcp open  ssl/echo
31691/tcp open  echo
31790/tcp open  ssl/unknown
31960/tcp open  echo
```

Port 31790 speaks SSL and is not an echo service, that is the target:

```
bandit16@bandit:~$ openssl s_client -connect localhost:31790
[TLS handshake output...]
---
[REDACTED]
Correct!
-----BEGIN RSA PRIVATE KEY-----
[REDACTED PRIVATE KEY]
-----END RSA PRIVATE KEY-----
```

Save the key and use it to connect as bandit17:

```
[greycipher@remnant ~]$ chmod 600 key
[greycipher@remnant ~]$ ssh -i key bandit17@bandit.labs.overthewire.org -p 2220
```

**Notes**

- `nmap -sV` detects service versions, not just open ports, crucial for distunguishing SSL from plain TCP services.
- SSH private key files must have permissions `600` (owner read/write only). SSH will refuse to use a key file with looser permissions.
- This level mirrors a real enumeration workflow: scan -> identify -> interesting service -> interact -> extract credentials.

---

## Level 17 -> 18

**Level Goal**

> There are two files: **password.old** and **password.new**. The password is the only line that changed between them.

**Solution**

```
[greycipher@remnant ~]$ ssh bandit17@bandit.labs.overthewire.org -p 2220
```

```
bandit17@bandit:~$ diff passwords.old passwords.new
42c42
< [OLD LINE]
---
> [REDACTED]
```

**Notes**

`diff` compares two files line by line and shows what changed. The `<` symbol marks lines from the first file, `>` marks lines from the second. In this output `42c42` means line 42 was changed, the `>` line is the new password. `diff` is a standard tool for change detection, log comparison and config auditing.

---

## Level 18 -> 19


**Level Goal**

> The password is stored in a file called **readme** in the home directory. Someone modified **.bashrc** to log you out immediately on SSH login.

**Solution**

Since `.bashrc` runs on interactive login and immediately exits, pass the command directly to SSH instead of opening a shell:

```
[greycipher@remnant ~]$ ssh bandit18@bandit.labs.overthewire.org -p 2220 cat readme
[REDACTED]
```

**Notes**

SSH accepts a command as a trailing argument and executes it on the remote host without spawning an interactive shell, which means `.bashrc` never runs. This technique is useful any time a shell is restricted or modified. In incident response, a tampered `.bashrc` or `.bash_profile` is a common persistence mechanism worth checking when investigating a compromised account.

---

## Level 19 -> 20

**Level Goal**

> Use the setuid binary in the home directory to read the password from **/etc/bandit_pass/bandit20**.

**Solution**

```
[greycipher@remnant ~]$ ssh bandit19@bandit.labs.overthewire.org -p 2220
```

```
bandit19@bandit:~$ ls -la
-rwsr-x--- 1 bandit20 bandit19 ... bandit20-do

bandit19@bandit:~$ ./bandit20-do
Run a command as another user.
Example: ./bandit20-do id

bandit19@bandit:~$ ./bandit20-do cat /etc/bandit_pass/bandit20
[REDACTED]
```

**Notes**

The `s` in `-rwsr-x---` us the setuid bit. When set on an executable, it runs with the permission of the file owner rather than the user who executes it. Here `bandit-20-do` is owned by `bandit20`, so any major attack surface in Linux privilege escalation, findind unexpected ones with `find / -perm -4000 2>/dev/null` is a standard step in any privilege escalation checklist.
