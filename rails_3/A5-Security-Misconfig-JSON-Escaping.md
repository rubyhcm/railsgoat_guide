# Description

Another one of the Rails security configurations relates to escaping HTML entities in JSON.

# Bug

When the following setting is set to false, HTML entities in JSON response will not be encoded.

```ruby
ActiveSupport::escape_html_entities_in_json = false
```

# Solution

Edit the html_entities file at config/initializers/html_entities.rb and set the following to true.

```ruby
ActiveSupport::escape_html_entities_in_json = true
```
Once the initializer is edited and the application is restarted, any HTML entities in JSON responses will be encoded.

# Hint

Think HTML entities, escaping and initializers.



