+++
title = "Bandit | Levels 16-20 | Networking & Privileges"
date = 2026-10-12T22:38:53+02:00
description = "Walkthrough of Levels16-20 in the OverTheWire Bandit wargame. Learn port scanning, SSL/TLS service identification, file diffing, bypassing modified shell configs and linux setuid binaries."

[taxonomies]
collection = ["otw-bandit"]

[extra]
collection_part = 4
+++

## Introduction

Levels 16 through 20 shift from data manipulation into networking and privileges
mechanisms. We will scan for open ports, identify SSL services, compare files
to find changes, bypass a tampered shell configuration and use a setuid binary
to read files as another user. These concepts map directly to real-world
enumeration, persistence detection and privilege escalation.

---

### Level 15 -> 16

#### Level Goal

> The password for the next level can be retrieved by submitting the password
> of the current level to port **30001** on localhost using SSL/TLS encryption.

#### Solution

```bash
[greycipher@remnant ~]$ ssh bandit15@bandit.labs.overthewire.org -p 2220
```

Plain `nc` does not support SSL. Use `openssl s_client` instead:

```bash
bandit15@bandit:~$ openssl s_client -connect localhost:30001
[TLS handshake output...]
---
[REDACTED]
Correct!
[REDACTED]
```

> If you see `DONE`, `RENEGOTIATING`, or `KEYUPDATE` after sending the password,
> add `-ign_eof` to the command: `openssl s_client -connect localhost:30001 -ign_oef`.

#### Notes

`openssl s_client` is a diagnostic tool for testing SSL/TLS connection. It is
the equivalent of `nc` but with encryption. In security work it is useful for
inspecting certificates, testing cipher suites and verifying TLS configuration
on servers. The difference between this level and the previous one, plain TCP
vs SSL/TLS, is the same difference between HTTP and HTTPS.

---

### Level 16 -> 17

### WIP
