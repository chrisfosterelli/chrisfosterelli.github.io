---
layout: post
permalink: dangerous-use-of-express-body-parser
title: "Dangerous use of express body-parser"
---

[Cross-Site Request Forgery](https://www.owasp.org/index.php/Cross-Site_Request_Forgery_(CSRF)), or CSRF, is a type of attack that developers are familiar with in traditional web applications, but often misunderstand or forget about when it comes to new REST API's. Fortunately, much of this misunderstanding and lack of consideration occurs because full page applications often _don't need_ to worry about CSRF. While many architectural differences in REST reduce the risk of CSRF attacks, that doesn't mean we can simply ignore them entirely. [Express's body-parser](https://www.npmjs.com/package/body-parser) module is a great recent example of this.

<!-- Content Breaker -->

## When you don't need CSRF protection

With Javascript XHR requests, it's not possible to make cross-origin requests the same way it is with forms due to the [same-origin policy](https://en.wikipedia.org/wiki/Same-origin_policy). If you attempt to make an AJAX request to another origin, the browser will disallow the request. HTML forms are not capable of generating anything other than form data, so they can't talk to REST backends. Therefore since only XHR's can send JSON data to an API, an API that accepts only JSON is protected from most CSRF attacks.

<img src="/image/dangerous-use-of-express-body-parser/csrf.png" style="margin-left: 0; margin-right: 0; width: 100%;" alt="CSRF Diagram">

## When you do need CSRF protection

So how does all of this relate to express's body-parser module? If you're using Nodejs and express, there is a very high chance that your building a server side API. The de facto way to parse JSON in express is body-parser, and the code [they](https://github.com/expressjs/body-parser#expressconnect-top-level-generic) [always](http://expressjs.com/en/api.html#req.body) [recommend](http://stackoverflow.com/a/24330353/372767) [using](https://codeforgeek.com/2014/09/handle-get-post-request-express-4/) is:

{% highlight javascript %}
var bodyParser = require('body-parser');
app.use(bodyParser.json());
app.use(bodyParser.urlencoded({
    extended: true
}));
{% endhighlight %}

The problem with the above code is that if you use it, **you need CSRF protection**.  In your route middleware, body-parser will map JSON data to the `req.body` object present in each request. However, when you call both `bodyParser.json()` and `bodyParser.urlencoded()` you are populating `req.body` with content from *both* JSON sources and form-encoded sources. Your API route handlers cannot tell the difference between what Content-Type populated the API.

This means that your API route could be called from either a Javascript XHR request _or_ a traditional HTML form. If your application uses cookies, then that means that an attacker could create a malicious HTML form with form-encoded data that submits to your API, triggering whatever operation they'd like with the permissions of the currently logged in user. This is your traditional CSRF attack, but made possible because you parse more data types than you realize.

## Making it safe 

If your app has this, there are multiple ways you can fix this:

* Use traditional CSRF tokens for all requests
* Remove the `bodyParser.urlencoded` middleware from API routes
* Check the `Content-Type` header of the request

Personally, I'd recommend removing the `bodyParser.urlencoded` middleware, since it's the most straightforward in my opinion. If you parse any sort of traditional HTML form, such as for user login, then you will need to be add the `bodyParser.urlencoded` middleware support on _that route_, but keep it off all the others! The documentation [covers how to do this](https://github.com/expressjs/body-parser#express-route-specific).

## Beware example code

In summary, this isn't an actual vulnerability or something that is wrong with body-parser. The author provided defaults that would cover the use case for 99% of the middleware's users. This is a configuration-based insecurity, and is great example of how even simple example code can lead to a vulnerability in your own application. Don't trust code blindly, be sure to understand what you're putting in your application and how it affects your security.

This is a side effect of my recent messing around with CSRF. If you'd like to see more articles like this, [follow me on Twitter](https://twitter.com/chrisfosterelli/).
