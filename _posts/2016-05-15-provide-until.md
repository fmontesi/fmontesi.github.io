---
layout: post
title: Provide-until now provided in Jolie
tags:
- tutorials
- microservices
- jolie
---

A pattern that arises often when writing a (micro)service is that of a workflow that _provides_ access to a resource _until_ something special happens. The new provide-until primitive in [Jolie](http://www.jolie-lang.org/) supports exactly this pattern. It doesn't add any expressivity to the language (we could do this before using bookkeeping variables, just like in any other language), but it makes the code pleasant to read in this kind of situations that come up so frequently.

Here's an example of a service that handles shopping carts. We can start a process in our service by invoking operation `createCart`. We then set a timeout (given by the constant `TIMEOUT`, omitted here). When the timeout expires, the `Time` service (from the Jolie standard library) is going to call us on operation `timeout`.
Now we are all set and we enter a `provide-until` block. Provide-until takes two input choices. The reasoning is very simple: the operations in the first input choice are provided (thus can be invoked multiple times) until any of the operations in the until block is called. Below, we can invoke the operations `add` and `remove` to add and remove, respectively, items in our shopping cart, until one of the following happens: the process timeouts (operation `timeout` is invoked); the user decides to delete the cart (operation `delete`); or, the user decide to proceed to checkout (operation `checkout`).

<pre>
<code class="language-jolie">/* Interfaces and ports ... */

cset { id: Add.id Remove.id Timeout.id Delete.id Checkout.id }

main
{
  createCart( request )( cart ) {
    cart.id = csets.id = new;
    cart.user = request.user
  };

  setNextTimeout@Time( TIMEOUT { .message.id = cart.id } );

  provide
    [ add( request )( cart ) {
      cart.items[ #cart.items ] << request.item
    } ]

    [ remove( request )( cart ) {
      undef( cart.items[ request.which ] )
    } ]

  until
    [ timeout() ]

    [ delete()() ]

    [ checkout( paymentInfo )() {
      // ...
    } ]
}</code></pre>

Provide-until is a little thing that I enjoyed developing lately. It originally comes from the desire of handling elegantly REST processes in Jolie, but it works just as well in any similar situation. I give a longer explanation for researchers of how Jolie works with the Web in this paper: [https://arxiv.org/abs/1410.3712](https://arxiv.org/abs/1410.3712).
