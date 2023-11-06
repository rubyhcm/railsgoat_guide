# Description

The constantize method is a Rails MetaProgramming method designed to dynamically find a constant that matches the string specified. This is often used to dynamically instantiate a class or module. When user-supplied input is a part of that equation, great precautions must be taken to ensure security holes are not introduced.

# Bug

Within the file app/controllers/benefit_forms_controller.rb:

```ruby
def download
  begin
    path = Rails.root.join('public', 'docs', params[:name])
    file = params[:type].constantize.new(path)
    send_file file, :disposition => 'attachment'
  rescue
    redirect_to user_benefit_forms_path(:user_id => current_user.user_id)
  end
end
```

The location of the file to render is dynamically generated based on user input (params[:name]). This means the user controls the location of the file to be retrieved. Additionally, the params[:type] (File) is not validated to make sure it matches up with expected values.

# Solution

### Constantize ATTACK:

In order to attack this weakness, navigate to the benefit forms page and observe the link to download either the health or dental documents.

    http://railsgoat.dev/download?name=Health_n_Stuff.pdf&type=File

Change the name parameter to something a little more fun like:

    http://railsgoat.dev/download?name=|touch+testthis.txt&type=Logger

Note that you should now have a testthis.txt file located in your Railsgoat directory.

### Constantize SOLUTION:

In this instance and as always, there are multiple ways to fix this. A simple method to secure this function by validating user input is as follows:

```ruby
# More secure version
def download
 file_assoc = {"1" => "Health_n_Stuff.pdf", "2" => "Dental_n_Stuff.pdf"}
 begin
   if file_assoc.has_key?(params[:name].to_s)
      path = Rails.root.join('public', 'docs', file_assoc[params[:name].to_s])
      if params[:type] == "File"
        file = params[:type].constantize.new(path)
        send_file file, :disposition => 'attachment'
      end
   else
     file =  Rails.root.join('public', 'docs', "Dental_n_Stuff.pdf")
     send_file file, :disposition => 'attachment'
   end
 rescue
   redirect_to user_benefit_forms_path(:user_id => current_user.user_id)
 end
end
```
The fix ultimately boils down to leveraging a hash, if the hash has the key provided by the user, the value associated with that key is the name of the file to be returned.

# Hint

It can be very helpful for employees to download benefit forms.