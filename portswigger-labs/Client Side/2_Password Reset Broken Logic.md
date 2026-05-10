# Password Reset Broken Logic

## Vulnerability

Account Takeover via insecure password reset logic.

The application validates the reset token but fails to verify whether the token belongs to the supplied username.

---

# Root Cause

The reset request contains:

```text
token
username
new-password
```

The server checks if the token is valid but trusts the user-controlled `username` parameter.

Broken logic:

```text
token + username
```

Secure logic should be:

```text
token → mapped user → update password
```

---

# Exploit Steps

## 1. Start Password Reset

Initiate password reset for your own account.
![image.png](attachment:8553fe06-f42f-48e1-8d1a-09f1b8841e62:image.png)
---

## 2. Receive Reset Token

Open the reset link received in email.
![image.png](attachment:d256f10a-94ff-4048-b8ad-228234b951c4:image.png)
---

## 3. Intercept Request

Intercept the password reset submission using Burp Suite.

Original request:

```http
POST /reset-password

token=abc123
username=wiener
new-password=test123
```

---

## 4. Modify Username

Change:

```text
username=wiener
```

to:

```text
username=carlos
```
![image.png](attachment:3baf7982-f5af-4178-9749-6867ddcdc050:image.png)
---

## 5. Submit Request

Forward the modified request.

The server accepts the valid token and resets Carlos’s password.

---

# Why It Works

The application:

- Validates token existence
- Does NOT validate token ownership

Because the token is not bound to a specific user, attackers can reuse their own token to reset another user's password.

---

# Impact

- Full Account Takeover
- Unauthorized password reset
- Privilege escalation if admin account targeted

---

# Key Learnings

- Business logic flaws can lead to critical vulnerabilities
- Reset tokens must always be tied to a specific account
- Never trust user-controlled parameters like `username`
- Authentication flows are common bug bounty targets

---

# Prevention

- Bind reset tokens to user IDs server-side
- Do not accept username/email during final reset request
- Identify account only through token
- Expire tokens after short duration
- Invalidate token after one use

---

# Secure Flow Example

```text
1. User requests password reset
2. Server generates token linked to user
3. Email contains only token
4. User submits new password
5. Server:
   - Validates token
   - Fetches mapped user internally
   - Updates password
```

---

# Bug Bounty Insight

Broken authentication logic is extremely common in real-world applications.

High-value areas to test:

- Password reset flows
- 2FA logic
- Email verification
- OAuth/account linking
- Session management