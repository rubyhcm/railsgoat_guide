# Description

DOM Based XSS (or as it is called in some texts, “type-0 XSS”) is an XSS attack wherein the attack payload is executed as a result of modifying the DOM “environment” in the victim’s browser used by the original client side script, so that the client side code runs in an “unexpected” manner. That is, the page itself (the HTTP response that is) does not change, but the client side code contained in the page executes differently due to the malicious modifications that have occurred in the DOM environment.

# Bug

The following code was taken from app/views/sessions/new.html.erb:

```javascript
<script>
 //document.write("<select style="width: 100px;">");
 //document.write("<OPTION value=1>English</OPTION>");
 //document.write("<OPTION value=2>Spanish</OPTION>");
 try {
     var hashParam = location.hash.split("#")[1];
     var paramName = hashParam.split('=')[0];
     var paramValue = decodeURIComponent(hashParam.split('=')[1]);
     document.write("<OPTION value=3>" +  paramValue  + "</OPTION>");
 } catch(err) {
 }
 //document.write("</select>");
</script>
```
The code (above) takes user input (params), and renders it back on the page without any output encoding or escaping.

# Solution

### DOM-Based Cross-Site Scripting ATTACK:

Ensure you are signed out of the application first. Make sure you are using something like Firefox as Safari/Chrome won't work for this exercise. Then, use the following link (substitute hostname for your actual hostname) to execute an alert box:

```html
http://127.0.0.1:3000/#test=<script>alert(1)</script>
```

### DOM-Based Cross-Site Scripting SOLUTION:

Leverage the Hogan function for escaping (found in the application.js file) to escape user input:

```javascript
<!-- support for multiple languages coming soon! -->
<script>
  //document.write("<select style="width: 100px;">");
  //document.write("<OPTION value=1>English</OPTION>");
  //document.write("<OPTION value=2>Spanish</OPTION>");
  try {
    var hashParam = location.hash.split("#")[1];
    var paramName = hashParam.split('=')[0];
    var paramValue = decodeURIComponent(hashParam.split('=')[1]);
    document.write("<OPTION value=3>" +   hoganEscape(paramValue)  + "</OPTION>");
  } catch(err) {
 }
  //document.write("</select>");
</script>
```
# Hint

You should view the source of the login page, might be something interesting there.