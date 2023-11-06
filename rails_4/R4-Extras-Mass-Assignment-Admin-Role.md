# Description

The application allows the admin attribute of a User model to be set through a mass assignment call. This vulnerability exists because a developer has indicated it is acceptable for any parameters to be pushed into the model instantiation (clarified in the code below). Any time a developer decides to use the `permit!` method on parameters, this introduces a mass assignment issue. In this case, because the User model has an admin attribute, it introduces vertical privilege escalation vulnerability.

# Bug

The bug is introduced within app/controllers/users_controller.rb file:

```ruby
def create
  user = User.new(user_params)
```

```ruby
def user_params
  params.require(:user).permit!
end
```

Any attribute (parameter) setting can be used during a mass assignment call when `permit!` is used. What this means is that conceptually, the following is allowed:

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
  user = User.new(user_params)
  user.build_benefits_data
  if user.save
    session[:user_id] = user.user_id
    redirect_to home_dashboard_index_path
  else
    @user = user
    flash[:error] = user.errors.full_messages.to_sentence
    redirect_to :signup
  end
end
```

### Mass Assignment SOLUTION:

The solution is fairly simple, remove the admin attribute from the attr_accessible method. The following code shows what we mean:

```ruby
# Note that the permit! method has been removed, and the correct syntax is used
# Also note that normally we wouldn't need user_id to be passed in but we've done that
# ...to demonstrate another vulnerability (guess you could call that a hint, huh?)
            
def user_params
  params.require(:user).permit(:email, :first_name, :last_name, :user_id, :password, :password_confirmation)
end
```

# Hint

Did you provide enough parameters when you created your account? 