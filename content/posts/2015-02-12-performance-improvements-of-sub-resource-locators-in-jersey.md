---
title: Performance Improvements of Sub-Resource Locators in Jersey
author: Michal Gajdoš
type: post
date: 2015-02-12T16:50:12+00:00
url: /performance-improvements-of-sub-resource-locators-in-jersey/
aliases: /2015/02/12/performance-improvements-of-sub-resource-locators-in-jersey/
categories:
  - Jersey
tags:
  - jax-rs
  - jersey
  - performance

---
There were many performance improvements since Jersey 2 first came out almost 2 years ago. Jakub and I decided to present some of the latest ones in our recent articles. Jakub took a look at general performance gains in Jersey in his <a href="https://blogs.oracle.com/japod/entry/jersey_2_performance">Jersey 2 Performance</a>. I&#8217;ll try to cover the change we introduced in Jersey 2.16 and that affected processing and performance of sub-resource locators in Jersey applications.

<!--more-->

> We&#8217;ve fought a lot of fights but the war isn&#8217;t over. We&#8217;re going to focus on performance improvements in Jersey even more.

This article is separated into three main sections:

  * [JAX-RS Resources and their Methods][1]
  * [Performance Testing and Results][2]
  * [Sub-Resource Locator Cache Configuration][3]

## <a name="methods"></a>Resources and their Methods

JAX-RS recognizes three types of methods that can be present in a resource class and invoked directly by JAX-RS: resource methods, sub-resource methods and sub-resource locators.

#### Resource Methods

