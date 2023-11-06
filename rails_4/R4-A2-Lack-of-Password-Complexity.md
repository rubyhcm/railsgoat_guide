# Description

Password complexity is incredibly important and highly debated subject. Other factors play a part in the stringency of the enforcement policy applied. If a username can be enumerated, a CAPTCHA on the login form is not present or other methods to deter a brute-force password guessing campaign are not in place, at least password complexity enforcement policy can make it a that much more difficult for an attacker to guess users passwords.

# Bug

Within app/models/user.rb

```ruby
validates :password, :presence => true,
                     :confirmation => true,
                     :length => {:within => 6..40},
                     :on => :create,
                     :if => :password
```

The application validates only the password length and nothing else. Developers can leverage the format option to apply a regular expression that checks the password has sufficient complexity.

# Solution

### Lack of Password Complexity - ATTACK

Leverage a tool such as BurpSuite's intruder to brute-force the passwords of the users. The highest privileged account that you an attacker can compromise is the admin. The password is very simple ("admin1234"), username is ("admin@metacorp.com").

### Lack of Password Complexity - SOLUTION

This regular expression validates the password has the following requirements:

* 1 digit
* 1 lowercase alphabet
* 1 uppercase alphabet
* 1 special character

```ruby
validates :password, :presence => true,
                      :confirmation => true,
                      :if => :password,
                      :format => {:with => /\A.*(?=.{10,})(?=.*\d)(?=.*[a-z])(?=.*[A-Z])(?=.*[\@\#\$\%\^\&\+\=]).*\z/}
```

# Hint

I wonder how strong the administrator's password is?