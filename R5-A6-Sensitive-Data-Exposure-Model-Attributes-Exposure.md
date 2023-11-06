# Description

The application's API returns a model object (user or users). Using respond_with, the API returns the full model object. It is simple but exposes information such as the user's password and other user attributes that you may wish to keep invisible.

# Bug

Within app/controllers/api/v1/users_controller.rb:

```ruby
def index
  # We removed the .as_json code from the model, just seemed like extra work.
  # dunno, maybe useful at a later time?
  #respond_with @user.admin ? User.all.as_json : @user.as_json
  
  respond_with @user.admin ? User.all : @user
end

def show
  respond_with @user.as_json
end
```

The as_json method referenced in the comments section of the index action exists within the user model in order to override and safely protect our model from only rendering certain attributes. It is unused (commented out), app/models/user.rb:

```ruby
# Instead of the entire user object being returned, we can use this to filter.
def as_json
  super(only: [:id, :email, :first_name, :last_name])
end
```

The as_json method referenced in the comments section of the index action exists within the user model in order to override and safely protect our model from only rendering certain attributes. It is unused (commented out), app/models/user.rb:

    HTTP/1.1 200 OK
    Content-Type: application/json; charset=utf-8
    X-UA-Compatible: IE=Edge
    ETag: "6b4caf343a20865de174b2b530b945dd"
    Cache-Control: max-age=0, private, must-revalidate
    X-Request-Id: c3b0a57861087c0b827aab231747ef0c
    X-Runtime: 0.051734
    Connection: close
    
    {"admin":false,"created_at":"2014-01-23T16:17:10Z","email":
    "mike@metacorp.com","first_name":"Mike","id":5,"last_name":"McCabe","password":
    "b46dd2888a0904972649cc880a93f4dd","updated_at":"2014-01-23T16:17:10Z","id":5}

Note that all attributes associated with this user are returned via the API.

# Solution

### Model Attributes Exposure - ATTACK

Use the API and review the data returned. Additional information on exploiting the API [available here](https://github.com/OWASP/railsgoat/wiki/Extras-Broken-Regular-Expression).

### Model Attributes Exposure - SOLUTION

Uncomment the as_json method within the user model. Additionally, call `.as_json` on any User model object you would like to return via the API or other means. Example:

```ruby
respond_with @user.admin ? User.all.as_json : @user.as_json
```

Upon uncommenting the `as_json` method within the User model, the `as_json` method will ensure the API output only returns those attributes you have allowed in the following code:

```ruby
def as_json
  super(only: [:id, :email, :first_name, :last_name])
end
```

The response from the API should look like:        

    HTTP/1.1 200 OK
    Content-Type: application/json; charset=utf-8
    X-UA-Compatible: IE=Edge
    ETag: "2333488e856669ac637e37cb4cf09cb6"
    Cache-Control: max-age=0, private, must-revalidate
    X-Request-Id: baa6a1c90004838793614e4c61633767
    X-Runtime: 0.092768
    Connection: close
    
    {"email":"mike@metacorp.com","first_name":"Mike","last_name":"McCabe","id":5}

# Hint

We have an API available... what does it return?