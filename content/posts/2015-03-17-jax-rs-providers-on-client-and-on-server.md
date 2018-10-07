---
title: JAX-RS Providers on Client and on Server
author: Michal Gajdoš
type: post
date: 2015-03-17T09:15:06+00:00
url: /jax-rs-providers-on-client-and-on-server/
aliases: /2015/03/17/jax-rs-providers-on-client-and-on-server/
categories:
  - Jersey
tags:
  - jax-rs
  - jersey
  - hk2

---
In this article I&#8217;d like to explain two things. First, what to do when you want to restrict JAX-RS providers to be used on, for example, client-side only. And second, what are the issues (and how to solve them) with injecting providers when you create and register instances of them directly.

<!--more-->

## Constraining JAX-RS providers to particular runtime

Some of the JAX-RS providers can be used on the server-side as well as on the client-side. The reusability of providers was among the goals when JAX-RS 2.0 Client API was proposed and designed. Unquestionably this being a useful feature of JAX-RS 2.0 there are cases in which you want to constrain some of the available providers to either client or server. Constraining providers, or <a href="https://jax-rs.github.io/apidocs/2.0.1/javax/ws/rs/core/Feature.html">Features</a> for that matter, is even more useful when writing a general purpose library that aims to support both sides but each in a slightly different manner.

Creators of JAX-RS 2.0 thought about needs like this and came up with two possible solutions:

### Declarative – @ConstrainedTo

<a href="https://jax-rs.github.io/apidocs/2.0.1/javax/ws/rs/ConstrainedTo.html">@ConstrainedTo</a> annotation can be placed on any JAX-RS provider type (be it interceptor, filter, message body provider or feature) and in Jersey you can place it on your custom providers annotated with <a href="https://jersey.github.io/apidocs/latest/jersey/org/glassfish/jersey/spi/Contract.html">@Contract</a> and registered within your application too. Annotated provider is restricted to be picked-up and used only in a specified run-time context, defined by <a href="https://jax-rs.github.io/apidocs/2.0.1/javax/ws/rs/RuntimeType.html">RuntimeType</a> enum value in the annotation.

For example, the <a href="http://en.wikipedia.org/wiki/Server-sent_events">Server-Sent Events</a> in Jersey are supported on both, client and server. All you need to do is registering <a href="https://jersey.github.io/apidocs/latest/jersey/org/glassfish/jersey/media/sse/SseFeature.html">SseFeature</a> and you&#8217;re good to go.

Usually, SSE in Jersey works like this: You create an <a href="https://jersey.github.io/apidocs/latest/jersey/org/glassfish/jersey/media/sse/OutboundEvent.html">OutboundEvent</a> in your server application and broadcast the message to all connected clients. On the client you receive an <a href="https://jersey.github.io/apidocs/latest/jersey/org/glassfish/jersey/media/sse/InboundEvent.html">InboundEvent</a> and read the data. As you may guess, the SSE extension module contains message body writer used on the server and message body reader for the client. So, to make sure the reader is not accidentally registered on server we annotate the provider as follows and that&#8217;s it:

{{< highlight java "hl_lines=1" >}}
@ConstrainedTo(RuntimeType.CLIENT)
class InboundEventReader implements MessageBodyReader<InboundEvent> {

    // ...
}
{{< / highlight >}}

### Programmatic – RuntimeType

Registering JAX-RS providers for a specified run-time programmatically is mainly used in <a href="https://jax-rs.github.io/apidocs/2.0.1/javax/ws/rs/core/Feature.html">Feature</a>s. Basically you obtain the current <a href="https://jax-rs.github.io/apidocs/2.0.1/javax/ws/rs/RuntimeType.html">RuntimeType</a> from <a href="https://jax-rs.github.io/apidocs/2.0.1/javax/ws/rs/core/Configuration.html">Configuration</a> and act upon it. In case of the mentioned _SseFeature_ the _configure_ method looks like:

{{< highlight java "hl_lines=8-9 13" >}}
public class SseFeature implements Feature {

    public boolean configure(final FeatureContext context) {
        if (context.getConfiguration().isEnabled(this.getClass())) {
            return false;
        }

        switch (context.getConfiguration().getRuntimeType()) {
            case CLIENT:
                context.register(EventInputReader.class);
                context.register(InboundEventReader.class);
                break;
            case SERVER:
                context.register(OutboundEventWriter.class);
                break;
        }
        return true;
    }
}
{{< / highlight >}}

The code above does the same thing as placing _@ConstrainedTo_ annotation on every provider listed in the switch block.

However, you&#8217;re able to determine the current run-time everywhere where you have access to _Configuration_. Imagine your provider behaves differently on client than it does on server. Also let&#8217;s assume you decide this difference is not that a big of an issue that would force you to create separate providers for both run-times. In this case you can take advantage of _RuntimeType_ (and during runtime), e.g.:

