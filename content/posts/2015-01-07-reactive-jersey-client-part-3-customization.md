---
title: Reactive Jersey Client, Part 3 – Customization
author: Michal Gajdoš
type: post
date: 2015-01-07T08:15:43+00:00
url: /reactive-jersey-client-part-3-customization/
aliases: /2015/01/07/reactive-jersey-client-part-3-customization/
categories:
  - Jersey
tags:
  - jersey
  - client
  - reactive

---
Reactive Jersey Client API is a generic API allowing end users to utilize the popular reactive programming model when using Jersey Client. This part of the series describes SPI and implementation of support for custom reactive libraries.

<!--more-->

  * [Reactive Jersey Client – Motivation][1]
  * [Reactive Jersey Client – Usage and Supported Reactive Libraries][2]
  * Reactive Jersey Client – Customization (SPI) _(this article)_

In case you want to bring support for some other library providing Reactive Programming Model into your application you can extend functionality of Reactive Jersey Client by implementing SPI available in <code class="literal">jersey-rx-client</code> module.

### Extend RxInvoker interface

Even though not entirely intuitive this step is required when a support for a custom reactive library is needed. As mentioned above few JAX-RS Client interfaces had to be modified in order to make possible to invoke HTTP calls in a reactive way. All of them except the <a class="link" href="https://jersey.github.io/apidocs/latest/jersey/org/glassfish/jersey/client/rx/RxInvoker.html">RxInvoker</a> extend the original interfaces from JAX-RS (e.g. <code class="literal">Client</code>). <code class="literal">RxInvoker</code> is a brand new interface (very similar to <code class="literal">SyncInvoker</code> and <code class="literal">AsyncInvoker</code>) that actually lets you to invoke HTTP methods in the reactive way.

{{< highlight java >}}
@Beta
public interface RxInvoker<T> {
 
    public <T> get();
 
    public <R> <T> get(Class<R> responseType);
 
    // ...
 
}
{{< / highlight >}}

As you can notice it&#8217;s too generic as it&#8217;s designed to support various reactive libraries without bringing any additional abstractions and restrictions. The first type parameter, <code class="literal">T</code>, is the asynchronous/event-based completion aware type (e.g. <code class="literal">Observable</code>). The given type should be parametrized with the actual response type. And since it&#8217;s not possible to parametrize type parameter it&#8217;s an obligation of the extension of <code class="literal">RxInvoker</code> to do that. That applies to simpler methods, such as <code class="literal">get()</code>, as well as to more advanced methods, for example <code class="literal">get(Class)</code>.

In the first case it&#8217;s enough to parametrize the needed type with <code class="literal">Response</code>, e.g. <code class="literal">Observable<Response> get()</code>. The second case uses the type parameter from the parameter of the method. To accordingly extend the <code class="literal">get(Class<R>)</code> method you need to parametrize the needed type with <code class="literal">R</code> type parameter, e.g. <code class="literal"><T> Observable<T> get(Class<T> responseType)</code>.

To following summarizes the requirements above and illustrate them in one code snippet.

{{< highlight java >}}
 {
 
    @Override
    public Observable<Response> get();
 
    @Override
    public <T> Observable<T> get(Class<T> responseType);
 
    // ...
 
}
{{< / highlight >}}

This is an excerpt from <code class="literal">RxObservableInvoker</code> that works with RxJava&#8217;s <code class="literal">Observable</code>.

### Implement the extended interface

Either you can implement the extension of <code class="literal">RxInvoker</code> from scratch or it&#8217;s possible to extend from <a class="link" href="https://jersey.github.io/apidocs/latest/jersey/org/glassfish/jersey/client/rx/spi/AbstractRxInvoker.html">AbstractRxInvoker</a> abstract class which serves as a default implementation of the interface. In the later case only <code class="literal">#method(...)</code> methods are needed to be implemented as the default implementation of other methods (HTTP calls) delegates to these methods.

