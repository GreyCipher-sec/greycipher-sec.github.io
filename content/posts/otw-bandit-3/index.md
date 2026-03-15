---
title: "Bandit | Levels 10-15 | Encoding & Compression"
date: 2026-03-08T17:48:38+01:00
lastmod: 2026-03-08T17:48:38+01:00
description: "Walkthrough of Levels 10-15 in the OverTheWire Bandit wargame. Learn how to decode base64, reverse ROT13, decompress layered archives, use SSH keys and communicate with local network services."
summary: "Walkthrough of Levels 10-15 in the OverTheWire Bandit wargame. Learn how to decode base64, reverse ROT13, decompress layered archives, use SSH keys and communicate with local network services."
categories: ["WarGames", "OverTheWire", "Linux"]
series: ["OverTheWire - Bandit"] # If you write several articles on the same subject
seriesOrder: 3
showSummary: true
showTableOfContents: true
showTaxonomies: true
showReadingTime: true
showWordCount: false
showDate: true
draft: false
---

# Introduction

Levels 10 through 15 introduce encoding schemes and data manipulation. You will decode base64, reverse ROT13 cipher, peel back multiple layers of compression, authenticate with an SSH key instead of a password, and send data directly to a network port. These are all skills that appread regularly in malware analysis and incident response.

---

## Level 10 -> 11

**Level Goal**

> The password for the next level is stored in the file **data.txt**, which contains base64 encoded data.

**Solution**

```
[greycipher@remnant ~]$ ssh bandit10@bandit.labs.overthewire.org -p 2220
```

```
bandit10@bandit:~$ cat data.txt
VGhlIHBhc3N3b3JkIGlzIGR0UjE3M2ZaS2IwUlJzREZTR3NnMlJXbnBOVmozcVJyCg==

bandit10@bandit:~$ base64 -d data.txt
The password is: [REDACTED]
```

**Notes**

Base64 is an encoding scheme that represents binary data as ASCII text. It is not encryption, anyone can decode it instantly. It shows up constantly in security work: email attachments, JWT tokens, obfuscated malware payloads, and encoded PowerShell commands all commonly use base64. Recognising it on sight (`=` padding at the end, alphanumeric characters) is a useful reflex.

---

## Level 11 -> 12

**Level Goal**

> The password for the next level is stored in the file **data.txt**, where all lowercase (a-z) and uppercase (A-Z) letters have been rotated by 13 positions.

**Solution**

```
[greycipher@remnant ~]$ ssh bandit11@bandit.labs.overthewire.org -p 2220
```

```
bandit11@bandit:~$ cat data.txt
Gur cnffjbeq vf [REDACTED]

bandit11@bandit:~$ cat data.txt | tr 'A-Za-z' 'N-ZA-Mn-za-m'
The password is: [REDACTED]
```

**Notes**

ROT13 is a Caesar cipher that shifts each letter by 13 positions. Since the alphabet has 26 letters, applying ROT13 twice returns the original text, encoding and decoding are the same operation. `tr` translates characters by mapping an input set to an output set. The pattern `'A-Za-z' 'N-ZA-Mn-za-m'` maps each letter to its ROT13 equivalent.

---

## Level 12 -> 13

**Level Goal**

> The password for the next level is stored in the file **data.txt**, which is a hexdump of a file that has been repeatedly compressed.

**Solution**

```
[greycipher@remnant ~]$ ssh bandit12@bandit.labs.overthewire.org -p 2220
```

Create a working directory and copy the file there:

```
bandit12@bandit:~$ mktemp -d
/tmp/tmp.Xyz123abc

bandit12@bandit:~$ cp data.txt /tmp/tmp.Xyz123abc/
bandit12@bandit:~$ cd /tmp/tmp.Xyz123abc
```

Reverse the hexdump back to binary, then peel compression layers one by one using `file` to identify each format:

