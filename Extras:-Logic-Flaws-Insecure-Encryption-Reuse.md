# Description

The Railsgoat application allows employees of Metacorp to choose the Remember Me option at login, which creates a cookie named auth-token. The encryption routine used to generate the auth-token allows the application to extract a user ID. When decrypted, a user ID is extracted and the user is authorized appropriately. This same encryption routine is used elsewhere in the application in a manner such that a clever attacker can generate an auth_token cookie with whatever user ID they prefer and authorize to the application as a different user.

# Bug

Within the file lib/encryption.rb, there are two encryption related methods that we have exposed:

```ruby
# Added a re-usable encryption routine, shouldn't be an issue!
def self.encrypt_sensitive_value(val="")
   aes = OpenSSL::Cipher::Cipher.new(cipher_type)
   aes.encrypt
   aes.key = key
   aes.iv = iv if iv != nil
   new_val = aes.update("#{val}") + aes.final
   Base64.strict_encode64(new_val).encode('utf-8')
end

def self.decrypt_sensitive_value(val="")
   aes = OpenSSL::Cipher::Cipher.new(cipher_type)
   aes.decrypt
   aes.key = key
   aes.iv = iv if iv != nil
   decoded = Base64.strict_decode64("#{val}")
   aes.update("#{decoded}") + aes.final
end
```

We have placed this code under the lib directory so that we have a re-usable encryption routine. This code is used to generate a user's `auth_token` cookie responsible for authorization and access. However, we've also used this same code when encrypting a user's bank account number. This means, a user can enter in any value they would like and will receive it's encrypted equivalent back from the application. Essentially, a user has the ability to generate the auth_token cookie for any user ID and authorize as that user.

Within the app/models/pay.rb file we have a before hook that will save a user's bank account number as an encrypted value:

```ruby
# callbacks
before_save :encrypt_bank_account_num

def encrypt_bank_account_num
  self.bank_account_num = Encryption.encrypt_sensitive_value(self.bank_account_num)
end
```

Additionally, we render that encrypted value (purposefully) when the show action is created within the app/controllers/pay_controller.rb file:

```ruby
def show
  respond_to do |format|
    format.json { render :json => {:user => current_user.pay.as_json} }
  end
end
```

Lastly, we re-use this same routine within the following code is used to create a user's auth_token cookie upon sign-up or creation (app/models/user.rb):

```ruby
before_create { generate_token(:auth_token) }

def generate_token(column)
 begin
   self[column] = Encryption.encrypt_sensitive_value(self.user_id)
 end while User.exists?(column => self[column])
end
```

# Solution

### Insecure Encryption Re-use ATTACK:

Navigate to the _Pay_ section of the application. Enter your bank account number but use the number 1 as your bank account number. Once the information is entered and submitted, you'll see the encrypted value of your bank account number (1) returned. URL encode the special characters (+ and ==) and use this value as your auth_token cookie. Navigate to your dashboard and you'll have the ability to access administrative functionality.

### Insecure Encryption Re-use SOLUTION:

Create an entirely new encryption routine or create the SHA hash with a different salt.

# Hint

My "Remember Me" cookie looks familiar, almost like one of those values you get when you enter your bank account number.