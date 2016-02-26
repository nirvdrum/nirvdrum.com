---
title: How to Perform Basic Authentication in Selenium
author: Kevin Menard
tags: [opensource, selenium]
layout: post
excerpt: |-
  Performing basic authentication in Selenium 1.x may seem impossible at first glance.  Selenium 1.x drives the browser via JavaScript, but is unable to do anything when the basic authentication dialog pops up.  One way to get around this is to embed the user credentials in the URL.  Traditionally this has worked well, but newer browsers have disabled that functionality due to the security risks inherently involved.  Fortunately, there is another way: send over the authentication credentials in the request header.
---

Performing basic authentication in Selenium 1.x may seem impossible at first glance.  Selenium 1.x drives the browser via JavaScript, but is unable to do anything when the basic authentication dialog pops up.  One way to get around this is to embed the user credentials in the URL.  Traditionally this has worked well, but newer browsers have disabled that functionality due to the security risks inherently involved.  Fortunately, there is another way: send over the authentication credentials in the request header.

Selenium as a Proxy Server
--------------------------

In order to customize the request headers in Selenium you must be using it as a proxy server.  Unless you use the `-avoidProxy` argument when you start up the Selenium server, it will serve as a proxy server bound to localhost (i.e., it is not an open proxy).  You also must use a browser launcher that configures the browser to use the Selenium server as an HTTP proxy.  I discuss this in more depth in my article on [how to accept self-signed SSL certificates in Selenium](http://mogotest.com/blog/2010/04/13/how-to-accept-self-signed-ssl-certificates-in-selenium).

It is a best practice to only perform basic authentication over HTTPS if you're accessing publicly accessible pages.  If all your work is on an internal network and it's a trusted environment, basic authentication over HTTP may be acceptable.  Just realize that basic authentication is plain text and easily sniffable.  If you're connecting to a server with a self-signed certificate you may want to read the aforementioned article on how to do so with Selenium.

Basic Authentication with Selenium
----------------------------------

It probably seems obvious, but you really need to make sure your credentials work.  If they don't, the browser is going to pop up a basic authentication dialog and your Selenium browser session is going to be effectively locked-up.  An example of how to do so in Ruby can be seen in Example 1.

{% highlight ruby %}
require 'rubygems'
require 'net/http'

# Check credentials via SSL.
http = Net::HTTP.new('www.example.com', 443)
http.use_ssl = true

# Make the request with the basic authentication credentials.
req = Net::HTTP::Get.new('/')
req.basic_auth('my_user', 'my_password')
response = http.request(req)

# 401 is the HTTP Unauthorized status code.
if response.code == 401
  puts "Basic authentication failed"
  exit(-1)
else
  log_in_with_selenium()
end  
{% endhighlight %}
<div class='caption'>Example 1: Ensuring your basic authentication credentials work (in Ruby).</div>

Once you've verified that your basic authentication credentials do in fact work, you can proceed to add the credentials to your Selenium request.  You do so by adding the "Authorization" header with a value of "Basic " followed by the base 64 encoded "username:password" pair.  The browser will cache this value for all requests within the same session, so you only need to authenticate on the first request.  Example 2 shows how this looks in Ruby using the selenium-client gem.

{% highlight ruby %}
require 'rubygems'
require 'base64'
require 'selenium/client'

# Note that the start URL is not on the domain that we need to authorize for.
# If you set the start URL there, you may see the authorization dialog before
# you're able to modify the request.
Selenium::Client::Driver.new \
        :host => 'localhost',
        :port => 4444,
        :browser => '*googlechrome',
        :url => 'http://localhost:4444',
        :timeout_in_seconds => 180,
        :highlight_located_element => false

# Add in the basic authentication header.
encoded = Base64.encode64("#{my_username}:#{my_password}").strip
browser.remote_control_command('addCustomRequestHeader',
                              ['Authorization', "Basic #{encoded}"])

# Request the page, fully authorized.
browser.open('http://www.example.com/')
{% endhighlight %}
<div class='caption'>Example 2: Adding basic authentication credentials to Selenium request (in Ruby).</div>

Conclusion
----------

Since basic authentication credentials can be supplied as an HTTP request header, you can perform basic authentication with Selenium with a little bit of work.  You need to use the Selenium server as a proxy for your browser requests, configure acceptance of self-signed SSL certificates if appropriate, and verify that the credentials work before issuing your request.  Once you do so, you'll be able to test sites without worrying about the browser locking up on the authentication dialog.  Note that this article only addresses basic authentication (the most popular of the HTTP authentication methods).  Digest authentication is a completely different beast and the method laid out in this article will not work for it.
