# Description

Overly verbose error messages that indicate whether or not a user exists can assist an attacker with brute-forcing accounts. In attempting to harvest valid usernames for a password-guessing campaign, these messages can prove very useful.

# Bug

Username and Password Enumeration

Within /app/models/user.rb:

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
You'll notice that the application generates two error messages (above).

Within /app/controllers/sessions_controller.rb:

```ruby
 10   def create
 11     path = params[:url].present? ? params[:url] : home_dashboard_index_path
 12     begin
 13       # Normalize the email address, why not
 14       user = User.authenticate(params[:email].to_s.downcase, params[:password])
 15       # @url = params[:url]
 16       rescue Exception => e
 17     end
 18
 19     if user
 20       if params[:remember_me]
 21         cookies.permanent[:auth_token] = user.auth_token if User.where(:id => user.id).exists?
 22       else
 23         session[:user_id] = user.id if User.where(:id => user.id).exists?
 24       end
 25       redirect_to path
 26     else
 27       # Removed this code, just doesn't seem specific enough!
 28       # flash[:error] = "Either your username and password is incorrect"
 29       flash[:error] = e.message
 30       render "new"
 31     end
 32   end
```
On line 16 you see the exception message object "e" is created. On line 29, the message is displayed.

One of these messages indicates the email address (username) doesn't exist on the system. The other indicates that the password is incorrect. Although the application will render both error messages, either one of the error messages would be harmful by itself. This type of information can be used by an attacker to harvest email addresses or usernames. Once that list is gathered, passwords can be guessed for each account. If the username being enumerated is actually an email address, a phishing campaign could ensue with emails made to look like they are originating from the vulnerable site.

# Solution

### Username and Password Enumeration - SOLUTION

Within /app/controllers/sessions_controller.rb

```ruby
def create
  begin
    user = User.authenticate(params[:email], params[:password])
  rescue Exception => e
  end
  
  if user
    session[:user_id] = user.id if User.where(:id => user.id).exists?
    redirect_to home_dashboard_index_path
  else
    flash[:error] =  "Either your username and password is incorrect" #e.message
    render "new"
  end
end
```

Although this fix is neither systemic nor does it address the problematic code at its core (within the user model), it does provide a quick solution. On line 12, we comment out the "e.message code" and instead provide a very generic error message that lacks specificity on what credential was incorrectly entered.

# Hint
Enter an email address that wouldn't likely exist into the login form. Analyze the result.

Can you leverage this to gain unauthorized access?
