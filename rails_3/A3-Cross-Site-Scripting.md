# Description

XSS flaws occur whenever an application takes untrusted data and sends it to a web browser without proper validation and escaping. XSS allows attackers to execute scripts in the victim’s browser which can hijack user sessions, deface web sites, or redirect the user to malicious sites.

# Bug

Stored Cross-Site Scripting - The following code was taken from app/views/layouts/shared/_header.html.erb

```erb
<li style="color: #FFFFFF">
  <!--
  I'm going to use HTML safe because we had some weird stuff
  going on with funny chars and jquery, plus it says safe so I'm guessing
  nothing bad will happen
  -->
  Welcome, <%= current_user.first_name.html_safe %>
</li>
```

Coincidentally, HTML safe is not safe from HTML Injection or "XSS" attacks. The name is deceiving. 

```ruby
# Psuedo-code to help conceptualize
def raw(dirty_string)
  dirty_string.to_s.html_safe
end
```
# Solution

### Stored Cross-Site Scripting ATTACK:

When registering, enter your JavaScript tag such as <script>alert("ohai")</script> in the First Name field. Upon login the header navigation bar will echo "Welcome" + your JS code. You can have your XSS code point the victim to a BeEF server and have some fun as well.

### Stored Cross-Site Scripting SOLUTION:

Often developers error on the side of using "html_safe" versus "raw" with the idea being one is safer than the other. In this example, simply removing the .html_safe call would both eliminate the attack (by default, Rails 3+ html encodes these dangerous chars). Rails 2.x would require that any potentially malicious content is wrapped within an h() tag. Potentially malicious content should be thought of anything that is dynamically generated. Also, it is important to note that if for some reason you wanted to render HTML code in literal form, you can use things like sanitize() or strip_tags().

# Hint

Apparently we had some issues rendering people's names with weird formatting or something, I dunno, I think I fixed it by safely encoding html and rendering the necessary content.

You're **Welcome**!