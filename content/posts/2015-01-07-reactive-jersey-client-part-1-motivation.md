---
title: Reactive Jersey Client, Part 1 – Motivation
author: Michal Gajdoš
type: post
date: 2015-01-07T08:05:38+00:00
url: /reactive-jersey-client-part-1-motivation/
aliases: /2015/01/07/reactive-jersey-client-part-1-motivation/
categories:
  - Jersey
tags:
  - jersey
  - client
  - reactive

---
Reactive Jersey Client API is a generic API allowing end users to utilize the popular reactive programming model when using Jersey Client. This part of the series describes motivation behind creating Reactive Jersey Client.

<!--more-->

The whole series consists of three articles:

  * Reactive Jersey Client – Motivation _(this article)_
  * [Reactive Jersey Client – Usage and Supported Reactive Libraries][1]
  * [Reactive Jersey Client – Customization (SPI)][2]

## The Problem

Imagine a travel agency whose information system consists of multiple basic services. These services might be built using different technologies (JMS, EJB, WS, …). For simplicity we presume that the services can be consumed using REST interface via HTTP method calls (e.g. using a JAX-RS Client). We can also presume that the basic services we need to work with are:

  * _Customers service_ – provides information about customers of the travel agency.
  * _Destinations service_ – provides a list of visited and recommended destinations for an authenticated customer.
  * _Weather service_ – provides weather forecast for a given destination.
  * _Quoting service_ – provides price calculation for a customer to travel to a recommended destination.

The task is to create a publicly available feature that would, for an authenticated user, display a list of 10 last visited places and also display a list of 10 new recommended destinations including weather forecast and price calculations for the user. Notice that some of the requests (to retrieve data) depend on results of previous requests. E.g. getting recommended destinations depends on obtaining information about the authenticated user first. Obtaining weather forecast depends on destination information, etc. This relationship between some of the requests is an important part of the problem and an area where you can take a real advantage of the reactive programming model.

One way how to obtain data is to make multiple HTTP method calls from the client (e.g. mobile device) to all services involved and combine the retrieved data on the client. However, since the basic services are available in the internal network only we&#8217;d rather create a public orchestration layer instead of exposing all internal services to the outside world. The orchestration layer would expose only the desired operations of the basic services to the public. To limit traffic and achieve lower latency we&#8217;d like to return all the necessary information to the client in a single response.

The orchestration layer is illustrated here:

[<img class="aligncenter wp-image-350 size-large" src="/uploads/rx-client-problem-1024x491.png" alt="Travel Agency Orchestration Service" width="660" height="316" srcset="/uploads/rx-client-problem-1024x491.png 1024w, /uploads/rx-client-problem-300x144.png 300w, /uploads/rx-client-problem.png 1569w" sizes="(max-width: 660px) 100vw, 660px" />][3]

The layer accepts requests from the outside and is responsible of invoking multiple requests to the internal services. When responses from the internal services are available in the orchestration layer they&#8217;re combined into a single response that is sent back to the client.

Now, let&#8217;s take a look at various ways how the orchestration layer can be implemented.

## A Naïve Approach

The simplest way to implement the orchestration layer is to use synchronous approach. For this purpose we can use JAX-RS Client Sync API. The implementation is simple to do, easy to read and straightforward to debug.

```java
final WebTarget destination = ...;
final WebTarget forecast = ...;
 
// Obtain recommended destinations.
List<Destination> recommended = Collections.emptyList();
try {
    recommended = destination.path("recommended").request()
            // Identify the user.
            .header("Rx-User", "Sync")
            // Return a list of destinations.
            .get(new GenericType<List<Destination>>() {});
} catch (final Throwable throwable) {
    errors.offer("Recommended: " + throwable.getMessage());
}
 
// Forecasts. (depend on recommended destinations)
final Map<String, Forecast> forecasts = new HashMap<>();
for (final Destination dest : recommended) {
    try {
        forecasts.put(dest.getDestination(),
                forecast.resolveTemplate("destination", dest.getDestination())
                        .request()
                        .get(Forecast.class));
    } catch (final Throwable throwable) {
        errors.offer("Forecast: " + throwable.getMessage());
    }
}
```

> The whole implementation can be found in Jersey workspace, e.g. on GitHub – <a href="https://github.com/jersey/jersey/blob/master/examples/rx-client-java8-webapp/src/main/java/org/glassfish/jersey/examples/rx/agent/SyncAgentResource.java#L84">SyncAgentResource</a> class in <a href="https://github.com/jersey/jersey/tree/master/examples/rx-client-java8-webapp">rx-client-java8-webapp</a>.

The downside of this approach is it&#8217;s slowness. You need to sequentially process all the independent requests which means that you&#8217;re wasting resources. You are needlessly blocking threads, that could be otherwise used for some real work.

