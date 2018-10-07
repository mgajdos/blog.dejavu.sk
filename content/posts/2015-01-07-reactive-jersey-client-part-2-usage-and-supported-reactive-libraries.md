---
title: Reactive Jersey Client, Part 2 – Usage and Supported Libraries
author: Michal Gajdoš
type: post
date: 2015-01-07T08:10:43+00:00
url: /reactive-jersey-client-part-2-usage-and-supported-reactive-libraries/
aliases: /2015/01/07/reactive-jersey-client-part-2-usage-and-supported-reactive-libraries/
categories:
  - Jersey
tags:
  - jersey
  - client
  - reactive

---
Reactive Jersey Client API is a generic API allowing end users to utilize the popular reactive programming model when using Jersey Client. This part of the series describes API, usage and lists all supported reactive libraries.

<!--more-->

  * [Reactive Jersey Client – Motivation][1]
  * Reactive Jersey Client – Usage and Supported Reactive Libraries _(this article)_
  * [Reactive Jersey Client – Customization (SPI)][2]

## Usage

Reactive Jersey Client API tries to bring a similar experience you have with the existing JAX-RS Client API. It builds on it with extending these JAX-RS APIs with a few new methods.

When you compare synchronous invocation of HTTP calls:

{{< highlight java >}}
Response response = ClientBuilder.newClient()
        .target("http://example.com/resource")
        .request()
        .get();
{{< / highlight >}}

with asynchronous invocation:

{{< highlight java >}}
 response = ClientBuilder.newClient()
        .target("http://example.com/resource")
        .request()
        .async()
        .get();
{{< / highlight >}}

it is apparent how to pretty conveniently modify the way how a request is invoked (from sync to async) only by calling <code class="literal">async</code> method on an <a class="link" href="http://jax-rs-spec.java.net/nonav/2.0/apidocs/javax/ws/rs/client/Invocation.Builder.html">Invocation.Builder</a>.

Naturally, it&#8217;d be nice to copy the same pattern to allow invoking requests in a reactive way. Just instead of <code class="literal">async</code> you&#8217;d call <code class="literal">rx</code> on an extension of <code class="literal">Invocation.Builder</code>, like

{{< highlight java >}}
 response = Rx.newClient(RxObservableInvoker.class)
        .target("http://example.com/resource")
        .request()
        .rx()
        .get();
{{< / highlight >}}

To achieve this a few new interfaces had to be introduced in the Reactive Jersey Client API. The first new interface is <a class="link" href="https://jersey.github.io/apidocs/latest/jersey/org/glassfish/jersey/client/rx/RxInvoker.html">RxInvoker</a> which is very similar to <a class="link" href="http://jax-rs-spec.java.net/nonav/2.0/apidocs/javax/ws/rs/client/SyncInvoker.html">SyncInvoker</a> and <a class="link" href="http://jax-rs-spec.java.net/nonav/2.0/apidocs/javax/ws/rs/client/AsyncInvoker.html">AsyncInvoker</a>. It contains all methods present in the two latter JAX-RS interfaces but the <code class="literal">RxInvoker</code> interface is more generic, so that it can be extended and used in particular implementations taking advantage of various reactive libraries. Extending this new interface in a particular implementation also preserves type safety which means that you&#8217;re not loosing type information when a HTTP method call returns an object that you want to process further.

As a user of the Reactive Jersey Client API you only need to keep in mind that you won&#8217;t be working with <code class="literal">RxInvoker</code> directly. You&#8217;d rather be working with an extension of this interface created for a particular implementation and you don&#8217;t need to be bothered much with why are things designed the way they are.

The important thing to notice here is that an extension of <code class="literal">RxInvoker</code> holds the type information and the Reactive Jersey Client needs to know about this type to properly propagate it among the method calls you&#8217;ll be making. This is the reason why other interfaces (described bellow) are parametrized with this type.

