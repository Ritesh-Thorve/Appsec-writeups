# Brute-forcing a Stay-Logged-In Cookie

## Lab Overview

The application uses a vulnerable **remember me** feature.

The persistent cookie format is:

```text
base64(username:md5(password))
```

Goal: Access Carlos's account by brute-forcing the stay-logged-in cookie.

Credentials:

```text
wiener:peter
```

Victim:

```text
carlos
```

---

# Steps

## 1. Login & Enable Remember Me

Login using:

```text
wiener:peter
```

Enable the **Stay logged in** option.

Observed cookies:

- `session`
- `stay-logged-in`

---

## 2. Decode Cookie

Decoded the `stay-logged-in` cookie from Base64.

Result format:

```text
username:md5(password)
```

Example:

```text
wiener:51dc30ddc473d43a6011e9ebba6ca770
```

---

## 3. Prepare Brute-force

Using Burp Suite Intruder:

- Added password list as payload
- Applied:
  - MD5 hash on payload
- Added prefix:

```text
carlos:
```

Final payload format:

```text
base64("carlos:md5(password)")
```

---

## 4. Remove Session Cookie

Removed the `session` cookie because it changes per login and interferes with testing.

Only used:

```text
stay-logged-in
```

---

## 5. Find Valid Cookie

Sent requests with different generated cookie values.

One response showed:

- Different response length
- Different status code

This indicated successful authentication.

---

## 6. Access Carlos Account

Used the valid cookie in Repeater/browser.

The server accepted the cookie and issued a valid session.

Successfully accessed Carlos’s account.

---

# Key Learnings

- Base64 is encoding, not encryption
- MD5 is weak and insecure
- Persistent login cookies should never trust client-side data
- Lack of rate limiting enables brute-force attacks
- Remember-me functionality can introduce authentication flaws

---

# Prevention

- Use random server-side session tokens
- Never store:
  
```text
username:hash(password)
```

inside cookies

- Use secure hashing:
  - bcrypt
  - Argon2
  - PBKDF2

- Implement:
  - Rate limiting
  - Account lockout
  - Secure cookie signing (HMAC)

```
### Proof:

1. login with proper credentials to check flow
![image.png](attachment:fa573550-2938-4c70-9c8d-90c54ad78b16:image.png)
2. Send stay-log-in parameter to decoder
wiener:51dc30ddc473d43a6011e9ebba6ca770(MD5 hash)
![image.png](attachment:2be827ce-d1d1-4a86-a7c5-33efce72b46a:image.png)
3.Intruder change remove the parameter value of session and add rules
![image.png](attachment:42267323-189b-400f-a034-009038292e45:image.png)
4. Intruder Result
![image.png](attachment:6182b5a4-c188-4d61-8e62-887f89082d59:image.png)
5. End result got session and then i change both values in browser (cookie)
![image.png](attachment:debe3ec5-e957-4856-b98c-e4f1f6803008:image.png)