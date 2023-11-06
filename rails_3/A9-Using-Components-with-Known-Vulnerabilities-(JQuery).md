# Description

JQuery Snippet contains at least one DOM-Based XSS vulnerability that can be confirmed in IE11. Unknowingly, the Railsgoat development team used this library. Credit for vulnerability discovery as well as submission to [@raesene](https://www.twitter.com/raesene). This was unintentional but goes to show how easily vulnerabilities can creep in when using third-party libraries.

# Bug

Within the file app/assets/javascripts/jquery.snippet.js:

```javascript
// snippet new window popup function
function snippetPopup(content) {
   top.consoleRef=window.open('','myconsole',
    'width=600,height=300'
     +',left=50,top=50'
     +',menubar=0'
     +',toolbar=0'
     +',location=0'
     +',status=0'
     +',scrollbars=1'
     +',resizable=1');
   top.consoleRef.document.writeln(
    '<html><head><title>Snippet :: Code View :: '+location.href+'</title></head>'
     +'<body bgcolor=white onLoad="self.focus()">'
     +'<pre>'+content+'</pre>'
     +'</body></html>'
   );
   top.consoleRef.document.close();
}
```

We can see that the location.href DOM property is used to dynamically generate a title for the text box pop-up. This value is string concatenated directly from the DOM without first performing some escaping routine or HTML encoding.

# Solution

### Using Components with Known Vulnerabilities (DOM XSS) - ATTACK

In order to demonstrate that you can indeed perform DOM XSS through this coding error, we will use a simple alert box. This does not appear to work in Chrome, Safari, or Firefox as they first URL encoded the script portion of the url before rendering which complicates browser interpretation. IE on the other hand, true to form, is totally vulnerable. The following example assumes you are running Railsgoat on localhost, port 3000. If this is the case, open IE, paste the URL (below) into IE.

```html
http://localhost:3000/tutorials/injection#</title></head><script>alert(1)</script>
```

The portion after the pound (#) symbol will close off the title and head portions of the HTML and then allow for properly generated JavaScript to be rendered and executed. After browsing to this URL, navigate to the tutorial where code snippets are shown and click on the "pop-up" link that appears after hovering over the code snippet. This should be all that is required to demonstrate DOM-XSS.

### Using Components with Known Vulnerabilities (DOM XSS) - SOLUTION

Use the hoganEscape() function defined in application.js to solve this problem. For instance:

```javascript
'<html><head><title>Snippet :: Code View :: '+hoganEscape(location.href) +'</title></head>' 
```

# Hint

Review the JQuery Code Snippet for any content that might be mirrored or reflected back and that is under our control.