If you take a closer look at the example you can notice that at the moment when all the recommended destinations are available for further processing we try to obtain forecasts for these destinations. Obtaining a weather forecast can be done only for a single destination with a single request, so we need to make 10 requests to the _Forecast service_ to get all the destinations covered. In a synchronous way this means getting the forecasts one-by-one. When one response with a forecast arrives we can send another request to obtain another one. This takes time. The whole process of constructing a response for the client can be seen in the following picture.

[<img class="aligncenter size-large wp-image-351" src="/uploads/rx-client-sync-approach-1024x316.png" alt="Time consumed to create a response for the client – synchronous way" width="660" height="204" srcset="/uploads/rx-client-sync-approach-1024x316.png 1024w, /uploads/rx-client-sync-approach-300x93.png 300w" sizes="(max-width: 660px) 100vw, 660px" />][4]

Let&#8217;s try to quantify this with assigning an approximate time to every request we make to the internal services. This way we can easily compute the time needed to complete a response for the client. For example, obtaining

  * _Customer details_ takes 150 ms
  * _Recommended destinations_ takes 250 ms
  * _Price calculation for a customer and destination_ takes 170 ms (each)
  * _Weather forecast for a destination_ takes 330 ms (each)

When summed up, 5400 ms is approximately needed to construct a response for the client.

