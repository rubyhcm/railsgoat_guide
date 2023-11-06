# Description

An OS command injection attack occurs when an attacker attempts to execute system level commands through a vulnerable application. Applications are considered vulnerable to the OS command injection attack if they utilize user input in a system level command.

# Bug

This manifestation of the bug occurs within the Benefits model. A system command is used to make a copy of the file the user has chosen to upload. User-supplied input is leveraged in creating this system command.

Within `app/controllers/benefit_forms_controller.rb`:

```ruby
def upload
  file = params[:benefits][:upload]
  if file
    flash[:success] = "File Successfully Uploaded!"
    Benefits.save(file, params[:benefits][:backup])
  else
    flash[:error] = "Something went wrong"
  end
  redirect_to user_benefit_forms_path(:user_id => current_user.user_id)
end
```
Within `app/models/benefits.rb`:

```ruby
class Benefits < ActiveRecord::Base

 def self.save(file, backup=false)
   data_path = Rails.root.join("public", "data")
   full_file_name = "#{data_path}/#{file.original_filename}"
   f = File.open(full_file_name, "wb+")
   f.write file.read
   f.close
   make_backup(file, data_path, full_file_name) if backup == "true"
 end

 def self.make_backup(file, data_path, full_file_name)
    if File.exists?(full_file_name)
      silence_streams(STDERR) { system("cp #{full_file_name} #{data_path}/bak#{Time.zone.now.to_i}_#{file.original_filename}") }
    end
 end

end
```
The command injection vulnerability is introduced when the user-supplied input (name of file) is interpolated or mixed in with a system command.

# Solution

### Command Injection - ATTACK

The filename portion of the benefits[upload] parameter is vulnerable to command injection. Navigate to the benefits section of the application, and choose a file to upload. Once the file is chosen, turn your intercepting proxy on, click start upload, and intercept the request. you will want to change the backup option to true (highlighted below) and inject your commands within the filename parameter (highlighted). Note: forward slashes ('/') are escaped by the original_filename method (used to extract the file name ).

```
POST /upload HTTP/1.1
Host: railsgoat.dev
User-Agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 10.8; rv:19.0) Gecko/20100101 Firefox/19.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8
Accept-Language: en-US,en;q=0.5
Accept-Encoding: gzip, deflate
Referer: http://railsgoat.dev/users/5/benefit_forms
Cookie: _railsgoat_session=[redacted for brevity]
Connection: keep-alive
Content-Type: multipart/form-data; boundary=--------54316025
Content-Length: 1731

----------54316025
Content-Disposition: form-data; name="utf8"

✓
----------54316025
Content-Disposition: form-data; name="authenticity_token"

zKnXZO1PGcM+rFweczO7H8IDQ6NHmc8Siud2ypM6ZeA=
----------54316025
Content-Disposition: form-data; name="benefits[backup]"

true
----------54316025
Content-Disposition: form-data; name="benefits[upload]"; filename="test.rb;+mkdir+thisisatest "
Content-Type: text/x-ruby-script
```

### Command Injection - SOLUTION

The solution is fairly simple and because this is so poorly done there are numerous ways to fix the vulnerability. One option, is to abstract a file creation method and pass it options such as the path and filename, then call it twice, once for the initial upload and another for the backup. Another option is to make a copy through the use of the FileUtils.

As an example:

```ruby
def self.make_backup(file, data_path, full_file_name)
  FileUtils.cp "#{full_file_name}", "#{data_path}/bak#{Time.zone.now.to_i}_#{file.original_filename}"
end
```

# Hint

Let's create a backup when uploading a file, wonder how they are naming it?
