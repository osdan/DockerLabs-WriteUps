# Aidor — DockerLabs Writeup

**Platform:** DockerLabs · **Difficulty:** Easy · **Target IP:** 172.17.0.2 ·

## Summary

Aidor is an easy-level machine based on an IDOR (Insecure Direct Object Reference) vulnerability in a web dashboard built with Flask. A simple self-registration process grants access to a dashboard that uses a user-supplied `id` parameter without verifying ownership, thereby exposing the personal information of all users directly within the page's HTML code. Cracking the associated hashes provides SSH access, while residual credentials found in the application's source code lead directly to root access.

**Attack chain:** Recon → Enumeration of `/register` and `/dashboard` (Flask app on port 5000) → Register account → IDOR on `/dashboard?id=N` leaks all usernames + password hashes → Crack hashes with John → SSH login as `aidor` → Read `app.py` source, exposing a commented-out root credential → Crack root hash → `su root` → root

## 1. Reconnaissance

```bash
ping -c 4 172.17.0.2
```
<img width="696" height="263" alt="image" src="https://github.com/user-attachments/assets/f9bef920-0ef4-4bcc-94a1-d2ec9fe2f213" />


```bash
nmap -p- -sS -sC -sV -vvv 172.17.0.2
```

```
PORT     STATE SERVICE REASON         VERSION
22/tcp   open  ssh     syn-ack ttl 64 OpenSSH 10.0p2 Debian 7 (protocol 2.0)
5000/tcp open  http    syn-ack ttl 64 Werkzeug httpd 3.1.3 (Python 3.13.5)
|_http-server-header: Werkzeug/3.1.3 Python/3.13.5
|_http-title: Iniciar Sesi\xC3\xB3n
| http-methods: 
|_  Supported Methods: POST HEAD GET OPTIONS
```

Two services exposed: SSH and a Flask web application on port 5000, with a login page titled "Iniciar Sesión".

<img width="778" height="894" alt="image" src="https://github.com/user-attachments/assets/dcdf093b-bfc6-4ffd-b2a8-4b91e28f9bfc" />


## 2. Web Enumeration

Directory brute-force against the Flask app:

```bash
dirb http://172.17.0.2:5000
```

<img width="523" height="499" alt="image" src="https://github.com/user-attachments/assets/b03fde9f-7d94-481c-8009-1c7418f8ef75" />


**Interesting paths found:**

| Path | Code |
|------|------|
| `/change_password` | 405 (POST only — not browsable) |
| `/console` | 400 |
| `/dashboard` | 302 |
| `/logout` | 302 |
| `/register` | 200 |

`/register` returns 200 without authentication — self-service account creation is open.

## 3. Initial Access via Self-Registration

![Empty registration form](images/05-register-empty.png)

Registered a throwaway account through `/register`:

```
username: Osdan
email: osdan@gmail.com
password: osdantest123@
```
<img width="477" height="433" alt="image" src="https://github.com/user-attachments/assets/2401f2ce-fc02-4594-b8db-627f58962e4d" />



Registration succeeded and redirected to `/dashboard?id=55` — the newly created account's ID appears directly in the URL as a plain integer.


<img width="926" height="980" alt="image" src="https://github.com/user-attachments/assets/36452146-8923-47bf-9f53-025d1980f5c8" />


## 4. IDOR — Discovering the Hash Leak

The `id` parameter in `/dashboard?id=N` (`N` being any numeric user ID) is trusted without verifying it belongs to the logged-in session. Scrolling down the same dashboard page also reveals a "Cambiar Contraseña" section that displays the **current password hash** of whichever `id` is loaded — the hash is rendered straight into the HTML:

```html
<div class="password-section">
    <h3><i class="fas fa-key"></i> Cambiar Contraseña</h3>
    <div class="password-info">
        <div class="current-password-display">
            <p><strong>Contraseña Actual (Hash):</strong></p>
            <div class="password-hash">5e884898da28047151d0e56f8dc6292773603d0d6aabbdd62a11ef721d1542d8</div>
        </div>
    </div>
    <form method="POST" action="/change_password" class="password-form">
        ...
    </form>
</div>
```

Note: `/change_password` itself only accepts `POST` (confirmed by the 405 in the dirb scan and a manual GET test) — the hash isn't served from that route. It's embedded directly in the `/dashboard?id=N` page, so no extra request is needed to read it.

## 5. Automating the Dump with Python

A script walks through IDs 1-100, pulling the username and hash out of each `/dashboard?id=N` response, and saves the results:
