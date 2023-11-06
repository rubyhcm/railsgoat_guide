# Description

The OWASP description - Many web applications do not properly protect sensitive data, such as credit cards, SSNs, and authentication credentials, with appropriate encryption or hashing. Attackers may steal or modify such weakly protected data to conduct identity theft, credit card fraud, or other crimes.

Railsgoat does hash user passwords. Unfortunately, it does so using an extremely weak algorithm (MD5). Generally speaking, a strong algorithm and per-user salt can greatly improve the security of a hashed value. Also important to note, hashing and encryption are not the same. Encryption is meant to be reversible using some secret information, hashing is not, hashing is a one-way function not meant to be reversible.

All that being said, there are groups within security organizations that devote themselves to threat models built around this topic so clearly, this description does not encompass all scenarios. However, our recommendation is better than hashing using MD5.

# Bug

Within app/models/user.rb:

```ruby
before_save :hash_password

def self.authenticate(email, password)
 auth = nil
 user = find_by_email(email)
 if user
   if user.password == Digest::MD5.hexdigest(password)
     auth = user
   else
    raise "Incorrect Password!"
   end
 else
    raise "#{email} doesn't exist!"
 end
 return auth
end

def hash_password
  if self.password.present?
      self.password = Digest::MD5.hexdigest(password)
  end
end
```

# Solution

### Password Storage - ATTACK

Using the passwords stored within db/seeds.rb file, create a wordlist and leverage a password cracking tool such as John The Ripper to crack those passwords.

### Password Storage - SOLUTION

A simple solution here would be to enforce a per-user salt in creating a BCrypt hash. You would need to alter the db schema to add a password_salt and password_hash columns to the table.

```ruby
def self.authenticate(email, password)
 user = find_by_email(email)
 if user and user.password_hash == BCrypt::Engine.hash_secret(password, user.password_salt)
     user
 else
    raise "Invalid Credentials Supplied"
 end
end

def hash_password
  if self.password.present?
    self.password_salt = BCrypt::Engine.generate_salt
    self.password_hash = BCrypt::Engine.hash_secret(self.password, self.password_salt)
  end
end
```

In addition to changing the code, you must alter the database, generate two separate migrations. One to remove the `password` column from the user table and another to add both the `password_hash` and `password_salt` columns.

```
# Remove Password Column
rails g migration RemovePasswordFromUsers password:string

# Add password_hash and password_salt columns to the user table
rails g migration AddPasswordHashAndSaltToUsers password_hash:text
password_salt:text
```

# Hint

How protected are those passwords in the database against cracking?