{{< highlight java "hl_lines=7-8" >}}
class MyWriterInterceptor implements WriterInterceptor {

    @Override
    public void aroundWriteTo(final WriterInterceptorContext context) throws IOException {
        // do something

        if (RuntimeType.SERVER
              == context.getConfiguration().getRuntimeType) {
            // do something more
        }

        context.proceed();
    }
}
{{< / highlight >}}

## How injection works for JAX-RS provider instances in Jersey

On server, you don&#8217;t need to do anything special to properly inject instances and classes of JAX-RS providers and resources (and basically everything managed by HK2 or CDI). Injection works as you would expect. There is no problem to inject anything because of the fact that there is only one server run-time (and only one [ServiceLocator][1]) for each application.

On client, you can still inject classes of JAX-RS providers without noticing a difference. However, the situation with provider instances is a little bit different.

Imagine you have a client request filter that you want to initialize with some data and inject it with an instance of _MyInjectedService_:

{{< highlight java "hl_lines=5 7-8 16 18" >}}
class MyLoggingFilter implements ClientRequestFilter {

    private static final Logger LOGGER = Logger.getLogger(MyLoggingFilter.class.getName());

    private final String loggingPrefix;

    @Inject
    private MyInjectedService service;

    public MyLoggingFilter(final String prefix) {
        this.loggingPrefix = prefix;
    }

    public void filter(final ClientRequestContext context) throws IOException {

        final String name = service.getName();

        LOGGER.info(prefix + name);

        // ...
    }
}
{{< / highlight >}}

Now, you create a JAX-RS client, register your provider with the client and create two web targets:

{{< highlight java "hl_lines=2 6" >}}
final Client client = ClientBuilder.newClient()
    .register(new MyLoggingFilter("JAX-RS Client #" + (i++) + ":"));

final WebTarget target1 = client.target("http://example.com");
final WebTarget target2 = client.target("http://example.com/json")
    .register(JacksonFeature.class);

// Throws NPE.
target1.request().get();
// Throws NPE as well.
target2.request().get();
{{< / highlight >}}

The second web target requires also _JacksonFeature_ to be able to work with JSON. When you try to invoke a request from either one of these web targets the _NullPointerException_ would be thrown. _Why?_

Well, _service_ field in _MyLoggingFilter_ would be _null_ because the filter instance doesn&#8217;t get injected. The reason behind this is that you (possibly) have multiple client run-times which means you have multiple HK2&#8217;s _ServiceLocator_s. Jersey, of course, knows which one should be used but when you think about it we cannot inject the filter. Every web-target (client runtime) has reference to the same instance of _MyLoggingFilter_ (it&#8217;s a singleton). In addition, provides ought to be thread-safe so you cannot inject the filter using one service locator and hope that the filter wouldn&#8217;t be, at the same time, used from another thread (from a different web-target). It just wouldn&#8217;t work &#8217;cause you&#8217;d be using wrongly injected service.

To be fair, this would work in the case if Jersey injected a dynamic proxy into _service_ field but we don&#8217;t do that. The reason – it&#8217;d be pretty slow.

There is another way to solve the injection introduced in Jersey 2.6. Use <a href="https://jersey.github.io/apidocs/latest/jersey/org/glassfish/jersey/client/ServiceLocatorClientProvider.html">ServiceLocatorClientProvider</a> to extract <a href="https://hk2.java.net/apidocs/org/glassfish/hk2/api/ServiceLocator.html">ServiceLocator</a> from context in any JAX-RS provider and with locator you can obtain anything previously registered there. The following example shows how to utilize _ServiceLocatorClientProvider_ in the rewritten version of _MyLoggingFilter_:

{{< highlight java "hl_lines=14-15 18" >}}
class MyLoggingFilter implements ClientRequestFilter {

    private static final Logger LOGGER = Logger.getLogger(MyLoggingFilter.class.getName());

    private final String loggingPrefix;

    public MyLoggingFilter(final String prefix) {
        this.loggingPrefix = prefix;
    }

    public void filter(final ClientRequestContext context) throws IOException {
        // use ServiceLocatorClientProvider to extract HK2 ServiceLocator
        // from request
        ServiceLocator locator = ServiceLocatorClientProvider
                                     .getServiceLocator(requestContext);
 
        // and ask for MyInjectedService:
        MyInjectedService service = locator.getService(MyInjectedService.class);

        final String name = service.getName();

        LOGGER.info(prefix + name);

        // ...
    }
}
{{< / highlight >}}

 [1]: https://hk2.java.net/2.4.0-b12/apidocs/org/glassfish/hk2/api/ServiceLocator.html