In addition to having a concrete <code class="literal">RxInvoker</code> implementation ready there is also a need to have an implementation of new reactive methods, <code class="literal">rx()</code> and <code class="literal">rx(ExecutorService)</code>. They&#8217;re defined in <a class="link" href="https://jersey.github.io/apidocs/latest/jersey/org/glassfish/jersey/client/rx/RxInvocationBuilder.html">RxInvocationBuilder</a> which extends the <a class="link" href="http://jax-rs-spec.java.net/nonav/2.0/apidocs/javax/ws/rs/client/Invocation.Builder.html">Invocation.Builder</a> from JAX-RS. Using the first method you can simply access the reactive request invocation interface to invoke the built request and the second allows you to specify the executor service to execute the current reactive request (and only this one).

To access the <code class="literal">RxInvocationBuilder</code> we needed to also extend JAX-RS <code class="literal">Client</code> (<a class="link" href="https://jersey.github.io/apidocs/latest/jersey/org/glassfish/jersey/client/rx/RxClient.html">RxClient</a>) and <code class="literal">WebTarget</code> (<a class="link" href="https://jersey.github.io/apidocs/latest/jersey/org/glassfish/jersey/client/rx/RxWebTarget.html">RxWebTarget</a>) to preserve the fluent Client API introduced in JAX-RS.

With all these interfaces ready the only question left behind is the way how to create an instance of Reactive Jersey Client. This functionality is beyond the actual JAX-RS API. It is not possible to create such a client via the standard <code class="literal">ClientBuilder</code> entry point. To resolve this, we introduced a new helper class, <a class="link" href="https://jersey.github.io/apidocs/latest/jersey/org/glassfish/jersey/client/rx/Rx.html">Rx</a>, which does the job. This class contains factory methods to create a new (reactive) client from scratch

  * <a class="link" href="https://jersey.github.io/apidocs/latest/jersey/org/glassfish/jersey/client/rx/Rx.html#newClient(java.lang.Class)">Rx.newClient(Class)</a>
  * <a class="link" href="https://jersey.github.io/apidocs/latest/jersey/org/glassfish/jersey/client/rx/Rx.html#newClient(java.lang.Class, java.util.concurrent.ExecutorService)">Rx.newClient(Class,ExecutorService)</a>

and it also contains methods to enhance an existing JAX-RS <code class="literal">Client</code> and <code class="literal">WebTarget</code>

  * <a class="link" href="https://jersey.github.io/apidocs/snapshot/jersey/org/glassfish/jersey/client/rx/Rx.html#from(javax.ws.rs.client.Client, java.lang.Class)">Rx.from(Client,Class)</a>
  * <a class="link" href="https://jersey.github.io/apidocs/snapshot/jersey/org/glassfish/jersey/client/rx/Rx.html#from(javax.ws.rs.client.Client, java.lang.Class, java.util.concurrent.ExecutorService)">Rx.from(Client,Class,ExecutorService)</a>
  * <a class="link" href="https://jersey.github.io/apidocs/snapshot/jersey/org/glassfish/jersey/client/rx/Rx.html#from(javax.ws.rs.client.WebTarget, java.lang.Class)">Rx.from(WebTarget,Class)</a>
  * <a class="link" href="https://jersey.github.io/apidocs/snapshot/jersey/org/glassfish/jersey/client/rx/Rx.html#from(javax.ws.rs.client.WebTarget, java.lang.Class, java.util.concurrent.ExecutorService)">Rx.from(WebTarget,Class,ExecutorService)</a>

It&#8217;s possible to provide an <a class="link" href="http://docs.oracle.com/javase/6/docs/api/java/util/concurrent/ExecutorService.html">ExecutorService</a> instance to tell the reactive client that all requests should be invoked using this particular executor. This behaviour can be suppressed by providing another <code class="literal">ExecutorService</code> instance for a particular request.

Similarly to the <code class="literal">RxInvoker</code> interface the <code class="literal">Rx</code> class is general and does not stick to any conrete implementation. When Reactive Clients are created using <code class="literal">Rx</code> factory methods, the actual invoker type parameter has to be provided (this is not the case with similar helper classes created for particular reactive libraries).

## Supported Reactive Libraries

### RxJava – Observable

