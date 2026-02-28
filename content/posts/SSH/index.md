---
title: "SSH - A Practical Introduction"
date: 2026-02-28T15:58:26+01:00
lastmod: 2026-02-28T15:58:26+01:00
description: "A practical guide to SSH covering basic connections, key authentication, the config file, and common troubleshooting for Linux users."
categories: ["Linux", "Tooling"]
series: [] # If you write several articles on the same subject
showSummary: true
showTableOfContents: true
showTaxonomies: true
showReadingTime: true
showWordCount: false
showDate: true
draft: false
---

If you are working through any wargame, CTF, or home lab, SSH is the
first tool you need to be comfortable with. This post covers everything
from basic connections to config aliases — the things you will actually
use.

---

## What SSH Is

SSH (Secure Shell) is a protocol for connecting to remote machines
over a network. It encrypts everything in transit, which means your
commands and any data you send cannot be read by anyone intercepting
the connection.

When you connect to a remote server — a VPS, a lab machine, a
wargame server — SSH is almost always how you do it.

---

## Basic Connection
```plaintext
ssh username@hostname
```

If the server runs SSH on a non-standard port (default is 22), specify
it with `-p`:
```plaintext
ssh username@hostname -p 2220
```

You will be prompted for a password. Type it and press Enter — the
terminal will not show any characters while you type, that is normal.

---

## The Known Hosts Warning

The first time you connect to a server you will see something like:
```plaintext
The authenticity of host 'bandit.labs.overthewire.org' can't be established.
ED25519 key fingerprint is SHA256:xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
Are you sure you want to continue connecting (yes/no/[fingerprint])?
```

This is SSH asking you to confirm you trust the server. Type `yes` and
the server's fingerprint gets saved to `~/.ssh/known_hosts`. Next time
you connect, SSH checks the fingerprint automatically — if it changed,
SSH will warn you, which could indicate something is wrong.

> Never blindly accept fingerprints on machines you don't control in
> a production context. For lab and wargame servers it is fine.

---

## Password vs Key Authentication

SSH supports two ways to authenticate:

**Password** — you type a password when connecting. Simple but weaker.
The server has to store password hashes and brute force is possible if
the password is weak.

**Key-based** — you generate a key pair (public + private). The public
key goes on the server, the private key stays on your machine. No
password is ever sent over the network. This is the standard for
anything serious.

Generate a key pair:
```plaintext
ssh-keygen -t ed25519 -C "your comment here"
```

This creates two files in `~/.ssh/`:
- `id_ed25519` — your private key, never share this
- `id_ed25519.pub` — your public key, this goes on servers

Copy your public key to a server:
```plaintext
ssh-copy-id username@hostname
```

From that point on you connect without a password.

---

## Useful Flags

| Flag | What it does |
|------|-------------|
| `-p` | Specify port |
| `-i` | Specify a private key file |
| `-v` | Verbose output, useful for debugging connection issues |
| `-L` | Local port forwarding |
| `-N` | Do not execute a remote command, useful with `-L` |
| `-T` | Disable pseudo-terminal allocation |

---

## The SSH Config File

Typing `ssh bandit0@bandit.labs.overthewire.org -p 2220` every time
gets old fast. The config file at `~/.ssh/config` lets you define
aliases for any host.

Create or edit the file:
```plaintext
[greycipher@remnant ~]$ nano ~/.ssh/config
```

Add an entry:
```plaintext
Host bandit0
    HostName bandit.labs.overthewire.org
    User bandit0
    Port 2220
```

Now you connect with just:
```plaintext
[greycipher@remnant ~]$ ssh bandit0
```

You can add as many entries as you want. For the Bandit series you
could add one per level, or use a wildcard:
```plaintext
Host bandit*
    HostName bandit.labs.overthewire.org
    Port 2220
```

Then `ssh bandit0`, `ssh bandit1` and so on all work automatically —
SSH fills in the hostname and port, you only specify the user via the
host alias.

---

## Copying Files Over SSH

**scp**: copies files like `cp` but over SSH:
```plaintext
# local to remote
scp file.txt username@hostname:/remote/path/

# remote to local
scp username@hostname:/remote/file.txt ./local/path/
```

**sftp**: interactive file transfer session:
```plaintext
sftp username@hostname
```

Once connected you get an `sftp>` prompt where you can `ls`, `cd`,
`get` and `put` files.

---

## Troubleshooting

**Connection refused**: the server is not running SSH, the port is
wrong, or a firewall is blocking it. Double check the port with `-p`.

**Permission denied**: wrong username, wrong password, or your key
is not on the server. Try `-v` to see exactly where authentication
fails.

**Host key verification failed**: the server's fingerprint changed
since you last connected. If you trust the server, remove the old
entry from `~/.ssh/known_hosts` and reconnect:
```plaintext
ssh-keygen -R hostname
```

## What's Next

If you are working through OverTheWire Bandit, you now have everything
you need to get started. SSH is also the foundation for tunneling,
proxying, and remote port forwarding — topics worth exploring once
the basics are solid.
