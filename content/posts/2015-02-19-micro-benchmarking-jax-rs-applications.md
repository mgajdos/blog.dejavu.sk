---
title: Micro-Benchmarking JAX-RS Applications
author: Michal Gajdoš
type: post
date: 2015-02-19T09:15:44+00:00
url: /micro-benchmarking-jax-rs-applications/
aliases: /2015/02/19/micro-benchmarking-jax-rs-applications/
categories:
  - Jersey
tags:
  - jax-rs
  - client
  - performance

---
Sometimes you want to examine what impact would a new JAX-RS filter have on performance of your application. Whether your custom message body provider is as fast as you though or you simply want to find out the throughput of your JAX-RS resources or client instances. Recently we were looking into this area and we&#8217;ve created few utilities that may make your life easier if you want to write micro-benchmarks for JAX-RS applications.

<!--more-->

Benchmarks I am going to focus on are based on <a href="http://openjdk.java.net/projects/code-tools/jmh/">JMH</a> tool from JDK. If you&#8217;ve never encountered JMH I highly recommend and encourage you to go through their <a href="http://hg.openjdk.java.net/code-tools/jmh/file/tip/jmh-samples/src/main/java/org/openjdk/jmh/samples/">samples</a> which will show you the main features of the tool. In under 30 minutes you&#8217;ll be able to write your own benchmarks.

But lets get back to JAX-RS and Jersey – the main goal of bringing the support for micro-benchmarks was to measure the values (e.g. throughput) as precisely as possible for both, server and client, without any unnecessary interferences.

We&#8217;ve introduced a new module, <a href="https://github.com/jersey/jersey/tree/master/test-framework/util">jersey-test-framework-util</a>, with some handy utilities to write your own benchmarks. The module is already available in the <a href="https://github.com/jersey/jersey/">workspace</a> and in some time it&#8217;s going to be also in <a href="https://java.net/jira/browse/JERSEY/fixforversion/17200/">Jersey 2.17</a>.

## Server

In my recent post, <a href="/2015/02/12/performance-improvements-of-sub-resource-locators-in-jersey/">Performance Improvements of Sub-Resource Locators in Jersey</a>, I was covering performance gains we got in Jersey when we introduced caches for sub-resource locators. I&#8217;ve decided to also create a micro-benchmark that would focus on the same area (normal resource methods vs. sub-resource locators) and that would let me compare the findings I made earlier with results from the benchmark.