<div class="page" title="Page 25">
  <div class="layoutArea">
    <div class="column">
      <p>
        Resource methods are methods of a resource class annotated with a request method designator (e.g. <a href="https://jax-rs.github.io/apidocs/2.0.1/javax/ws/rs/GET.html">@GET</a>, <a href="https://jax-rs.github.io/apidocs/2.0.1/javax/ws/rs/POST.html">@POST</a>, &#8230;). These are well know, widely used to handle incoming request and pretty straightforward to understand so no further explanation is required.
      </p>
      
      <p>
        An example of resource method may be:
      </p>
      
      {{< highlight java "hl_lines=4-7" >}}
@Path("resource")
public class Resource {

    @GET
    public String get() {
        return "Hello from Resource Method";
    }
}
{{< / highlight >}}
    </div>
  </div>
</div>

Now lets take a closer look at <a href="https://jersey.github.io/apidocs/latest/jersey/index.html">@Path</a> annotation. In addition to putting it on a class you can annotate also methods with this annotation. Methods of a resource class that are annotated with _@Path_ are either sub-resource methods or sub-resource locators.

Sub-resource methods handle a HTTP request directly whilst sub-resource locators return an object that will handle a HTTP request. The presence or absence of a request method designator (e.g. _@GET_) differentiates between the two.

#### Sub-Resource Methods

If the request method designator is present on a resource method then we&#8217;re talking about sub-resource methods. These methods are treated like a normal resource method except the method is only invoked for request URIs that match a URI template created by concatenating the URI template of the resource class with the URI template of the method.

Sub-resource methods are also pretty common and an example of such a method could be:

{{< highlight java "hl_lines=4-8" >}}
@Path("resource")
public class Resource {

    @GET
    @Path("hello")
    public String get() {
        return "Hello from Sub-Resource Method";
    }
}
{{< / highlight >}}

#### Sub-Resource Locators

Locators are methods used to dynamically resolve the object that will handle the request. Any returned object is treated as a resource class instance and used to either handle the request or to further resolve the object that will handle the request.  Sub-resource locators may have all the same parameters as a normal resource method except that they must not have an entity parameter.

An example of a sub-resource locator would be:

{{< highlight java "hl_lines=4-7" >}}
@Path("resource")
public class Resource {

    @Path("{name}")
    public SubResource sub(@PathParam("name") final String name) {
        return new SubResource(name);
    }
}

public class SubResource {

    private final String name;

    public SubResource(final String name) {
        this.name = name;
    }

    @GET
    public String get() {
        return "Hello, " + name + ", from Sub-Resource Locator";
    }
}
{{< / highlight >}}

_Useful?_ Definitely. _Usable?_ Yes!, but slightly slow in Jersey prior to version 2.16.

The performance problems in previous versions of Jersey was caused by updating (and not caching some parts of the) Jersey&#8217;s Resource Model over and over again when a sub-resource locator was used to handle a request at runtime.

Resource Model is used when requests are processed and routed to a matching resource class and resource method. It is created and stored at the deployment time for resources registered in your application (root resources). When the base model is created Jersey does some further processing. The model is enhanced, programmatically using <a href="https://jersey.github.io/documentation/latest/resource-builder.html#d0e11146">Model Processors</a>, by adding new resource methods to it (e.g. HTTP OPTION methods are added to each resource class by Jersey). The last step of preparing the resource processing model for runtime is to validate it. This step is present to be sure the model is in accordance with JAX-RS expectations.

These steps are rather obvious and perfectly fine to do once, not so fine to repeat them every time a sub-resource locator is involved in handling request.

When you have a sub-resources in your application the application&#8217;s resource model is updated and information about the sub-resource class is added to it so the processing of a request is handled properly. This means that all the steps that are done for application&#8217;s resource model are done for sub-resource locator class as well.

Fortunately the sub-resource model (enhanced and validated), for classes and instances, can be created once and then reused in subsequent calls. This was the approach to improve the bad performance of sub-resource locators – we introduced a cache for sub-resource locators which made the performance similar to using normal resource methods (configuration of the cache is explained in the last section, [Cache Configuration][3]).

## <a name="results"></a>Performance Testing

To be able to evaluate performance gains, between three versions of Jersey (2.0, 2.15, 2.16), we created a simple Jersey application (see resource classes below) and deployed it on Grizzly HTTP container. To saturate the server with requests we had two, <a href="https://github.com/wg/wrk">wrk</a> based, clients on two other separate nodes.

We measured (on the server) the number of requests per second the server was able to process. We ran five series, each containing 20 runs, of POST, PUT and GET requests and _text/plain_ media type. So, at the end we had 100 _requests/sec_ measurements for each of the HTTP methods.

For the sake of this article the absolute _request/sec_ numbers aren&#8217;t that important. Rather we focused on the difference in number of calls between tested Jersey versions and we also looked at the difference between performance of sub-resource locators and normal resource methods in Jersey 2.16.

The following two resource classes were used during testing,

{{< highlight java "hl_lines=13-16" >}}
@Path("text")
@Consumes(MediaType.TEXT_PLAIN)
@Produces(MediaType.TEXT_PLAIN)
public class TextEntityResource {

    private static final com.yammer.metrics.core.Timer postTimer =
            Metrics.newTimer(TextEntityResource.class, "posts", TimeUnit.MILLISECONDS, TimeUnit.SECONDS);
    private static final com.yammer.metrics.core.Timer getTimer =
            Metrics.newTimer(TextEntityResource.class, "gets", TimeUnit.MILLISECONDS, TimeUnit.SECONDS);
    private static final com.yammer.metrics.core.Timer putTimer =
            Metrics.newTimer(TextEntityResource.class, "puts", TimeUnit.MILLISECONDS, TimeUnit.SECONDS);

    @Path("sub-class")
    public Class<?> subresource() {
        return TextEntityResource.class;
    }

    @POST
    public String echo(final String text) {
        final TimerContext timer = postTimer.time();
        try {
            return text;
        } finally {
            timer.stop();
        }
    }

    @PUT
    public void put(final String text) {
        final TimerContext timer = putTimer.time();
        timer.stop();
    }

    @GET
    public String get() {
        final TimerContext timer = getTimer.time();
        try {
            return "text";
        } finally {
            timer.stop();
        }
    }
}
{{< / highlight >}}

for obtaining _requests/sec_ ratio of sub-resource locator based resources, where individual request were pointed at **/text/sub-class** path, and

{{< highlight java >}}
@Path("text")
@Consumes(MediaType.TEXT_PLAIN)
@Produces(MediaType.TEXT_PLAIN)
public class TextEntityResource {

    private static final com.yammer.metrics.core.Timer postTimer =
            Metrics.newTimer(TextEntityResource.class, "posts", TimeUnit.MILLISECONDS, TimeUnit.SECONDS);
    private static final com.yammer.metrics.core.Timer getTimer =
            Metrics.newTimer(TextEntityResource.class, "gets", TimeUnit.MILLISECONDS, TimeUnit.SECONDS);
    private static final com.yammer.metrics.core.Timer putTimer =
            Metrics.newTimer(TextEntityResource.class, "puts", TimeUnit.MILLISECONDS, TimeUnit.SECONDS);

    @POST
    public String echo(final String text) {
        final TimerContext timer = postTimer.time();
        try {
            return text;
        } finally {
            timer.stop();
        }
    }

    @PUT
    public void put(final String text) {
        final TimerContext timer = putTimer.time();
        timer.stop();
    }

    @GET
    public String get() {
        final TimerContext timer = getTimer.time();
        try {
            return "text";
        } finally {
            timer.stop();
        }
    }
}
{{< / highlight >}}

to get **requests/sec** comparison between sub-resource locator based requests (**/text/sub-class**) and normal resource method based requests (**/text**).

### Jersey 2.0 vs. 2.15 vs. 2.16

All the results are available and accessible in the following Google Docs Spreadsheet:

  * <a href="https://docs.google.com/spreadsheets/d/1PirHkj5BLs5VmsIBWFQIOYmiPWS2CtcMtP2ajRhOCR8">Jersey 2.0/2.15/2.16 Sub-Resource Locator Performance</a>

Lets together take a look and compare HTTP _GET_, _text/plain_ requests invoked on sub-resource locators in Jersey 2.0, 2.15 and 2.16.

<div class="chart">
</div>

As you can see the version 2.0 is the least performant one with ~2950 requests/sec. Jersey 2.15 is not much bette, ~3880 requests/sec. On the other hand, with locator caching, Jersey 2.16 was able to perform very nicely with ~24130 requests/sec.

The relative increase between

  * **2.0** and **2.16** (2946.82 vs 24126.09) is **718%**
  * **2.15** and **2.16** (3880.7 vs 24126.09) is **521%**.

> The performance increase for <a href="https://docs.google.com/spreadsheets/d/1PirHkj5BLs5VmsIBWFQIOYmiPWS2CtcMtP2ajRhOCR8/pubchart?oid=714286734&format=interactive">POST</a> and <a href="https://docs.google.com/spreadsheets/d/1PirHkj5BLs5VmsIBWFQIOYmiPWS2CtcMtP2ajRhOCR8/pubchart?oid=1066006244&format=interactive">PUT</a> methods are similar and you can take a look at the charts in the mentioned spreadsheet.

These number are pretty impressive but how does sub-resource locators perform against normal resource methods in Jersey 2.16?

<div class="chart">
</div>

Not bad! Normal resource methods gave us **~25100** requests per second and for sub-resource locators we got **~24130** requests per second. This is difference of **~4%** in favor of normal resource methods.

To summarize the findings we were able to get huge performance gain for sub-resource locators between Jersey 2.15 and 2.16. Also, number of handled requests when using sub-resource locators are as close as possible to number of handled requests when an application use normal resource methods in Jersey 2.16.

## <a name="configuration"></a>Cache Configuration

The sub-resource locator cache is initialized for the first time in an application when a first sub-resource locator is being processed. If you don&#8217;t have locators in your application you don&#8217;t need to worry about (slightly) bigger memory footprint.

Jersey offers two eviction policies to make sure the locator cache doesn&#8217;t grow too much – cache size policy and entry age policy.

When you want to limit the size of the cache provide the desired value with the <a href="https://github.com/jersey/jersey/blob/master/core-server/src/main/java/org/glassfish/jersey/server/ServerProperties.java#L652">ServerProperties.SUBRESOURCE_LOCATOR_CACHE_SIZE</a> property. The default cache size is 64 entries.

The age eviction policy can be set via property <a href="https://github.com/jersey/jersey/blob/master/core-server/src/main/java/org/glassfish/jersey/server/ServerProperties.java#L675">ServerProperties.SUBRESOURCE_LOCATOR_CACHE_AGE</a> and is defined as the time (in seconds) since the last access (read) to the entry in the cache. The default value is not set. The age eviction policy can be used to lower the memory footprint when your application (or at least locator methods) is idle.

So, for example if I wanted to use cache that would contain 1000 entries at most and each entry would live for maximum of 10 minutes my application class would look like:

{{< highlight java "hl_lines=9 10" >}}
@ApplicationPath("api")
public class MyApplication extends ResourceConfig {

    public MyApplication() {
        // Register resources and providers.
        // ...

        // Properties.
        property(ServerProperties.SUBRESOURCE_LOCATOR_CACHE_SIZE, 1000);
        property(ServerProperties.SUBRESOURCE_LOCATOR_CACHE_AGE, 60 * 10);
    }
}
{{< / highlight >}}

#### Programmatically created resources

Some of you may already know that besides the declarative (JAX-RS) way of writing resources Jersey provides also a <a href="https://jersey.github.io/documentation/latest/resource-builder.html">programmatic way</a> for creating them. For example, &#8220;Hello World&#8221; application may look like:

{{< highlight java >}}
public static class MyResourceConfig extends ResourceConfig {
 
    public MyResourceConfig() {
        final Resource.Builder resourceBuilder = Resource
                .builder("helloworld");

        resourceBuilder
                .addMethod("GET")
                .handledBy(new Inflector<ContainerRequestContext, Response>() {

                    @Override
                    public Response apply(ContainerRequestContext data) {
                        return Response.ok("Hello World!").build();
                    }
                });
 
        final Resource resource = resourceBuilder.build();
        registerResources(resource);
    }
}
{{< / highlight >}}

Few of you may also know that it&#8217;s possible, in addition to return resource class or resource instance, to return an instance of <a href="https://jersey.github.io/apidocs/latest/jersey/org/glassfish/jersey/server/model/Resource.html">Resource</a> from a sub-resource locator as well. This is useful in cases when you really need to dynamically create resources that are exposed on a certain path.

Unfortunately, caching of this kind of sub-resource locators is very limited at the moment and in many cases the performance of using them is similar to performance of sub-resource locators in version 2.15 (or lower).

However, if you use only few programmatically created resources, that are returned from locators, it&#8217;s possible to leverage caching by returning the same Jersey _Resource_ instances for the same input parameters of a sub-resource locator:

{{< highlight java "hl_lines=4 5 9" >}}
@Path("resource")
public class Resource {

    private final static Resource INSTANCE1 = // ...
    private final static Resource INSTANCE2 = // ...

    @Path("{sub}")
    public Resource sub(@PathParam("sub") final String sub) {
        return "hello".equals(sub) ? INSTANCE1 : INSTANCE2;
    }
}
{{< / highlight >}}

Jersey _Resource_ caching is not enabled by default but you can turn it on by setting <a href="https://github.com/jersey/jersey/blob/master/core-server/src/main/java/org/glassfish/jersey/server/ServerProperties.java#L691">ServerProperties.SUBRESOURCE_LOCATOR_CACHE_JERSEY_RESOURCE_ENABLED</a> property to true.

> If you have any comments or suggestions, please leave a message in the discussion below or send me a note at <michal@dejavu.sk>.

 [1]: #methods
 [2]: #results
 [3]: #configuration