```plaintext
bandit12@bandit:/tmp/tmp.XyZ123abc$ xxd -r data.txt > data

bandit12@bandit:/tmp/tmp.XyZ123abc$ file data
data: gzip compressed data

bandit12@bandit:/tmp/tmp.XyZ123abc$ mv data data.gz && gunzip data.gz
bandit12@bandit:/tmp/tmp.XyZ123abc$ file data
data: bzip2 compressed data

bandit12@bandit:/tmp/tmp.XyZ123abc$ mv data data.bz2 && bunzip2 data.bz2
bandit12@bandit:/tmp/tmp.XyZ123abc$ file data
data: gzip compressed data

bandit12@bandit:/tmp/tmp.XyZ123abc$ mv data data.gz && gunzip data.gz
bandit12@bandit:/tmp/tmp.XyZ123abc$ file data
data: POSIX tar archive

bandit12@bandit:/tmp/tmp.XyZ123abc$ mv data data.tar && tar xf data.tar
bandit12@bandit:/tmp/tmp.XyZ123abc$ file data5.bin
data5.bin: POSIX tar archive

bandit12@bandit:/tmp/tmp.XyZ123abc$ tar xf data5.bin
bandit12@bandit:/tmp/tmp.XyZ123abc$ file data6.bin
data6.bin: bzip2 compressed data

bandit12@bandit:/tmp/tmp.XyZ123abc$ mv data6.bin data6.bz2 && bunzip2 data6.bz2
bandit12@bandit:/tmp/tmp.XyZ123abc$ file data6
data6: POSIX tar archive

bandit12@bandit:/tmp/tmp.XyZ123abc$ tar xf data6
bandit12@bandit:/tmp/tmp.XyZ123abc$ file data8.bin
data8.bin: gzip compressed data

bandit12@bandit:/tmp/tmp.XyZ123abc$ mv data8.bin data8.gz && gunzip data8.gz
bandit12@bandit:/tmp/tmp.XyZ123abc$ file data8
data8: ASCII text

bandit12@bandit:/tmp/tmp.XyZ123abc$ cat data8
The password is: [REDACTED]
```

**Notes**

- `xxd -r` reverses the hexdump back to binary.
- `file` is essential to understand the filetype and use the right tool.
- the key workflow for layered compressionis: `file` -> rename with correct extension -> decompress -> repeat.
- `mktemp -d` creates a uniquely named temporary directory.

---

## Level 13 -> 14

**Level Goal**

> The password for the next level is stored in **/etc/bandit_pass/bandit14** and can only be read by user bandit14. You get a private SSH key to log into the next level instead of a password.

**Solution**

```
[greycipher@remnant ~]$ ssh bandit13@bandit.labs.overthewire.org -p 2220
```

OverTheWire doesn't allow localhost login, to use the key you will have to log out, copy the key using SCP and then use that from the local machine to log in.

```
[greycipher@remnant ~]$ scp -P 2220 bandit13@bandit.labs.overthewire.org:/home/bandit13/sshkey.private .

[greycipher@remnant ~]$ chmod 700 sshkey.private

[greycipher@remnant ~]$ ssh -i sshkey.private bandit14@bandit.labs.overthewire.org -p 2220

bandit14@bandit:~$ cat /etc/bandit_pass/bandit14
[REDACTED]
```

**Notes**

The `-i` flag tells SSH to use a specific private key file instead of prompting for a password. Key-based authentication is the standard in production environment, most servers disable password authentication entirely and only accept keys. If you have not read the [SSH introduction post](/posts/ssh) yet, now is a good time.

---

## Level 14 -> 15

**Level Goal**

> The password for the next level can be retrieved by submitting the password of the current level to port 30000 on localhost.

**Solution**

```
[greycipher@remnant ~]$ ssh bandit14@bandit.labs.overthewire.org -p 2220
```

```
bandit14@bandit:~$ echo "[REDACTED]" | nc localhost 30000
Correct!
[REDACTED]
```

**Notes**

`nc` (netcat) opens a raw TCP connection to a host and port. It is often called the "Swiss Army Knife" of networking, you can use it to send data to services, listen for incoming connections, transfer files, and test port availability. `echo` pipes the password string directly into `nc` so it gets sent as soon as the connection opens.
