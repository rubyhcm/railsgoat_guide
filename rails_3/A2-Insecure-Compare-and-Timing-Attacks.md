# Description

A timing attack can exist in several forms. This specific case relates to username (email address) enumeration. By leveraging an automated tool, an attacker can review any subtle variation in response times after submitting a login request to determine if the application is performing a computationally intense function. Meaning, if a function is run once a user is discovered, even if the password is incorrect, this information provides the user with valid or invalid usernames.

# Bug

Within app/models/user.rb

```ruby
def self.authenticate(email, password)
 auth = nil
  user = find_by_email(email)
  raise "#{email} doesn't exist!" if !(user)
   if user.password == Digest::MD5.hexdigest(password)
     auth = user
   else
    raise "Incorrect Password!"
   end
 return auth
end
```
Ignore for a moment that the application actually tells you whether or not an email address exists :-). Instead, let's look at what would happen if this error message wasn't so specific. Even if the error message vulnerability was mitigated (because it indicates whether or not a user exists), there will be some variations in the application's response between a user that exists and one that does not (however so slight, considering MD5 is in use).

To understand why, let's follow the flow of this code example. Firstly, the application looks for a user by email. If not found, nothing else really happens. No further processing, password comparison, etc. If a user _is_ found, we will perform a password comparison and process as normal.

# Solution

### Insecure Timing Attacks - SOLUTION

Within app/models/user.rb:

```ruby
def self.authenticate(email, password)
  user = find_by_email(email) || User.new(:password => "")
  if Rack::Utils.secure_compare(user.password, Digest::MD5.hexdigest(password))
    return user
  else
    raise "Incorrect username or password"
  end
end
```

To mitigate this attack and shore up our weakness, we do two things. The first is to find a user by email, if they don't exist, create a new user object in memory (not in the database) and assign it a blank password value. This means, regardless of whether or not a user exists, we will have a user to perform some processing on. The next is, we take the input from the user and match it against the user object's password leveraging secure_compare. This is a function (secure_compare) used to ensure that when a comparison happens, it will always take the same amount of time.

In summary, we have ensured that regardless of whether or not a user exists, a password comparison will always occur and it will take the same amount of time to complete.

# Hint

Timing is everything. Authenticating is important too.