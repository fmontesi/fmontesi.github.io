---
layout: post
title: Service Decorators
tags:
- tutorials
- microservices
- jolie
---

*TL;DR*: decorators for services are pretty easy to write in [Jolie](http://www.jolie-lang.org/),
thanks to a native feature called [aggregation](http://docs.jolie-lang.org/#!documentation/architectural_composition/aggregation.html).
Be careful in the deployment phase for managing performance hits!

# Service Decorators

Just read [this article about using decorators](https://dzone.com/articles/is-inheritance-dead) over inheritance in
object-oriented programs. I assume that you know decorators in the following. If
you don't, get informed (e.g., by reading that article) before proceeding.

The Decorator design pattern offers a way to cleanly modify the behaviour of an
object by using composition instead of inheritance. I won't enter in the merits
of composition against inheritance in object-orientation (both have their own,
depending on the other features of the language), because I'm interested in
microservices here. In microservices, composition is typically the only option
you have. That's because your microservices may be written in different
languages, or even paradigms. And even if their codebases were somehow
compatible, inheritance would still be out of place: your microservices can only
depend on the communication APIs of other services, not on their implementation
details (e.g., by looking at their source code).

Since decorator acts through composition, this pattern is potentially
interesting for service programming. But as the author of the article linked above mentions,
using it can be frustrating since it requires a lot of boilerplate code. This makes it also
error-prone.
As a running example, let's look at the e-mail service interface proposed in that article, rewritten in Jolie:
<pre>
<code class="language-jolie">interface EmailServiceIface {
RequestResponse:
  send(Email)(void),
  listEmails(Range)(EmailInfoList),
  downloadEmail(EmailInfo)(Email)
}</code></pre>


## Decorating an operation

Now, suppose that this service is available to us as `EmailService`. Assume also
that we want to write a new service that decorates `EmailService` with the
following logic: whenever `send` is called, we check if we have sent an
"important" e-mail (e.g., the subject contains a specific keyword telling us
that the e-mail is important, or the addressee is in a special list); if an
important e-mail has been sent successfully, we backup its content by calling
another service, for indexing and safe-keeping.

A naive implementation is to write the decorator by re-implementing interface `EmailService`, like this:
<pre>
<code class="language-jolie">inputPort EmailServiceImportant {
Location: MyLocation Protocol: MyProtocol
Interfaces: EmailServiceIface
}

main
{
  [ send( request )( response ) {
    send@EmailService( request )();
    check@Important( { .subject = request.subject, .to = request.to } )( important );
    if ( important ) {
      backup@BackupService( request )()
    }
  } ]

  [ listEmails( request )( response ) {
    listEmails@EmailService( request )( response )
  } ]

  [ downloadEmail( request )( response ) {
    downloadEmail@EmailService( request )( response )
  } ]
}</code></pre>

The code for `listEmails` and `downloadEmail` is boilerplate, since we're just
forwarding requests and responses. The author of the [article](https://dzone.com/articles/is-inheritance-dead)
suggests that it would be nice if languages supported a native feature that makes
writing this code unnecessary. Luckily, we have it in this case!

Forwarding is a native feature in Jolie, since building proxies is the bread and
butter of service composition (think of load balancers, caches, etc.).
We can rewrite our decorator like this:
<pre>
<code class="language-jolie">inputPort EmailServiceImportant {
Location: MyLocation Protocol: MyProtocol
Aggregates: EmailService // Aggregates instead of Interfaces!
}

main
{
  [ send( request )( response ) {
    send@EmailService( request )();
    check@Important( { .subject = request.subject, .to = request.to } )( important );
    if ( important ) {
      backup@BackupService( request )()
    }
  } ]
}</code></pre>

No boilerplate, same behaviour as our previous decorator!


## Interface Decoration

What if you want to change the behaviours of _many_ operations, not just one?
What if this behaviour change is always the same? For example, suppose that we want
to write a decorator that keeps track of all events: whenever an operation is called,
we write this in an external log.

Here's a naive implementation:
<pre>
<code class="language-jolie">inputPort EmailServiceLogger {
Location: MyLocation Protocol: MyProtocol
Interfaces: EmailServiceIface
}

main
{
  [ send( request )( response ) {
    send@EmailService( request )();
    log@Logger( request )()
  } ]

  [ listEmails( request )( response ) {
    listEmails@EmailService( request )( response );
    log@Logger( request )()
  } ]

  [ downloadEmail( request )( response ) {
    downloadEmail@EmailService( request )( response );
    log@Logger( request )()
  } ]
}</code></pre>

Ouch, boilerplate again! What we need is the capability of writing that logging
code just once, and applying it to the entire API of `EmailService`. That's what
[courier processes](http://docs.jolie-lang.org/#!documentation/architectural_composition/couriers.html)
in Jolie are for. Here's the improved code, using `courier`:
<pre>
<code class="language-jolie">inputPort EmailServiceLogger {
Location: MyLocation Protocol: MyProtocol
Aggregates: EmailService // Aggregates again
}

courier EmailServiceLogger // courier enables primitives for whole-interface behaviour
{
  [ interface EmailServiceIface( request )( response ) ] { // Apply to all operations in the interface
    forward( request )( response ); // forward is a primitive: it forwards the message to the aggreated (in this case, decorated) service
    log@Logger( request )()
  }
}</code></pre>

No boilerplate again!


## Circuit breakers

What's a cool example of a decorator? Circuit breaker! Yup. A sketch in Jolie using aggregation can be found
[in this paper](https://arxiv.org/abs/1609.05830).

## Conclusions

As cool as decorators are, don't forget that you are adding a layer of
indirection. Many times, you don't have a choice and your code benefits so much
that you should pay the price. But in microservices, be *very* careful about how
efficient your extra layer will be. If you stack too many decorators and they
are all communicating remotely via sockets, you'll soon have a performance
problem. This is a deployment challenge: in Jolie, using a different
communication medium doesn't alter your behavioural code. So choose your
communication media wisely when you deploy! If you have control over your stack
of decorators, consider whether it would be better to put them all in one place and
have them communicate using shared-memory (`local` location in Jolie).
