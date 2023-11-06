# Description

Security misconfiguration can happen at any level of an application stack, including the platform, web server, application server, database, framework, and custom code. Developers and system administrators need to work together to ensure that the entire stack is configured properly. Automated scanners are useful for detecting missing patches, misconfigurations, use of default accounts, unnecessary services, etc.

# Bug

Rails has quite a few security related configurations. One of which relates to enforcing mass assignment protection.

```ruby
config.active_record.whitelist_attributes=false
```

This configuration forces an application developer to whitelist attributes that can be modified with mass-assignment. When this configuration is set to false **any attribute can be mass-assigned**.

# Solution

The solution for this issue is quite simple. In your application.rb file set the configuration as follows.

```ruby
config.active_record.whitelist_attributes=true
```
Once this configuration is updated to true and the application is restarted, any attributes to be mass-assigned will have to be defined as attr_accessible.

# Hint

It has to do with mass-assignment, whitelisting and configuration.