The example, <a href="https://github.com/jersey/jersey/tree/master/examples/helloworld-benchmark">helloworld-benchmark</a>, exposes one <a href="https://github.com/jersey/jersey/blob/master/examples/helloworld-benchmark/src/main/java/org/glassfish/jersey/examples/helloworld/HelloWorldResource.java#L55">resource class</a> containing three resource methods (for GET, POST and PUT) and also a sub-resource locator (at _&#8220;locator&#8221;_ path) which then again points to the same resource class.

Much more interesting is the benchmark class, <a href="https://github.com/jersey/jersey/blob/master/examples/helloworld-benchmark/src/main/java/org/glassfish/jersey/examples/helloworld/HelloWorldBenchmark.java#L78">HelloWorldBenchmark</a>, which illustrates one approach of writing benchmarks for JAX-RS server-side applications:

{{< highlight java "linenos=inline,hl_lines=20 25-28 37" >}}
@BenchmarkMode(Mode.Throughput)
@OutputTimeUnit(TimeUnit.SECONDS)
@Warmup(iterations = 8, time = 1, timeUnit = TimeUnit.SECONDS)
@Measurement(iterations = 8, time = 1, timeUnit = TimeUnit.SECONDS)
@Fork(1)
@State(Scope.Benchmark)
public class HelloWorldBenchmark {

    @Param(value = {"helloworld", "helloworld/locator"})
    private String path;

    @Param(value = {"GET", "POST", "PUT"})
    private String method;

    private volatile ApplicationHandler handler;
    private volatile ContainerRequest request;

    @Setup
    public void start() throws Exception {
        handler = new ApplicationHandler(new Application());
    }

    @Setup(Level.Iteration)
    public void request() {
        request = ContainerRequestBuilder
                .from(path, method)
                .entity("GET".equals(method) ? null : HelloWorldResource.CLICHED_MESSAGE, handler)
                .build();
    }

    @TearDown
    public void shutdown() {
    }

    @Benchmark
    public Future<ContainerResponse> measure() throws Exception {
        return handler.apply(request);
    }

    public static void main(final String[] args) throws Exception {
        final Options opt = new OptionsBuilder()
                // Register our benchmarks.
                .include(HelloWorldBenchmark.class.getSimpleName())
                .build();

        new Runner(opt).run();
    }
}
{{< / highlight >}}

You can see that the benchmark is not based on any container, e.g. Servlet or Grizzly, like tests backed-up by <a href="https://jersey.github.io/documentation/latest/test-framework.html">Jersey Test Framework</a>. Requests are directly invoked on <a href="https://github.com/jersey/jersey/blob/master/core-server/src/main/java/org/glassfish/jersey/server/ApplicationHandler.java#L166">ApplicationHandler</a> class from Jersey Server module, the main entry point for server applications. This eliminates any client or network influences and lets you measure directly the server performance. Plus, all JAX-RS features, like filters and interceptors, are still taken into consideration when the benchmark is executed.

The first step when creating a benchmark similar to the one above is to create an instance of _ApplicationHandler_ and give it an instance (or a class) of tested JAX-RS application (see line 20). The _@Setup_ annotation on _start()_ method ensures new handler is created for each run of the benchmark.

This benchmark executes six runs in total – we have two paths we want test, _&#8220;helloworld&#8221;_ and _&#8220;helloworld/locator&#8221;_ (see parameter definition at lines 9-10), and three HTTP methods, see lines 12-13. When you combine the number of paths with number of HTTP methods to be invoked you get the total number of runs.

_ApplicationHandler_ handles <a href="https://github.com/jersey/jersey/blob/master/core-server/src/main/java/org/glassfish/jersey/server/ContainerRequest.java#L104">container request context</a> instances and creates a response for the request. To create such a container request we introduced the <a href="https://github.com/jersey/jersey/blob/master/test-framework/util/src/main/java/org/glassfish/jersey/test/util/server/ContainerRequestBuilder.java#L69">container request builder</a> that would help you to create any request you need. You may find it slightly unusual at first, especially if you&#8217;re used to JAX-RS Client API but it&#8217;s not that difficult. An example of use can be seen on lines 25-28. You need to provide (relative) path to a resource, HTTP method you want to invoke and an optional entity. Then you can build the request context and pass it to the handler. The _@Setup(Level.Iteration)_ tells JMH to create a new request context for each iteration.

The benchmark itself is defined in method _measure()_ (see _@Benchmark_ annotation) and all it does is applying the container request context to the handler and returning the response (see line 37).

This is all we need to do in terms of JAX-RS. Now we need to define some properties of the benchmark. We want to measure throughput in seconds or &#8216;number of operations per second&#8217; (lines 1 and 2). The warmup of each run would be 8 iterations, each would take 1 second (in the benchmarked code itself, line 3). The actual benchmark would behave the same as warmup (line 4) and it&#8217;ll be executed in a forked JVM (line 5).

An example results of the benchmark looks like the follows and it basically confirms the results we got during regular performance testing.

Jersey 2.15:

{{< highlight bash >}}
# Run complete. Total time: 00:01:42

Benchmark                    (method)              (path)   Mode  Cnt      Score      Error  Units
HelloWorldBenchmark.measure       GET          helloworld  thrpt    8  31509.431 ± 2127.133  ops/s
HelloWorldBenchmark.measure       GET  helloworld/locator  thrpt    8   1364.246 ±  283.353  ops/s
HelloWorldBenchmark.measure      POST          helloworld  thrpt    8  36839.525 ± 7194.788  ops/s
HelloWorldBenchmark.measure      POST  helloworld/locator  thrpt    8   1417.278 ±  346.776  ops/s
HelloWorldBenchmark.measure       PUT          helloworld  thrpt    8  35924.090 ± 4610.261  ops/s
HelloWorldBenchmark.measure       PUT  helloworld/locator  thrpt    8   1469.342 ±  315.749  ops/s
{{< / highlight >}}

Jersey 2.17-SNAPSHOT:

{{< highlight bash >}}
# Run complete. Total time: 00:01:41

Benchmark                    (method)              (path)   Mode  Cnt      Score      Error  Units
HelloWorldBenchmark.measure       GET          helloworld  thrpt    8  74343.864 ± 4814.979  ops/s
HelloWorldBenchmark.measure       GET  helloworld/locator  thrpt    8  54137.102 ± 8996.766  ops/s
HelloWorldBenchmark.measure      POST          helloworld  thrpt    8  45173.853 ± 5349.363  ops/s
HelloWorldBenchmark.measure      POST  helloworld/locator  thrpt    8  37144.797 ± 4699.782  ops/s
HelloWorldBenchmark.measure       PUT          helloworld  thrpt    8  45945.974 ± 4116.752  ops/s
HelloWorldBenchmark.measure       PUT  helloworld/locator  thrpt    8  36345.667 ± 5929.480  ops/s
{{< / highlight >}}

## Client

With JMH and Jersey you can also measure performance of JAX-RS clients (filters, interceptors, message body readers/writers, etc.). This is possible by using special <a href="https://jersey.github.io/documentation/latest/client.html#d0e4740">client connector</a>  available in the new Jersey Test Framework Util module. It&#8217;s called <a href="https://github.com/jersey/jersey/blob/master/test-framework/util/src/main/java/org/glassfish/jersey/test/util/client/LoopBackConnector.java#L68">LoopBackConnector</a> and the only thing it does is it translates request created via JAX-RS Client API into a Response. Request headers and entity are copied into the response to make sure the message body providers and parameter providers are executed and measured as well.

Creating a client for benchmark testing is easy. Simply use one of the following snippets and you&#8217;re ready.

{{< highlight java >}}
client = ClientBuilder.newClient(LoopBackConnectorProvider.getClientConfig())
{{< / highlight >}}

or

{{< highlight java >}}
ClientConfig config = new ClientConfig()
    .connectorProvider(new LoopBackConnectorProvider());
ClientBuilder.newClient(config);
{{< / highlight >}}

The Benchmark code may then look like:

{{< highlight java >}}
@BenchmarkMode(Mode.Throughput)
@OutputTimeUnit(TimeUnit.SECONDS)
@Warmup(iterations = 16, time = 1, timeUnit = TimeUnit.SECONDS)
@Measurement(iterations = 16, time = 1, timeUnit = TimeUnit.SECONDS)
@Fork(1)
@State(Scope.Benchmark)
public class ClientBenchmark {

    private volatile Client client;

    @Setup
    public void start() throws Exception {
        client = ClientBuilder.newClient(LoopBackConnectorProvider.getClientConfig());
    }

    @TearDown
    public void shutdown() {
        client.close();
    }

    @Benchmark
    public Response get() throws Exception {
        return client.target("foo").request().get();
    }

    @Benchmark
    public Response post() throws Exception {
        return client.target("foo").request().post(Entity.text("bar"));
    }

    @Benchmark
     public Response asyncBlock() throws Exception {
        return client.target("foo").request().async().get().get();
    }

    @Benchmark
    public Future<Response> asyncIgnore() throws Exception {
        return client.target("foo").request().async().get(new InvocationCallback<Response>() {
            @Override
            public void completed(final Response response) {
                // NOOP
            }

            @Override
            public void failed(final Throwable throwable) {
                // NOOP
            }
        });
    }

    @Benchmark
    public Future<Response> asyncEntityIgnore() throws Exception {
        return client.target("foo").request().async().post(Entity.text("bar"), new InvocationCallback<Response>() {
            @Override
            public void completed(final Response response) {
                // NOOP
            }

            @Override
            public void failed(final Throwable throwable) {
                // NOOP
            }
        });
    }

    public static void main(final String[] args) throws Exception {
        final Options opt = new OptionsBuilder()
                // Register our benchmarks.
                .include(ClientBenchmark.class.getSimpleName())
                .build();

        new Runner(opt).run();
    }
}
{{< / highlight >}}

You can see that the benchmark is testing synchronous and asynchronous GET and POST methods. The whole class (and some other tests) is available in our new <a href="https://github.com/jersey/jersey/tree/master/tests/performance/benchmarks">benchmarks</a> module amongst the other performance tests.

> If you have any comments or questions, please leave a message in the discussion below or send me a note via email, <michal@dejavu.sk>.