{{< highlight java >}}
final class JerseyRxObservableInvoker
                extends AbstractRxInvoker<Observable>
                implements RxObservableInvoker {

    JerseyRxObservableInvoker(final Invocation.Builder builder,
                              final ExecutorService executor) {
        super(builder, executor);
    }

    @Override
    public <T> Observable<T> method(final String name,
                                    final Entity<?> entity,
                                    final Class<T> responseType) {
        // Invoke as sync JAX-RS client request and subscribe/observe
        // on a scheduler initialized with executor service.

        final Scheduler scheduler = Schedulers.from(getExecutorService());

        return Observable.create(new Observable.OnSubscribe<T>() {
            @Override
            public void call(final Subscriber<? super T> subscriber) {
                if (!subscriber.isUnsubscribed()) {
                    try {
                        final T response = getBuilder()
                            .method(name, entity, responseType);

                        if (!subscriber.isUnsubscribed()) {
                            subscriber.onNext(response);
                        }
                        if (!subscriber.isUnsubscribed()) {
                            subscriber.onCompleted();
                        }
                    } catch (final Throwable throwable) {
                        if (!subscriber.isUnsubscribed()) {
                            subscriber.onError(throwable);
                        }
                    }
                }
            }
        }).subscribeOn(scheduler).observeOn(scheduler);
    }

    // ...

}
{{< / highlight >}}

For a complete implementation take a look at <a href="https://github.com/jersey/jersey/blob/master/incubator/rx/rx-client-rxjava/src/main/java/org/glassfish/jersey/client/rx/rxjava/JerseyRxObservableInvoker.java#L68">JerseyRxObservableInvoker</a>.

### Implement and register RxInvokerProvider

To create an instance of particular <code class="literal">RxInvoker</code> an implementation of <a class="link" href="https://jersey.github.io/apidocs/latest/jersey/org/glassfish/jersey/client/rx/spi/RxInvokerProvider.html">RxInvokerProvider</a> SPI interface is needed. When a concrete <code class="literal">RxInvoker</code> is requested the runtime goes through all available providers and finds one which supports the given invoker type. It is expected that each provider supports mapping for distinct set of types and subtypes so that different providers do not conflict with each other.

{{< highlight java >}}
public final class RxObservableInvokerProvider
                       implements RxInvokerProvider {
 
    @Override
    public <T> T getInvoker(final Class<T> invokerType,
                            final Invocation.Builder builder,
                            final ExecutorService executor) {

        if (RxObservableInvoker.class.isAssignableFrom(invokerType)) {
            return invokerType.cast(
                new JerseyRxObservableInvoker(builder, executor));
        }
        return null;
    }
}
{{< / highlight >}}

Reactive Jersey Client looks for all available <code class="literal">RxInvokerProvider</code>s via the standard <code class="literal">META-INF/services</code> mechanism. It&#8217;s enough to bundle <code class="literal">org.glassfish.jersey.client.rx.spi.RxInvokerProvider</code> file with your library and reference your implementation (by fully qualified class name) from it.

{{< highlight java >}}
org.glassfish.jersey.client.rx.rxjava.RxObservableInvokerProvider
{{< / highlight >}}

## Further reading

  * [Reactive Jersey Client – Motivation][1]
  * [Reactive Jersey Client – Usage and Supported Reactive Libraries][2]

## Resources

  * <a href="https://github.com/jersey/jersey/tree/master/incubator/rx/rx-client-rxjava/src/main/java/org/glassfish/jersey/client/rx/rxjava">jersey-rx-client-rxjava</a> (sample implementation)
  * <a class="link" href="https://github.com/jersey/jersey/tree/master/examples/rx-client-webapp">Travel Agency (Orchestration Layer) Example using Reactive Jersey Client API</a>
  * <a class="link" href="https://github.com/jersey/jersey/tree/master/examples/rx-client-java8-webapp">Travel Agency (Orchestration Layer) Example using Reactive Jersey Client API (Java 8)</a>

 [1]: /2015/01/07/reactive-jersey-client-part-1-motivation "Rx Client – Motivation"
 [2]: /2015/01/07/reactive-jersey-client-part-2-usage-and-supported-reactive-libraries "Rx Client – Usage and Supported Reactive Libraries"