<a class="link" href="https://github.com/ReactiveX/RxJava">RxJava</a>, contributed by Netflix, is probably the most advanced reactive library for Java at the moment. It&#8217;s used for composing asynchronous and event-based programs by using observable sequences. It uses the <a class="link" href="http://en.wikipedia.org/wiki/Observer_pattern">observer pattern</a> to support these sequences of data/events via it&#8217;s <a class="link" href="http://reactivex.io/RxJava/javadoc//rx/Observable.html">Observable</a> entry point class which implements the Reactive Pattern. <code class="literal">Observable</code> is actually the parameter type in the RxJava&#8217;s extension of <code class="literal">RxInvoker</code>, called <a class="link" href="https://jersey.github.io/apidocs/latest/jersey/org/glassfish/jersey/client/rx/rxjava/RxObservableInvoker.html">RxObservableInvoker</a>.

Requests are by default invoked at the moment when a subscriber is subscribed to an observable (it&#8217;s a cold <code class="literal">Observable</code>). If not said otherwise a separate thread (JAX-RS Async Client requests) is used to obtain data. This behavior can be overridden by providing an <a class="link" href="http://docs.oracle.com/javase/6/docs/api/java/util/concurrent/ExecutorService.html">ExecutorService</a> when a reactive <code class="literal">Client</code> or <code class="literal">WebTarget</code> is created or when a particular requests is about to be invoked.

To create a <code class="literal">Client</code> or <code class="literal">WebTarget</code> aware of reactive HTTP calls we can either use the more generic method

{{< highlight java >}}
// New Client
RxClient<RxObservableInvoker> newRxClient = Rx.newClient(RxObservableInvoker.class);
 
// From existing Client
RxClient<RxObservableInvoker> rxClient = Rx.from(client, RxObservableInvoker.class);
 
// From existing WebTarget
RxTarget<RxObservableInvoker> rxWebTarget = Rx.from(target, RxObservableInvoker.class);
{{< / highlight >}}

or  <a class="link" href="https://jersey.github.io/apidocs/latest/jersey/org/glassfish/jersey/client/rx/rxjava/RxObservable.html">RxObservable</a> helper class

{{< highlight java >}}
// New Client
RxClient<RxObservableInvoker> newRxClient = RxObservable.newClient();
 
// From existing Client
RxClient<RxObservableInvoker> rxClient = RxObservable.from(client);
 
// From existing WebTarget
RxTarget<RxObservableInvoker> rxWebTarget = RxObservable.from(target);
{{< / highlight >}}

In addition to specifying the invoker type and client/web-target instances, when using the factory methods in the entry points mentioned above, an <code class="literal">ExecutorService</code> can be specified that will be used to execute requests on separate threads. In the case of RxJava the executor service is utilized to create a <a class="link" href="http://reactivex.io/RxJava/javadoc//rx/Scheduler.html">Scheduler</a> that is later leveraged in both <a class="link" href="http://reactivex.io/RxJava/javadoc//rx/Observable.html#observeOn(rx.Scheduler)">Observable#observeOn(rx.Scheduler)</a> and <a class="link" href="http://reactivex.io/RxJava/javadoc//rx/Observable.html#subscribeOn(rx.Scheduler)">Observable#subscribeOn(rx.Scheduler)</a>.

To put it in context an example of obtaining <code class="literal">Observable</code> with JAX-RS <code class="literal">Response</code> from a remote service may look like

{{< highlight java >}}
 observable = RxObservable.newClient()
        .target("http://example.com/resource")
        .request()
        .rx()
        .get();
{{< / highlight >}}

> More complex example, implementation of the problem described in the first part, can be found in Jersey workspace, e.g. on GitHub –  <a href="https://github.com/jersey/jersey/blob/master/examples/rx-client-java8-webapp/src/main/java/org/glassfish/jersey/examples/rx/agent/ObservableAgentResource.java#L97">ObservableAgentResource</a> class in <a href="https://github.com/jersey/jersey/tree/master/examples/rx-client-java8-webapp">rx-client-java8-webapp</a>.

To obtain the extension module containing Reactive Jersey Client with RxJava support look at the coordinates: <a href="http://search.maven.org/#search%7Cga%7C1%7C%22org.glassfish.jersey.ext.rx%3Ajersey-rx-client-rxjava%22">org.glassfish.jersey.ext.rx:jersey-rx-client-rxjava</a>.

### <a name="java8"></a>Java 8 – CompletionStage and CompletableFuture

Java 8 natively contains asynchronous/event-based completion aware types, <a class="link" href="http://docs.oracle.com/javase/8/docs/api/java/util/concurrent/CompletionStage.html">CompletionStage</a> and <a class="link" href="http://docs.oracle.com/javase/8/docs/api/java/util/concurrent/CompletableFuture.html">CompletableFuture</a>. These types can be then combined with <a class="link" href="http://docs.oracle.com/javase/8/docs/api/java/util/stream/Stream.html">Stream</a>s to achieve similar functionality as provided by RxJava. <code class="literal">CompletionStage</code> is the parameter type in the Java 8 extension of <code class="literal">RxInvoker</code>, called <a class="link" href="https://jersey.github.io/apidocs/latest/jersey/org/glassfish/jersey/client/rx/java8/RxCompletionStageInvoker.html">RxCompletionStageInvoker</a>.

Requests are by default invoked immediately. If not said otherwise the <a class="link" href="http://docs.oracle.com/javase/8/docs/api/java/util/concurrent/ForkJoinPool.html#commonPool--">ForkJoinPool#commonPool()</a> pool is used to obtain a thread which processed the request. This behavior can be overridden by providing an <a class="link" href="http://docs.oracle.com/javase/6/docs/api/java/util/concurrent/ExecutorService.html">ExecutorService</a> when a reactive <code class="literal">Client</code> or <code class="literal">WebTarget</code> is created or when a particular request is about to be invoked.

> To use this module the application has to be compiled (with <code class="literal">javac</code> <code class="literal">-target</code> option set to <code class="literal">1.8</code>) and run in a Java 8 environment. If you want to use Reactive Jersey Client with <code class="literal">CompletableFuture</code> in pre-Java 8 environment, see [JSR-166e – CompletableFuture][3].

To create a <code class="literal">Client</code> or <code class="literal">WebTarget</code> aware of reactive HTTP calls we can use <a class="link" href="https://jersey.github.io/apidocs/latest/jersey/org/glassfish/jersey/client/rx/java8/RxCompletionStage.html">RxCompletionStage</a> helper class.

{{< highlight java >}}
// New Client
RxClient<RxCompletionStageInvoker> newRxClient = RxCompletionStage.newClient();
 
// From existing Client
RxClient<RxCompletionStageInvoker> rxClient = RxCompletionStage.from(client);
 
// From existing WebTarget
RxTarget<RxCompletionStageInvoker> rxWebTarget = RxCompletionStage.from(target);
{{< / highlight >}}

In addition to specifying the invoker type and client/web-target instances, when using the factory methods in the entry points mentioned above, an <code class="literal">ExecutorService</code> instance could be specifies that should be used to execute requests on a separate thread.

An example of obtaining <code class="literal">CompletionStage</code> with JAX-RS <code class="literal">Response</code> from a remote service may look like

{{< highlight java >}}
 stage = RxCompletionStage.newClient()
        .target("http://example.com/resource")
        .request()
        .rx()
        .get();
{{< / highlight >}}

> More complex example, implementation of the problem described in the first part, can be found in Jersey workspace, e.g. on GitHub –  <a href="https://github.com/jersey/jersey/blob/master/examples/rx-client-java8-webapp/src/main/java/org/glassfish/jersey/examples/rx/agent/CompletionStageAgentResource.java#L100">CompletionStageAgentResource</a> class in <a href="https://github.com/jersey/jersey/tree/master/examples/rx-client-java8-webapp">rx-client-java8-webapp</a>.

To obtain the extension module containing Reactive Jersey Client with Java 8 support look at the coordinates: <a href="http://search.maven.org/#search%7Cga%7C1%7C%22org.glassfish.jersey.ext.rx%3Ajersey-rx-client-java8%22">org.glassfish.jersey.ext.rx:jersey-rx-client-java8</a>.

### Guava – ListenableFuture and Futures {.title}

<a class="link" href="https://code.google.com/p/guava-libraries/">Guava</a>, contributed by Google, also contains a type, <a class="link" href="http://docs.guava-libraries.googlecode.com/git-history/v14.0.1/javadoc/com/google/common/util/concurrent/ListenableFuture.html">ListenableFuture</a>, which can be decorated with listeners that are notified when the future completes. The <code class="literal">ListenableFuture</code> can be combined with <a class="link" href="http://docs.guava-libraries.googlecode.com/git-history/v14.0.1/javadoc/com/google/common/util/concurrent/Futures.html">Futures</a> to achieve asynchronous/event-based completion aware processing. <code class="literal">ListenableFuture</code> is the parameter type in the Guava&#8217;s extension of <code class="literal">RxInvoker</code>, called <a class="link" href="https://jersey.github.io/apidocs/latest/jersey/org/glassfish/jersey/client/rx/guava/RxListenableFutureInvoker.html">RxListenableFutureInvoker</a>.

Requests are by default invoked immediately. If not said otherwise the <a class="link" href="http://docs.oracle.com/javase/8/docs/api/java/util/concurrent/Executors.html#newCachedThreadPool--">Executors#newCachedThreadPool()</a> pool is used to obtain a thread which processed the request. This behavior can be overridden by providing a <a class="link" href="http://docs.oracle.com/javase/6/docs/api/java/util/concurrent/ExecutorService.html">ExecutorService</a> when a reactive <code class="literal">Client</code> or <code class="literal">WebTarget</code> is created or when a particular requests is about to be invoked.

To create a <code class="literal">Client</code> or <code class="literal">WebTarget</code> aware of reactive HTTP calls we can use <a class="link" href="https://jersey.github.io/apidocs/snapshot/jersey/org/glassfish/jersey/client/rx/guava/RxListenableFuture.html">RxListenableFuture</a> helper class.

{{< highlight java >}}
// New Client
RxClient<RxListenableFutureInvoker> newRxClient = RxListenableFuture.newClient();
 
// From existing Client
RxClient<RxListenableFutureInvoker> rxClient = RxListenableFuture.from(client);
 
// From existing WebTarget
RxTarget<RxListenableFutureInvoker> rxWebTarget = RxListenableFuture.from(target);
{{< / highlight >}}

In addition to specifying the invoker type and client/web-target instances, when using the factory methods in the entry points mentioned above, an <code class="literal">ExecutorService</code> can be specified that will be used to execute requests on a separate thread.

An example of obtaining <code class="literal">ListenableFuture</code> with JAX-RS <code class="literal">Response</code> from a remote service may look like

{{< highlight java >}}
 stage = RxListenableFuture.newClient()
        .target("http://example.com/resource")
        .request()
        .rx()
        .get();
{{< / highlight >}}

> More complex example, implementation of the problem described in the first part, can be found in Jersey workspace, e.g. on GitHub –  <a href="https://github.com/jersey/jersey/blob/master/examples/rx-client-java8-webapp/src/main/java/org/glassfish/jersey/examples/rx/agent/ListenableFutureAgentResource.java#L92">ListenableFutureAgentResource</a> class in <a href="https://github.com/jersey/jersey/tree/master/examples/rx-client-java8-webapp">rx-client-java8-webapp</a>.

To obtain the extension module containing Reactive Jersey Client with Guava support look at the coordinates: <a href="http://search.maven.org/#search%7Cga%7C1%7C%22org.glassfish.jersey.ext.rx%3Ajersey-rx-client-guava%22">org.glassfish.jersey.ext.rx:jersey-rx-client-guava</a>.

### <a name="jsr166e"></a>JSR-166e – CompletableFuture

When Java 8 is not an option but the functionality of <a class="link" href="http://docs.oracle.com/javase/8/docs/api/java/util/concurrent/CompletionStage.html">CompletionStage</a> and <a class="link" href="http://docs.oracle.com/javase/8/docs/api/java/util/concurrent/CompletableFuture.html">CompletableFuture</a> is required a <a class="link" href="http://g.oswego.edu/dl/concurrency-interest/">JSR 166</a> library can be used. It&#8217;s a back-port of classes from<code class="literal">java.util.concurrent</code> package added to Java 8. Contributed and maintained by Doug Lea. <code class="literal">CompletableFuture</code> is the parameter type in the JSR-166e&#8217;s extension of<code class="literal">RxInvoker</code>, called <a class="link" href="https://jersey.github.io/apidocs/latest/jersey/org/glassfish/jersey/client/rx/jsr166e/RxCompletableFutureInvoker.html">RxCompletableFutureInvoker</a>.

Requests are by default invoked immediately. If not said otherwise the <a class="link" href="http://gee.cs.oswego.edu/dl/jsr166/dist/jsr166edocs//jsr166e/ForkJoinPool.html#commonPool()">ForkJoinPool.html#commonPool()</a> pool is used to obtain a thread which processed the request. This behavior can be overridden by providing an <a class="link" href="http://docs.oracle.com/javase/6/docs/api/java/util/concurrent/ExecutorService.html">ExecutorService</a> when a reactive <code class="literal">Client</code> or <code class="literal">WebTarget</code> is created or when a particular requests is about to be invoked.

> If you&#8217;re compiling and running your application in Java 8 environment consider use Reactive Jersey Client with [Java 8 – CompletionStage and CompletableFuture][4] support instead.

To create a <code class="literal">Client</code> or <code class="literal">WebTarget</code> aware of reactive HTTP calls we can use <a class="link" href="https://jersey.github.io/apidocs/snapshot/jersey/org/glassfish/jersey/client/rx/jsr166e/RxCompletableFuture.html">RxCompletableFuture</a> helper class.

{{< highlight java >}}
// New Client
RxClient<RxCompletableFutureInvoker> newRxClient = RxCompletableFuture.newClient();
 
// From existing Client
RxClient<RxCompletableFutureInvoker> rxClient = RxCompletableFuture.from(client);
 
// From existing WebTarget
RxTarget<RxCompletableFutureInvoker> rxWebTarget = RxCompletableFuture.from(target);
{{< / highlight >}}

In addition to specifying the invoker type and client/web-target instances, when using the factory methods in the entry points mentioned above, an <code class="literal">ExecutorService</code> can be specified that is further used to execute requests on a separate thread.

To put it in context an example of obtaining `CompletableFuture` with JAX-RS <code class="literal">Response</code> from a remote service may look like

{{< highlight java >}}
 stage = RxCompletableFuture.newClient()
        .target("http://example.com/resource")
        .request()
        .rx()
        .get();
{{< / highlight >}}

To obtain the extension module containing Reactive Jersey Client with Java 8 support look at the coordinates: <a href="http://search.maven.org/#search%7Cga%7C1%7C%22org.glassfish.jersey.ext.rx%3Ajersey-rx-client-jsr166e%22">org.glassfish.jersey.ext.rx:jersey-rx-client-jsr166e</a>.

## Further reading

  * [Reactive Jersey Client – Motivation][1]
  * [Reactive Jersey Client – Customization (SPI)][2]

## Resources

  * <a class="link" href="https://github.com/jersey/jersey/tree/master/examples/rx-client-webapp">Travel Agency (Orchestration Layer) Example using Reactive Jersey Client API</a>
  * <a class="link" href="https://github.com/jersey/jersey/tree/master/examples/rx-client-java8-webapp">Travel Agency (Orchestration Layer) Example using Reactive Jersey Client API (Java 8)</a>

 [1]: /2015/01/07/reactive-jersey-client-part-1-motivation "Rx Client – Motivation"
 [2]: /2015/01/07/reactive-jersey-client-part-3-customization "Rx Client – Customization (SPI)"
 [3]: #jsr166e
 [4]: #java8