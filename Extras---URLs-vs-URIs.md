# Description and Solution

In our discussion of how to solve the problem of [Unvalidated
Redirects](rails_3/A10-Unvalidated-Redirects-and-Forwards-(redirect_to).md), we
chose to whitelist possible redirect URLs by making sure that they are part of
our Rails application:


```ruby
def post_authentication_redirect_path(default_path: home_dashboard_index_path)
  path = params[:url] || default_path
  Rails.application.routes.recognize_path(path)
rescue ActionController::RoutingError
  default_path
end
```
 
While that code should work for our purposes, a previous version of RailsGoat
actually recommended a different solution:

```ruby
path = home_dashboard_index_path
begin
 if params[:url].present?
  path = URI.parse(params[:url]).path
 end
rescue
end
```

This code is meant to parse a URL (such as `http://evil.com/open-redirect`) and
snip the domain, so that only the path path (`/open-redirect`) remains. This
seemed sufficient until a user discovered that the `URI.parse` method let some
other attacks through. Here are the results of some other malicious URIs that
actually work:

```ruby
[1] pry(main)> URI.parse("@evil.com/open-redirect").path
=> "@evil.com/open-redirect"

[2] pry(main)> URI.parse("////evil.com/open-redirect").path
=> "//evil.com/open-redirect"

[3] pry(main)> URI.parse("-evil.com/open-redirect").path
=> "-evil.com/open-redirect"

[4] pry(main)> URI.parse(".evil.com/open-redirect").path
=> ".evil.com/open-redirect"
```

If you try these URIs in your application, you'll see that your browser will
happily redirect you to the malicious site. The problem here is that our
`URI.parse` method doesn't differentiate between URIs and URLs. You may not
even realize that they're different, and in this case that lack of
differentiation allows users to maliciously attack our site.

Briefly, a URL (Uniform Resource Locator) is _just one type_ of URI (or Uniform
Resource Indicator); the one we're most familiar with. A URL has a pretty well
defined format, so we're used to thinking that all parseable URIs will match
this format in some way. However, the other attack examples we provided are
legitimate URIs _that are not legitimate URLs_, and take advantage of the fact
that our user's browser will try to interpret those URIs as a sensible URL.

As mentioned in the original exploit, one solution to this problem is to verify
that the requested URI is part of our Rails app:

```ruby
Rails.application.routes.recognize_path(path)
```

If we need to redirect users to paths that are outside our Rails application,
or to other domains, this approach won't work. In that case, another workable
approach is to product a whitelist of URL patterns that are known to be good:

```ruby
WHITELISTED_URLS = [
  /\Ahttp:\/\/google.com\/.*\z/,
  /\Ahttp:\/\/mysite.com\/.*\z/,
]

unless WHITELISTED_URLS.any? { |pattern| path =~ pattern }
  raise ActionController::RoutingError 
end

redirect_to path
```

As you can see, this approach can be hard to read and maintain. For instance,
as written, users can be redirected to `google.com`, but not `news.google.com`.
Sloppy regexes are also frequently insecure on their own. If you take this
approach, remember to anchor your regexes with \A and \z, or you'll likely be
[attacked via your Regexes](Extras\:-Broken-Regular-Expression.md).

We can make this a little cleaner (while still fixing the URI-versus-URL issue)
by using URI parse to detect properly formatted URLs:

```ruby
WHITELISTED_HOSTS = [
  "google.com",
  "mysite.com",
]

parsed_host = URI.parse(params[:url]).host

unless WHITELISTED_HOSTS.include?(parsed_host)
  raise ActionController::RoutingError
end

redirect_to params[:url]
```

If an attacker attempts to provide a URI like `@evil.com/open-redirect`, our
new approach will return a `nil` host and properly raise an error.

One final approach is to avoid being passed arbitrary URLs at all. While our
login method should be able to redirect you back to any location you came from,
this won't always be true. Whenever possible, redirect to hardcoded locations
(e.g. `home_dashboard_index_path`) to avoid this attack entirely:

```ruby
redirect_to home_dashboard_index_path
```