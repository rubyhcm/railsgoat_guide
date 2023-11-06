# Description

The Railsgoat application stores and transmits Social Security Numbers insecurely.

# Bug

The Railsgoat application stores user's Social Security Numbers in plain-text within the database and because of this, it fails to adequately protect these numbers from theft. Additionally, the user's full SSN is sent back to the user within an HTTP response from the application.

The WorkInfo model (app/models/work_info.rb) is missing code to encrypt this data prior to storage. Additionally, while code exists to render only the last 4 numbers of an SSN (shown below), at no time is it used.

```ruby
# We should probably use this
def last_four
  "***-**-" << self.decrypt_ssn[-4,4]
end
```

# Solution

### SSN Storage - SOLUTION

There is a lot of guidance on adequately protecting sensitive data at rest and using a layered defensive approach. Make no mistake, this should not be your sole means of securing sensitive data. That being said, there are at least four precautions that should be taken.

* The sensitive data is encrypted everywhere, including backups
* Only authorized users can access decrypted copies of the data
* Use a strong algorithm
* Strong key is generated, protected from unauthorized access, and key change is planned for.

In the following code, we demonstrate switching from the storage of full SSN(s) in clear-text to storing them in the AES-256 encrypted format. The first thing to do is build the encrypt and decrypt functions. These can be found within app/models/work_info.rb.

```ruby
def encrypt_ssn
 aes = OpenSSL::Cipher::Cipher.new(cipher_type)
 aes.encrypt
 aes.key = key
 aes.iv = iv if iv != nil
 self.encrypted_ssn = aes.update(self.SSN) + aes.final
 self.SSN = nil
end

def decrypt_ssn
   aes = OpenSSL::Cipher::Cipher.new(cipher_type)
   aes.decrypt
   aes.key = key
   aes.iv = iv if iv != nil
   aes.update(self.encrypted_ssn) + aes.final
end

def key
  raise "Key Missing" if !(KEY)
  KEY
end

def iv
  raise "No IV for this User" if !(self.key_management.iv)
  self.key_management.iv
end

def cipher_type
  'aes-256-cbc'
end
```

Also within the WorkInfo model, we add the following line of code...

```ruby
before_save :encrypt_ssn
```

The remaining pieces are:

* We "seed" the database with per-user initialization vectors (IV) and store them within the key_management table
* Separate production and development encryption keys. Production keys should be stored in an HSM, environment variable, etc. but never within the source code. Development keys are irrelevant if not being used for real data
* Change the view where SSNs are called and rendered to the user so that the "last_four" method is called instead
* For new user's who are registering, we create an initialization specific to their account

```ruby
# SEED DATA
work_info.each do |wi|
 list = [:user_id, :SSN]
 info = WorkInfo.new(wi.reject {|k| list.include?(k)})
 info.user_id = wi[:user_id]
 info.build_key_management({:user_id => wi[:user_id], :iv => SecureRandom.hex(32) })
 info.SSN = wi[:SSN]
 info.save
end
```

```ruby
# SEPARATE PROD AND DEV KEYS (config/initializers/key.rb)
if Rails.env.production?
  # Specify env variable/location/etc. to retrieve key from
elsif Rails.env.development?
  KEY = "123456789101112123456789101112123456789101112"
end
```

```erb
# CHANGE VIEW TO CALL LAST FOUR METHOD (app/views/work_info/index.html.erb)
<td class="ssn"><%= @user.work_info.last_four %></td>
```

```ruby
def build_benefits_data
 build_retirement(POPULATE_RETIREMENTS.shuffle.first)
 build_paid_time_off(POPULATE_PAID_TIME_OFF.shuffle.first).schedule.build(POPULATE_SCHEDULE.shuffle.first)
 build_work_info(POPULATE_WORK_INFO.shuffle.first)
 # Uncomment below line to use encrypted SSN(s)
 work_info.build_key_management(:iv => SecureRandom.hex(32))
 performance.build(POPULATE_PERFORMANCE.shuffle.first)
end
```

# Hint

My SSN seems pretty important, hope it's kept safe!