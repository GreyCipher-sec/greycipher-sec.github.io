---
title: "Bandit | Levels 11-15 | Encoding & Compression"
date: 2026-03-08T17:48:38+01:00
lastmod: 2026-03-08T17:48:38+01:00
description: "A short summary for SEO and Blowfish overview"
summary: ""
categories: []
series: [] # If you write several articles on the same subject
seriesOrder:
showSummary: true
showTableOfContents: true
showTaxonomies: true
showReadingTime: true
showWordCount: false
showDate: true
draft: true
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

Base64 is an encoding scheme that represents binary data as ASCII text. It is not encryption — anyone can decode it instantly. It shows up constantly in security work: email attachments, JWT tokens, obfuscated malware payloads, and encoded PowerShell commands all commonly use base64. Recognising it on sight (`=` padding at the end, alphanumeric characters) is a useful reflex.