Synchronous approach is better to use for lower number of requests (where the accumulated time doesn&#8217;t matter that much) or for a single request that depends on the result of previous operations.

## Optimized Approach

The amount of time needed by the synchronous approach can be lowered by invoking independent requests in parallel. We&#8217;re going to use JAX-RS Client Async API to illustrate this approach. The implementation in this case is slightly more difficult to get right because of the nested callbacks and the need to wait at some points for the moment when all partial responses are ready to be processed. The implementation is also a little bit harder to debug and maintain. The nested calls are causing a lot of complexity here.

{{< highlight java >}}
final WebTarget destination = ...;
final WebTarget forecast = ...;
 
// Obtain recommended destinations. (does not depend on visited ones)
destination.path("recommended").request()
    // Identify the user.
    .header("Rx-User", "Async")
    // Async invoker.
    .async()
    // Return a list of destinations.
    .get(new InvocationCallback<List<Destination>>() {
        @Override
        public void completed(final List<Destination> recommended) {
            final CountDownLatch innerLatch = new CountDownLatch(recommended.size());

            // Forecasts. (depend on recommended destinations)
            final Map<String, Forecast> forecasts =
                Collections.synchronizedMap(new HashMap<>());

            for (final Destination dest : recommended) {
                forecast.resolveTemplate("destination", dest.getDestination())
                    .request()
                    .async()
                    .get(new InvocationCallback<Forecast>() {
                        @Override
                        public void completed(final Forecast forecast) {
                            forecasts.put(dest.getDestination(), forecast);
                            innerLatch.countDown();
                        }

                        @Override
                        public void failed(final Throwable throwable) {
                            errors.offer("Forecast: " + throwable.getMessage());
                            innerLatch.countDown();
                        }
                    });
            }

            // Have to wait here for dependent requests ...
            try {
                if (!innerLatch.await(10, TimeUnit.SECONDS)) {
                    errors.offer("Inner: Waiting for requests to complete has timed out.");
                }
            } catch (final InterruptedException e) {
                errors.offer("Inner: Waiting for requests to complete has been interrupted.");
            }

            // Continue with processing.
        }

        @Override
        public void failed(final Throwable throwable) {
            errors.offer("Recommended: " + throwable.getMessage());
        }
    });
{{< / highlight >}}

> The whole implementation can be found in Jersey workspace, e.g. on GitHub – <a href="https://github.com/jersey/jersey/blob/master/examples/rx-client-java8-webapp/src/main/java/org/glassfish/jersey/examples/rx/agent/AsyncAgentResource.java#L104">AsyncAgentResource</a> class in <a href="https://github.com/jersey/jersey/tree/master/examples/rx-client-java8-webapp">rx-client-java8-webapp</a>.

The example is a bit more complicated from the first glance. We provided an <a class="link" href="http://jax-rs-spec.java.net/nonav/2.0/apidocs/javax/ws/rs/client/InvocationCallback.html">InvocationCallback</a> to async <code class="literal">get</code> method. One of the callback methods (<code class="literal">completed</code> or <code class="literal">failed</code>) is called when the request finishes. This is a pretty convenient way to handle async invocations when no nested calls are present. Since we have some nested calls (obtaining weather forecasts) we needed to introduce a <a class="link" href="http://docs.oracle.com/javase/6/docs/api/java/util/concurrent/CountDownLatch.html">CountDownLatch</a> synchronization primitive as we use asynchronous approach in obtaining the weather forecasts as well. The latch is decreased every time a request, to the _Forecasts service_, completes successfully or fails. This indicates that the request actually finished and it is a signal for us that we can continue with processing (otherwise we wouldn&#8217;t have all required data to construct the response for the client). This additional synchronization is something that was not present when taking the synchronous approach, but it is needed here.

Also the error processing can not be written as it could be in an ideal case. The error handling is scattered in too many places within the code, that it is quite difficult to create a comprehensive response for the client.

On the other hand taking asynchronous approach leads to code that is as fast as it gets. The resources are used optimally (no waiting threads) to achieve quick response time. The whole process of constructing the response for the client can be seen in an image below. It only took 730 ms instead of 5400 ms which we encountered in the previous approach.

[<img class="aligncenter size-large wp-image-349" src="/uploads/rx-client-async-approach-1024x450.png" alt="Time consumed to create a response for the client – asynchronous way" width="660" height="290" srcset="/uploads/rx-client-async-approach-1024x450.png 1024w, /uploads/rx-client-async-approach-300x132.png 300w" sizes="(max-width: 660px) 100vw, 660px" />][5]

As you can guess, this approach, even with all it&#8217;s benefits, is the one that is really hard to implement, debug and maintain. It&#8217;s a safe bet when you have many independent calls to make but it gets uglier with an increasing number of nested calls.

## Reactive Approach

Reactive approach is a way out of the so-called _Callback Hell_ which you can encounter when dealing with Java&#8217;s <code class="literal">Future</code>s or invocation callbacks. Reactive approach is based on a data-flow concept and the execution model propagate changes through the flow. An example of a single item in the data-flow chain can be a JAX-RS Client HTTP method call. When the JAX-RS request finishes then the next item (or the user code) in the data-flow chain is notified about the continuation, completion or error in the chain. You&#8217;re more describing what should be done next than how the next action in the chain should be triggered. The other important part here is that the data-flows are composable. You can compose/transform multiple flows into the resulting one and apply more operations on the result.

{{< highlight java >}}
final WebTarget destination = ...;
final WebTarget forecast = ...;
 
// Recommended places.
final Observable<Destination> recommended = RxObservable.from(destination)
        .path("recommended")
        .request()
        // Identify the user.
        .header("Rx-User", "RxJava")
        // Reactive invoker.
        .rx()
        // Return a list of destinations.
        .get(new GenericType<List<Destination>>() {})
        // Handle Errors.
        .onErrorReturn(throwable -> {
            errors.offer("Recommended: " + throwable.getMessage());
            return Collections.emptyList();
        })
        // Emit destinations one-by-one.
        .flatMap(Observable::from)
        // Remember emitted items for dependant requests.
        .cache();
 
// Forecasts. (depend on recommended destinations)
final RxWebTarget<RxObservableInvoker> rxForecast = RxObservable.from(forecast);
final Observable<Forecast> forecasts = recommended.flatMap(destination ->
        rxForecast
                .resolveTemplate("destination", destination.getDestination())
                .request()
                .rx()
                .get(Forecast.class)
                .onErrorReturn(throwable -> {
                    errors.offer("Forecast: " + throwable.getMessage());
                    return new Forecast(destination.getDestination(), "N/A");
                }));
 
final Observable<Recommendation> recommendations = Observable
        .zip(recommended, forecasts, Recommendation::new);
{{< / highlight >}}

> The whole implementation can be found in Jersey workspace, e.g. on GitHub – <a href="https://github.com/jersey/jersey/blob/master/examples/rx-client-java8-webapp/src/main/java/org/glassfish/jersey/examples/rx/agent/ObservableAgentResource.java#L97">ObservableAgentResource</a> class in <a href="https://github.com/jersey/jersey/tree/master/examples/rx-client-java8-webapp">rx-client-java8-webapp</a>.

The APIs are described in more detail in the next part [Reactive Jersey Client, Part 2 – Usage and Supported Libraries][6]{.row-title}.

As you can see the code achieves the same work as the previous two examples. It&#8217;s more readable than the pure asynchronous approach even though it&#8217;s equally fast. It&#8217;s as easy to read and implement as the synchronous approach. The error processing is also better handled in this way than in the asynchronous approach.

When dealing with a large amount of requests (that depend on each other) and when you need to compose/combine the results of these requests, the reactive programming model is the right technique to use.

## Further reading

  * [Reactive Jersey Client – Usage and Supported Reactive Libraries][1]
  * [Reactive Jersey Client – Customization (SPI)][2]

## Resources

  * <a class="link" href="https://github.com/jersey/jersey/tree/master/examples/rx-client-webapp">Travel Agency (Orchestration Layer) Example using Reactive Jersey Client API</a>
  * <a class="link" href="https://github.com/jersey/jersey/tree/master/examples/rx-client-java8-webapp">Travel Agency (Orchestration Layer) Example using Reactive Jersey Client API (Java 8)</a>

 [1]: /2015/01/07/reactive-jersey-client-part-2-usage-and-supported-reactive-libraries "Rx Client – Usage and Supported Reactive Libraries"
 [2]: /2015/01/07/reactive-jersey-client-part-3-customization "Rx Client – Customization (SPI)"
 [3]: /uploads/rx-client-problem.png
 [4]: /uploads/rx-client-sync-approach.png
 [5]: /uploads/rx-client-async-approach.png
 [6]: /2015/01/07/reactive-jersey-client-part-2-usage-and-supported-reactive-libraries